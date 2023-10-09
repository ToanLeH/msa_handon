# Create Infrastructure with Mongo, RabbitMq and Postgres

# Create **docker-compose.yaml** file
- Create **docker-compose.yaml** file with below configuration to create Mongo, Rabbitmq and Postregres
- For more info how to config mongo please refer to this link https://hub.docker.com/_/mongo
- For more info how to config Rabbitmq please refer to this link https://hub.docker.com/_/rabbitmq
- For more info how to config Postgres please refer to this link https://hub.docker.com/_/postgres
``` yml
version: "3.8"

services:
  mongo:
    image: mongo
    container_name: mongo
    ports:
      - 27017:27017
    volumes:
      - mongodbdata:/data/db
    networks:
      - msa-network

  rabbitmq:
    image: rabbitmq:management
    container_name: rabbitmq
    ports:
      - 5672:5672
      - 15672:15672
    volumes:
      - rabbitmqdata:/var/lib/rabbitmq
    hostname: rabbitmq
    networks:
      - msa-network

  postgresql:
    image: postgres:14-alpine
    environment:
      - POSTGRES_DB=msapostgres
      - POSTGRES_USER=guest
      - POSTGRES_PASSWORD=guest
    ports:
      - 5432:5432
    volumes:
      - postgresqldata:/var/lib/postgresql/data
    networks:
      - msa-network

volumes:
  mongodbdata:
  rabbitmqdata:
  postgresqldata:

networks:
  msa-network:
```

- Run below command to create those containers
``` bash
docker compose up
```

- in case you want to remove those containers, run below command
``` bash
docker compose down
```

# Verifying all containers are running
- ensure above 3 containers are running via docker extension from visual studio code