# The interface contract

This file is the border between the two tracks.

Claude writes `backend/`, `frontend/` and the Dockerfiles. xael writes every
file under `k8s/`. The two never read each other's source. They read this file.

**Neither track changes a value here alone.** A change is a merge request, and
both agree before it merges. Every value below appears in code on one side and
in a manifest on the other. If one side edits a value in silence, the other side
breaks, and the error shows up as an empty dashboard rather than as a crash.

Everything runs in the namespace **`lgtm`**.

---

## 1. Images

Claude owns the Dockerfiles, so Claude owns these two names and tags. The
manifests use them exactly as written.

| Image | Built from | Runs as | Port |
|---|---|---|---|
| `lgtm/backend:v1` | `backend/Dockerfile` | user `10001`, not root | 8080 |
| `lgtm/frontend:v1` | `frontend/Dockerfile` | `nginx:alpine` default | 80 |

Rules that both tracks follow:

- **No tag is `latest`.** K3s runs containerd, and a `latest` tag makes
  Kubernetes pull from a registry. There is no registry here, so the pod fails
  with `ErrImagePull`.
- Every manifest sets `imagePullPolicy: IfNotPresent`.
- **The tag stays `v1`. It never carries the commit.** A tag that moves with
  every commit would force the manifests to hold a placeholder and a deploy step
  to fill it in, and then no one could read a manifest and know what runs. Raw
  YAML that `kubectl apply -f` accepts as written is worth more than that.
- **So `make deploy` ends with `kubectl rollout restart`.** This is the price of
  a tag that never moves: the image string in the Deployment does not change, so
  `kubectl apply` sees nothing to do and leaves the old pod running the old
  binary. The restart makes new pods, and a new pod reads whatever `v1` points
  to in containerd now. Without this line you debug code that is not running.
- An image reaches the cluster in two steps, never one:

  ```
  docker build -t lgtm/backend:v1 backend/
  docker save lgtm/backend:v1 | sudo k3s ctr images import -
  ```

Third party images belong to the manifests. They are pinned to a version there,
never to `latest`: `otel/opentelemetry-collector-contrib`, `prom/prometheus`,
`grafana/tempo`, `grafana/loki`, `grafana/grafana`, `ipfs/kubo`.

> The Collector must be the **contrib** image. The base image has no
> `prometheusremotewrite` exporter, and the metrics pipeline needs it.

---

## 2. Services and ports

Every name below is a Kubernetes Service in the namespace `lgtm`. The full DNS
name is `<service>.lgtm.svc.cluster.local`.

| Service | Port | Protocol | Who calls it |
|---|---|---|---|
| `lgtm-backend` | 8080 | HTTP | Traefik |
| `lgtm-frontend` | 80 | HTTP | Traefik |
| `otel-collector` | 4317 | OTLP over gRPC | the backend, Kubo |
| `otel-collector` | 4318 | OTLP over HTTP | spare, for tests |
| `prometheus` | 9090 | HTTP | Grafana, the Collector |
| `tempo` | 3200 | HTTP | Grafana |
| `tempo` | 4317 | OTLP over gRPC | the Collector |
| `loki` | 3100 | HTTP | Grafana, the Collector |
| `grafana` | 3000 | HTTP | Traefik |
| `kubo` | 5001 | HTTP RPC | the backend **only** |
| `kubo` | 8080 | HTTP gateway | Traefik, the backend |
| `kubo` | 4001 | TCP and UDP swarm | other IPFS nodes |

Two ports need care:

- **`kubo:5001` never leaves the cluster.** That port adds and pins data. No
  Ingress points at it. The NetworkPolicy of issue #22 allows exactly one
  source: the backend pod.
- **`lgtm-backend:8080` and `kubo:8080` share a number** and nothing else. They
  are different pods, so the two never meet.

---

## 3. Host names and routes

`vm/bootstrap.sh` writes three names into `/etc/hosts`, all pointing at
`127.0.0.1`. This works because ServiceLB binds Traefik to port 80 of the guest.

| Host | Path | Goes to |
|---|---|---|
| `lgtm.local` | `/api` | `lgtm-backend:8080` |
| `lgtm.local` | `/` | `lgtm-frontend:80` |
| `grafana.lgtm.local` | `/` | `grafana:3000` |
| `ipfs.lgtm.local` | `/ipfs` | `kubo:8080` |

The frontend and the backend share the host `lgtm.local` on purpose. The browser
stays on one origin, so the project needs no CORS header anywhere.

Every Ingress sets `ingressClassName: traefik`.

> `.local` belongs to mDNS. `/etc/hosts` still wins, because `files` comes
> before `mdns4_minimal` in `/etc/nsswitch.conf`. When a name fails to resolve,
> read that file first.

### Reaching the three names from outside the machine

The subject fixes the three names and shows the IPFS link with no port:

> Your web application must be accessible at `lgtm.local`.
> Grafana must be accessible at `grafana.lgtm.local`.
> IPFS gateway must be accessible at `ipfs.lgtm.local`.
> User gets a link: `ipfs.lgtm.local/ipfs/QmXx...`

