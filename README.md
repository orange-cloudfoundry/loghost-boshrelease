<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [loghost-boshrelease](#loghost-boshrelease)
    - [Introduction](#introduction)
    - [Components](#components)
        - [Concentrator](#concentrator)
            - [Format](#format)
            - [Generated files](#generated-files)
            - [Rotation](#rotation)
            - [Forwarding and clustering](#forwarding-and-clustering)
        - [Dns](#dns)
        - [Exporter](#exporter)
        - [Alerts](#alerts)
        - [Dashboards](#dashboards)
    - [Usage](#usage)
        - [Step 1: Deploy loghost](#step-1-deploy-loghost)
        - [Step 2: Forward all logs to loghost instance](#step-2-forward-all-logs-to-loghost-instance)
        - [Step 3: Add alerts and dashboard to prometheus](#step-3-add-alerts-and-dashboard-to-prometheus)
    - [Reference](#reference)
        - [Ops-files](#ops-files)
        - [Metrics](#metrics)

<!-- markdown-toc end -->

# loghost-boshrelease

This is a BOSH release to gather, store and analyze syslog events forwarded by bosh VMs. It
currently uses [RSyslog] which is pre-installed by the stemcell.

Only Linux stemcells are supported at the moment.

## Introduction

Usually, platform logs are sent to ELK stacks which store and index events on-the-fly.
Finally, users can build fancy Kibana dashboards by extracting metrics from elasticsearch queries.

With the development of micro-services architectures, the number of emitted logs recently exploded,
making these ELKs very hardware and, therefore, money consuming.
Even more, these stacks are often built with heavy redundancy and high availability even when most of the emitted events are not critical.

The idea here is having a much more lightweight architecture, providing only the most essential
features of log processing:
- midterm storage for debug and production incident analysis
- hardware-efficient generation of metrics
- redundancy and availability matching the actual criticality of the logs

This is achieved by using both good old technologies such as [RSyslog] and modern
tools like [Prometheus].
The bridge between logs and metrics is provided by a brilliant tool [grok_exporter].

## Components

### Concentrator

The job `loghost_concentrator` configures local `rsyslogd` to store received logs to persistent disk.

#### Format

Only syslog events received in [RFC5424] format with `instance@47450` in
[Structured Data ID][structured-data] are handled. The `47450` private enterprise number is the one
generated by the [syslog-release] generally used to forward VM log events to a given endpoint.

#### Generated files

Received logs are stored on persistent disk in root directory `/var/vcap/store/loghost` where
`{path}` depends on parsed [Structured Data ID][structured-data] fields of the event.

Assuming logs are forwarded by [syslog-release], the parsed fields are:
- `$.director`: the configured name of bosh director
- `$.deployment`: the name of deployment from which event was sent
- `$.group`: the name of the instance from which event was sent

Finally, logs are stored under `/var/vcap/store/loghost/{$.director}/{$.deployment}/{$.group}.log`

#### Rotation

The job also configures local `logrotate` in order to rotate and compress logs every hour.
Rotated logs are stored in the same directories with the `-%Y%m%d%H.gz` suffix.

The number of kept rotations can be configured `loghost_concentrator.logrotate.max-hours` property
with a default value of `360` (ie: 15 days).

#### Forwarding and clustering

The job also provides the possibility to re-forward received syslog event under specified
conditions; this can be useful for:
- Clusterize multiple concentrators in order to create a kind of backup across independent BOSH directors
- Forward business or security critical events to an external log handling platform

![clustering]

Forwarding is configured from the `loghost_concentrator.syslog.forward` property by defined
target objects as follows:

```yml
<target-name>:
  conditions:
  - <condition>
  - ...
  targets:
    - address: hostname
      port: port
      transport: tcp|udp
    - ...
```

**Where:**

- `<condition>` are valid [rainerscript] expressions with parenthesis. Multiple conditions can be
  given, all must be true to trigger the forward
- `<targets>`: is a list of syslog endpoints where matching events are forwarded. When multiple
  targets are defined, matching events will be forwarded to all endpoints

**Example:**

```yml
jobs:
  - name: loghost_concentrator
    release: loghost
    properties:
      loghost_concentrator:
        syslog:
          forward:
            my-forward-target:
              conditions:
              - ($.director   isequal "local-director-name")
              - ($.deployment isequal "cf")
              targets:
                - address: target1.hostname.example.com
                  port: 514
                  transport: tcp
                - address: target2.hostname.example.com
                  port: 514
                  transport: tcp
```

### DNS

Assuming that your deployment uses [bosh-dns], the job `loghost_dns` can be used to define new
aliases.

DNS aliases are configured from the `loghost_dns.aliases` key with the same syntax as the
`aliases` key of [bosh-dns job][bosh-dns-job].

**Example:**

```yaml
jobs:
  - name: loghost_dns
    release: loghost
    properties:
      loghost_dns:
        aliases:
          my.alias.internal:
          - 127.0.0.1
          my.other.alias.internal:
          - *.collector-z1.default.logsearch.bosh
          - *.collector-z2.default.logsearch.bosh
```

### Exporter

The `loghost_exporter` job installs and configures the [grok_exporter]. This brilliant program
processes log files and computes [Prometheus] metrics according to parse rules given in
[grok] format.

Parsing rules are defined by the `loghost_exporter.metrics` key with the exact same syntax defined
by the [grok_exporter-metrics].

In addition, `loghost_exporter.directors` and `loghost_exporter.deployments` keys must be configured
to give the list of logs files that the exported should watch.

> **Note**: A limitation in the grok_exporter implementation forces watched directories to pre-exist
> at exporter startup. Because rsyslog files are created on the fly when events are received, the
> job creates required directories in its `pre-start` script.

In addition to user defined metrics, the exporter provides
[builtin metrics][grok-builtin-metrics].

Ops-files provided in the release also provide metrics, as described in the [usage section](#usage).

### Alerts

The job `loghost_alerts` defines the following alerts for your [prometheus-boshrelease] deployment:
- `LoghostNoLogReceived`: triggers if exporter reports no processed logs in the last 15 minutes
- `LoghostDroppedMessages`: triggers when there is an increase of "failed to write to target.example.net:6067" in the logs

When `loghost_alerts.security.enabled` key is set to `true` (default `false`), the job also defines
the following alerts:
- `SecurityTooManySystemAuthFailures`: triggers when `audispd` reports too many auth failures.
  `audispd` logs are generated by all virtual machines deployed by bosh
- `SecurityTooManyUaaClientFailures`:  triggers when `uaa` component reports too many
   client authentication failures
- `SecurityTooManyUaaUserFailures`: triggers when `uaa` component reports too many
   user authentication failures
- `SecurityTooManyDiegoSshFailures`: triggers when `ssh_proxy` component running on (`scheduler`
   instance) reports too many SSH authentication failures to containers
- `SecurityTooManyDiegoSshSuccess`: triggers when `ssh_proxy` component running on (`scheduler`
   instance) reports too many SSH authentications to containers

Alert thresholds and evaluation time can be configured from job's spec.

### Dashboards

The job `loghost_dashboards` adds [Grafana] dashboards for your [prometheus-boshrelease]
deployment.

- a global overview giving the system status, number of processed logs per rules, deployments and
  instances

- a security dashboard overview giving information on authentications when
  `loghost_dashboards.security.enabled` key is enabled.


## Usage

### Step 1: Deploy loghost

First, you must add `loghost` instance to the deployment of your choice. You can use the following
ops-files:
- `manifests/operations/loghost-concentrator-enable.yml`
- `manifests/operations/loghost-exporter-enable.yml`
- `manifests/operations/loghost-exporter-enable-security.yml`

It will add the instance `loghost` with basic features enabled:
- received log written to `/var/vcap/store/loghost`
- [grok_exporter] reading and generating metrics from received logs

### Step 2: Forward all logs to loghost instance

The simplest way to forward all logs at once is to create a `runtime-config.yml` using the [syslog-release].


With file `runtime-syslog-forward.yml`:

```yaml
addons:
- exclude:
    instance_groups:
    - loghost
  jobs:
  - name: syslog_forwarder
    properties:
      syslog:
        address: q-s0.loghost.default.((deployment)).bosh
        director: ((director_name))
        transport: udp
    release: syslog
  name: syslog_forwarder
releases:
- name: syslog
  sha1: 658fe5d6f049ec50383c09c0b227261251bfd4eb
  url: https://artifactory/cloudfoundry/syslog/syslog-11.6.1-ubuntu-xenial-621.tgz
  version: 11.6.1
```

Upload to bosh director: `bosh update-runtime-config --name syslog-forward runtime-syslog-forward.yml`

### Step 3: Add alerts and dashboard to prometheus

Add the following ops-files to your prometheus deployment:

- `manifests/operations/prometheus/loghost-enable.yml`
- `manifests/operations/prometheus/loghost-enable-security.yml`

It will:

- define scrape config based on `bosh_exporter` discovery
- define new alerts
- add dashboards to Grafana

## Reference

### Ops-files

| name                                   | description                                                                           |
|----------------------------------------|---------------------------------------------------------------------------------------|
| loghost-concentrator-enable.yml        | add  instance with `loghost_concentrator` job listening on udp                        |
| loghost-concentrator-enable-tcp.yml    | configure `loghost_concentrator` to listen on tcp addition to udp                     |
| loghost-dns-enable.yml                 | add `loshost_dns` job with empty aliases list                                         |
| loghost-exporter-enable.yml            | add `loghost_exporter` job which spawns `grok_exporter` with a default set of metrics |
| loghost-exporter-enable-security.yml   | add security metrics to `loghost_exporter` job, grok rules for `uaa` and `audispd`    |
| prometheus/loghost-enable.yml          | add discovery scraping of `grok_exporter`, default alerts and dashboards              |
| prometheus/loghost-enable-security.yml | add security alerts and dashboards                                                    |

### Metrics

In addition to [grok_exporter] [grok-builtin-metrics], the release defines:

| name                                     | dimensions                                        | type      | description                                                  |
|------------------------------------------|---------------------------------------------------|-----------|--------------------------------------------------------------|
| loghost_total                            | director, deployment, group                       | (Counter) | log processed                                                |
| loghost_error_total                      | director, deployment, group                       | (Counter) | log detected as level *error*                                |
| loghost_auth_failures                    | director, deployment, group, source, ip           | (Counter) | system authentication failures                               |
| loghost_auth_failures_last_5m            | director, deployment, group, source, ip           | (Gauge)   | system authentication failures in the last 5 minutes (*)     |
| loghost_auth_success                     | director, deployment, group, source, ip, username | (Counter) | system authentication success                                |
| loghost_auth_success_last_5m             | director, deployment, group, source, ip, username | (Gauge)   | system authentication success in the last 5 minutes (*)      |
| loghost_uaa_client_login_success         | director, deployment, group, ip, clientid         | (Counter) | UAA client authentication success                            |
| loghost_uaa_client_login_success_last_5m | director, deployment, group, ip, clientid         | (Gauge)   | UAA client authentication success in the last 5 minutes (*)  |
| loghost_uaa_client_login_failure         | director, deployment, group, ip, clientid         | (Counter) | UAA client authentication failures                           |
| loghost_uaa_client_login_failure_last_5m | director, deployment, group, ip, clientid         | (Gauge)   | UAA client authentication failures in the last 5 minutes (*) |
| loghost_uaa_user_login_success           | director, deployment, group, ip, username         | (Counter) | UAA user authentication success                              |
| loghost_uaa_user_login_success_last_5m   | director, deployment, group, ip, username         | (Gauge)   | UAA user authentication success in the last 5 minutes (*)    |
| loghost_uaa_user_login_failure           | director, deployment, group, ip, username         | (Counter) | UAA user  authentication failures                            |
| loghost_uaa_user_login_failure_last_5m   | director, deployment, group, ip, username         | (Gauge)   | UAA user failures in the last 5 minutes (*)                  |

With dimension values:
- `director`, `deployment`, `group`: BOSH director name, deployment name and instance group name
   from where the log was originally emitted
- `source`: the `exe` field of type=`USER.*` message of `audispd`
- `ip`: the remote address from which the authentication was attempted
- `clientid`: the `clientid` used to authenticate a client on `UAA`
- `username`: the `username` used to authenticate a user on `UAA`

> **(*) Tech note**: Because metrics dimensions values are created over time depending on encountered
> logs, we cannot rely on `rate` or `increase` prometheus function to compute the number of failures
> on a period of time. As a bypass, we manually compute this metric with a hackish record rule
> defined as:
> ```
>   sum(<metric> or <metric>{} * 0) by (<dimensions...>)
>   -
>   sum(<metric> offset 5m or <metric>{} * 0) by (<dimensions...>)
> ```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->

[RSyslog]: http://www.rsyslog.com/
[RFC5424]: https://tools.ietf.org/html/rfc5424
[syslog-release]: https://github.com/cloudfoundry/syslog-release#format
[Grafana]: https://grafana.com/
[prometheus-boshrelease]: https://github.com/bosh-prometheus/prometheus-boshrelease
[grok-builtin-metrics]: https://github.com/fstab/grok_exporter/blob/master/BUILTIN.md
[structured-data]: https://tools.ietf.org/html/rfc5424#section-6.3.2
[clustering]: ./doc/clustering.png
[rainerscript]: https://www.rsyslog.com/doc/v8-stable/rainerscript/index.html
[bosh-dns]: https://github.com/cloudfoundry/bosh-dns-release
[bosh-dns-job]: https://github.com/cloudfoundry/bosh-dns-release/blob/master/jobs/bosh-dns/spec
[grok_exporter]: https://github.com/fstab/grok_exporter
[Prometheus]: https://prometheus.io/
[grok]: https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html
[grok_exporter-metrics]: https://github.com/fstab/grok_exporter/blob/master/CONFIG.md#metrics-section
