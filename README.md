#  **README — Taller de Docker**

# Taller de Docker

**Crear imágenes propias, contenedores y balanceo de carga con Traefik**

Este taller tiene como objetivo aprender los conceptos fundamentales de Docker:

* Crear imágenes propias
* Ejecutar contenedores
* Definir servicios con Docker Compose
* Integrar un balanceador de carga (Traefik)
* Usar múltiples réplicas de un mismo servicio
* Subir imágenes a Docker Hub

---

#  **1. Crear mi primera imagen**

Docker nos permite crear nuestras propias imágenes basándonos en imágenes existentes.
Primero creamos el *build context*:

```bash
mkdir -p ~/Sites/hello-world
cd ~/Sites/hello-world
echo "hello" > hello
```

Creamos un archivo llamado **Dockerfile**:

```Dockerfile
FROM busybox
COPY /hello /
RUN cat /hello
```

Construimos la imagen:

```bash
docker build -t helloapp:v1 .
```

Verificamos la imagen creada:

```bash
docker images
```

*Captura mostrando la imagen helloapp:v1:*

![captura](https://github.com/isadiazac/DYAS-Docker/blob/main/Captura%20de%20pantalla%202025-11-17%20211444.png)

---

# **2. Crear una aplicación Python en Docker**

Creamos el nuevo build context:

```bash
mkdir -p ~/Sites/friendlyhello
cd ~/Sites/friendlyhello
```

Creamos **app.py**:

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)
app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello World!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(
        hostname=socket.gethostname(),
        visits=visits
    )

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

Creamos **requirements.txt**:

```
Flask
Redis
```

Y el **Dockerfile**:

```Dockerfile
FROM python:3-slim
WORKDIR /app
COPY . /app
RUN pip install --trusted-host pypi.python.org -r requirements.txt
EXPOSE 80
ENV NAME World
CMD ["python", "app.py"]
```

Construimos la imagen:

```bash
docker build -t friendlyhello .
```

---

# **3. Probar el contenedor**

```bash
docker run --rm -p 4000:80 friendlyhello
```

Abrimos en el navegador:

[http://localhost:4000](http://localhost:4000)

*Captura mostrando “Hello World!” y el hostname:*
![captura](https://github.com/isadiazac/DYAS-Docker/blob/main/Captura%20de%20pantalla%202025-11-17%20211607.png)

---

# **4. Crear la aplicación con Docker Compose**

Creamos **docker-compose.yml**:

```yaml
services:
  web:
    build: .
    ports:
      - "4000:80"
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "./data:/data"
    command: redis-server --appendonly yes
```

Probamos:

```bash
docker compose up
```

---

# **5. Agregar Traefik como balanceador de carga**

Modificamos el archivo docker-compose.yml:

```yaml
services:
  web:
    image: isadiac/friendlyhello:latest
    depends_on:
      - redis
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=PathPrefix(`/`)"
      - "traefik.http.services.web.loadbalancer.server.port=80"

  redis:
    image: redis
    volumes:
      - "./data:/data"
    command: redis-server --appendonly yes
    labels:
      - "traefik.enable=false"

  traefik:
    image: traefik:v2.3
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:4000"
    ports:
      - "4000:4000"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    labels:
      - "traefik.enable=true"
```

---

# **6. Ejecutar 5 réplicas del servicio web**

```bash
docker compose up -d --scale web=5
```

---

# **7. Probar el balanceo de carga**

Cada vez que recargamos la página en:

[http://localhost:4000](http://localhost:4000)

deberíamos ver **un hostname distinto**.

*Evidencias del hostname cambiando:*

![captura](https://github.com/isadiazac/DYAS-Docker/blob/main/Captura%20de%20pantalla%202025-11-17%20214340.png)

También verificamos en el dashboard de Traefik:

👉 [http://localhost:8080/dashboard/#/](http://localhost:8080/dashboard/#/)

*Captura del Dashboard mostrando Routers, Services y Middlewares:*
![captura](https://github.com/isadiazac/DYAS-Docker/blob/main/Captura%20de%20pantalla%202025-11-17%20214328.png)

---

# **8. Subir la imagen a Docker Hub**

1. Crear repositorio en Docker Hub
2. Iniciar sesión:

```bash
docker login
```

3. Etiquetar la imagen:

```bash
docker tag friendlyhello username/friendlyhello
```

4. Subirla:

```bash
docker push username/friendlyhello
```

---

# **Ejercicios finales**

* Modificar docker-compose.yml para usar tu propia imagen desde Docker Hub -> isadiac
* Modificar docker-compose.yml para usar la imagen de un compañero -> papo8888
![captura](https://github.com/isadiazac/DYAS-Docker/blob/main/Captura%20de%20pantalla%202025-11-17%20214859.png)
![captura](https://github.com/isadiazac/DYAS-Docker/blob/main/Captura%20de%20pantalla%202025-11-17%20214328.png)
* Reconstruir la imagen con versión nueva usando etiquetas

---

Si quieres, también puedo hacerte
👉 **un README con portada**,
👉 **un diseño más visual**,
👉 **o una versión en PDF lista para entregar**.
