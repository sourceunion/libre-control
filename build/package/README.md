# Container Definitions (Dockerfiles)

This directory contains the instructions for building the immutable container images that power the LibreControl microservices architecture.

## Strategy

We utilize **Multi-Stage Builds** for all components. This approach allows us to:

1. **Compile** the Go binaries in a heavy image containing the full toolchain.
2. **Discard** the toolchain and source code.
3. **Copy** only the resulting binary to a lightweight runtime image, significantly reducing the attack surface and image size.