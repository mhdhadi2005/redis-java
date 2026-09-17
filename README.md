# Redis Server in Java

A Java implementation of a Redis server built on top of the [CodeCrafters "Build Your Own Redis" Challenge](https://codecrafters.io/challenges/redis). It implements the core Redis wire protocol (RESP), an in-memory key-value store, and master/replica replication — plus Docker and Kubernetes deployment setups.

[![progress-banner](https://backend.codecrafters.io/progress/redis/b7dbc9e6-45a6-4033-97e0-c93c0a469df9)](https://app.codecrafters.io/users/codecrafters-bot?r=2qF)

## Features

- **RESP protocol** — parses and serializes Redis's wire format (`Components/Service/RespSerializer.java`)
- **TCP server** — binds and accepts client connections (`Components/Server/MasterTcpServer.java`)
- **In-memory key-value store** — with key expiry support (`Components/Repository/Store.java`)
- **Commands supported**: `PING`, `ECHO`, `SET` (with `PX` expiry), `GET`, `INCR`, `DEL`, `INFO`, `REPLCONF`, `WAIT`
- **Master/replica replication** — a server can run as a master and stream commands to connected replicas (`Components/Infra/Slave.java`, `Components/Server/SlaveTcpServer.java`)
- **Dockerized** — runs as a container, replica mode included
- **Kubernetes deployment** — Helm chart and a local KIND cluster setup for running it on k8s
- **CI/CD** — an Azure Pipelines config for automated builds

## Project structure

```
src/main/java/
├── Main.java                      # Entry point
├── Config/AppConfig.java          # Spring configuration
└── Components/
    ├── Server/                    # TCP server (master + replica modes), Redis config
    ├── Service/                   # Command handling, RESP (de)serialization
    ├── Repository/                # The actual key-value store
    └── Infra/                     # Client/replica connection handling, connection pool
```

## Running locally

Requires Java and Maven (`mvn`) installed.

```sh
./your_program.sh
```

This compiles and starts the server, listening on port `6379` by default. You can then connect with `redis-cli` or any Redis client.

## Running with Docker

```sh
docker build -t myredis .
docker run -p 6379:6379 myredis
```

To run a second instance as a replica of the first:

```sh
# Find your host's gateway IP if running the master on the host, not in Docker
ip route show | grep default | awk '{print $3}'

docker run -p 6480:6480 myredis --port 6480 --replicaof "<master-ip> 6379"
```

## Deploying to Kubernetes (via KIND + Helm)

<details>
<summary>Expand for full setup steps</summary>

**1. Create a local KIND cluster**

```sh
mkdir kind && cd kind
```

```yml
# k.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30007
    hostPort: 30007
```

```sh
kind create cluster --config k.yml --name cluster-name
cd ..
```

**2. Build and load the image into the cluster**

```sh
docker build -t beelzekamibub/redis:latest .
kind load docker-image beelzekamibub/redis:latest --name cluster-name
```

**3. Define the Pod and Service**

```yml
# pod.yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: redis
  name: redis
  namespace: default
spec:
  containers:
    - image: beelzekamibub/redis:latest
      imagePullPolicy: Never
      name: redis
```

```yml
# service.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: redis
  name: redis-service
spec:
  ports:
    - port: 6379
      protocol: TCP
      targetPort: 6379
      nodePort: 30007
  selector:
    run: redis
  type: NodePort
```

```sh
alias k=kubectl
k apply -f pod.yml
k apply -f service.yml
```

**4. Package as a Helm chart**

The `redis-chart/` directory in this repo already contains the templatized chart (built from `pod.yml`/`service.yml`, with the templates folder cleaned up and no chart dependencies). To work with it:

```sh
helm template redis-chart
helm lint redis-chart
helm install redis-app redis-chart

# to tear down
helm uninstall redis-app
```

</details>

## CI (Azure Pipelines)

`azure-pipelines.yml` defines the build pipeline. `dind/` contains a self-hosted Docker-in-Docker build agent, if you want to run pipeline jobs on your own infrastructure instead of Microsoft-hosted agents:

```sh
cd dind
docker build --tag "azp-agent:linux" .

docker run --privileged \
  -e AZP_URL="<your-azure-devops-org-url>" \
  -e AZP_TOKEN="<your-pat-token>" \
  -e AZP_POOL="agent" \
  -e AZP_AGENT_NAME="Docker Agent - Linux" \
  --name "azp-agent-linux" \
  azp-agent:linux
```

## Background

This started as a solution to CodeCrafters' Redis challenge (build a toy Redis clone handling `PING`, `SET`, `GET`, RESP encoding, and event loops), then was extended with replication, containerization, and cloud-deployment tooling.
