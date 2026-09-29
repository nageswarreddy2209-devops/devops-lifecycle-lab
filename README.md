# DevOps lifecycle lab: Flask, containers, and Kubernetes

A student-friendly starting point for learning CI/CD with OCI DevOps, OCIR,
and an existing Oracle Kubernetes Engine (OKE) cluster.

**This is a learning lab, not a production-ready platform.** The Flask development
server is deliberately retained from the original exercise. The `prod` namespace
demonstrates promotion; it does not provide production-grade isolation.

## Lifecycle

1. Commit application changes to Git.
2. CI validates code, builds a container, and runs endpoint tests.
3. Push the image to OCIR using the full Git commit SHA as its tag.
4. Deliver the image artifact and trigger the release pipeline.
5. Deploy that tag to `flask-test` and verify health.
6. Wait for a human approval.
7. Promote the same image tag to `flask-prod`, without rebuilding.

Commit tags provide traceability but are not inherently immutable. Do not overwrite
release tags; stronger release controls should deploy a verified image digest.

## What is included

- `app.py`: Flask application with `/` and `/health` endpoints.
- `tests/test_app.py`: basic endpoint tests.
- `Dockerfile`: container image for local use and CI.
- `build_spec.yaml`: configurable OCI managed-build example.
- `k8s/flask-test.yaml`, `k8s/flask-prod.yaml`: deployment/service templates with
  two replicas, health probes, resource limits, and rolling updates.

The public template removes the original tenancy identifiers and private Git
history. Its configuration is intentionally different from the running training
tenancy. OCI resources, credentials, IAM policies, and pipeline stages are **not**
created by cloning this repository. There is no Jenkinsfile or GitHub Actions
workflow yet; publishing to GitHub does not change the existing OCI pipeline source.

