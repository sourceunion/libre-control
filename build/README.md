# Build & Orchestration

> [!CAUTION]
> Ensure that the `docker-compose.yml` maintains strict network segregation. The services should never map ports to the host machine's public interface in a production simulation.

This directory houses the infrastructure-as-code (IaC) assets required to containerize, orchestrate, and deploy the LibreControl environment.

## Structure

- **`package/`**: Contains the Dockerfiles for each distinct microservice.
- **`docker-compose.yml`**: The topology manifest defining the network isolation, volume mapping, and service dependencies.

## Usage

> [!TIP]
> These Dockerfiles are designed to be run from the **project root**, not from within this directory. This is required so the Docker daemon can access the shared `pkg/` and `internal/` directories.

To spin up the entire grid:

```
# From the project root
docker-compose -f build/docker-compose.yml up -d --build
```