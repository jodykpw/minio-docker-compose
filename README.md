# MinIO Deployment with Docker, Traefik, and GitLab CI/CD

This repository contains a GitLab CI/CD pipeline and a Docker Compose configuration for automating the deployment and management of a MinIO service, including logging integration via Fluentd, and routing via Traefik.

## Requirements

1. **GitLab CI/CD**: This pipeline is designed to run on GitLab CI/CD.
2. **SSH Setup**: SSH access to the remote server must be configured:
   - A valid SSH private key stored in GitLab CI/CD variables.
   - The remote server's IP address stored in `REMOTE_IPADDRESS`.
   - Ensure the user has sufficient permissions to manage Docker containers and volumes on the remote server.
3. **Docker and Docker Compose** installed on the remote server.
4. **Traefik** installed and configured as an external service.
5. **Fluentd** or **Fluent Bit** must be set up for logging integration with either Elasticsearch or Loki.

## Pipeline Variables

The following variables are used in the pipeline:

- `REMOTE_SERVER_DEPLOY_FOLDER`: Directory where the project will be deployed on the remote server.
- `SSH_DOCKER_USER`: Username for SSH access to the remote server.
- `REMOTE_IPADDRESS`: IP address of the remote server.
- `SSH_PRIVATE_KEY`: SSH private key for authentication.
- `MINIO_ROOT_USER`: Username for the MinIO service.
- `MINIO_ROOT_PASSWORD`: Password for the MinIO service.

## Pipeline Stages

The pipeline consists of several stages:

1. **Test**:
   - `test_ssh_connection`: Verifies SSH connectivity to the remote server.

2. **Deploy**:
   - `ssh_scp_docker-compose-up`: Deploys the MinIO service to the remote server using Docker Compose. It copies the project files via `rsync` and starts the Docker containers.
   
3. **Shutdown**:
   - `ssh_scp_docker-compose-down`: Stops and removes the Docker containers defined in the `docker-compose.yml` file.

4. **Destroy**:
   - `ssh_scp_docker-compose-remove`: Stops the containers and removes associated volumes.

5. **Cleanup**:
   - `cleanup_deployment_failure`: Cleans up the remote deployment folder in the event of a failed deployment.

## Docker Compose Configuration

The `docker-compose.yml` file defines the setup for MinIO with Traefik routing, volume persistence, and Fluentd logging.

### Services

#### MinIO

- **Image**: `minio/minio:RELEASE.2024-09-09T16-59-28Z`
- **Ports**:
  - `9000`: MinIO data service port.
  - `9001`: MinIO console port.
- **Environment Variables**:
  - `MINIO_ROOT_USER`: Root user for MinIO.
  - `MINIO_ROOT_PASSWORD`: Root password for MinIO.
  - `MINIO_PROMETHEUS_AUTH_TYPE`: Set to `public` for Prometheus metrics.
  - `MINIO_PROMETHEUS_URL`: URL for the Prometheus server.
  - `MINIO_PROMETHEUS_JOB_ID`: Identifier for Prometheus metrics collection.
- **Command**: Runs MinIO with the server and console addresses specified.
- **Volumes**:
  - `minio_data`: Mounted to persist MinIO data.
- **Networks**:
  - `traefik`: An external Traefik network is used for routing.
- **Healthcheck**: A simple TCP health check on port 9000 with retries.

### Traefik Integration

The service is integrated with Traefik using Docker labels:

- The router is configured to respond to the domain `minio-docker-1.domain.com`.
- Requests are forwarded to the MinIO console on port `9001`.
- Traefik uses the `web` entry point for handling incoming HTTP requests.

### Fluentd Logging

MinIO logs are sent to Fluentd for further processing:

- **Logging Driver**: `fluentd`.
- **Fluentd Address**: Logs are forwarded to Fluentd at `127.0.0.1:24224`.
- **Tags**: The logs are tagged with `elastic.minio`, indicating they will be processed and sent to an Elasticsearch backend.

## Usage

### 1. Deployment

To deploy the MinIO service, manually trigger the `ssh_scp_docker-compose-up` job from the GitLab CI/CD pipeline. This will:

- Upload project files to the remote server using `rsync`.
- Start the MinIO service on Docker.

### 2. Service Shutdown

To stop and remove the MinIO service, manually trigger the `ssh_scp_docker-compose-down` job.

### 3. Destroying Services and Volumes

To remove the containers and associated volumes, trigger the `ssh_scp_docker-compose-remove` job.

### 4. Cleanup on Failure

If a deployment fails, the `cleanup_deployment_failure` job will automatically remove the project folder on the remote server.

## Healthcheck

MinIO is monitored with a TCP health check on port `9000`. If the service is unresponsive, the system will retry the connection three times before marking it as failed.

## Prometheus Integration

MinIO is configured to expose Prometheus metrics publicly, with the metrics endpoint URL specified in the environment variables. Ensure Prometheus is running at the configured URL to scrape metrics from MinIO.

### Customising .env

It is **not recommended** to store sensitive information such as passwords directly in the `.env` file and push it to the repository, as this can expose critical data. Instead, sensitive values such as passwords should be stored in GitLab CI/CD environment variables for secure management.

The `.env` file can still be used for non-sensitive configuration settings. For example:

```env
# Non-sensitive configurations can be kept here
COMPOSE_PROJECT_NAME=minio
```

For sensitive configurations like username and passwords, store these in GitLab CI/CD variables, these GitLab CI/CD variables will automatically be injected into the pipeline during the deployment process, ensuring security and preventing the exposure of sensitive information in your codebase.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🇬🇧🇭🇰 Author Information

* Author: Jody WAN
* Linkedin: https://www.linkedin.com/in/jodywan/
* Website: https://www.jodywan.com