# Multi-Node-Application-Deployment-on-Docker-Swarm

This project deploys a multi-service "Cats vs Dogs" voting application on a Docker Swarm cluster. The architecture is composed of five services distributed across two overlay networks.

## Architecture

The application is broken down into the following services:

- **vote**: A Python web application for users to cast their votes. It pushes new votes onto a Redis queue.
- **redis**: An in-memory key-value store that holds the incoming votes temporarily in a queue.
- **worker**: A .NET application that consumes votes from the Redis queue and persists them into a PostgreSQL database.
- **db**: A PostgreSQL database for persistent storage of the vote data.
- **result**: A Node.js web application that queries the database and displays the voting results in real-time.

## Network Configuration

To ensure secure and organized communication, the services are isolated into two separate overlay networks:

- **frontend**: Handles communication between the vote app, the redis queue, and the worker.
- **backend**: Handles communication between the worker, the db, and the result app.

The **worker** service is connected to both networks, acting as a bridge to move data from the front-end queue to the back-end database.

## 1. Create Networks

```
docker network create -d overlay backend
docker network create -d overlay frontend
```

## 2. Deploy Services
### Vote App (Frontend)
```
docker service create \
  --name vote \
  -p 5000:80 \
  --replicas 4 \
  --network frontend \
  dockersamples/examplevotingapp_vote
```

### Redis Queue (Frontend)
```
docker service create \
  --name redis \
  --replicas 4 \
  --network frontend \
  redis:alpine
```
### Worker (Connects Frontend and Backend)
```
docker service create \
  --name worker \
  --replicas 4 \
  --network frontend \
  --network backend \
  dockersamples/examplevotingapp_worker
```
### PostgreSQL Database (Backend)
```
docker service create \
  --name postgres \
  --network backend \
  --constraint 'node.hostname==manager2' \
  --mount type=bind,source=/root/postgres,destination=/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=12345 \
  postgres:15-alpine
```

### Result App (Backend)
```
docker service create \
  --name result \
  -p 5001:80 \
  --network backend \
  dockersamples/examplevotingapp_result
```

### Accessing the Application
- Voting Interface: ```http://<YOUR_SWARM_IP>:5000```

- Results Interface: `http://<YOUR_SWARM_IP>:5001`