**So no URL in this project ever carries a port.** The machine therefore owns an
address of its own, and Traefik binds port 80 **inside the guest**, where the
port costs no privilege. Nothing in this project ever needs root on the host.

| Interface | Address | Carries |
|---|---|---|
| NAT | given by the provider | `vagrant ssh` only |
| private network | `192.168.56.10`, fixed | every HTTP name below |
| bridged | given by the router, opt-in | the IPFS swarm port 4001 |

A person browsing from the host writes three lines in the hosts file of the
**host**, not of the guest:

```
192.168.56.10 lgtm.local
192.168.56.10 grafana.lgtm.local
192.168.56.10 ipfs.lgtm.local
```

The address is fixed in the `Vagrantfile`, so those three lines are written once
and never again. `192.168.56.0/21` is the range VirtualBox allows for a
host-only network with no extra setup, so the same address works on both
providers. Traefik reads the `Host` header to choose the route, so the three
names share the one address.

> **Why not forward port 80 from the host.** Because a port below 1024 needs
> `CAP_NET_BIND_SERVICE`, so `vagrant up` would have to run under `sudo` — every
> time, downloading boxes as root and moving libvirt to root's session. Binding
> the port inside the guest costs nothing and reaches the same portless URL.

**The bridged interface is opt-in, and IPFS is why it exists.** `LGTM_BRIDGE=eth0
vagrant up` turns it on. Without it the node sits behind NAT, and a public
gateway may fail to reach our CID — which the subject permits us to *explain*,
but the test still wants a real answer. Wireless interfaces usually refuse to
bridge, so use a cable for that demonstration.

> **One privileged container, and it is not ours.** K3s binds port 80 with
> ServiceLB, which runs a `svclb-traefik` pod with the right to bind a
> privileged port. That is how K3s ships. The backend container is a different
> thing: it runs as user `10001` with a read only root file system, and nothing
> here changes that. Expect the question at the defence and answer it with this
> paragraph.

---

## 4. The HTTP surface of the backend

The frontend and the Ingress both depend on these paths.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/run` | Compile the code, run it, upload it, answer |
| `GET` | `/healthz` | The liveness probe and the readiness probe |

Request:

```json
{ "code": "fn main() { println!(\"hello\"); }" }
```

Answer, on success and on failure alike, with HTTP 200:

```json
{
  "output": "hello\n",
  "cid": "bafy...",
  "link": "http://ipfs.lgtm.local/ipfs/bafy...",
  "timings": { "compile_ms": 812, "execute_ms": 4, "upload_ms": 37 },
  "error": null
}
```

**The backend composes `link`, not the page.** The frontend is static files behind
nginx: it can read no environment and no manifest, so the only way for it to
build that address alone would be to hold the gateway host in its JavaScript —
the same value as `IPFS_GATEWAY_URL` here and as `ipfs.lgtm.local` in section 3,
in a third place, with nothing to catch the day one of them moves. `link` is
empty exactly when `cid` is.

When the run fails, `error` holds an object and the frontend styles it by kind:

```json
{ "error": { "kind": "compile", "message": "..." } }
```

`kind` is one of `compile`, `runtime`, `timeout`, `output_limit`, `internal`,
`request`. The frontend shows a compile error and a runtime error differently,
which the subject requires, so the kind must arrive from the backend and must
never be guessed from the text of the message.

> **There is no `memory` kind, and the memory limit is still enforced.** A guest
> that asks for more than its 10 MiB is refused, and Rust answers a refused
> allocation by printing `memory allocation of N bytes failed` and calling
> `abort()`. That is a trap, the same mechanism as any panic, and the runtime
> reports the two identically. Naming this one `memory` would mean reading the
> text the program printed, which is the one thing the rule above forbids. So it
> arrives as `runtime`, and the reason the user needs is already in `output`.

A failed upload leaves `cid` and `link` empty and does **not** set `error`. The
run succeeded; only the sharing failed.

**Only a run that reached its end is shared.** A source that did not compile has
no output to put beside it, and a run that trapped or was cut short would put a
file on IPFS that does not say so. One rule, and no edge case to argue about at
the defence.

**A request that never became a run answers `400`, not `200`.** A body that is
not JSON, an empty `code`, and a source above **64 KiB** are all mistakes of the
caller rather than results of a run, and none of the six other kinds can carry
them without lying. The answer keeps the same shape and uses the kind `request`,
so the frontend parses one thing and not two. The 200 rule above covers runs
that started and failed, which is a different thing.

---

## 5. The environment of the backend

The backend reads exactly these five variables. It reads nothing else from the
environment.

| Variable | Value in the cluster |
|---|---|
| `OTEL_SERVICE_NAME` | `lgtm-backend` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector.lgtm.svc.cluster.local:4317` |
| `OTEL_METRICS_EXEMPLAR_FILTER` | `trace_based` |
| `IPFS_API_URL` | `http://kubo.lgtm.svc.cluster.local:5001` |
| `IPFS_GATEWAY_URL` | `http://ipfs.lgtm.local` |

