# Start the services defined in the docker-compose.yml file
docker compose -f docker-compose.yml up -d

# Stop a specific service
docker compose -f docker-compose.postgres.yml down

# Stop and also remove volumes (deletes data)
docker compose -f docker-compose.postgres.yml down -v
Other useful commands:

# See running containers
docker ps

# View logs of a container
docker logs postgres

# Stop all running containers at once
docker stop $(docker ps -q)