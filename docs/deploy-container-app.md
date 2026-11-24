# Deploy Container App

Reusable workflow that deploys an image tag from Azure Container Registry to an Azure Container App.

1. Checks that the image tag exists in the registry.
2. Updates the container app to `<registry>.azurecr.io/<image>:<version>`.
3. Moves `latest` to the same image, as a fallback for infrastructure jobs.
4. Notifies Slack (optional).
5. Deletes old images, keeping `latest`, the deployed tag and the 5 newest other tags.

## Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    uses: DirektoratetForByggKvalitet/felles-workflows/.github/workflows/deploy-container-app.yml@main
    with:
      environment: dev
      image: ftpb-email-api
      version: 2.0.42
      registry: myregistrydev
      resource-group: my-resource-group
      container-app: my-container-app
    secrets: inherit
```

The caller must grant `id-token: write`, which Azure login uses for OIDC.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `environment` | yes | `dev`, `test` or `prod`. Also the GitHub environment the jobs run in. |
| `image` | yes | Image name in the registry. |
| `version` | yes | Image tag to deploy. |
| `registry` | yes | Registry name, without `.azurecr.io`. |
| `resource-group` | yes | Resource group of the container app. |
| `container-app` | yes | Name of the container app. |
| `auto-notify-slack` | no | Default `true`. Set to `false` when deploying several apps from one repo and send your own notification. |
| `override-branch-rules` | no | Default `false`. Allows deploying prod from a branch other than `main`. |

## Secrets

| Secret | Required |
| --- | --- |
| `ARM_CLIENT_ID` | yes |
| `ARM_TENANT_ID` | yes |
| `ARM_SUBSCRIPTION_ID` | yes |
| `ARKITEKTUM_SLACK_CICD_WEBHOOK_URL` | no, only needed when `auto-notify-slack` is `true` |

The Azure identity needs a federated credential for `environment:<environment>` in the calling repository.

## Things to know

- **Prod is deployed from `main` only**, unless `override-branch-rules` is set. For a hard guarantee, also restrict deployment branches on the `prod` environment in the calling repo.
- **Concurrency** is grouped per container app and environment (`deploy-<container-app>-<environment>`). Don't use the same group name in the caller, or the runs will deadlock.
- **Each environment has its own registry**, so moving `latest` and cleaning up only affect that environment.
- **Shared image repositories:** cleanup only protects the tag that was just deployed. If several container apps run the same image repository, cleaning up after one app can delete the tag another app is running. Give each app its own image repository, or extend the cleanup to keep every tag in use.
- **Cleanup failures** fail the run, but are not reported to Slack.
