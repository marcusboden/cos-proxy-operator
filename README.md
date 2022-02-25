# COS Proxy charm

This Juju machine charm that provides a single integration point in the machine world
with the Kubernetes-based [COS bundle](https://charmhub.io/cos-lite).

This charm is designed to be easy to integrate in bundles and Juju-driven appliances,
and reduce the amount of setup needed to integrate with the Kubernetes-based COS to
just connect the COS Proxy charm with it.

Proxying support is provided for:

* Prometheus
* Grafana Dashboards
* NRPE (through [nrpe_exporter](https://github.com/canonical/nrpe_exporter), which sends
NRPE results to Prometheus

## Deployment

The cos-proxy charm is used as a connector between a Juju model hosting
applications on machines, and COS charms running within Kubernetes.
In the following example our machine charms will be running on an OpenStack
cloud, and the Kubernetes is Microk8s running on a separate host.  There must be
network connectivity from each of the endpoints to the Juju controller.

For example, we have two models.  One, named 'reactive', hosting machine charms
running on OpenStack.  There is a Telegraf application, cs:telegraf, collecting
metrics from units, and we wish to relate that to Prometheus and Grafana running
in another model named cos, running Kubernetes.

Here's the steps to create the models:

```
$ juju clouds
Only clouds with registered credentials are shown.
There are more clouds, use --all to see them.

Clouds available on the controller:
Cloud             Regions  Default      Type
microk8s-cluster  1        localhost    k8s
serverstack       1        serverstack  openstack

juju add-model reactive serverstack
juju add-model cos microk8s-cluster
```

Next we'll deploy an example application on the `reactive` model:

```
juju deploy -m reactive cs:ubuntu --series focal -n 3
juju deploy -m reactive cs:telegraf
juju relate -m reactive telegraf:juju-info ubuntu:juju-info
```

To relate Telegraf to Prometheus in order to add scrape targets and alerting
rules, we must use a cross model relation.

Offer the relation in the cos model:

```
juju offer cos.prometheus:metrics-endpoint
```

Deploy the cos-proxy charm in a new machine unit on the target model:

```
juju deploy -m reactive cos-proxy  # or ./cos-proxy_ubuntu-20.04-amd64.charm --resource nrpe-exporter=./nrpe_exporter
juju relate -m reactive telegraf:prometheus-client cos-proxy:prometheus-target
juju relate -m reactive telegraf:prometheus-rules cos-proxy:prometheus-rules
```

Add the cross model relation:

```
juju consume -m reactive cos.prometheus
juju relate -m reactive prometheus cos-proxy:downstream-prometheus-scrape
```

Now we can do the same for Grafana

```
juju offer cos.grafana:grafana_dashboard
juju relate -m reactive telegraf:dashboards cos-proxy:dashboards
```

Add the cross model relation:

```
juju consume -m reactive cos.prometheus
juju consume -m reactive cos.grafana
juju relate -m reactive prometheus cos-proxy:downstream-prometheus-scrape
juju relate -m reactive grafana cos-proxy:downstream-grafana-dashboards
```

## NRPE Exporting

The `nrpe_exporter` binary used as a resource can be built from the [nrpe_exporter](https://github.com/canonical/nrpe_exporter)
repository by cloning and running `make`. `docker` is a build dependency, since `nrpe_exporter` (and `nrpe` itself)
use older SSL ciphers, and a connection cannot be negotiated in pure Go.

NRPE targets may appear on multiple relations. To capture all jobs, `cos-proxy` should be related to
**BOTH** an existing reactive NRPE subordinate charm, as well as the application which that charm is subordinated to,
as the `monitors` interface may appear on either, with the principal charm providing "host-level" checks, and
the subordinate `nrpe` providing application-level ones.
