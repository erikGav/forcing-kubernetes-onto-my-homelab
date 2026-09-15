# Forcing Kubernetes Onto My Homelab

In [my last article](https://medium.com/@work.erik.gavrilov/what-i-learned-running-a-homelab-like-a-startup-08a86598a93a) I wrote that everyone knows what their tools do, and not everyone knows the moment to reach for them. This one turns that on me.

*Why do you need a better tool for a simple job? It's like reaching for a chainsaw to cut a piece of wood when a hand saw would do the job fine.*

A chainsaw is absurd on one plank and the only sane choice on a cord of firewood. The question is where it flips. I had never had to answer that for Kubernetes. Nothing I run has ever been big enough to force it, so I forced it. I took the `apps` stack from my NUC, 95 lines of Compose running five services, and rebuilt it on Kubernetes. Then I measured what that cost. Line counts, boot times, failure behavior, memory at idle.

This ran on a second machine with the same data layout, not on the live server. Every number below comes from that cluster.

## Contents

1. [The stack](#the-stack)
2. [Picking the distribution](#picking-the-distribution)
3. [The defaults k3s ships](#the-defaults-k3s-ships)
4. [Reading the compose file in Kubernetes terms](#reading-the-compose-file-in-kubernetes-terms)
5. [One bind mount, four ways to write it](#one-bind-mount-four-ways-to-write-it)
6. [qBittorrent's peer port and the LoadBalancer Service](#qbittorrents-peer-port-and-the-loadbalancer-service)
7. [Rolling updates, tested with a broken image tag](#rolling-updates-tested-with-a-broken-image-tag)
8. [Restarts, limits, and the parts Compose already had](#restarts-limits-and-the-parts-compose-already-had)
9. [The same five services as a Helm chart](#the-same-five-services-as-a-helm-chart)
10. [What the cluster costs while idle](#what-the-cluster-costs-while-idle)
11. [The conditions that would justify the overhead](#the-conditions-that-would-justify-the-overhead)
12. [What it cost against what it gave](#what-it-cost-against-what-it-gave)

## The stack

A dashboard (Heimdall), a git-backed wiki (Gollum), a torrent client (qBittorrent), and two file tools, BentoPDF and ConvertX. Four of the five write to a directory on the host; BentoPDF is the exception. qBittorrent is the only one publishing a port that isn't HTTP. They all join an external network called `proxy`, where nginx-proxy-manager terminates TLS for `*.nuxlet.com` and routes by hostname.

```yaml
services:
  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Jerusalem
      - ALLOW_INTERNAL_REQUESTS=false #optional
    volumes:
      - /home/homeserver/portainer/heimdall/config:/config
    networks:
      - proxy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Jerusalem
      - WEBUI_PORT=8080
    volumes:
      - /home/homeserver/portainer/qbittorrent/config:/config
      - /home/homeserver/Downloads:/downloads
      - /home/homeserver/portainer/qbittorrent/init.d:/custom-cont-init.d:ro
    ports:
      - 6881:6881
      - 6881:6881/udp
    networks:
      - proxy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  gollum:
    image: gollumwiki/gollum:latest
    container_name: gollum
    command: ["-c", "/wiki/gollum.rb"]
    volumes:
      - /home/homeserver/homelab:/wiki
    networks:
      - proxy
    restart: unless-stopped
    ulimits:
      core: 0
    healthcheck:
      test: ["CMD", "ruby", "-e", "require 'net/http'; Net::HTTP.get_response(URI('http://localhost:4567/'))"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

  bentopdf:
    image: ghcr.io/alam00000/bentopdf-simple:latest
    container_name: bentopdf
    networks:
      - proxy
    restart: unless-stopped
    healthcheck:
      test: ['CMD', 'wget', '--spider', '-q', 'http://localhost:8080']
      interval: 30s
      timeout: 10s
      retries: 3

  convertx:
    image: ghcr.io/c4illin/convertx:latest
    container_name: convertx
    environment:
      - ALLOW_UNAUTHENTICATED=true
    volumes:
      - /home/homeserver/portainer/convertx/data:/app/data
    networks:
      - proxy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  proxy:
    external: true
```

One file, and `docker compose up -d` on the other end of it. That's the input; everything below is what happened to it.

## Picking the distribution

The cluster had to be something a business would run in production. Run the experiment on a toy and the overhead numbers measure the toy.

**kind** went first, and I had Heimdall serving through ingress-nginx on it before I dropped it. Its documentation says it's built for testing Kubernetes itself and for CI, and the setup shows it. Each node is a Docker container, so node ports don't reach the host until they're declared in the cluster config, and that config is read only at creation time; adding one exposed port means deleting the cluster and building another. `type: LoadBalancer` never gets an address, because there's no provider to answer. The ingress controller has to run on whichever node has the published ports, and the manifest that used to pin it there dropped the `nodeSelector`. Every one of those follows from running nodes as Docker containers. None of them follow from Kubernetes.

Weight was never my objection to **kubeadm**. Three control-plane components plus etcd run fine on a NUC. Reversibility was. `kubeadm reset` leaves CNI config in `/etc/cni/net.d`, iptables and ipvs rules, and kubelet state behind, then prints instructions telling you to clear the rest yourself. That machine already has a job.

I kept **k3s**. Install is one command, uninstall is `/usr/local/bin/k3s-uninstall.sh`, and the installer writes that script for you. One binary carries the server, the agent, and containerd. It's a CNCF-certified Kubernetes distribution, which is what makes the numbers worth reading: same API, different packaging, so what bites here bites on a cluster someone pays for. The cluster below is v1.36.3+k3s1 on one node.

## The defaults k3s ships

Under a minute after the installer finished, with nothing deployed yet:

```
$ kubectl -n kube-system get svc traefik
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
traefik   LoadBalancer   10.43.147.210   192.168.1.4   80:30804/TCP,443:30342/TCP

$ kubectl get ingressclass
NAME                CONTROLLER
traefik (default)   traefik.io/ingress-controller

$ kubectl get storageclass
NAME                   PROVISIONER             VOLUMEBINDINGMODE
local-path (default)   rancher.io/local-path   WaitForFirstConsumer
```

Traefik shows up in two of those because it's two separate things. The Service is what holds ports 80 and 443 on the node, and it carries an `EXTERNAL-IP` rather than `<pending>`, which means something answered the request for an address. The IngressClass is what makes an `Ingress` object mean anything at all; write one with no controller watching for it and you have inert YAML. Add metrics-server, running in `kube-system` alongside them, and that's four things I didn't install.

kind ships the storage class too, the same local-path provisioner. The other three it leaves to you.

That's convenience, and it's also the part to argue with, because two of those defaults are substitutes rather than equivalents.

klipper-lb, the component that handed Traefik `192.168.1.4`, is a DaemonSet, one pod per node, that intercepts the port on the host and forwards it. The address is the node's own IP, so it lasts exactly as long as the node does, and no health check anywhere in that path can move traffic off it.

local-path writes into `/var/lib/rancher/k3s/storage` on whichever node ran the pod first, which is why its binding mode is `WaitForFirstConsumer`:

```
$ kubectl -n kube-system get cm local-path-config -o jsonpath='{.data.config\.json}'
{"nodePathMap":[{"node":"DEFAULT_PATH_FOR_NON_LISTED_NODES","paths":["/var/lib/rancher/k3s/storage"]}]}
```

Nothing replicates. Add a second node, let a pod reschedule, and the data stays where it was. On one machine both defaults are fine, because there's nowhere else for anything to go. k3s makes single-node Kubernetes feel solved, and single node is the case where Compose already worked.

## Reading the compose file in Kubernetes terms

Nothing gets migrated until every line has a known destination. Heimdall's definition is 19 lines. Working out where each one lands is most of the work, and the lines that land nowhere are the interesting ones.

| compose | Kubernetes | cost |
|---|---|---|
| `image:` | `spec.containers[].image` | same line |
| `container_name: heimdall` | nothing | pod names get a generated suffix; `docker exec -it heimdall` becomes `kubectl exec deploy/heimdall` |
| `environment:` (4 entries) | `env:` as name/value pairs | 4 lines become 9 |
| `volumes: - /host:/config` | a PersistentVolume, a PersistentVolumeClaim, a `volumes` entry, a `volumeMounts` entry | 1 line becomes 37 |
| `ports: - 6881:6881` | a Service, and a decision about its type | 1 line becomes an object |
| `networks: - proxy` | nothing | every pod already reaches every pod |
| `restart: unless-stopped` | nothing | `Always` is the only value a Deployment accepts |
| `healthcheck:` | `startupProbe`, `readinessProbe`, `livenessProbe` | one check becomes three, with three meanings |
| `command:` | `args:` | Kubernetes `command` is ENTRYPOINT, not CMD |
| `ulimits:` | nothing | no equivalent field exists |

Four of those rows deserve more than a row.

`restart: unless-stopped` has no target because a Deployment doesn't offer the choice. The API says so:

```
$ kubectl apply --dry-run=server -f deploy-with-onfailure.yaml
spec.template.spec.restartPolicy: Unsupported value: "OnFailure": supported values: "Always"
```

Compose gives four restart policies. A Deployment gives one. The line isn't translated, it's deleted, and the behavior it described becomes non-negotiable.

`healthcheck:` becomes three probes, and only one of them decides anything about traffic. `startupProbe` holds the other two off while a slow container boots. `livenessProbe` restarts the container when it stops answering. `readinessProbe` is the one that matters for a deploy: a pod failing it stays running, but the Service takes it out of the endpoint list, so nothing routes to it. Compose marks a container unhealthy and leaves it in place. That difference is the whole basis of the rolling update further down.

`command:` looks like it maps to `command:`, and that's the trap. In Kubernetes, `command` overrides the image's ENTRYPOINT and `args` overrides its CMD. Compose's `command` is CMD. Gollum's `command: ["-c", "/wiki/gollum.rb"]` has to become `args:`, and writing it as `command:` replaces the entrypoint instead of passing arguments to it. The container starts and does the wrong thing rather than failing, which is the worst way for a mistake to behave.

`ulimits: core: 0` on Gollum has nowhere to go at all:

```
$ kubectl explain pod.spec --recursive | grep -ci ulimit
0
```

There's no pod-level or container-level ulimit field. Getting that behavior back means node configuration or a privileged init container, which is more machinery than one line of YAML was worth. I dropped it.

Compose has one noun. Kubernetes splits it into six, and four of them are unreachable on one machine: a DaemonSet places one pod per node, a StatefulSet gives each replica its own volume, and Job and CronJob replace `docker run --rm` under host cron. What's left is a Pod, which nobody writes by hand, and a Deployment, which is the object that has to justify all of it.

## One bind mount, four ways to write it

```yaml
volumes:
  - /home/homeserver/portainer/heimdall/config:/config
```

A host path, a container path, a colon. Kubernetes has four ways to say that, and picking between them is the first decision in this migration that can lose data.

`hostPath` is the literal translation, eight lines instead of one, identical behavior. Its `type` field is the part that matters: `Directory` refuses to start if the path is missing, `DirectoryOrCreate` makes an empty one and starts anyway, and the default `""` checks nothing. A pod that comes up healthy while serving an empty config directory is a specific kind of bad morning.

A `local` PersistentVolume is the same path with the abstraction on top: a PV pointing at the existing directory, a PVC bound to it by name, and `storageClassName: ""` on both so the default provisioner keeps its hands off. Its reclaim policy defaults to `Retain`, which leaves the directory on disk when the claim goes away. The API refuses to accept it without a scheduling rule:

```
$ kubectl apply --dry-run=server -f pv-without-affinity.yaml
spec.nodeAffinity: Required value: Local volume requires node affinity
```

A bind mount is pinned to one machine too, and Compose never made anyone write that down. Kubernetes won't take the volume until the pinning is stated as a rule the scheduler can read, which moves the constraint out of my head and into a file where a second person can find it. The `1Gi` capacity next to it is decoration; nothing enforces it on a local volume, and a directory that outgrows it keeps growing.

The third option is the dynamic one, a ten-line PVC against `local-path`, and it's the one that would have cost me the data. It doesn't adopt the existing directory, it creates a new empty one under `/var/lib/rancher/k3s/storage`. Its reclaim policy is `Delete`:

```
$ kubectl get sc local-path -o jsonpath='{.reclaimPolicy}'
Delete
```

Deleting the claim deletes the directory. `docker compose down` never touched a bind mount, and removing named volumes took an explicit `-v`. The safety default is inverted here, and the object that triggers it is one people delete casually while iterating.

The fourth option, NFS or a CSI driver, is the only one that survives a second node, and it's a service to run and to back up before it holds a single file. I went with `local` PV plus PVC across all six volumes. Counting the PV, the claim, the `volumes` block, and the `volumeMounts` entry, that's 37 lines to say what Compose said in one. qBittorrent has three volumes, so it pays that three times. The read-only mount is the only thing in the whole migration that got shorter: `:ro` became `readOnly: true`.

Fair test of whether the paperwork bought anything: I deleted every object in the namespace and reapplied. All six directories were still on disk, all five pods came back with their data. `Retain` did what it says.

## qBittorrent's peer port and the LoadBalancer Service

Four of the five services are a web UI and nothing else, so they end up behind the same ingress controller and cost one `Ingress` object each. qBittorrent has a second port, 6881 for peer traffic, and an Ingress can't carry it. HTTP routing needs a Host header to route on, and BitTorrent doesn't have one.

So those two compose lines become a second Service:

```yaml
ports:
  - 6881:6881
  - 6881:6881/udp
```

```
qbittorrent-lb   LoadBalancer   10.43.92.84   192.168.1.4   6881:30790/TCP,6881:30790/UDP
```

On k3s, `type: LoadBalancer` is answered by klipper-lb, which doesn't open a socket at all. Its startup log prints the whole implementation:

```
+ iptables -t nat -I PREROUTING -p TCP --dport 6881 -j DNAT --to 10.43.92.84:6881
+ iptables -t nat -I POSTROUTING -d 10.43.92.84/32 -p TCP -j MASQUERADE
+ iptables -t nat -I PREROUTING -p UDP --dport 6881 -j DNAT --to 10.43.92.84:6881
+ iptables -t nat -I POSTROUTING -d 10.43.92.84/32 -p UDP -j MASQUERADE
```

Four rules, and the address it hands out is the node's own IP. On kind the same Service sits at `<pending>` forever, because nothing there answers the request. On a cloud, it provisions a load balancer and bills for it per Service, which is why one ingress controller fronts many hostnames instead of every app claiming a load balancer of its own. The peer port is the one case here with no alternative.

## Rolling updates, tested with a broken image tag

A Deployment exists to roll from one version to the next without dropping traffic, so I broke a deploy on purpose and counted the failed requests.

First, BentoPDF. No volumes, so it keeps the default `RollingUpdate` strategy. I pointed it at an image tag that doesn't exist and hit it once a second for 30 seconds:

```
$ kubectl -n apps set image deploy/bentopdf bentopdf=ghcr.io/alam00000/bentopdf-simple:does-not-exist
  ok=30 fail=0

$ kubectl -n apps get pods -l app=bentopdf
bentopdf-5dddc74fc6-76fjg   1/1   Running        0   77s
bentopdf-786dd77dc6-hfptc   0/1   ErrImagePull   0   32s
```

Thirty out of thirty. The new pod never became ready, so it never received traffic, and the old one kept serving the whole time. That's the feature working as advertised, on the one app in the stack that can use it.

Then Heimdall, same broken change. Heimdall has a `ReadWriteOnce` config volume. Old and new pods can't hold it at once, which forces `strategy: Recreate`:

```
  t+1: 502   t+2: 502   t+3: 502
  t+4: 503 ... t+30: 503
  ok=0 fail=30
```

Zero out of thirty. Recreate stops the old pod before starting the new one, so a change that never had a chance of working still took the service down first and asked questions later.

Compose, given the identical broken tag, refused the change:

```
$ docker compose up -d
 Image ghcr.io/alam00000/bentopdf-simple:does-not-exist Pulling
 Image ghcr.io/alam00000/bentopdf-simple:does-not-exist Error manifest unknown
Error response from daemon: manifest unknown

$ docker ps
bl-bentopdf   Up 7 seconds (healthy)
```

It pulls before it touches the running container, the pull fails, and nothing happens. The old container is still up and still answering 200.

So on this stack, for four of five services, Compose handled the failed deploy better than Kubernetes did. Kubernetes served all thirty requests on the fifth. The dividing line isn't the orchestrator, it's whether the app owns a writable directory, and every app I care about owns one.

Where Kubernetes won outright was the recovery. One command, and eight seconds:

```
$ kubectl rollout undo deploy/heimdall
recovered after 8s
```

I never had to know what the old tag was. The Deployment kept the ReplicaSet it rolled off, so `undo` had somewhere to roll back to. Compose keeps no history at all: rolling back means finding the tag I replaced, editing the file, and running `up -d` again, and how long that takes depends on whether I wrote the old tag down.

## Restarts, limits, and the parts Compose already had

Delete a pod and the replacement is running in about two seconds:

```
t+1s: bentopdf-5dddc74fc6-ccwxw  1/1  Terminating
t+2s: bentopdf-5dddc74fc6-76fjg  0/1  Running
t+3s: bentopdf-5dddc74fc6-76fjg  1/1  Running
```

`restart: unless-stopped` covers the case where a process dies. What a Deployment adds on top is recovery from things that delete the container rather than crash it, and recovery from a node going away. There is one node. If it goes away, the control plane that would reschedule the pods goes with it.

Limits land on the same cgroup Compose's `mem_limit` writes to. I capped ConvertX at 32Mi to watch it happen:

```
convertx-55959957d9-nhz6v   0/1   OOMKilled   1 (2s ago)   3s
    Reason: OOMKilled     Exit Code: 137
    QoS Class: Burstable
```

The part Compose doesn't have is `requests`, and requests are input to the scheduler. On one node the scheduler has one answer to every question, so what the limits got me was the ability to OOM my own container in a new file format.

## The same five services as a Helm chart

The five services are 95 lines of Compose. I wrote them out as plain manifests first, to get a number rather than an impression, and only excerpts of those files appear above: 642 lines across six files and 29 objects. That's 6.8 times the lines, and it's the worst case, because almost all of it is the same shape written out five times.

So I collapsed it into a Helm chart: one template that loops over a map of apps, and a values file that describes each app in the shape the Compose file did.

```
Chart.yaml            5 lines
values.yaml          54 lines
templates/app.yaml  157 lines
                    ---
                    216 lines, rendering 679
```

216 hand-written lines against 642. A sixth app costs 4 lines in `values.yaml` if it's stateless like BentoPDF and 16 if it looks like qBittorrent, against the 63 and 224 those two took as hand-written files. Against the Compose file it's still 2.3 times bigger, and the template itself is a different job from writing YAML; conditionals like `{{ if $app.volumes }}Recreate{{ else }}RollingUpdate{{ end }}` are code, and they fail like code. The chart deployed all five apps and had every hostname answering 10 seconds later.

One more thing worth knowing before you trust `helm upgrade` as a reset button. The 32Mi limit I put on ConvertX with `kubectl set resources` survived a `helm upgrade` against the unchanged chart. The live object kept the field, and I had to patch the Deployment directly. Helm reconciles what's in the chart; it doesn't notice what you added by hand.

## What the cluster costs while idle

```
560.7 MB   /usr/local/bin/k3s server
 70.4 MB   /usr/bin/dockerd
```

Six running pods in `kube-system`, 98 Mi between them, hold up five application pods using 491 Mi: CoreDNS, Traefik, metrics-server, the local-path provisioner, and two klipper-lb DaemonSet pods. Two more completed Helm jobs sit there as the record of how Traefik got installed.

Startup and teardown, measured the same way on the same hardware, from command to every service answering HTTP:

```
docker compose up -d        4s
kubectl apply -f k8s/      10s

docker compose down        11s
kubectl delete -f k8s/     52s
```

Neither of those numbers matters at this scale, and the gap is narrower than I expected going in. Half a gigabyte of resident memory for the control plane is the cost that shows up, on a machine where the whole apps stack fits in 491 Mi.

## The conditions that would justify the overhead

The migration worked. All five services run on k3s, each behind an Ingress, and the four that keep state kept it, in the directories I pointed them at. Nothing about the result is worse than what I had. It's bigger, and everything it bought me has a name and a number attached.

I got one zero-downtime deploy out of five apps, and only for the one that stores nothing. I got an eight-second rollback. I got a scheduler that refuses to run a pod when its data isn't on the node, instead of a bind mount that would have quietly served an empty directory. Against that: 642 lines instead of 95, half a gig of control plane, and six infrastructure pods before a single app starts.

The threshold isn't a container count. Every one of those wins turns on a specific condition, and I can list them:

**A second machine.** Three of the four storage options are pinned to one node and the fourth costs a fileserver. Everything Kubernetes does that Compose can't starts existing once there's somewhere else to schedule to: a pod that outlives the machine under it, a rolling update with room to run both copies at once, a scheduler with more than one answer.

**Apps that can run more than one replica.** Heimdall, qBittorrent, and ConvertX each own one writable config directory, so the replica count is pinned at 1 and `Recreate` is forced. Stateless services get rolling updates; stateful ones get the same downtime Compose gives, plus the YAML.

**Deploys frequent enough that you roll one back.** The eight seconds is a saving per rollback, not per deploy, and my test flattered it. Every image in the compose file is `:latest`, and the API server gives `:latest` a pull policy of `Always`:

```
$ kubectl apply --dry-run=server -f two-containers.yaml
lscr.io/linuxserver/heimdall:latest  -> imagePullPolicy=Always
lscr.io/linuxserver/heimdall:1.0.0   -> imagePullPolicy=IfNotPresent
```

`rollout undo` restores the previous pod template, and the previous template also says `:latest`, so it pulls the same broken image again. I got a clean rollback because I broke the deploy with a tag that never existed. Rolling back a bad upstream build needs pinned digests first, which Compose would have needed too, and which I haven't done.

**More than one person.** An Ingress in git beats four fields in the nginx-proxy-manager UI the moment somebody else has to know what changed, and loses to it at 11pm when I want one hostname exposed right now.

None of those four is true of one NUC that I administer alone. Two of them go true the day I add a second machine. So the chainsaw was never too much tool. It's the right tool for a job I don't have yet, and the hand saw is getting through the one plank in front of it.

## What it cost against what it gave

Everything measured above, side by side.

| | Docker Compose | k3s |
|---|---|---|
| Lines to describe the stack | 95 | 642 raw, 216 via Helm |
| Files | 1 | 6, or 3 in the chart |
| Objects | 5 services and a network | 29 |
| Control plane memory, idle | 70.4 MB | 560.7 MB |
| Infrastructure containers | 0 | 6 pods, 98 Mi |
| Cold start to all five serving | 4s | 10s |
| Teardown | 11s | 52s |
| Broken image tag, stateless app | pull fails, old container keeps serving | 30 of 30 requests served |
| Broken image tag, app with a volume | pull fails, old container keeps serving | 0 of 30 requests served |
| Rollback | edit the file, re-run `up -d` | `kubectl rollout undo`, 8s |
| Missing data directory | mounts an empty one, starts | refuses to schedule |
| Restart on process death | `restart: unless-stopped` | Deployment, 2s |
| Restart on node death | nothing | nothing, the control plane is on that node |
