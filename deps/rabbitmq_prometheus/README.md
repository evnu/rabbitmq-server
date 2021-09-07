[![Build](https://img.shields.io/github/workflow/status/rabbitmq/rabbitmq-prometheus/Test)](https://github.com/rabbitmq/rabbitmq-prometheus/actions?query=workflow%3ATest)
[![Grafana Dashboards](https://img.shields.io/badge/Grafana-6%20dashboards-blue)](https://grafana.com/orgs/rabbitmq)

# Prometheus Exporter of Core RabbitMQ Metrics

## Getting Started

This is a Prometheus exporter of core RabbitMQ metrics, developed by the RabbitMQ core team.
It is largely a "clean room" design that reuses some prior work from Prometheus exporters done by the community.

## Project Maturity

This plugin is new as of RabbitMQ `3.8.0`.

## Documentation

See [Monitoring RabbitMQ with Prometheus and Grafana](https://www.rabbitmq.com/prometheus.html).


## Installation

This plugin is included into RabbitMQ 3.8.x releases. Like all [plugins](https://www.rabbitmq.com/plugins.html), it has to be
[enabled](https://www.rabbitmq.com/plugins.html#ways-to-enable-plugins) before it can be used:

To enable it with [rabbitmq-plugins](http://www.rabbitmq.com/man/rabbitmq-plugins.1.man.html):

``` shell
rabbitmq-plugins enable rabbitmq_prometheus
```

## Usage

See the [documentation guide](https://www.rabbitmq.com/prometheus.html).

Default port used by the plugin is `15692` and the endpoint path is at `/metrics`.
To try it with `curl`:

```shell
curl -v -H "Accept:text/plain" "http://localhost:15692/metrics"
```

In most environments there would be no configuration necessary.

See the entire list of [metrics](metrics.md) exposed via the default port.


## Configuration

This exporter supports the following options via a set of `prometheus.*` configuration keys:

 * `prometheus.return_per_object_metrics` returns [individual (per object) metrics that are not aggregated](https://www.rabbitmq.com/prometheus.html#metric-aggregation) (default is `false`).
 * `prometheus.path` defines a scrape endpoint (default is `"/metrics"`).
 * `prometheus.tcp.*` controls HTTP listener settings that match [those used by the RabbitMQ HTTP API](https://www.rabbitmq.com/management.html#configuration)
 * `prometheus.ssl.*` controls TLS (HTTPS) listener settings that match [those used by the RabbitMQ HTTP API](https://www.rabbitmq.com/management.html#single-listener-https)

Sample configuration snippet:

```ini
# these values are defaults
prometheus.return_per_object_metrics = false
prometheus.path = /metrics
prometheus.tcp.port =  15692
```

When metrics are returned per object, nodes with 80k queues have been measured to take 58 seconds to return 1.9 million metrics in a 98MB response payload.
In order to not put unnecessary pressure on your metrics system, metrics are aggregated by default.

When debugging, it may be useful to return metrics per object (unaggregated).

This can be done by scraping the `/metrics/per-object` endpoint:
```shell
curl -v -H "Accept:text/plain" "http://localhost:15692/metrics/per-object"
```

This can also be enabled as the default behavior of the `/metrics` endpoint on-the-fly,
without restarting or configuring RabbitMQ, using the following command:

```
rabbitmqctl eval 'application:set_env(rabbitmq_prometheus, return_per_object_metrics, true).'
```

To go back to aggregated metrics on-the-fly, run the following command:

```
rabbitmqctl eval 'application:set_env(rabbitmq_prometheus, return_per_object_metrics, false).'
```

## Reducing number of metrics

As mentioned in the previous section, returning a lot of metrics is a computationally intensive process. 

Default endpoints `/metrics` and `/metrics/per-object` expose every possible, including low-level stats like Erlang VM stats and higher-level things like queue stats.

Using aggregation is one way to reduce the number of metrics and somewhat reduce CPU usage. 

It's also possible to completely disable some groups of higher-level metrics that provide excessive level of detail, or pose no interest at all in given circumstances. For queue metrics it's also possible to filter on a per-vhost basis - that can be useful if there is a way to choose less interesting queues (like one-off transient queues for RPC) and don't include them in the output.

Those customizations can be applied to default endpoints using a configuration file. The following config snippet will make default endpoints to expose just enough information to get a number of messages and and a number of consumers for each queue, but only in the default vhost `/`:

```ini
prometheus.core_metrics.default_families.1 = queue_coarse_metrics
prometheus.core_metrics.default_families.2 = queue_consumer_count
prometheus.core_metrics.default_vhosts.1 = /
```

Even when the number of metrics for the default endpoints is reduced, it's possible to get any combination of those configurable metrics via a separate endpoint, where HTTP `GET`-parameters determine what exactly should be returned. E.g. scraping `/metrics/core?vhost=vhost-1&vhost=vhost-2&family=queue_coarse_metrics&family=queue_consumer_count`. will only return requested metrics (and not, for example, low-level Erlang VM metrics). It supports the following parameters:

* Zero or more `family` - if given, only these metric families will be returned. The full list is documented in [metrics](metrics.md), and it's the same names that are being used in the config file.
* Zero or more `vhost` - if it's given, queue related metrics (`queue_coarse_metrics`, `queue_coarse_metrics` and `queue_metrics`) will be returned only for given vhost(s).
* Optional `per-object` - if it's `1` or `true`, per-object values without aggregation will be returned.

## Contributing

See [CONTRIBUTING.md](https://github.com/rabbitmq/rabbitmq-prometheus/blob/master/CONTRIBUTING.md).


## Makefile

This project uses [erlang.mk](https://erlang.mk/), running `make help` will return erlang.mk help.

To see all custom targets that have been documented, run `make h`.

For Bash shell autocompletion, run `eval "$(make autocomplete)"`, then type `make a<TAB>` to see all Make targets starting with the letter `a`, e.g.:

```sh
$ make a<TAB
ac               all.coverdata    app-build        apps             apps-eunit       asciidoc-guide   autocomplete
all              app              app-c_src        apps-ct          asciidoc         asciidoc-manual
```


## Copyright

(c) 2007-2020 VMware, Inc. or its affiliates.