## Run and test locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m unittest discover -s tests -v
python app.py
```

In another terminal:

```bash
curl --fail http://localhost:8080/
curl --fail http://localhost:8080/health
```

Or build and run the container:

```bash
docker build -t flask-devops-app:local .
docker run --rm -p 127.0.0.1:8080:8080 flask-devops-app:local
```

## Configure your own OCI lab

Cloud resources can incur charges. Use your own authorized training compartment,
check quotas and IAM permissions, and remove resources you create when finished.
Never delete a shared cluster or shared namespace as lab cleanup.

### 1. Registry and cluster

Create a private OCIR repository named `flask-devops-app`. Use an existing permitted
OKE cluster or provision your own. Configure kubectl access and create the namespaces:

```bash
kubectl create namespace flask-test
kubectl create namespace flask-prod
```

Provision a Kubernetes image-pull secret named `ocir-pull-secret` in **each**
namespace using an approved secret-management process. It must authorize pulls
from your OCIR repository. No secret manifest is included here.

### 2. CI configuration

Create an OCI DevOps project and managed-build pipeline, connect your source
repository, and select `build_spec.yaml`. A public GitHub repository still needs
the appropriate OCI source connection configuration. The runner needs Docker
(or a compatible CLI) and Python 3; it needs network access to the source repo,
base-image registry, Python package index, and OCIR.

Set these runtime values for your account:

| Name | Purpose |
| --- | --- |
| `REGISTRY_HOST` | Regional OCIR hostname, for example `iad.ocir.io` |
| `REGISTRY_NAMESPACE` | Your tenancy's Object Storage/OCIR namespace |
| `REGISTRY_USERNAME` | Full OCIR login username, including identity domain where required |
| `REGISTRY_TOKEN` | OCI auth-token secret, never a console password or token OCID |
| `IMAGE_TAG` | Exported automatically from `OCI_PRIMARY_SOURCE_COMMIT_HASH` |

Prefer OCI Vault for `REGISTRY_TOKEN`. Plain pipeline parameters are **not a secret
store**: readers may see values, and parameter forwarding can copy them into
deployment records. Do not log the token or commit it. If your lab has no Vault
access, review the exposure with your administrator before using a temporary token
and revoke it when the exercise is complete.

Create an OCIR image artifact named `flask-app-image` with parameter substitution
enabled and source URI:

```text
${REGISTRY_HOST}/${REGISTRY_NAMESPACE}/flask-devops-app:${IMAGE_TAG}
```

Add a Deliver Artifacts stage after the managed build, mapping the build output
`flask-app-image` to that artifact. The example performs an explicit registry push
and retains OCI's delivery stage to illustrate both steps. A hardened pipeline
can use native artifact delivery with properly scoped resource-principal IAM
instead of the explicit token-based login.

### 3. Release configuration

Create an OKE deployment environment referencing your cluster. Configure its
resource-principal IAM/RBAC and network access to the cluster, scoped to the
required resources. Do not blindly grant tenancy-wide administrator access.

Create a release pipeline with these parameters:

- `REGISTRY_HOST` and `REGISTRY_NAMESPACE`: same values as CI.
- `IMAGE_TAG`: no usable default; supply the successful build's exact SHA.

Register each file in `k8s/` as a Kubernetes manifest artifact with parameter
substitution enabled. Configure the stages **sequentially**:

1. Deploy-test: test manifest, namespace `flask-test`.
2. Approve-production: manual approval, at least one authorized approver.
3. Deploy-prod: production manifest, namespace `flask-prod`.

Add a Trigger Deployment stage after CI's Deliver Artifacts stage targeting this
release pipeline. Forward the build parameters so the release receives `IMAGE_TAG`.
Review the credential-forwarding warning above before enabling "pass all parameters".

### 4. Verify the release

Run CI once. Check all build stages and the automatic release's image tag. Confirm
test pods are ready, then inspect the app locally through port-forwarding:

```bash
kubectl rollout status deployment/flask-devops-app -n flask-test --timeout=180s
kubectl get pods,svc -n flask-test
kubectl port-forward -n flask-test service/flask-devops-app 8080:80
```

Use the curl commands above from a second terminal. Only approve production after
the test checks pass, then repeat the rollout check in `flask-prod`.

Both Services are `ClusterIP`, so **the app is not exposed to the internet**.
Public access requires a separately configured ingress/load balancer, network
rules, DNS, and TLS. Port-forwarding defaults to localhost and is for testing.

## Jenkins extension (future exercise)

Jenkins can replace OCI's build pipeline while retaining the same release pipeline:
check out Git, test/build on an agent, push a SHA-tagged image, then call OCI to
create a deployment with `IMAGE_TAG`. Store registry credentials in Jenkins
Credentials and give the agent narrowly scoped OCI authorization. Disable the
old CI trigger when switching so one commit does not cause duplicate releases.

## Troubleshooting and next lessons

- Registry login failure: check full username, valid auth-token secret, and region.
- Registry push 403: check repository existence, compartment, and IAM permissions.
- ImagePullBackOff: check the image tag and pull secret in the target namespace.
- Deployment authorization failure: check pipeline identity, IAM, and cluster RBAC.
- A release "in progress" at the approval stage is expected until approval/rejection.
- Never repeat a timed-out create command until checking whether the first run exists.

Before real production use: add a production WSGI server, non-root execution,
dependency/image scans, digest-based promotion, stronger tests, network policies,
separate environment access boundaries, monitoring, rollback procedures, and TLS.

## References

- [OCI build specifications](https://docs.oracle.com/en-us/iaas/Content/devops/using/build_specs.htm)
- [Trigger Deployment stage](https://docs.oracle.com/en-us/iaas/Content/devops/using/triggerdeploy_stage.htm)
- [OCI DevOps IAM policies](https://docs.oracle.com/en-us/iaas/Content/devops/using/devops_iampolicies.htm)
- [Jenkins integration](https://docs.oracle.com/en-us/iaas/Content/devops/using/devops_plugin.htm)
