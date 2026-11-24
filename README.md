# Tube Archivist Helm Chart

![Downloads](https://img.shields.io/github/downloads/tonylea/helm-chart-tubearchivist/total)

Deploys [Tube Archivist](https://www.tubearchivist.com) from its official image, [`bbilly1/tubearchivist`](https://github.com/tubearchivist/tubearchivist), published by the project for amd64 and arm64 in one tag.

The chart is opinionated: it fixes the image, the user and most of the settings, and leaves values only for what changes with where it's installed.
Tube Archivist runs as one replica, replaced with `Recreate` rather than rolled, because it keeps a SQLite database in `/cache` and runs its own task scheduler, so two pods would clash.
The chart also runs Tube Archivist's own Redis, which holds its task queue and is not shared with anything else.
Redis serves no Prometheus metrics itself, so [`redis_exporter`](https://github.com/oliver006/redis_exporter) runs beside it in the same pod and serves them on port 9121.
Its video index lives in an Elasticsearch that the chart does not install.
The project calls its arm64 builds "essentially untested"; if they misbehave, run the pod on an amd64 node.

## Values

| Value                                | Default                                                                                                                                                                                                                 |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `host`                               | none; required. The hostname Tube Archivist is reached at, without a scheme. `TA_HOST` is set to `https://<host>`, which must match the address typed in the browser, or requests fail with a 400 or a logout redirect. |
| `timezone`                           | `UTC`. An IANA zone name, used for the schedules.                                                                                                                                                                       |
| `existingSecret`                     | none; required. See [Requirements](#requirements).                                                                                                                                                                      |
| `elasticsearch.url`                  | none; required. With scheme and port, such as `https://elasticsearch:9200`.                                                                                                                                             |
| `elasticsearch.user`                 | `tubearchivist`                                                                                                                                                                                                         |
| `elasticsearch.snapshotDir`          | none; required. Must equal Elasticsearch's `path.repo`.                                                                                                                                                                 |
| `media.existingClaim`                | none; required. The claim mounted at `/youtube`.                                                                                                                                                                        |
| `resources`                          | none. Requests of 50m CPU and 768Mi memory, with no limits, are a starting point until usage is measured.                                                                                                               |
| `persistence.cache.size`             | `5Gi`. It grows with the library, since it holds a thumbnail for every video and the art for every channel.                                                                                                             |
| `persistence.cache.storageClass`     | none; the cluster's default class.                                                                                                                                                                                      |
| `persistence.download.existingClaim` | none; required. The claim mounted at `/cache/download`.                                                                                                                                                                 |
| `redis.resources`                    | none. Requests of 10m CPU and 64Mi memory, with no limits, are a starting point until usage is measured.                                                                                                                |
| `redis.persistence.size`             | `128Mi`, the recommended size. Redis holds only task messages, task status and locks, so even a large backlog is a few MB.                                                                                              |
| `redis.persistence.storageClass`     | none; the cluster's default class.                                                                                                                                                                                      |
| `redis.exporter.resources`           | none. Requests of 10m CPU and 32Mi memory, with no limits, are a starting point until usage is measured.                                                                                                                |
| `redis.serviceMonitor.enabled`       | `false`. A ServiceMonitor for the exporter.                                                                                                                                                                             |
| `redis.serviceMonitor.interval`      | `30s`                                                                                                                                                                                                                   |
| `redis.dashboard.enabled`            | `false`. The exporter's own Grafana dashboard, as a ConfigMap labelled `grafana_dashboard: "1"`.                                                                                                                        |
| `httpRoute.enabled`                  | `false`                                                                                                                                                                                                                 |
| `httpRoute.parentRefs`               | `[]`; required when `httpRoute.enabled` is set.                                                                                                                                                                         |

## Fixed settings

- **`ES_DISABLE_VERIFY_SSL` is on.**
  Elasticsearch on Kubernetes usually serves a self-signed certificate, and Tube Archivist has no setting for a CA certificate.
- **`TA_AUTO_UPDATE_YTDLP` is `release`.**
  YouTube's changes often break yt-dlp, and Tube Archivist's releases come months apart, so each start installs the latest yt-dlp release from PyPI.
  The pod needs internet access for it, as it does for YouTube.
- **`REDIS_CON` points at the chart's own Redis.**
- **Both pods run as non-root users**: Tube Archivist as `1000:1000`, Redis and its exporter as `999:999`.

## Requirements

### Elasticsearch

Version 8.x or 9.x, with security on and `path.repo` set to the same path as `elasticsearch.snapshotDir`.
Tube Archivist checks the version and `path.repo` at startup, and won't start without them.

The user it logs in as needs this role, shown as an Elasticsearch `roles.yml` entry:

```yaml
tubearchivist:
  cluster:
    - monitor
    - monitor_snapshot
    - manage_slm
    - cluster:admin/repository/put
    - cluster:admin/snapshot/*
  indices:
    - names: ["ta_*"]
      privileges: [read, write, manage]
```

- `monitor` covers the version, cluster health and node settings checks.
- `monitor_snapshot` covers reading the snapshot repository and listing snapshots.
- `manage_slm` covers the daily snapshot policy, `ta_daily`: reading, creating, deleting and running it.
- `cluster:admin/repository/put` registers the `ta_snapshot` repository.
  It's granted as its own action, rather than the broad `manage` privilege Elastic lists for that API.
- `cluster:admin/snapshot/*` covers taking, deleting and restoring snapshots, again without `manage`.
- Every index Tube Archivist uses starts with `ta_`, including the `ta_<name>_v<N>` copies it makes when it rebuilds an index.
  `read` covers search, point in time and count; `write` covers documents, bulk, update by query and delete by query; `manage` covers creating and deleting indexes, aliases, mappings, refresh, and creating the snapshot policy.

### Secret

`existingSecret` names a Secret in the release's namespace with these keys:

| Key                | What it holds                                                                           |
| ------------------ | --------------------------------------------------------------------------------------- |
| `TA_USERNAME`      | The first user's name. Only read on the first start: changing it later does nothing.    |
| `TA_PASSWORD`      | The first user's password. Only read on the first start: change it later in the web UI. |
| `ELASTIC_PASSWORD` | The password of `elasticsearch.user`.                                                   |

### Storage

| Mount             | Claim                                | Requirements                                                                                                                                                                                                                        |
| ----------------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/youtube`        | `media.existingClaim`                | The finished videos. Tube Archivist creates a folder per channel ID, and creates, renames and deletes files in it, so the pod's user (or the user the share maps it to) must be able to do all of those. Its size is the library's. |
| `/cache`          | The chart's `cache`                  | The SQLite database, artwork and yt-dlp's cache. SQLite needs working file locks, so this is block or local storage, never NFS or SMB. It holds state, so it needs to persist and is worth keeping redundant.                       |
| `/cache/download` | `persistence.download.existingClaim` | Downloads in progress, copied to `/youtube` when each finishes. It must hold the largest single video. It needs no redundancy and needn't survive a restart: a lost partial download is fetched again.                              |
| `/data` (Redis)   | The chart's `redis`                  | Redis's snapshots. Block or local storage.                                                                                                                                                                                          |

### Gateway API

The `HTTPRoute` CRD, when `httpRoute.enabled` is set.

### Monitoring

The Prometheus Operator's CRDs, when `redis.serviceMonitor.enabled` is set, and a Grafana dashboard sidecar that reads ConfigMaps labelled `grafana_dashboard: "1"`, when `redis.dashboard.enabled` is set.

## Usage

```sh
helm repo add tubearchivist https://tonylea.github.io/helm-chart-tubearchivist
helm install tubearchivist tubearchivist/tubearchivist --values my-values.yaml
```

```yaml
host: tubearchivist.example.com
timezone: Europe/London
existingSecret: tubearchivist
elasticsearch:
  url: https://elasticsearch-es-http.elasticsearch.svc:9200
  snapshotDir: /usr/share/elasticsearch/data/snapshot
media:
  existingClaim: youtube
persistence:
  download:
    existingClaim: youtube-downloads
httpRoute:
  enabled: true
  parentRefs:
    - name: internal
      namespace: ingress
```

## Development

Pull requests run:

- `helm lint --strict`, and the rendered chart checked by kubeconform against Kubernetes 1.37.
- helm-unittest's tests in `tests/`.
- An install with chart-testing in a kind cluster, using `ci/test-values.yaml`.
  `ci/kind` first sets up what it needs: a single-node Elasticsearch 9.5.5 with `path.repo` set, the Secret, and claims for `/youtube` and `/cache/download`.

Commits follow Conventional Commits.
Releases are versioned and tagged automatically from `main`, and published to the Helm repository on this repo's GitHub Pages.
