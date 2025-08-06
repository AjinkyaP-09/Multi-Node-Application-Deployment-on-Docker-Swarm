# Multi-Node-Application-Deployment-on-Docker-Swarm

This project deploys a multi-service "Cats vs Dogs" voting application on a Docker Swarm cluster. The architecture is composed of five services distributed across two overlay networks.

<img width="1919" height="913" alt="Screenshot 2025-08-06 115357" src="https://github.com/user-attachments/assets/bbc51c72-3e94-4657-84fc-d608c814a476" />

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
<img width="675" height="281" alt="Screenshot 2025-08-06 115305" src="https://github.com/user-attachments/assets/1722a8e1-46f8-48fe-98bd-7e7929e89f71" />

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
<img width="1012" height="296" alt="Screenshot 2025-08-06 112932" src="https://github.com/user-attachments/assets/bf1f1948-0cb8-463b-b1d2-a4af46e49262" />

### Redis Queue (Frontend)
```
docker service create \
  --name redis \
  --replicas 4 \
  --network frontend \
  redis:alpine
```
<img width="1123" height="263" alt="Screenshot 2025-08-06 121332" src="https://github.com/user-attachments/assets/3be1932c-bea2-4661-8aac-d02c1aac0b2a" />

### Worker (Connects Frontend and Backend)
```
docker service create \
  --name worker \
  --replicas 4 \
  --network frontend \
  --network backend \
  dockersamples/examplevotingapp_worker
```
<img width="959" height="258" alt="Screenshot 2025-08-06 113359" src="https://github.com/user-attachments/assets/4b31c9e8-4059-4eb1-b98e-84f8884a95f9" />

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
<img width="954" height="244" alt="Screenshot 2025-08-06 114350" src="https://github.com/user-attachments/assets/82e96aa0-4794-4a5b-bdd5-e71fe1d94ec5" />

### Result App (Backend)
```
docker service create \
  --name result \
  -p 5001:80 \
  --network backend \
  dockersamples/examplevotingapp_result
```
<img width="983" height="204" alt="Screenshot 2025-08-06 114511" src="https://github.com/user-attachments/assets/87e18e94-8b74-41cc-991b-bfffc2217606" />

### Accessing the Application
- Voting Interface: ```http://<YOUR_SWARM_IP>:5000```
<img width="1622" height="534" alt="Screenshot 2025-08-06 121349" src="https://github.com/user-attachments/assets/697604a6-95d8-42d1-b04a-7facecd43d9c" />

- Results Interface: `http://<YOUR_SWARM_IP>:5001`
<img width="1906" height="976" alt="Screenshot 2025-08-06 114621" src="https://github.com/user-attachments/assets/c8239aa4-7516-4a12-a520-c3604be78018" />

GitHub Repository of App : https://github.com/dockersamples/example-voting-app
