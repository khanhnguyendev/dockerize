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

Note: Replace placeholders such as `<image_name>`, `<container_id>`, `<host_port>`, `<container_port>`, etc., with your actual values. This cheat sheet covers basic Docker commands and Docker Compose commands.

# Do not run containers as the root user
- `Privilege Escalation`: If an attacker gains control of a process within a container running as the root user, they might be able to escalate their privileges and potentially compromise the host system. Running containers with non-root users helps contain potential security breaches within the container.

- `Host File System Access`: A container running as root has more access to the host file system, which increases the risk of unintended modifications or deletions of important system files. Running containers with non-root users limits the impact of potential file system-related security issues.

- `Security Vulnerabilities`: Some software inside containers may have security vulnerabilities. Running as a non-root user limits the potential damage that an attacker can do in case they exploit such vulnerabilities.

- `Principle of Least Privilege`: The principle of least privilege suggests that processes and users should have the minimum level of access required to perform their tasks. Running containers with the root user violates this principle and increases the attack surface.