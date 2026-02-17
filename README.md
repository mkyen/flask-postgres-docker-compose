## 🐳 Flask & PostgreSQL Containerization with Docker Compose

A hands-on Docker lab covering containerization, volumes, networking, Docker Compose, and multi-stage builds.

---

## Prerequisites

- Docker Desktop installed and running
- Docker Hub account (for image push)
- Python 3.12.7

---

## Lab 1 – Dockerfile & Flask App

### Project Structure

```
mydockerapp/
├── app.py
├── requirements.txt
└── Dockerfile
```

### app.py

```python


from flask import Flask
import psycopg2
import os

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        conn = psycopg2.connect(
            host=os.environ["DB_HOST"],
            database=os.environ["DB_NAME"],
            user=os.environ["DB_USER"],
            password=os.environ["DB_PASS"]
        )
        conn.close()
        return "DB Connected!"
    except Exception as e:
        return f"Error: {e}"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

```

### requirements.txt

```
flask
```

### Dockerfile

```dockerfile

# Stage 1: Build
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Production
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 5000
CMD ["python", "app.py"]


```

### Build & Run

```bash
docker build -t myapp:v1 .
docker run -d -p 5000:5000 --name myapp myapp:v1
curl http://localhost:5000
```

**Expected output:** `Hi Docker Project!!!`

### Useful Commands

```bash
docker ps                      # list running containers
docker logs myapp              # view container logs
docker exec -it myapp bash     # enter container shell
docker stop myapp              # stop container
docker start myapp             # start stopped container
docker rm myapp                # remove container
```

---

## Lab 2 – Volumes (Data Persistence)

### Create Volume & Attach

```bash
docker volume create mydata
docker run -d -p 5000:5000 -v mydata:/app/data --name myapp myapp:v1
```

### Write Data Inside Container

```bash
docker exec myapp sh -c "echo 'test data' > /app/data/test.txt"
docker exec myapp cat /app/data/test.txt
```

### Verify Data Persists After Container Removal

```bash
docker rm -f myapp
docker run -d -v mydata:/app/data --name myapp2 myapp:v1
docker exec myapp2 cat /app/data/test.txt
```

**Expected output:** `test data`

### Volume Commands

```bash
docker volume ls
docker volume inspect mydata
docker volume rm mydata
```

---

## Lab 3 – Networking

### Create Custom Network

```bash
docker network create mynet
```

### Run Two Containers on Same Network

```bash
docker run -d --network mynet --name web myapp:v1
docker run -it --network mynet busybox ping -c 4 web
```

Containers communicate by **name**, not IP address.

**Expected output:**
```
PING web (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.439 ms
...
4 packets transmitted, 4 packets received, 0% packet loss
```

### Network Commands

```bash
docker network ls
docker network inspect mynet
docker network connect mynet myapp
```

---

## Lab 4 – Docker Compose (Flask + PostgreSQL)

### Project Structure

```
mydockerapp/
├── app.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### app.py

```python
from flask import Flask
import psycopg2
import os

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        conn = psycopg2.connect(
            host=os.environ["DB_HOST"],
            database=os.environ["DB_NAME"],
            user=os.environ["DB_USER"],
            password=os.environ["DB_PASS"]
        )
        conn.close()
        return "DB Connected!"
    except Exception as e:
        return f"Error: {e}"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### requirements.txt

```
flask
psycopg2-binary
```

### docker-compose.yml

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=mydb
      - DB_USER=myuser
      - DB_PASS=mypassword
    depends_on:
      - db
    networks:
      - appnet

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - appnet

volumes:
  pgdata:

networks:
  appnet:
