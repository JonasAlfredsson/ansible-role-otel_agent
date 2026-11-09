# ansible-role-otel_agent

This role will deploy the [OTel Collector][1] as an "[agent][3]" on the targeted
machine for enrichment of all metrics collected from the local services. What
this means is that all services running on the targeted machine should send all
of their OTel stuff directly to _this_ collector, which can then add extra host
information, buffer and retry sending the data to the backend database. This
can significantly simplify auth, encryption and retry logic to a single place
instead of handling it in every separate service.

This role is a rewrite and simplification of the [opentelemetry_collector][2]
role managed by Grafana, but you will still find a lot of similarities between
them. I did this just to learn and to make it better suited for my setup.


## Installation

This repository does not have any dependencies, so just move into your `roles/`
folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-otel_agent.git otel_agent
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: all
  name: Install OTel Agent
  gather_facts: yes
  become: true
  tasks:
    - import_role:
        name: otel_agent
```


## Usage

The role in itself is quite simple, and the main variables you might want to
change are those related to choosing the version and directories:

```yaml
otel_agent_version: "latest"
otel_agent_installation_dir: "/opt/otel-collector-agent"
otel_agent_config_dir: "/etc/otel-collector-agent"
```

However, what is more interesting is to configure the OTel collector to behave
as you want. Thanks to the config file being YAML it is enables extreme freedom
here where you can just populate the following variables with the exact settings
you want:

```yaml
otel_agent_receivers: {}
otel_agent_processors: {}
otel_agent_exporters: {}
otel_agent_extensions: {}
otel_agent_service: {}
otel_agent_connectors: {}
```

## Example Configuration
This freedom might be overwhelming, so I will just share an example of my
setup.

### Recievers
This section contains all the configuration that I have added to the
`otel_agent_receivers` variable.

A "reciever" is just something that is able to collect data in some way, either
actively or by just listening for incoming messages.

**otlp** is the standard listener, which is needed if you want all your local
services to be able to send data to this agent that can then pass the data
along to the actual backend. See the [Exporters](#exporters) section for that
part.
```yaml
otlp:
  protocols:
    grpc:
      endpoint: 127.0.0.1:4317
      max_concurrent_streams: 1024
    http:
      endpoint: 127.0.0.1:4318
```

> Read the [Docker Workaround](#docker-workaround) section to get a solution
> on how to NOT expose the collector's ports outside localhost, while still
> routing the Docker container traffic to it.

**host_metrics** is a receiver which is able to collect data about your host
machine. I have actually split it into two separate receivers, one "fast" and
one "slow" to not have to call on the "heavy" queries that are needed in
collecting some of the metrics here.

> NOTE: the name after the `/` is just an identifier and has no function beyond
> that, so call them whatever you want.
```yaml
host_metrics/fast:
  # These are quick to gather, and change often, so we check these every 10s.
  collection_interval: 10s
  scrapers:
    cpu: null
    load: null
    memory: null
    network:
      exclude:
        interfaces: [lo, docker0, veth.*, br-.*, cni*]
        match_type: regexp

host_metrics/slow:
  # These are slower to gather, or don't change often, so just check every 1m.
  collection_interval: 1m
  scrapers:
    disk: null
    filesystem:
      exclude_fs_types:
        fs_types: [devtmpfs, overlay, squashfs]
        match_type: strict
      exclude_mount_points:
        match_type: regexp
        mount_points: [/proc, /sys, /dev, /run, /var/lib/docker/*]
    paging: null
    processes: null
```

**journal** is able to just tail the system log and send them to the backend.
But to make the collector faster to start we just attach to the stream, and
do not try to parse the system log from the beginning on every service restart.
```yaml
journald/system:
  convert_to_semantic_conventions: true
  start_at: end
```
However, the `dmesg` log starts way before the collector is ready, so here we
just print the kernel log and start from the beginning every time, which should
be fine since these log lines are far fewer.
```yaml
journald/dmesg:
  convert_to_semantic_conventions: true
  dmesg: true
  start_at: beginning
```

### Processors
This section contains all the configuration that I have added to the
`otel_agent_processors` variable.

A "processor" allows you to process the data in some way before it is sent to
the backend. This includes things like buffering the messages, as well as adding
host specific information to all the data leaving this collector. How these are
activated can then be seen in the [Services](#services) section.

**batch** just tries to group multiple messages into one large payload to send
to the backend. This reduces the amount of packages that need to be sent, which
means lower CPU usage and network load.
```yaml
batch:
  send_batch_size: 8192
  timeout: 1s
```

**memory_limiter** is a safety net to drop incoming messages in case the host's
RAM usage is nearing the limit, so that the collector is not the reason to
something gets killed.
```yaml
memory_limiter:
  check_interval: 1s
  limit_percentage: 75
  spike_limit_percentage: 15
```

**resource_detection** is the processor that, in this case, adds
`host.name: $(hostname)` and `os.type: linux` to all outgoing messages, so it
is easy to identify from where they are coming.
```yaml
resource_detection:
  detectors: [env, system]
  system:
    hostname_sources: [os]
  timeout: 2s
```

### Exporters
This section contains all the configuration that I have added to the
`otel_agent_exporters` variable.

An "exporter" is a location where you want any collected data to be sent to,
and in my case I only have a single backend endpoint. However, here we configure
retries and the queue, which works in tandem with the **batch**
[processor](#processors) to buffer packages in case of network issues.

```yaml
exporters:
  otlp_grpc/backend:
    compression: gzip
    endpoint: http://192.168.0.2:4317
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_elapsed_time: 5m
      max_interval: 30s
    sending_queue:
      enabled: true
      num_consumers: 4
      queue_size: 1000
    tls:
      insecure: true
```

### Services
This section contains all the configuration that I have added to the
`otel_agent_service` variable.

The most important part here is the "pipelines" bit since it is where you
connect all the different things you have been defining up until now. Here
you can see that all the different types (logs/metrics/traces) all export to
the same place, i.e. my `otlp_grpc/backend`. However, what is being handled in
each of these sections is defined by what is in the `recievers` part, and should
only contain those things that are relevant for each section.

Finally it is the `processors` which will manipulate the data in each section,
and the ordering is important here. That is the reason to why the
`memory_limiter` is first, since you need to drop messages as quickly as
possible if the system is having memory issues instead of starting to process
it. This is also why `batch` is at the end, since it is pointless to try to
buffer messages before they are finished being processed.

```yaml
pipelines:
  logs:
    exporters: [otlp_grpc/backend]
    processors: [memory_limiter, resource_detection, batch]
    receivers: [otlp, journald/system, journald/dmesg] # Log related recievers

  metrics:
    exporters: [otlp_grpc/backend]
    processors: [memory_limiter, resource_detection, batch]
    receivers: [otlp, host_metrics/fast, host_metrics/slow] # Metrics related recievers

  traces:
    exporters: [otlp_grpc/backend]
    processors: [memory_limiter, resource_detection, batch]
    receivers: [otlp] # Traces related recievers
```

### Docker Workaround
TODO



[1]: https://hub.docker.com/r/otel/opentelemetry-collector-contrib
[2]: https://github.com/grafana/grafana-ansible-collection/tree/main/roles/opentelemetry_collector
[3]: https://opentelemetry.io/docs/collector/deploy/agent/
