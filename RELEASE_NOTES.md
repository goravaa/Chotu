# Chotu v0.1.0

This is the first tagged Chotu release. It packages the two Go daemons used by the bootstrap process and a standalone container image for the status service.

## Release contents

- `chotu-setup-linux-amd64` and `chotu-setup-linux-arm64`: setup daemon binaries.
- `chotu-status-linux-amd64` and `chotu-status-linux-arm64`: status service binaries.
- `chotu-setup-SHA256SUMS` and `chotu-status-SHA256SUMS`: checksums for their corresponding binaries.
- `garvw/chotu-status:v0.1.0`: amd64 container image for the status service.

The AWS Terraform bootstrap downloads the two binaries for the machine architecture from this GitHub release and runs them as systemd services. The container image is published for standalone use; Terraform does not use it. The inference container is the separately configured, digest-pinned vLLM image.

## Scope and limitations

The documented deployment provisions one AWS EC2 GPU machine. Its setup daemon configures Docker and NVIDIA container support, starts the configured vLLM model, configures OpenCode when enabled, synchronizes the workspace with S3 when enabled, and reports setup and machine status through the status service. Terraform’s AWS defaults expose SSH only; the model endpoint and status page are reached through SSH tunnels.

This release does not establish that the Vast.ai or GCP modules are production-ready, and it does not provide multi-machine orchestration, a public model endpoint, or a general hosted coding service. Model quality and hardware compatibility depend on the selected model and machine. With the default configuration, workspace data is synchronized to S3, so “private” does not mean data stays only on the GPU machine. Review the selected cloud, bucket, model, and enabled integrations before deployment.
