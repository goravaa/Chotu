# chotu

Rent a private GPU machine for coding work. Use it. Give it back.

## Demo

[Watch the Chotu demo](https://youtu.be/XPaYZT3ZX5Q)

## Why this exists

I don't own a GPU machine. A few times a month I hit the ceiling of what I can run locally, and some of that work involves private client data I don't want to paste into someone else's chat box. So instead of buying hardware or sending data to a hosted chat service, chotu spins up a single cloud GPU box, sets itself up, and tears itself down when I'm done. The workspace can sync with the S3 bucket configured for the deployment, so data handling depends on that configuration and the services you enable.

## What it does

One `terraform apply` gets you a working box with:

- A local LLM endpoint (vLLM serving Qwen3-Coder-30B) reachable only over SSH, with no public ports
- OpenCode wired to it, so it behaves like a normal coding setup
- Your workspace synchronized with the configured S3 bucket, so the box starts with your code
- A status page that shows setup progress while the box converges, then system health after
- Spot support, so a disposable run costs noticeably less than on demand

When you're done, `terraform destroy` and it's gone: no lingering VM, no data left behind anywhere except your bucket.

## How to use

You need three things installed and working: the AWS CLI, configured so `aws sts get-caller-identity` prints your account, terraform, and this repo cloned locally. Then:

```
cd terraform
terraform init
terraform plan    # check what gets built
terraform apply   # wait ~5 minutes for setup to converge
```

Apply prints an `ssh_config_snippet`. Paste it into `~/.ssh/config`, then:

```
ssh chotu
```

With that session open: `http://localhost:8081` is the status page (setup progress, then system health), `localhost:8000` is the model endpoint. When you're done, keep the bucket and destroy everything else:

```
# Forget the bucket resources so destroy leaves the data alone
terraform state rm 'module.aws[0].aws_s3_bucket.workspace[0]' 'module.aws[0].aws_s3_bucket_versioning.workspace[0]' 'module.aws[0].aws_s3_bucket_public_access_block.workspace[0]'
terraform destroy
```

The bucket stays in AWS. On the next deploy, set `s3_create_bucket = false` to adopt it. A plain `terraform destroy` without the `state rm` fails on purpose (`prevent_destroy` on the bucket) rather than deleting workspace data.

Tweaks live in `terraform/variables.tf`: instance type, region, spot vs on demand, bucket, and model. No other files need changes for a normal deploy.

## Releases

The current tagged version is **v0.1.0**. It provisions one AWS EC2 GPU machine from the Terraform AWS module, downloads the `chotu-setup` and `status-service` Linux binaries from the matching GitHub release, and starts them as systemd services. The release publishes the `status-service` Docker image for standalone use, but Terraform bootstraps the host binaries instead of deploying that image. The inference container is the separately pinned vLLM image configured in Terraform.

This release supports the AWS machine path described above. The repository has Vast.ai and GCP Terraform modules, but they are not documented here as production-ready release targets. The setup is a single machine, and the local model’s quality, supported GPU hardware, and configured integrations limit what it can do. It does not provide a general hosted service or guarantee data never leaves the machine: workspace synchronization to S3 is enabled by default and data sent to configured external tools follows their own handling policies. Review the configuration before using private data.

See [CHANGELOG.md](CHANGELOG.md) and [RELEASE_NOTES.md](RELEASE_NOTES.md) for this release’s scope. Maintainers can publish a new version by pushing a `vMAJOR.MINOR.PATCH` tag; see [docs/releasing.md](docs/releasing.md) for secrets and manual retry instructions.

## What it won't do (yet)

Honest version: a 30B quantized model is not going to architect your system or untangle your hardest bug. For that kind of work it's mediocre. What it's genuinely good at is routine work over private data, such as linting, writing and running tests, and churning through data analysis scripts. That work is boring but can stay under your control. If that's your bottleneck, this removes it for a few dollars.

## How it works

Terraform provisions one EC2 GPU instance (g5/g6) and cloud init writes the machine config. A bootstrap script installs two small Go daemons: one that converges the box (drivers, Docker, model pull, sync) and one that serves the status page. Access is only available over SSH with generated keys. The status page and model endpoint come through tunnels. The setup daemon is idempotent, so a fresh box always lands in the same state.

## Where it's going

- More clouds: Vast.ai and GCP modules, so I can chase cheaper or faster GPUs per job instead of being tied to one provider
- Bigger models (70B+) for when the task actually needs the reasoning, priced per run via spot
- Generally: whatever removes the next bottleneck in my own development speed

Built for freelance client work where I need local model inference and control over where workspace data is stored and synchronized.