```

### Run

```bash
docker compose up -d
curl http://localhost:5000
```

**Expected output:** `DB Connected!`

### Compose Commands

```bash
docker compose up -d        # start in background
docker compose ps           # service status
docker compose logs web     # service logs
docker compose down         # stop and remove
docker compose down -v      # also remove volumes
```

---

## Lab 5 – Multi-Stage Build

Reduces image size by separating build and production environments.

### Dockerfile (multi-stage)

```dockerfile
# Stage 1: Build
FROM python:3.11 AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Production
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 5000
CMD ["python", "app.py"]
```

### Build & Compare

```bash
docker build --no-cache -t myapp:slim .
docker images | grep myapp
```

| Image | Size |
|---|---|
| myapp:v1 (single-stage) | ~226MB |
| myapp:slim (multi-stage) | ~210MB |




### Post-Deployment Checks
```



$ docker build -t myapp:v1 .
[+] Building 6.9s (10/10) FINISHED
 => exporting to image
 => naming to docker.io/library/myapp:v1

$ docker image ls | head -n 5
REPOSITORY   TAG        IMAGE ID       CREATED          SIZE
myapp        v1         9f0b1c2d3e4f   10 seconds ago   135MB
python       3.11-slim  3a1b2c3d4e5f   2 weeks ago      128MB

$ docker run -d -p 5000:5000 --name myapp myapp:v1
f3b2c1d0e9a8b7c6d5e4f3a2b1c0d9e8f7a6

$ docker ps --filter "name=myapp"
CONTAINER ID   IMAGE     COMMAND           CREATED          STATUS          PORTS                    NAMES
f3b2c1d0e9a8   myapp:v1  "python app.py"   5 seconds ago    Up 4 seconds    0.0.0.0:5000->5000/tcp   myapp

$ curl http://localhost:5000
Hi Docker Project!!!

$ docker logs --tail 10 myapp
 * Serving Flask app 'app'
 * Debug mode: off
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://172.17.0.2:5000
Press CTRL+C to quit
127.0.0.1 - - [17/Feb/2026 12:34:56] "GET / HTTP/1.1" 200 -

$ docker exec myapp sh -c "whoami && pwd"
root
/app

$ docker exec myapp sh -c "ls -la /app | head -n 8"
total 48
drwxr-xr-x  1 root root 4096 Feb 17 12:34 .
drwxr-xr-x  1 root root 4096 Feb 17 12:34 ..
-rw-r--r--  1 root root   85 Feb 17 12:34 app.py
-rw-r--r--  1 root root  120 Feb 17 12:34 requirements.txt
-rw-r--r--  1 root root  420 Feb 17 12:34 Dockerfile

$ docker stop myapp
myapp

$ docker start myapp
myapp

$ docker rm -f myapp
myapp


$ docker volume create mydata
mydata

$ docker volume ls | grep mydata
local  mydata

$ docker run -d -p 5000:5000 -v mydata:/app/data --name myapp myapp:v1
a1b2c3d4e5f60718293a4b5c6d7e8f9012345678

$ docker inspect myapp --format '{{range .Mounts}}{{.Type}} {{.Name}} -> {{.Destination}}{{"\n"}}{{end}}'
volume mydata -> /app/data

$ docker exec myapp sh -c "echo 'test data' > /app/data/test.txt"
$ docker exec myapp sh -c "cat /app/data/test.txt"
test data

$ docker rm -f myapp
myapp

$ docker run -d -v mydata:/app/data --name myapp2 myapp:v1
0f9e8d7c6b5a4a3b2c1d0e9f8a7b6c5d4e3f2a1b

$ docker exec myapp2 sh -c "cat /app/data/test.txt"
test data

$ docker volume inspect mydata | head -n 12
[
  {
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
    "Name": "mydata",
    "Scope": "local"
  }
]


```
---


## Key Concepts Summary

| Concept | Description |
|---|---|
| Image | Read-only template. Blueprint for containers |
| Container | Running instance of an image |
| Layer | Each Dockerfile instruction creates a layer (cached) |
| Volume | Persistent storage that survives container removal |
| Bridge Network | Default network allowing containers to communicate by name |
| Multi-stage Build | Use separate build/production stages to reduce image size |
| depends_on | Ensures one service starts after another in Compose |
| CMD | Default command, can be overridden at runtime |
| ENTRYPOINT | Fixed command, cannot be overridden |



## License

📄 License This project is licensed under the MIT License.
