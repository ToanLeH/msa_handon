# Prerequisites
- Visual Studio Code
- WSL2 ubuntu 20.04
- Docker Engine and Docker Compose
- Required to install dotnet 7 SDKs/Runtime

# High Level Architecture
- Please refer below for High Level Architecture of MSA Fundamental projects

![MSA-Fundamental-High-Level_Architecture.jpg](/.attachments/MSA-Fundamental-High-Level_Architecture-d50549ed-8c3a-457d-8077-5113c015667b.jpg)


# Learning Objectives
- Create skeleton MicroServices Application

- The following will be the Scenarios within this practical labs
    -  [Create required InIrastructure (Mongo, Postgres, Rabbitmq) using docker compose](./00-infrastructure/0-learning-objectives.md)
    -  [Create common module to shared code between services](./01-common-module/0-learning-objectives.md)
    -  [Create skeleton product services integrate with Mongo Database](./02-product-service/0-learning-objectives.md)
    -  [Create skeleton order services integrate with Postgres Database](./03-order-service/0-learning-objectives.md)
    -  [Synchronous Communication - REST](./04-communication-rest/0-learning-objectives.md)
    -  [Sync Data between Services - Domain Event](./05-sync-domain-event/0-learning-objectives.md)
    -  [Saga Pattern - Choreography](./06-saga-choreography/0-learning-objectives.md)
    -  [Saga Pattern - Orchestration](./07-saga-orchestration/0-learning-objectives.md)
    -  [Authentication - Create skeleton identity services using in memory Duende Identity Server](./08-identity-server/0-learning-objectives.md)
    -  [Reverse Proxy - Add Reverse Proxy for services](./09-reverse-proxy/0-learning-objectives.md)
    -  [Dockerize services - create Dockerfile to containerize services](./10-docker-app/0-learning-objectives.md)