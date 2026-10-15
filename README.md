# fruit-database-gitops

Kubernetes manifests that deploy the [fruit-database](https://github.com/mostmark/fruit-database)
Quarkus demo application together with a MySQL database on OpenShift.

## Contents

| Path | Description |
|---|---|
| `manifests/` | One file per resource |
| `fruit-database.yaml` | All manifests from `manifests/` in a single file |

Resources:

- **MySQL**: `Secret`, `PersistentVolumeClaim` (1Gi, default storage class), `Deployment` (`registry.redhat.io/rhel8/mysql-80`) and `Service` (`mysql:3306`)
- **Application**: `Deployment` (`quay.io/mostmark/fruit-database:1.0`), `Service` and an edge-terminated `Route` (HTTP redirects to HTTPS)
- **Network policies**:
  - `deny-all-incoming` blocks all incoming traffic to every pod in the namespace
  - `allow-from-openshift-ingress` lets the OpenShift router reach the application
  - `allow-app-to-mysql` lets the application reach MySQL on TCP 3306

No resource sets a namespace, so everything is created in the current project.

## Deploy

From the command line:

```shell
oc new-project fruit-database
oc apply -f fruit-database.yaml
```

or apply the individual files with `oc apply -f manifests/`.

From the OpenShift console, select the project, click **+** (Import YAML) and paste the contents of `fruit-database.yaml`.

Then open the application:

```shell
echo "https://$(oc get route fruit-database -o jsonpath='{.spec.host}')"
```

## Demo: simulate a network fault

Remove the policy that lets the application reach the database:

```shell
oc delete networkpolicy allow-app-to-mysql
oc scale deployment/fruit-database --replicas=0
oc scale deployment/fruit-database --replicas=1
```

The application pod fails with `java.net.SocketTimeoutException: Connect timed out` and goes into
`CrashLoopBackOff`, and the route returns 503. Scaling to zero and back makes sure no previously running
pod keeps serving. Restore with `oc apply -f manifests/networkpolicy-allow-app-to-mysql.yaml`.

## Updating

When a file in `manifests/` changes, regenerate `fruit-database.yaml` so the two stay in sync.
