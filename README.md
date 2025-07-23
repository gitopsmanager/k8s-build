# 🚀 Reusable GitHub Workflow: Multi-Cloud Docker Build with Kaniko

This reusable GitHub Actions workflow builds and pushes Docker images using [Kaniko](https://github.com/GoogleContainerTools/kaniko), with support for:

- ✅ AWS ECR via Pod Identity or AWS secrets
- ✅ Azure ACR via Azure service principal
- ✅ Single build, dual push support (e.g., AWS + Azure)
- ✅ GitHub-hosted and self-hosted runners (e.g., GitHub ARC)
- ✅ Automatic fallback tag using GitHub run ID

---

## ⚙️ Inputs

| Name             | Required | Type    | Description |
|------------------|----------|---------|-------------|
| `path`           | ❌        | string  | Path to Docker build context (default: `.`) |
| `image`          | ✅        | string  | Docker image name (e.g. `gitopsmanager`) |
| `tag`            | ❌        | string  | Optional tag. If omitted, uses GitHub `run_id`. If provided, both it and the `run_id` will be pushed |
| `build_file`     | ❌        | string  | Path to the Dockerfile (relative to context) |
| `extra_args`     | ❌        | string  | Additional flags passed to Kaniko |
| `aws_region`     | ❌        | string  | AWS region for ECR (default: `eu-west-1`) |
| `aws_registry`   | ❌        | string  | AWS ECR registry hostname |
| `azure_registry` | ❌        | string  | Azure ACR registry hostname |
| `push_aws`       | ❌        | boolean | Set `true` to push to AWS ECR (default: `false`) |
| `push_azure`     | ❌        | boolean | Set `true` to push to Azure ACR (default: `false`) |
| `runner`         | ❌        | string  | Runner label (default: `ubuntu-latest`) |

---

## 🔐 Secrets

These are only needed if your runners do not use pod identity (AWS) or managed identity (Azure):

| Secret                 | Description |
|------------------------|-------------|
| `AWS_ACCESS_KEY_ID`    | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY`| AWS IAM secret key |
| `AWS_SESSION_TOKEN`    | Optional for STS/SSO temporary credentials |
| `AZURE_CLIENT_ID`      | Azure app registration client ID |
| `AZURE_CLIENT_SECRET`  | Azure client secret |
| `AZURE_TENANT_ID`      | Azure AD tenant ID |

---

## 🛠 Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| [actions/checkout](https://github.com/actions/checkout) | `v4` | Clones your repo for context |
| [azure/login](https://github.com/Azure/login) | `v1` | Secure Azure authentication for ACR |
| [Kaniko (Chainguard)](https://github.com/chainguard-images/kaniko-project) | `v1.25.0` | Container-based Docker image builder |

---

## 🧑‍💻 Runner Compatibility

This workflow was originally built for **GitHub ARC (self-hosted)** runners, but is fully compatible with **GitHub-hosted runners** such as:

```yaml
runs-on: ubuntu-latest
```

You can configure runner labels via the `runner` input.

---

## ⚠️ GitHub ARC Runner Note

If you are using GitHub ARC (Actions Runner Controller) with self-hosted runners, be aware:

- The default `latest` GitHub ARC runner image **does not include Docker**
- This workflow uses `docker run` to execute the Kaniko container
- You must either:
  - Use a custom ARC runner image that includes Docker
  - Or mount the Docker socket from the host (not recommended in multi-tenant environments)

Example pod override for mounting the Docker socket:
```yaml
volumeMounts:
  - name: docker-sock
    mountPath: /var/run/docker.sock
volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock

---

## 🏗 How It Works: Single Build, Multi-Registry Push

This workflow performs the build **once** using Kaniko with `--no-push`, which:

- Builds the image from your `Dockerfile` into Kaniko’s cache
- Avoids rebuilding the image for each registry
- Then pushes to one or both targets (AWS ECR and/or Azure ACR) using `--destination`

> ✅ This ensures efficient image building without repeating work for multiple clouds.

---

## 🧪 Beta Release & Versioning

This workflow is currently in **beta**. While the structure is stable and production-ready for most use cases, we recommend starting with pinned versions (e.g. `@v1`) as best practice.

### Versioning Strategy:

- `v1`: Current stable release (latest in v1.x series)
- `v1.0.0`, `v1.0.1`, etc.: Pinned versions for reproducible builds
- Future breaking changes will be published under `v2`, `v3`, etc.

> We encourage feedback and contributions as we move toward a fully stable release cycle.

---

## 🧪 Example Usage

```yaml
jobs:
  build:
    uses: your-org/k8s-build/.github/workflows/kaniko-build.yaml@v1
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

## 📄 License

MIT © 2025 Affinity7 Consulting Ltd
