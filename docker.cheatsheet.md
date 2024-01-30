# Docker Command Cheat Sheet

| Purpose                                      | Command                                          |
|----------------------------------------------|--------------------------------------------------|
| **Build an image from a Dockerfile**          | `docker build -t <image_name> .`                 |
| **Run a container**                           | `docker run -d -p <host_port>:<container_port> <image_name>` |
| **Run an interactive container**              | `docker run -it <image_name>`                     |
| **Run a container with volume mount**         | `docker run -v /host/path:/container/path <image_name>` |
| **Run a container and remove on exit**        | `docker run --rm -it <image_name>`                |
| **List running containers**                   | `docker ps`                                      |
| **List all containers (including stopped)**  | `docker ps -a`                                   |
| **Stop a running container**                  | `docker stop <container_id>`                      |
| **Remove a container**                        | `docker rm <container_id>`                        |
| **List images**                              | `docker images`                                  |
| **Remove an image**                           | `docker rmi <image_id>`                           |
| **Inspect container details**                 | `docker inspect <container_id>`                  |
| **Inspect image details**                     | `docker inspect <image_name>`                    |
| **View container logs**                       | `docker logs <container_id>`                     |
| **Enter a running container's shell**         | `docker exec -it <container_id> /bin/bash`       |
| **View Docker version**                       | `docker --version`                               |
| **Pull an image from Docker Hub**             | `docker pull <image_name>`                       |
| **Push an image to Docker Hub**               | `docker push <image_name>`                       |
| **Docker Compose: Start services**            | `docker-compose up -d`                           |
| **Docker Compose: Stop services**             | `docker-compose down`                            |
| **Docker Compose: View logs**                 | `docker-compose logs`                            |
| **Docker Compose: Build and start**           | `docker-compose up --build -d`                   |
| **Show the History of an image**              | `docker history --human --format "{{.CreatedBy}} => {{.Size}}" <image_name>`|

Note: Replace placeholders such as `<image_name>`, `<container_id>`, `<host_port>`, `<container_port>`, etc., with your actual values. This cheat sheet covers basic Docker commands and Docker Compose commands.
