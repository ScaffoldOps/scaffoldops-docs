# GitHub Actions and self-hosted runners

Each app has `pr-checks.yml`, `develop-pipeline.yml`, `main-pipeline.yml` and reusable `deploy-k8s.yml`. PR checks run Maven verify with tests skipped plus separate clean test jobs. Develop/main pushes verify/test, build, push and deploy to DEV/PRE. Docker push rebuilds rather than assuming an image is shared between runner jobs.

| Component | Published image | Tags/deployed version |
| --- | --- | --- |
| generator-api | `victodomvar/scaffoldops-generator-api` | Maven version + short SHA, branch alias; deploy immutable tag |
| generator-worker | `victodomvar/scaffoldops-generator-worker` | `latest`, full SHA; deploy latest |
| deployment-worker | `victodomvar/scaffoldops-deployment-worker` | `latest`, full SHA; deploy latest |

Both worker branch pipelines share `latest`; concurrent DEV/PRE builds can replace that tag. PRE is a workflow target, not a fully provisioned current environment. The generator-worker reusable workflow applies a Kustomization fixed to DEV even when invoked for PRE; see [findings](findings.md).

## Runner setup

Register a Linux x64 self-hosted runner for each repository or an organization runner with repository access. Use the installation/registration commands supplied by GitHub repository Settings → Actions → Runners, including the short-lived registration token, then start the runner under the intended Linux user. Runner installation directories are operational assets and must not be committed.

The runner needs Git, Java 17/Maven, Linux Docker CLI/daemon access, network access to dependency registries/Docker Hub, kubectl and a kubeconfig reaching Minikube. Worker pipelines require `/usr/bin/docker`; workflow deployment selects context `minikube`. Check `docker version`, `kubectl cluster-info` and permissions as the runner user. Infra jobs use labels `[self-hosted, Linux, X64]` and hard-code `KUBECONFIG=/home/victor/.kube/config`; adapt that assumption when moving runners. Provision app secrets and target RBAC before deployment.

## Infrastructure workflows

`platform-infra.yml` validates local-dev and dev renders on PRs and main/develop pushes and checks live Kubernetes access. `deploy.yml` is manual, accepts `environment` (`dev` or `pre`) and `runBootstrapJobs`; it validates/applies the overlay, optionally recreates bootstrap Jobs and waits for rollout. PRE is rejected because its overlay is absent.

```bash
gh workflow run deploy.yml --repo ScaffoldOps/platform-infra --ref develop   -f environment=dev -f runBootstrapJobs=true
gh run list --repo ScaffoldOps/platform-infra --workflow deploy.yml
gh run list --repo ScaffoldOps/generator-api --workflow develop-pipeline.yml
```

App `deploy-k8s.yml` supports only `workflow_call`, so `gh workflow run deploy-k8s.yml` is not supported. Trigger app delivery by an authorized push to its configured branch or rerun an existing pipeline with `gh run rerun RUN_ID --repo ScaffoldOps/COMPONENT`. Pipeline delivery is an actual cluster change.