**The backend stops at start if one of the five is missing.** It prints the name
of the variable and exits with a code that is not zero. It never starts with a
default and it never runs without telemetry. A typo in a manifest then appears
in `kubectl logs` within seconds, instead of appearing days later as a dashboard
that draws nothing.

Three of the five deserve a word:

- **`OTEL_SERVICE_NAME` must be `lgtm-backend`, exactly.** This same text sits
  inside the Tempo datasource, in the `tracesToLogsV2` query
  `{service_name="lgtm-backend"}`. When the two differ, the button "Logs for
  this span" returns nothing at all, and Grafana reports no error. Changing
  this value means changing the datasource in the same merge request.
- **`OTEL_METRICS_EXEMPLAR_FILTER` must be `trace_based`.** Without it the SDK
  attaches no trace identifier to the histogram, and the click from a graph to a
  trace never works.
- **`IPFS_GATEWAY_URL` is a name from `/etc/hosts`, not a Service name.** The
  browser opens this link, and the browser runs outside the cluster. It carries
  no port, because the subject prints the link without one. Section 3 explains
  what that costs.

The backend listens on `:8080`. That is fixed in the code, not read from the
environment, because the Dockerfile, the Service and the probes all state it.

---

## 6. The metrics the dashboards read

The backend records three instruments. The Collector renames them on the way to
Prometheus: it turns every dot into an underscore and adds the unit at the end.

| Instrument in the code | Kind | Base name in Prometheus |
|---|---|---|
| `lgtm.executions.total` | counter | `lgtm_executions_total` |
| `lgtm.execution.duration` | histogram | `lgtm_execution_duration_seconds` |
| `lgtm.execution.duration.last` | gauge | `lgtm_execution_duration_last_seconds` |
| `lgtm.ipfs.propagation.duration` | histogram | `lgtm_ipfs_propagation_duration_seconds` |

Every duration is in seconds. The last row belongs to the bonus.

A histogram is three series, not one. Add the suffix to the base name:
`_bucket`, `_sum` and `_count`.

The buckets of `lgtm_execution_duration_seconds` are seconds, and the backend
sets them: `0.1 0.25 0.5 1 2 3 4 5 7.5 10 15 30`. The defaults of the SDK are
milliseconds and would put every run of this playground in the first bucket, so
`histogram_quantile` would answer the same number forever and the panel would
still draw. A panel that needs different resolution changes the boundaries in
`backend/internal/run/metrics.go`, not the query.

The counter carries one attribute, `result`, whose value is `success` or
`failure`. Nothing else is an attribute. Code text, a CID and a trace identifier
all have too many possible values, and each one would create a new time series.

Two rules protect this section:

- **Read the real names before writing a panel.** Run
  `curl prometheus:9090/api/v1/label/__name__/values` and read the answer. The
  first three rows above were measured on 2026-08-31, by recording the real
  instruments through the Collector and reading what it served: all three match,
  and `lgtm_executions_total` gains no second `_total`. That measurement used the
  Collector alone, because Prometheus is #24 and is not deployed yet. Re-read the
  names once it is. If any differ, this file changes first, and the panels follow
  this file.
- **The histogram is the only instrument that carries exemplars.** The backend
  records it inside the active span. The exemplar then rides on the `_bucket`
  series and nowhere else. A panel that reaches exemplars must therefore query
  `_bucket`, which `histogram_quantile` does. Rewriting that panel as `_sum`
  divided by `_count` looks simpler and silently kills the drilldown.

---

## 7. The spans the dashboards read

One run produces one trace with exactly four spans. The Tempo panel searches
them by name, so the names are part of this contract.

| Span | Parent | Made by |
|---|---|---|
| `POST /api/run` | none, this is the root | the `otelhttp` middleware |
| `compile` | the root | the backend |
| `execute` | the root | the backend |
| `ipfs.upload` | the root | the backend |

Attributes:

| Attribute | On which span | Value |
|---|---|---|
| `code.sha256` | every span | the hash of the source the user sent |
| `ipfs.cid` | the root span | the directory CID, once it is known |

On every failure the backend calls `span.RecordError` **and**
`span.SetStatus(codes.Error, ...)`. Recording the error alone leaves the span
green in Tempo, and a green failed run is worse than no trace.

The bonus adds two more spans under the root: `ipfs.provide` and
`ipfs.propagation`.

---

## 8. The order the manifests are applied

`make deploy` applies the manifests in this order and waits for each step. A
deploy that does not wait starts Grafana before its datasources exist, and
Grafana then shows three broken datasources and no reason why.

1. the namespace
2. Prometheus, Tempo, Loki — the three stores
3. the Collector, which writes to all three
4. Kubo
5. the backend and the frontend
6. Grafana
7. the Ingress objects
8. the NetworkPolicy, last, so a wrong rule cannot hide a startup failure

**The number in front of a file name groups the files of one component. It is
not the apply order.** The order is the list above, and `make deploy` is the only
supported way to apply these manifests. `kubectl apply -f k8s/` reads the
directory by name, which starts Kubo after the backend that calls it and the
Collector after the two services that export to it.
