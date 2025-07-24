# 🚀 Reusable GitHub Workflow: Multi-Cloud Docker Build with BuildKit

This reusable GitHub Actions workflow builds and pushes Docker images using [BuildKit](https://github.com/moby/buildkit), with support for:

- ✅ AWS ECR via Pod Identity or AWS credentials  
- ✅ Azure ACR via Azure service principal  
- ✅ Build once, push to AWS, Azure, or both  
- ✅ GitHub-hosted and GitHub ARC (self-hosted) runners  
- ✅ Automatic fallback tag using GitHub run ID  

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

## 🧪 Versioning Strategy

This workflow is currently in **beta**. While it's stable and production-ready for most use cases, we recommend starting with pinned versions (e.g. `@v1`) as best practice.

| Version    | Description                         |
|------------|-------------------------------------|
| `v1`       | Stable major version                |
| `v1.0.0`   | Exact pinned release for reproducibility |
| `v2`, `v3` | Reserved for future breaking changes |

---

## 📄 License

MIT © 2025 Affinity7 Consulting Ltd
