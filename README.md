# Monitoring setup for Gerrit

This project provides a setup for monitoring Gerrit instances. The setup is
based on Prometheus and Grafana running in Kubernetes. In addition, logging will
be provided by Grafana Loki.

The setup is provided as a helm chart. It can be installed using Helm
(This README expects Helm version 3.0 or higher).

The charts used in this setup are the chart provided in the open source and can be
found on GitHub:

- [Prometheus](https://github.com/helm/charts/tree/master/stable/prometheus)
- [Grafana](https://github.com/helm/charts/tree/master/stable/grafana)
- [Loki](https://github.com/grafana/loki/tree/master/production/helm/loki)

This project just provides `values.yaml`-files that are already configured to
work with the `metrics-reporter-prometheus`-plugin of Gerrit to make the setup
easier.

## Dependencies

### Software

- Gerrit \
Gerrit requires the following plugin to be installed:
  - [metrics-reporter-prometheus](https://gerrit.googlesource.com/plugins/metrics-reporter-prometheus/)

- Promtail \
Promtail has to be installed with access to the `logs`-directory in the Gerrit-
site. A configuration-file for Promtail will be provided in this setup. Find
the documentation for Promtail
[here](https://github.com/grafana/loki/blob/master/docs/clients/promtail/README.md)

- Helm \
To install and configure Helm, follow the
[official guide](https://helm.sh/docs/intro/quickstart/#install-helm).

- ytt \
ytt is a templating tool for yaml-files. It is required for some last moment
configuration. Installation instructions can be found
[here](https://k14s.io/#install-from-github-release).

- Pipenv \
Pipenv sets up a virtual python environment and installs required python packages
based on a lock-file, ensuring a deterministic Python environment. Instruction on
how Pipenv can be installed, can be found
[here](https://github.com/pypa/pipenv#installation)

### Infrastructure

- Kubernetes Cluster \
A cluster with at least 3 free CPUs and 4 GB of free memory are required. In
addition persistent storage of about 30 GB will be used.

- Ingress Controller \
The charts currently expect a Nginx ingress controller to be installed in the
cluster.

- Object store \
Loki will store the data chunks in an object store. This store has to be callable
via the S3 API.

## Add dashboards

To have dashboards deployed automatically during installation, export the dashboards
to a JSON-file or create JSON-files describing the dashboards in another way.
Put these dashboards into the `./dashboards`-directory of this repository. During
the installation the dashboards will be added to a configmap and with this
automatically installed to Grafana.

## Configuration

While this project is supposed to provide a specialized and opinionated monitoring
setup, some configuration is highly dependent on the specific installation.
These options have to be configured in the `./config.yaml` before installing and
are listed here:

| option                                   | description                                                                        |
|------------------------------------------|------------------------------------------------------------------------------------|
| `gerritServers.[0].host`                 | Hostname (incl. port, if required) of the Gerrit server to monitor                 |
| `gerritServers.[0].username`             | Username of Gerrit user with 'View Metrics' capabilities                           |
| `gerritServers.[0].password`             | Password of Gerrit user with 'View Metrics' capabilities                           |
| `gerritServers.[0].promtail.storagePath` | Path to directory, where Promtail is allowed to save files (e.g. `positions.yaml`) |
| `gerritServers.[0].promtail.logPath`     | Path to directory containing the Gerrit logs (e.g. `/var/gerrit/logs`)             |
| `namespace`                              | The namespace the charts are installed to                                          |
| `tls.skipVerify`                         | Whether to skip TLS certificate verification                                       |
| `tls.caCert`                             | CA certificate used for TLS certificate verification                               |
| `prometheus.server.host`                 | Prometheus server ingress hostname                                                 |
| `prometheus.server.username`             | Username for Prometheus                                                            |
| `prometheus.server.password`             | Password for Prometheus                                                            |
| `prometheus.server.tls.cert`             | TLS certificate                                                                    |
| `prometheus.server.tls.key`              | TLS key                                                                            |
| `prometheus.alertmanager.slack.apiUrl`   | API URL of the Slack Webhook                                                       |
| `prometheus.alertmanager.slack.channel`  | Channel to which the alerts should be posted                                       |
| `loki.host`                              | Loki ingress hostname                                                              |
| `loki.username`                          | Username for Loki                                                                  |
| `loki.password`                          | Password for Loki                                                                  |
| `loki.s3.protocol`                       | Protocol used for communicating with S3                                            |
| `loki.s3.host`                           | Hostname of the S3 object store                                                    |
| `loki.s3.accessToken`                    | The EC2 accessToken used for authentication with S3                                |
| `loki.s3.secret`                         | The secret associated with the accessToken                                         |
| `loki.s3.bucket`                         | The name of the S3 bucket                                                          |
| `loki.s3.region`                         | The region in which the S3 bucket is hosted                                        |
| `loki.tls.cert`                          | TLS certificate                                                                    |
| `loki.tls.key`                           | TLS key                                                                            |
| `grafana.host`                           | Grafana ingress hostname                                                           |
| `grafana.tls.cert`                       | TLS certificate                                                                    |
| `grafana.tls.key`                        | TLS key                                                                            |
| `grafana.admin.username`                 | Username for the admin user                                                        |
| `grafana.admin.password`                 | Password for the admin user                                                        |
| `grafana.ldap.enabled`                   | Whether to enable LDAP                                                             |
| `grafana.ldap.host`                      | Hostname of LDAP server                                                            |
| `grafana.ldap.port`                      | Port of LDAP server (Has to be `quoted`!)                                          |
| `grafana.ldap.password`                  | Password of LDAP server                                                            |
| `grafana.ldap.bind_dn`                   | Bind DN (username) of the LDAP server                                              |
| `grafana.ldap.accountBases`              | List of base DNs to discover accounts (Has to have the format `"['a', 'b']"`)      |
| `grafana.ldap.groupBases`                | List of base DNs to discover groups (Has to have the format `"['a', 'b']"`)        |
| `grafana.dashboards.editable`            | Whether dashboards can be edited manually in the UI                                |

### Encryption

The configuration file contains secrets. Thus, to be able to share the configuration,
e.g. with the CI-system, it is meant to be encrypted. The encryption is explained
[here](./documentation/config-management.md).

The `gerrit-monitoring.py install`-command will decrypt the file before templating,
if it was encrypted with `sops`.

## Installation

Before using the script, set up a python environment using `pipenv install`.

The installation will use the environment of the current shell. Thus, make sure
that the path for `ytt`, `kubectl`and `helm` are set. Also the `KUBECONFIG`-variable
has to be set to point to the kubeconfig of the target Kubernetes cluster.

This project provides a script to quickly install the monitoring setup. To use
it, run:

```sh
pipenv run python ./gerrit-monitoring.py \
  --config config.yaml \
  install \
  [--output ./dist] \
  [--dryrun] \
  [--update-repo]
```

The command will use the given configuration (`--config`/`-c`) to create the
final files in the directory given by `--output`/`-o` (default `./dist`) and
install/update the Kubernetes resources and charts, if the `--dryrun`/`-d` flag
is not set. If the `--update-repo`-flag is used, the helm repository will be updated
before installing the helm charts. This is for example required, if a chart version
was updated.

## Configure Promtail

Promtail has to be installed with access to the directory containing the Gerrit
logs, e.g. on the same host. The installation as described above will create a
configuration file for Promtail, which can be found in `./dist/promtail.yaml`.
Use it to configure Promtail by using the `-config.file=./dist/promtail.yaml`-
parameter, when starting Promtail. Using the Promtail binary directly this would
result in the following command:

```sh
$PATH_TO_PROMTAIL/promtail \
  -config.file=./dist/promtail.yaml
```

If TLS-verification is activated, the CA-certificate used for verification
(usually the one configured for `tls.caCert`) has to be present in the
directory configured for `promtail.storagePath` in the `config.yaml` and has to
be called `promtail.ca.crt`.

The Promtail configuration provided here expects the logs to be available in
JSON-format. This can be configured by setting `log.jsonLogging = true` in the
`gerrit.config`.

## Uninstallation

To remove the Prometheus chart from the cluster, run

```sh
helm uninstall prometheus --namespace $NAMESPACE
helm uninstall loki --namespace $NAMESPACE
helm uninstall grafana --namespace $NAMESPACE
kubectl delete -f ./dist/configuration
```

To also release the volumes, run

```sh
kubectl delete -f ./dist/storage
```

NOTE: Doing so, all data, which was not backed up will be lost!

Remove the namespace:

```sh
kubectl delete -f ./dist/namespace.yaml
```

The `./gerrit-monitoring.py uninstall`-script will automatically remove the
charts installed in the configured namespace and delete the namespace as well:

```sh
pipenv run python ./gerrit-monitoring.py \
  --config config.yaml \
  uninstall
```
