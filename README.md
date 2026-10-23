# Metrics Service

## Purpose

`metrics-service` is the app platform observability repository.

It is not an application service in the usual backend sense. It is a
platform-infrastructure repository that owns Docker images and deployment logic
for the observability stack used around the app platform.

Current stack components in this repository:

- Jaeger
- Elasticsearch
- Logstash
- Kibana
- Prometheus
- Grafana
- Postgres Exporter

## Repository role

This repository currently acts as a production-oriented deploy/build source for
the observability stack.

It does **not** currently contain:

- a local `docker-compose.yml`
- a checked-in local dev bootstrap script
- an `.env.example` describing a local runtime contract

So this repo should be treated primarily as a platform deployment repository,
not as the main local-development orchestration entrypoint for observability.

## Repository structure

```text
.github/workflows/main.yml               CI/CD build and remote deploy workflow
docker/production/elasticsearch/         Elasticsearch image and config
docker/production/grafana/               Grafana image
docker/production/jaeger/                Jaeger image
docker/production/kibana/                Kibana image and config
docker/production/logstash/              Logstash image, pipeline, and TLS materials
docker/production/pg-exporter/           Postgres exporter image
docker/production/prometheus/            Prometheus image and scrape config
```

## Current deployment model

The primary runtime source of truth is:

- `.github/workflows/main.yml`

Current flow on push to `main`:

1. build Docker images for all observability components
2. export them into tar archives
3. copy those archives to the target server
4. connect over SSH
5. recreate containers on the remote server with explicit `docker run` commands

The workflow also:

- creates or reuses the shared `fin_network`
- assigns fixed container IPs on `172.28.0.0/16`
- provisions Elasticsearch security users and roles
- creates ILM policy and index template for app logs

## Runtime topology

The deploy workflow currently creates these named containers:

- `fin_jaeger`
- `fin_elasticsearch`
- `fin_logstash`
- `fin_kibana`
- `fin_prometheus`
- `fin_grafana`
- `fin_pg_exporter`

These containers are attached to `fin_network`.

Important current IP assignments from the workflow:

- `fin_jaeger` -> `172.28.0.14`
- `fin_elasticsearch` -> `172.28.0.15`
- `fin_logstash` -> `172.28.0.16`
- `fin_kibana` -> `172.28.0.17`
- `fin_prometheus` -> `172.28.0.18`
- `fin_grafana` -> `172.28.0.19`
- `fin_pg_exporter` -> `172.28.0.20`

These addresses matter because other infra repos may assume them.

## Component notes

### Prometheus

Prometheus configuration lives in:

- `docker/production/prometheus/prometheus.yml`

Current scrape targets include:

- `fin_user:8000`
- `fin_trade:8000`
- `fin_restriction:8000`
- `fin_currency:8000`
- `fin_account:8000`
- `fin_merchant:8000`
- `fin_pay:8000`
- `fin_pg_exporter:9187`

Important nuance:

- `trade-service` is still present as a scrape target even though there is no
  matching repository in the current app workspace

### Logstash

Logstash configuration lives in:

- `docker/production/logstash/config/logstash.yml`
- `docker/production/logstash/pipeline/logstash.conf`

Current behavior:

- accepts GELF input on port `12201`
- parses JSON payloads from the incoming GELF message
- writes logs to Elasticsearch using the `logstash_internal` user
- uses ILM alias `app-logs`

### Elasticsearch and Kibana

Elasticsearch and Kibana are not passive images here.

The workflow actively bootstraps:

- `kibana_system` password
- `logstash_internal` user
- `logstash_writer` role
- `app-logs-policy`
- `app-logs-template`

### Jaeger

Jaeger runs in all-in-one mode with Badger storage and `QUERY_BASE_PATH=/jaeger`.

## Security-sensitive materials

This repository currently contains checked-in Logstash TLS materials under:

- `docker/production/logstash/certs/`

That includes:

- CA certificate
- server certificate
- private key

Treat changes in this area as security-sensitive and review them carefully.

## Local development note

For workspace-local observability, prefer root `infra/` when you want a local
developer stack tailored to the app workspace.

This repository is better treated as the reusable production-oriented source for
the observability components themselves.

## Current repository caveats

- there is no checked-in local `docker-compose.yml`
- there is no `.env.example`
- the main source of truth is the GitHub Actions deploy workflow, not a local
  compose stack
- Prometheus still references `fin_trade:8000`
- the repository contains checked-in TLS private-key material for Logstash
