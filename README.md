# 🚀 Reusable GitHub Workflow for Multi-Cloud Docker Image Builds (Docker or BuildKit)

> 🚀 Now powered by **BuildKit** for blazing-fast container builds.

Build and push Docker images to AWS ECR or Azure ACR using Docker (on GitHub-hosted runners) or BuildKit (on self-hosted runners). Supports both pod-managed identity and credential-based authentication.





### Key Features

- 🐳 **Docker** on GitHub-hosted runners  
- 🏗️ **BuildKit** on GitHub ARC (self-hosted) runners  
- ✅ AWS ECR via pod identity or IAM credentials  
- ✅ Azure ACR via workload identity, node identity, or service principal  
- ✅ Build once, push to AWS, Azure, or both  
- ✅ Automatically tags images with the GitHub run ID, and optionally with a custom tag if provided.

> ⚠️ **Alpha Notice**  
> This workflow is currently in **alpha** while Azure support is being finalized.  
> AWS ECR builds via Docker (hosted) and BuildKit (self-hosted) are working and production-ready.  
> Azure ACR support will be validated and stabilized before tagging a `v1.0.0` release.


---

## ⚙️ Runtime Requirements

| Environment         | Requirements                                                                 |
|---------------------|------------------------------------------------------------------------------|
| GitHub-hosted       | No setup required – uses Docker with `setup-buildx-action`                  |
| Self-hosted (ARC)   | Requires a running `buildkitd` daemon listening on `tcp://localhost:12345`  |

