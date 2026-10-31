# Releasing Chotu

The release workflow publishes the Linux daemon binaries used by `bootstrap/bootstrap.sh`, their SHA-256 files, and the standalone amd64 `status-service` container. It then creates a GitHub release from `RELEASE_NOTES.md`. Terraform installs the release binaries directly; it does not pull the status-service image. The configured vLLM image is independently pinned in Terraform and is not built or published by this workflow.

## Registry credentials

The workflow publishes `garvw/chotu-status` to Docker Hub. In the GitHub repository, open **Settings → Secrets and variables → Actions → New repository secret** and add:

- `DOCKERHUB_USERNAME`: the Docker Hub account allowed to publish `garvw/chotu-status`.
- `DOCKERHUB_TOKEN`: a Docker Hub access token with permission to push to that repository.

The token is read only by the Docker login action from GitHub Actions secrets. Do not commit it or put it in a workflow file. GitHub release creation uses the workflow’s `contents: write` permission.

## Publish a version

1. Update `RELEASE_NOTES.md` and `CHANGELOG.md` for the new version. Keep the documented features grounded in the code and Terraform configuration.
2. Commit and push those changes to the default branch.
3. Create and push an annotated or lightweight tag matching `vMAJOR.MINOR.PATCH`, for example `v0.2.0`.
4. Check the **Actions → Release** run. A successful run publishes the image `garvw/chotu-status:vMAJOR.MINOR.PATCH` and creates the GitHub release with both architectures of each daemon binary and checksum files.

The workflow does not publish a floating `latest` image tag. The Chotu image and GitHub release use the same `v`-prefixed version. Terraform’s `chotu_release_version` must be set to that tag to deploy it.

## Retry a release manually

If the tag already exists and its run failed or did not start (as with the existing `v0.1.0` tag, which predates this workflow), open **Actions → Release → Run workflow**, select the default branch containing this workflow, enter the version without `v` (for example `0.1.0`), and run it. The workflow checks out `refs/tags/v<version>` and rebuilds from that tag. Review the run logs and confirm the GitHub release assets and Docker Hub tag before retrying again.

If publishing fails at Docker login or push, confirm both secrets and the account’s push access to `garvw/chotu-status`; then rerun the workflow with the same version. If the binary build or release upload fails, inspect that step and rerun the same version after correcting the cause. A manual run does not create or move a Git tag.
