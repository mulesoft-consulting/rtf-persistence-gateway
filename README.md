# RTF Persistence Gateway

`TL;DR`
- Quickly deploy PostgreSQL to back RTF Persistent Object Store at the `best cost`
- Caters for both RTF appliance and BYO k8s (AKA BYOK)
- For testing purpose ONLY, production use cases require thorough considerations

This repo consists of `k8s` manifests to create `pv`, `pvc`, `cm`, `deployment` `svc` and `crd` to support a single instance PostgreSQL database workload to back RTF Persistence Gateway (Persistent Object Store).

## Create k8s Deployment, Secret and CRD

We'll be creating a simple PostgreSQL deployment in the same k8s cluster where RTF is running (bad idea for production but the best option at the best cost for testing) to back Persistent Object Store.

Choose the right manifests for target k8s distributions
- RTF appliance: k8s v1.13.x (due to upstream gravity 5.5.x)
- BYO k8s (EKS, GKE, AKS): k8s v1.18+

### Create `k8s` deployment, service, configmap and supporting Persistent Volume

```bash
# clone the repo
git clone <repo>
# choose k8s distro - e.g. byok
cd <repo>/byo_k8s

# create pv, pvc, cm, deployment, svc
kubectl apply -f k8s-manifests/
```

### Create the CRD

> NOTE: It is recommended to adopt declarative management of k8s objects.

```bash
# Persistence Gateway Secret and CRD
kubectl apply -f persistence-gateway-crd/
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

## Troubleshooting

In the unlikely event that exploring PostgreSQL is required, use the workflow:
- attach into postgresql pod: `kubectl exec -ti postgres-<rs-id>-<pod-id> -- /bin/bash`
- connect to postgresql via `psql` CLI: `psql -h localhost -p 5432 -d store -U mulesoft`
- check if relevant databases / tables exist
- check rows in the table
- etc.

> NOTE: `-d dbname` and `-U username` are defined in the CRD's secret as `persistence-gateway-creds` (`base64` encoded). 

Example output
```bash
controller-1:/$ kubectl exec -ti postgres-dcdcc6546-cpffd -- /bin/bash
root@postgres-dcdcc6546-cpffd:/# psql -h localhost -p 5432 -d store -U mulesoft
psql (13.4 (Debian 13.4-1.pgdg100+1))
Type "help" for help.

store=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | mulesoft | UTF8     | en_US.utf8 | en_US.utf8 | 
 store     | mulesoft | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | mulesoft | UTF8     | en_US.utf8 | en_US.utf8 | =c/mulesoft          +
           |          |          |            |            | mulesoft=CTc/mulesoft
 template1 | mulesoft | UTF8     | en_US.utf8 | en_US.utf8 | =c/mulesoft          +
           |          |          |            |            | mulesoft=CTc/mulesoft
(4 rows)

store=# \d
              List of relations
 Schema |     Name      |   Type   |  Owner   
--------+---------------+----------+----------
 public | items         | table    | mulesoft
 public | items_id_seq  | sequence | mulesoft
 public | stores        | table    | mulesoft
 public | stores_id_seq | sequence | mulesoft
(4 rows)

store=# \d stores;
                                          Table "public.stores"
       Column        |          Type          | Collation | Nullable |              Default               
---------------------+------------------------+-----------+----------+------------------------------------
 id                  | integer                |           | not null | nextval('stores_id_seq'::regclass)
 name                | character varying(255) |           | not null | 
 org_id              | character varying(255) |           | not null | 
 env_id              | character varying(255) |           | not null | 
 default_ttl_seconds | integer                |           | not null | 
 is_fixed_ttl        | boolean                |           | not null | 
Indexes:
    "stores_pkey" PRIMARY KEY, btree (id)
    "uk_stores" UNIQUE CONSTRAINT, btree (name, org_id, env_id)
Referenced by:
    TABLE "items" CONSTRAINT "items_store_id_fkey" FOREIGN KEY (store_id) REFERENCES stores(id)

store=# \d items;
                                         Table "public.items"
    Column    |            Type             | Collation | Nullable |              Default              
--------------+-----------------------------+-----------+----------+-----------------------------------
 id           | integer                     |           | not null | nextval('items_id_seq'::regclass)
 store_id     | integer                     |           | not null | 
 key          | character varying(255)      |           | not null | 
 partition    | character varying(255)      |           | not null | 
 value_type   | character varying(10)       |           | not null | 
 number_value | integer                     |           |          | 
 string_value | text                        |           |          | 
 binary_value | bytea                       |           |          | 
 last_updated | timestamp without time zone |           |          | 
 is_fixed_ttl | boolean                     |           | not null | 
 ttl          | timestamp without time zone |           | not null | 
Indexes:
    "items_pkey" PRIMARY KEY, btree (id)
    "idx_items_ttl" btree (ttl)
    "uk_items" UNIQUE CONSTRAINT, btree (key, store_id, partition)
Foreign-key constraints:
    "items_store_id_fkey" FOREIGN KEY (store_id) REFERENCES stores(id)

store=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 mulesoft  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}

store=# 
```

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