On GitHub ARC (self-hosted runners), this workflow assumes that a [BuildKit](https://github.com/moby/buildkit) daemon is running as a **sidecar container** in the same pod as the ARC runner.

The `buildkitd` sidecar should:
- Be reachable at `localhost:12345`
- Be configured with **minimal required Linux capabilities** (see note below)
- Not require privileged mode or Docker socket access

---

## 🛡️ Security Note

ℹ️ In our internal deployments, `buildkitd` runs in a hardened sidecar with minimal Linux capabilities (`SYS_ADMIN`, `CHOWN`, `FOWNER`, `DAC_OVERRIDE`) and does **not** require privileged mode or access to the Docker socket.

---

## 🧑‍💻 Runner Compatibility

This workflow is compatible with both:

- **GitHub-hosted runners**, e.g. `ubuntu-latest`
- **Self-hosted ARC runners**, e.g. `aws-linux-self-hosted-build-runner`

Use the `runner` input to specify the appropriate runner label.

---

## 🔧 Inputs

| Name             | Required | Type    | Description |
|------------------|----------|---------|-------------|
| `path`           | ❌        | string  | Path to Docker build context (default: `.`) |
| `image`          | ✅        | string  | Docker image name (e.g. `gitopsmanager`) |
| `tag`            | ❌        | string  | Optional tag. If omitted, uses GitHub `run_id`. If provided, both it and the `run_id` will be pushed |
| `build_file`     | ❌        | string  | Path to the Dockerfile (relative to context) |
| `extra_args`     | ❌        | string  | Additional flags passed to buildkit |
| `aws_region`     | ❌        | string  | AWS region for ECR |
| `aws_registry`   | ❌        | string  | AWS ECR registry hostname |
| `azure_registry` | ❌        | string  | Azure ACR registry hostname |
| `push_aws`       | ❌        | boolean | Set `true` to push to AWS ECR (default: `false`) |
| `push_azure`     | ❌        | boolean | Set `true` to push to Azure ACR (default: `false`) |
| `runner`         | ❌        | string  | Runner label (default: `ubuntu-latest`) |
| `use_azure_secrets` | ❌    | boolean | Whether Azure credentials are passed manually (default: `false`) |

---

## 🔐 Secrets

These are only needed if you're not using pod/managed identity:

| Secret                 | Description |
|------------------------|-------------|
| `AWS_ACCESS_KEY_ID`    | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY`| AWS IAM secret key |
| `AZURE_CLIENT_ID`      | Azure app client ID |
| `AZURE_CLIENT_SECRET`  | Azure client secret |
| `AZURE_TENANT_ID`      | Azure AD tenant ID |

---

---

## 🔐 Identity Options

This workflow supports multiple authentication mechanisms depending on your cloud platform and runner environment.

### ✅ AWS Authentication Options

| Method                    | Supported? | Notes |
|---------------------------|------------|-------|
| **Pod Identity (IRSA)**   | ✅ Yes      | Recommended for self-hosted ARC runners on EKS. Automatically used when AWS credentials are not supplied. |
| **Node IAM Role**         | ✅ Yes      | Supported if the ARC runner pod runs on an EC2 instance with an IAM role attached. |
| **AWS Secrets**           | ✅ Yes      | Provide `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. If not provided, AWS SDK falls back to pod or instance identity. |

> ℹ️ You **do not need a flag** to indicate how AWS credentials are supplied.  
> The AWS CLI and SDKs automatically use available environment variables, pod identity, or instance roles based on availability and precedence.

✅ Regardless of the authentication method, `aws ecr get-login-password` is still required to authenticate Docker to ECR.

---

### ✅ Azure Authentication Options

| Method                             | Supported? | Notes |
|------------------------------------|------------|-------|
| **Azure Workload Identity (Pod Identity)** | ✅ Yes | Recommended for self-hosted ARC runners on AKS. Set `use_azure_secrets: false`. |
| **Node Managed Identity**         | ✅ Yes      | Supported when the ARC runner runs on a VM/VMSS with a system-assigned or user-assigned identity. Use `use_azure_secrets: false`. |
| **Azure Secrets**                 | ✅ Yes      | Provide `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, and `AZURE_TENANT_ID`. Must explicitly set `use_azure_secrets: true`. |

> ⚠️ Azure requires an explicit `use_azure_secrets` flag.  
> If set to `false`, the workflow assumes federated identity or node-managed identity.

---

### 🧪 Credential Selection Behavior

| Cloud  | Secrets Provided? | Fallback Behavior                    |
|--------|--------------------|--------------------------------------|
| AWS    | Optional           | Falls back to pod or node identity   |
| Azure  | Must be explicit   | Requires `use_azure_secrets: true` if using secrets |

---

## 🧪 Example Usage

```yaml
jobs:
  build:
    uses: your-org/k8s-build/.github/workflows/buildkit-build.yaml@v1
    with:
      path: .
      image: gitopsmanager
      tag: latest
      aws_registry: 123456789.dkr.ecr.eu-west-1.amazonaws.com
      azure_registry: myregistry.azurecr.io
      push_aws: true
      push_azure: true
      runner: aws-linux-self-hosted-build-runner
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```

---

## 🛠 Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| [actions/checkout](https://github.com/actions/checkout) | `v4` | Clones your repo for context |
| [azure/login](https://github.com/Azure/login) | `v1` | Secure Azure authentication for ACR |
| [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action) | `v3` | Enables BuildKit for GitHub-hosted runners |
| [docker/build-push-action](https://github.com/docker/build-push-action) | `v5` | Performs the BuildKit build and push |


---

---

## 📦 Alpha Status & Planned Versioning Strategy

> ⚠️ **This workflow is currently in alpha.**  
> It has been tested on AWS (Docker + BuildKit), but Azure ACR support is still undergoing validation.  
> Once Azure support is confirmed, a stable `v1.0.0` release will be published.

### 🚧 Current State

- No official version tags yet
- Use `@main` for now while testing
- Breaking changes may still occur until `v1.0.0` is tagged

### 🔢 Planned Versioning Strategy

| Version Tag | Purpose                                 |
|-------------|------------------------------------------|
| `v1`        | Floating tag pointing to latest stable `v1.x` |
| `v1.0.0`    | First stable release once all platforms are tested |
| `v1.0.1+`   | Patches and improvements (non-breaking) |
| `v2`        | Reserved for future breaking changes     |

### 🧪 Example Usage (for testing)

```yaml
jobs:
  build:
    uses: gitopsmanager/k8s-build/.github/workflows/build.yaml@main
```

---

To be notified when stable tags are available, watch the [Releases tab](https://github.com/gitopsmanager/k8s-build/releases).


---

## 📄 License

MIT © 2025 Affinity7 Consulting Ltd
