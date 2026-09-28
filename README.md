# Microservices TODO App – Dockerized with Docker Compose

This project is a TODO application built with a **microservice architecture**. Each service is written in a different language or framework (Go, Python, Vue.js, Java and Node.js), so it is a good example of a polyglot system.

> **Important:** the microservices themselves already existed (they come from the PRFT DevOps training project). My work in this repository was to **containerize every service** by writing the `Dockerfile` per service and a `docker-compose.yml` file that runs the whole system with a single command.

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [What I Did (My Contribution)](#2-what-i-did-my-contribution)
3. [Prerequisites](#3-prerequisites)
4. [Project Structure](#4-project-structure)
5. [Step-by-Step Configuration](#5-step-by-step-configuration)
   - [Step 1 – Analyze each microservice](#step-1--analyze-each-microservice)
   - [Step 2 – Write one Dockerfile per service](#step-2--write-one-dockerfile-per-service)
   - [Step 3 – Write the docker-compose.yml](#step-3--write-the-docker-composeyml)
6. [How to Run the Project](#6-how-to-run-the-project)
7. [How to Test the Application](#7-how-to-test-the-application)
8. [Useful Docker Commands](#8-useful-docker-commands)
9. [Troubleshooting](#9-troubleshooting)
10. [Known Limitations and Possible Improvements](#10-known-limitations-and-possible-improvements)

---

## 1. Architecture

The application has five microservices and one Redis instance:

| Service | Technology | Port | Responsibility |
|---|---|---|---|
| `frontend` | Vue.js (Node 14) | 8080 | Web user interface |
| `auth-api` | Go 1.21 | 8081 | Login and JWT token generation |
| `todos-api` | Node.js 14 | 8082 | CRUD operations for TODO items |
| `users-api` | Java 8 / Spring Boot | 8083 | User profiles |
| `log-message-processor` | Python 3.6 | – | Reads messages from Redis and prints them to the console |
| `redis` | Redis (alpine) | 6379 | Message queue between `todos-api` and `log-message-processor` |

### How the services communicate

```
                +-----------+
   Browser ---> | frontend  | :8080
                +-----+-----+
                      |
          +-----------+------------+
          |                        |
   POST /login              GET/POST/DELETE /todos
          |                        |
    +-----v------+           +-----v------+        publish        +-------+
    |  auth-api  | :8081     | todos-api  | :8082 ---------------> | redis | :6379
    +-----+------+           +------------+                        +---+---+
          |                                                            |
    GET /users/:name                                              subscribe
          |                                                            |
    +-----v------+                                        +------------v-----------+
    | users-api  | :8083                                  | log-message-processor  |
    +------------+                                        +------------------------+
```

1. The user opens the **frontend** and logs in.
2. The frontend calls **auth-api**, which asks **users-api** for the user data and returns a **JWT token**.
3. The frontend sends the token to **todos-api** to create, list or delete TODOs.
4. On every `CREATE` or `DELETE`, **todos-api** publishes a message to a **Redis** channel (`log_channel`).
5. **log-message-processor** listens to that channel and prints each message to standard output.

The original architecture diagram is in [`arch-img/Microservices.png`](arch-img/Microservices.png).

---

## 2. Contribution

- Analyzed every microservice to understand its language, build tool, port and environment variables.
- Wrote **5 Dockerfiles** (one per service):
  - `frontend/Dockerfile`
  - `auth-api/Dockerfile`
  - `todos-api/Dockerfile`
  - `users-api/Dockerfile` (multi-stage build)
  - `log-message-processor/Dockerfile`
- Wrote **one `docker-compose.yml`** that:
  - builds all the images,
  - starts a Redis container,
  - connects all services in the same network,
  - defines ports, environment variables and startup order (`depends_on`).
- Made sure that all services share the same `JWT_SECRET` so tokens can be validated by every API.

---

## 3. Prerequisites

You only need these tools installed on your machine:

- [Docker](https://docs.docker.com/get-docker/) (version 20.10 or higher)
- [Docker Compose](https://docs.docker.com/compose/install/) (included in Docker Desktop; on Linux it can be the `docker compose` plugin)
- [Git](https://git-scm.com/)

Check your installation:

```bash
docker --version
docker compose version
git --version
```

> You do **not** need Go, Java, Maven, Node.js or Python installed locally. Everything is built inside the containers.

---

## 4. Project Structure

```
Microservices-todo/
├── docker-compose.yml          # Orchestrates all the services  
├── arch-img/                   # Architecture diagram
├── frontend/
│   ├── Dockerfile              
│   └── ...                     # Vue.js source code
├── auth-api/
│   ├── Dockerfile              
│   └── ...                     # Go source code
├── todos-api/
│   ├── Dockerfile              
│   └── ...                     # Node.js source code
├── users-api/
│   ├── Dockerfile              
│   └── ...                     # Spring Boot source code
└── log-message-processor/
    ├── Dockerfile              
    └── ...                     # Python source code
```

---

## 5. Step-by-Step Configuration

### Step 1 – Analyze each microservice

Before writing any Docker file, I read the source code of each service to find out:

| Question | Where I looked |
|---|---|
| Which language and version? | `go.mod`, `package.json`, `pom.xml`, `requirements.txt` |
| How is it built and started? | `npm start`, `mvn package`, `go build`, `python main.py` |
| Which port does it use? | `main.go`, `server.js`, `application.properties` |
| Which environment variables does it need? | `os.Getenv(...)`, `process.env...`, `os.environ[...]` |

This is what I found:

| Service | Build / start command | Port variable | Other environment variables |
|---|---|---|---|
| `auth-api` | `go build` → `./auth-api` | `AUTH_API_PORT` | `USERS_API_ADDRESS`, `JWT_SECRET` |
| `todos-api` | `npm install` → `npm start` | `TODO_API_PORT` (default `8082`) | `JWT_SECRET`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_CHANNEL` |
| `users-api` | `mvn package` → `java -jar` | `server.port=8083` (in `application.properties`) | `JWT_SECRET` |
| `log-message-processor` | `pip install` → `python main.py` | – | `REDIS_HOST`, `REDIS_PORT`, `REDIS_CHANNEL` |
| `frontend` | `npm install` → `npm start` | `PORT` (default `8080`) | `AUTH_API_ADDRESS`, `TODOS_API_ADDRESS` |

---

### Step 2 – Write one Dockerfile per service

I followed the same general pattern for the services: **copy the dependency file first, install dependencies, and then copy the source code**. This way Docker can reuse the cached layer for the dependencies when only the source code changes, and builds are much faster.

#### 2.1 `auth-api/Dockerfile` (Go)

```dockerfile
FROM golang:1.21-alpine
WORKDIR /app
RUN apk add --no-cache git
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -v -o auth-api
EXPOSE 8000
CMD ["./auth-api"]
```

Explanation:
- `golang:1.21-alpine` is a small image that already contains the Go compiler.
- `git` is installed because Go may need it to download some modules.
- `go mod download` downloads the dependencies in a separate layer (cache).
- `go build -o auth-api` compiles the application into one binary.
- The real port is defined at runtime with `AUTH_API_PORT` (see the Compose file).

#### 2.2 `todos-api/Dockerfile` (Node.js)

```dockerfile
FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["npm", "start"]
```

Explanation:
- Node 14 is used because this old project was written for that version.
- `package*.json` copies both `package.json` and `package-lock.json`.
- `npm install` installs the dependencies before copying the source code (cache).

#### 2.3 `frontend/Dockerfile` (Vue.js)

```dockerfile
FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["npm", "start"]
```

Explanation:
- It uses the same pattern as `todos-api`.
- `npm start` runs `node build/dev-server.js`, which starts the Vue development server on port `8080`.
- The dev server also works as a **proxy**: it redirects `/login` to `auth-api` and `/todos` to `todos-api`, using the `AUTH_API_ADDRESS` and `TODOS_API_ADDRESS` variables.

#### 2.4 `users-api/Dockerfile` (Java / Spring Boot, multi-stage)

```dockerfile
FROM maven:3.6.3-jdk-8 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:8-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8083
CMD ["java", "-jar", "app.jar"]
```

Explanation:
- This is a **multi-stage build** with two stages:
  1. **`builder`**: uses Maven + JDK 8 to compile the code and create the `.jar` file. `-DskipTests` makes the build faster.
  2. **Final stage**: uses only a small JRE 8 image and copies the `.jar` from the first stage.
- Result: the final image is much smaller because it does not contain Maven, the JDK or the source code.
- The service listens on port `8083` (defined in `application.properties`).

#### 2.5 `log-message-processor/Dockerfile` (Python)

```dockerfile
FROM python:3.6-alpine
WORKDIR /app
RUN apk add --no-cache gcc musl-dev
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

Explanation:
- `gcc` and `musl-dev` are needed on Alpine Linux because some Python packages must be compiled during installation.
- This service has no `EXPOSE` because it does not receive HTTP requests. It only **reads** from Redis.

---

### Step 3 – Write the docker-compose.yml

The Compose file describes how all the containers work together.

```yaml
version: '3.8'
services:
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  frontend:
    build: ./frontend
    ports:
      - "8080:8080"
    environment:
      - AUTH_API_ADDRESS=http://auth-api:8081
      - TODOS_API_ADDRESS=http://todos-api:8082
    depends_on:
      - todos-api
      - users-api
      - auth-api

  auth-api:
    build: ./auth-api
    ports:
      - "8081:8081"
    environment:
      - AUTH_API_PORT=8081
      - USERS_API_ADDRESS=http://users-api:8083
      - JWT_SECRET=PRF_TOKEN
    depends_on:
      - users-api

  todos-api:
    build: ./todos-api
    ports:
      - "8082:8082"
    environment:
      - JWT_SECRET=PRF_TOKEN
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_CHANNEL=log_channel
      - AUTH_API_ADDRESS=http://auth-api:8081
    depends_on:
      - redis
      - auth-api

  users-api:
    build: ./users-api
    ports:
      - "8083:8083"
    environment:
      - JWT_SECRET=PRF_TOKEN

  log-message-processor:
    build: ./log-message-processor
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_CHANNEL=log_channel
    depends_on:
      - redis
```

#### Key decisions explained

**1. Service names as hostnames.**
Docker Compose creates a private network and a DNS entry for every service. This is why the services can talk to each other with names like `http://auth-api:8081` or `REDIS_HOST=redis` instead of IP addresses or `localhost`.

**2. Port mapping (`"HOST:CONTAINER"`).**
Each service is exposed to my machine so I can test it directly:

| URL on my machine | Service |
|---|---|
| http://localhost:8080 | Frontend |
| http://localhost:8081 | Auth API |
| http://localhost:8082 | TODOs API |
| http://localhost:8083 | Users API |
| localhost:6379 | Redis |

**3. Environment variables.**
Each service receives its configuration from the `environment` section, which follows the "config in the environment" idea of the [Twelve-Factor App](https://12factor.net/config). No configuration is hard-coded inside the images.

- `AUTH_API_PORT=8081` tells the Go service which port to listen on.
- `USERS_API_ADDRESS`, `AUTH_API_ADDRESS` and `TODOS_API_ADDRESS` tell each service where to find the others.
- `REDIS_HOST`, `REDIS_PORT` and `REDIS_CHANNEL` connect `todos-api` (publisher) and `log-message-processor` (subscriber) to the same Redis channel.

**4. Shared `JWT_SECRET`.**
`auth-api` **creates** the JWT tokens and `todos-api` and `users-api` **validate** them. If the secret is different in any service, the token validation fails (usually with a `401 Unauthorized` error). That is why all three use the same value: `PRF_TOKEN`.

> In `users-api`, the value in `application.properties` (`jwt.secret=myfancysecret`) is overridden by the `JWT_SECRET` environment variable, because Spring Boot maps environment variables to properties automatically.

**5. Startup order with `depends_on`.**
It tells Compose to start the containers in the right order (for example, Redis before `todos-api`). Be aware that `depends_on` only waits until the container has **started**, not until the application inside is **ready**. See [Troubleshooting](#9-troubleshooting).

---

## 6. How to Run the Project

### 1. Clone the repository

```bash
git clone https://github.com/Angdicode/Microservices-todo.git
cd Microservices-todo
```

### 2. Build and start all the services

```bash
docker compose up --build
```

- `--build` forces Docker to build the images. The first build can take several minutes (Maven and npm need to download many dependencies).
- If you want to run it in the background, add `-d`:

```bash
docker compose up --build -d
```

### 3. Check that everything is running

```bash
docker compose ps
```

You should see six containers with the status `running`: `redis`, `frontend`, `auth-api`, `todos-api`, `users-api` and `log-message-processor`.

### 4. Stop the project

```bash
docker compose down
```

---

## 7. How to Test the Application

### Test in the browser

1. Open **http://localhost:8080**.
2. Log in with one of the default users:

| Username | Password |
|---|---|
| `admin` | `admin` |
| `johnd` | `foo` |

3. Create a few TODO items and delete some of them.

### Check the Redis queue processing

Every time you create or delete a TODO, the `log-message-processor` prints a message. To see it:

```bash
docker compose logs -f log-message-processor
```

You should see messages with the operation name (`CREATE` or `DELETE`), the username and the TODO id. This proves that `todos-api` → `redis` → `log-message-processor` works correctly.

### Test the APIs with curl (optional)

```bash
# 1. Get a JWT token from auth-api
curl -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}'

# 2. Use the token to list TODOs
curl http://localhost:8082/todos \
  -H "Authorization: Bearer <YOUR_TOKEN>"

# 3. Create a TODO
curl -X POST http://localhost:8082/todos \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"content":"My first todo"}'
```

---

## 8. Useful Docker Commands

| Command | What it does |
|---|---|
| `docker compose up --build` | Builds the images and starts all services |
| `docker compose up -d` | Starts the services in the background |
| `docker compose down` | Stops and removes the containers and the network |
| `docker compose ps` | Shows the status of the containers |
| `docker compose logs -f <service>` | Follows the logs of one service |
| `docker compose build --no-cache <service>` | Rebuilds one image without cache |
| `docker compose restart <service>` | Restarts one service |
| `docker compose exec <service> sh` | Opens a shell inside a running container |

---

## 9. Troubleshooting

| Problem | Possible cause | Solution |
|---|---|---|
| `port is already allocated` | Another program uses port 8080–8083 or 6379 | Stop that program or change the left side of the port mapping (for example `"9090:8080"`) |
| `401 Unauthorized` when using the APIs | Different `JWT_SECRET` values | Make sure all services use the same `JWT_SECRET` |
| `todos-api` cannot connect to Redis on startup | Redis was not ready yet (`depends_on` does not wait for readiness) | Run `docker compose restart todos-api`. The service also has a retry strategy for Redis. |
| The frontend cannot log in | `auth-api` is not ready or `AUTH_API_ADDRESS` is wrong | Check `docker compose logs auth-api` |
| Changes in the code are not visible | Old image in cache | Run `docker compose up --build` again |
| Build fails while downloading dependencies | Network problem | Try again, or run `docker compose build --no-cache <service>` |

---

## 10. Known Limitations and Possible Improvements

This setup is designed for **local development and learning**. These are the things I would improve for a real environment:

- **Secrets:** `JWT_SECRET` is written in plain text in `docker-compose.yml`. It is better to use a `.env` file or Docker secrets.
- **Health checks:** add `healthcheck` blocks and use `depends_on` with `condition: service_healthy`, so a service starts only when its dependency is really ready.
- **Frontend in production:** the frontend runs the Vue development server (`npm start`). For production, build the static files with `npm run build` and serve them with Nginx in a multi-stage Dockerfile.
- **Smaller and safer images:** use a compiled multi-stage build also for Go (final image based on `alpine` or `scratch`) and run containers with a non-root user.
- **Old runtimes:** Node 14, Python 3.6 and Java 8 are out of official support. They were chosen for compatibility with the original code; upgrading them would need code changes and testing.
- **Data persistence:** `todos-api` keeps the TODOs in memory, so data is lost when the container restarts.
- **`EXPOSE` values:** the `EXPOSE` lines in the `auth-api` (8000) and `todos-api` (8080) Dockerfiles are only documentation and do not match the real ports (8081 and 8082). The real ports are set with environment variables and the `ports` section in Compose, so the application works, but these lines should be corrected.
- **Observability:** the services already support Zipkin tracing through the `ZIPKIN_URL` variable. A Zipkin container could be added to Compose.
- **CI/CD:** add a pipeline (for example GitHub Actions) that builds and pushes the images to a registry.

---

