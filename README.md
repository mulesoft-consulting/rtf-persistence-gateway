# RTF Persistence Gateway

`TL;DR`
- Quickly deploy PostgreSQL to back RTF Persistent Object Store at the `best cost`
- Works for both RTF appliance model and BYO k8s
- For testing ONLY, not recommended for production

`k8s` manifests to create `pv`, `pvc`, `cm`, `deployment` and `svc` to support a single instance PostgreSQL database workload to back RTF Persistence Gateway (Persistent Object Store).

## Create k8s Deployment, Secret and CRD

We'll be creating a simple PostgreSQL deployment in the same k8s cluster (bad idea for production but the best option at the best cost for testing) to back Persistent Object Store.

Choose the right manifests for target k8s distributions
- RTF appliance: k8s v1.13.x (due to upstream gravity 5.5.x -_-z)
- BYO k8s (EKS, GKE, AKS): k8s v1.18+

### Create `k8s` deployment, service, configmap and supporting Persistent Volume

```bash

git clone <repo>
cd <repo>/byo_k8s

# create pv, pvc, cm, deployment, svc
kubectl apply -f k8s-manifests/
```

### Create the CRD

```bash
# Persistence Gateway Secret and CRD
kubectl apply -f rtf-persistence-gateway/
```

### Verify

```bash
# postgres pod in default namespace should be Running
kubectl get po

# check persistent gateway pods's log
kubetl -n rtf logs -f persistence-gateway-<rs-uuid>-<pod-uuid>

# or  
kubectl -n rtf logs deployment/persistence-gateway
```

Expect to see HTTP response code 200 which indicating passing the health check

```bash
# RTF appliance
2021/04/26 00:08:27 Connecting to PostgreSQL backend...
2021/04/26 00:08:27 Connect: max open connectinons set to 20
2021/04/26 00:08:27 Starting watcher for /var/run/secrets/rtf-object-store/persistence-gateway-creds
2021/04/26 00:08:27 Watching for changes on /var/run/secrets/rtf-object-store/persistence-gateway-creds
2021/04/26 00:08:27 Succesfully connected to the PostgreSQL backend.
10.250.77.1 - - [26/Apr/2021:00:08:50 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.13+"
10.250.77.1 - - [26/Apr/2021:00:09:05 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.13+"
10.250.77.1 - - [26/Apr/2021:00:09:20 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.13+"
10.250.77.1 - - [26/Apr/2021:00:09:35 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.13+"
10.250.77.1 - - [26/Apr/2021:00:09:50 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.13+"

# BYO k8s on EKS
2021/04/25 23:25:57 Connecting to PostgreSQL backend...
2021/04/25 23:25:57 Connect: max open connectinons set to 20
2021/04/25 23:25:57 Starting watcher for /var/run/secrets/rtf-object-store/persistence-gateway-creds
2021/04/25 23:25:57 Watching for changes on /var/run/secrets/rtf-object-store/persistence-gateway-creds
2021/04/25 23:25:57 Succesfully connected to the PostgreSQL backend.
192.168.44.174 - - [25/Apr/2021:23:26:25 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.19+"
192.168.44.174 - - [25/Apr/2021:23:26:40 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.19+"
192.168.44.174 - - [25/Apr/2021:23:26:55 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.19+"
192.168.44.174 - - [25/Apr/2021:23:27:10 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.19+"
192.168.44.174 - - [25/Apr/2021:23:27:25 +0000] "GET /api/v1/status/ready HTTP/1.1" 200 0 "" "kube-probe/1.19+"

```

> NOTE: Mule Applications deployed to RTF clusters needs to have the `Use Persistent Object Storage` selected. Deployment via REST API requires the option `"persistentObjectStore": true` in the HTTP POST data JSON body.

## Considerations

For testing ONLY, not for production.

`hostPath` is used to persist PostgreSQL data, `HostPath` volume is for single node testing only, will NOT work in a multi-node cluster, consider using `local` volume instead.

> NOTE: by default the k8s secret is using the following DB connection string: `postgres://mulesoft:mulesoft@postgres.default.svc.cluster.local:5432/store`, make adjustment if necessary.

To support production RTF clusters, it is recommended to use external Production grade PostgreSQL database (HA), DB as a service ;-)

Alternative Production deployment options
- external containerized PostgreSQL (e.g. Docker Compose or Podman Compose style)
- external production grade HA PostgreSQL cluster (provided by Infrastructure team)
- Production grade [PostgreSQL Operator](https://github.com/zalando/postgres-operator) to create and manage PostgreSQL clusters (as StatefulSet vs Deployment)

> NOTE: It is intended for testing purpose ONLY. Local (to k8s cluster) has its Pros and Cons.

## Reference

Runtime Fabric Docs - [Persistence Gateway (Persistent Object Store)](https://docs.mulesoft.com/runtime-fabric/latest/persistence-gateway)
