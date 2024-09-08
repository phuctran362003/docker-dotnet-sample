# Docker .NET Sample

This repository contains a simple example of a .NET web application that can be run using Docker and Kubernetes.

## Prerequisites

Make sure you have the following tools installed:

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- [Kubernetes CLI (kubectl)](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [.NET SDK](https://dotnet.microsoft.com/download)

## Getting Started

### Building and Running Locally

To build and run the .NET application in Docker, use the following steps:

1. **Clone the repository**:
    ```bash
    git clone https://github.com/your-repo/docker-dotnet-sample.git
    cd docker-dotnet-sample
    ```

2. **Build and run using Docker Compose**:
    ```bash
    docker compose up --build
    ```

3. The application will be available at `http://localhost:8080`.

### Dockerfile Explanation

The `Dockerfile` defines a multi-stage build for the .NET web application:

- **Stage 1: Build** – This stage builds the .NET project and publishes the application files.
  
  ```dockerfile
  FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS build
  ARG TARGETARCH
  COPY . /source
  WORKDIR /source/src
  RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
      dotnet publish -a ${TARGETARCH/amd64/x64} --use-current-runtime --self-contained false -o /app
  RUN dotnet test /source/tests
  ```

- **Stage 2: Development** – This stage is for local development, where the source files are copied, and the application is run using `dotnet run`.

  ```dockerfile
  FROM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS development
  COPY . /source
  WORKDIR /source/src
  CMD dotnet run --no-launch-profile
  ```

- **Stage 3: Final** – This stage prepares the production image, using a non-root user and running the application.

  ```dockerfile
  FROM mcr.microsoft.com/dotnet/aspnet:6.0-alpine AS final
  WORKDIR /app
  COPY --from=build /app .
  ARG UID=10001
  RUN adduser \
      --disabled-password \
      --gecos "" \
      --home "/nonexistent" \
      --shell "/sbin/nologin" \
      --no-create-home \
      --uid "${UID}" \
      appuser
  USER appuser
  ENTRYPOINT ["dotnet", "myWebApp.dll"]
  ```

### Deploying to Kubernetes

To deploy the application to Kubernetes, follow these steps:

1. **Apply the Kubernetes manifests**:
    ```bash
    kubectl apply -f docker-dotnet-kubernetes.yaml
    ```

2. **Check the status of the deployment**:
    ```bash
    kubectl get pods
    ```

3. **Access the application**: The service will be exposed on port `8080` using `NodePort`. You can access it at `http://localhost:8080` (or through your Kubernetes node's IP).

### Docker Compose

The `compose.yaml` file defines how to run the application with Docker Compose. It spins up the application along with all necessary dependencies. 

To run it, use:

```bash
docker compose up --build
```

The application will then be available at `http://localhost:8080`.

### Deploying Your Application to the Cloud

To deploy to the cloud, first build your Docker image:

```bash
docker build -t myapp .
```

If your cloud provider uses a different CPU architecture (for example, if you're on Mac M1 and the cloud provider uses `amd64`), you can build for that platform:

```bash
docker build --platform=linux/amd64 -t myapp .
```

Then, push the image to your registry:

```bash
docker push myregistry.com/myapp
```

### Kubernetes YAML Files

The `docker-dotnet-kubernetes.yaml` file defines Kubernetes resources to deploy the application and a PostgreSQL database:

- **Deployment for the application**:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      service: server
    name: server
    namespace: default
  spec:
    replicas: 1
    selector:
      matchLabels:
        service: server
    strategy: {}
    template:
      metadata:
        labels:
          service: server
      spec:
        initContainers:
          - name: wait-for-db
            image: busybox:1.28
            command: ['sh', '-c', 'until nc -zv db 5432; do echo "waiting for db"; sleep 2; done;']
        containers:
          - image: phuctran362003/docker-dotnet-sample
            name: server
            imagePullPolicy: Always
            ports:
              - containerPort: 80
                hostPort: 8080
                protocol: TCP
  ```

- **Deployment for PostgreSQL database**:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      service: db
    name: db
    namespace: default
  spec:
    replicas: 1
    selector:
      matchLabels:
        service: db
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          service: db
      spec:
        containers:
          - env:
              - name: POSTGRES_DB
                value: example
              - name: POSTGRES_PASSWORD
                value: example
            image: postgres
            name: db
            ports:
              - containerPort: 5432
                protocol: TCP
  ```

- **Services for the application and database**:
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      service: server
    name: server
    namespace: default
  spec:
    type: NodePort
    ports:
      - name: "8080"
        port: 8080
        targetPort: 80
        nodePort: 30001
    selector:
      service: server

  apiVersion: v1
  kind: Service
  metadata:
    labels:
      service: db
    name: db
    namespace: default
  spec:
    ports:
      - name: "5432"
        port: 5432
        targetPort: 5432
    selector:
      service: db
  ```

### References

- [Docker's .NET Guide](https://docs.docker.com/language/dotnet/)
- The [dotnet-docker](https://github.com/dotnet/dotnet-docker/tree/main/samples) repository contains relevant samples and documentation.
