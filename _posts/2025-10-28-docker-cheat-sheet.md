---
title: "My Docker cheat sheet"
date: 2025-10-28
---

My personal Docker / Docker Compose cheat sheet. This is just for quick reference.

## Core Docker commands

| Command                                 | Description
| --------------------------------------- | -------------------------------------------------------------------------------
| `docker ps`                             | List running containers. Add `-a` to show stopped containers.
| `docker images`                         | List local images.
| `docker pull <image>`                   | Download an image from a registry.
| `docker run <image>`                    | Run a new container from an image. (Use flags like `-d`, `-p`, `--name`, `-v`.)
| `docker exec -it <container> <command>` | Run an interactive command inside a running container.
| `docker logs <container>`               | Print container logs. Add `-f` to follow.
| `docker stop <container>`               | Gracefully stop a running container.
| `docker kill <container>`               | Force-stop a running container.
| `docker start <container>`              | Start an existing (stopped) container.
| `docker restart <container>`            | Restart a container.
| `docker rm <container>`                 | Remove a stopped container. Add `-f` to force removal.
| `docker rmi <image>`                    | Remove a local image.
| `docker stats`                          | Live resource usage statistics for containers.
| `docker system prune -a`                | Clean unused containers, images, networks (use carefully).


## Docker image management

| Command                             | Description                                              |
| ----------------------------------- | -------------------------------------------------------- |
| `docker build -t <name>:<tag> .`    | Build an image from a `Dockerfile` in current directory. |
| `docker tag <image> <repo>:<tag>`   | Tag an image for pushing to a registry.                  |
| `docker push <repo>:<tag>`          | Push a tagged image to a registry.                       |
| `docker save -o <file>.tar <image>` | Export an image to a tar archive.                        |
| `docker load -i <file>.tar`         | Import an image from a tar archive.                      |


## Docker Compose essentials

| Command                                   | What it does                                                                                          |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `docker compose up`                       | Create and start all services defined in `docker-compose.yml`.                                        |
| `docker compose up -d`                    | Run services in detached mode.                                                                        |
| `docker compose down`                     | Stop and remove containers, networks, and default volumes created by `up`.                            |
| `docker compose build`                    | Build or rebuild services defined in the Compose file.                                                |
| `docker compose ps`                       | List containers for the Compose project.                                                              |
| `docker compose logs`                     | Show logs for all services. Add `-f` to follow.                                                       |
| `docker compose exec <service> <command>` | Run a command in a running service container.                                                         |
| `docker compose run <service> <command>`  | Run a **one-off** command in a new container for that service (useful for migrations, tests, shells). |
| `docker compose restart`                  | Restart services.                                                                                     |
| `docker compose pull`                     | Pull updated images for services.                                                                     |
| `docker compose config`                   | Validate and view the combined Compose configuration.                                                 |


## Some examples

```bash
# Run nginx in detached mode, map port 8080 -> 80, name the container
docker run -d -p 8080:80 --name my-nginx nginx

# Get an interactive shell inside a running container
docker exec -it my-container bash

# Build an image from Dockerfile and tag it
docker build -t myapp:latest .

# One-off interactive shell for the "web" service (temporary container removed with --rm)
docker-compose run --rm web bash

# Start compose services in background
docker-compose up -d

# Follow logs for a specific compose service
docker-compose logs -f db

# Remove stopped containers and unused images/networks
docker system prune -a

```

## Common flags

| Flag        | Description                                                                                                                   | Example                                                               |
| --------    | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `-d`        | Run the container in the **background** (detached mode), so your terminal isn’t blocked.                                      | `docker run -d nginx`                                                 |
| `-p`        | Map a **host port** to a **container port** - enables access from outside. Format: `host:container`.                          | `docker run -d -p 8080:80 nginx` → access via `http://localhost:8080` |
| `--name`    | Assign a custom name to the container (otherwise Docker gives it a random name).                                              | `docker run -d --name my-nginx nginx`                                 |
| `-v`        | Mount a local directory or file into the container (for persistent data or code sharing). Format: `host_path:container_path`. | `docker run -d -v $(pwd)/site:/usr/share/nginx/html nginx`            |
| `-e`        | Set environment variables inside the container.                                                                               | `docker run -e ENV=production myapp`                                  |
| `--rm`      | Automatically remove the container after it exits.                                                                            | `docker run --rm ubuntu echo "Hello"`                                 |
| `-it`       | Interactive + TTY (useful for shells).                                                                                        | `docker run -it ubuntu bash`                                          |
| `--network` | Connect to a specific Docker network.                                                                                         | `docker run --network my-network myapp`                               |
