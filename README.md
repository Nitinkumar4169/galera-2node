# 2-node Galera (Ansible) + ProxySQL & garbd (Kubernetes)

Two MariaDB Galera data nodes on VMs, built by Ansible. ProxySQL and the garbd
arbitrator run **inside your Talos Kubernetes cluster**, from one generated YAML
file. garbd is the third quorum vote, so `wsrep_cluster_size` reads **3**.

```
                 Kubernetes (Talos)                        VMs (Ansible-managed)
        ┌────────────────────────────────┐          ┌─────────────────────────────┐
 app ──►│ ProxySQL x2  (Service :6033)   │──:3306──►│ galera1 ◄──sync──► galera2  │
        │ garbd x1  (hostNetwork :4567)  │──:4567──►│        ▲   (data lives here)│
        └────────────────────────────────┘          └────────┼────────────────────┘
                                                      garbd joins as member #3
```

## How it works

**Why garbd?** Galera decides who may serve data by majority vote. With only 2
nodes, losing one leaves 1 of 2 votes, which is not a majority, so the survivor
stops serving to avoid split-brain. garbd stores no data; it only votes. With
3 votes (galera1, galera2, garbd) you can lose any ONE and the rest keep
serving. Lose two and the survivor stops, which is the safe behavior.

**How the two halves stay in sync.** There is one source of truth:
`inventory.ini` + `group_vars/all.yml`. Ansible uses them to configure the VMs,
then renders `build/proxysql-garbd.yaml` from the *same* values: the same two
IPs become ProxySQL's backends and garbd's join address, and the same
`galera_cluster_name` becomes garbd's `--group`. Change a value, re-run, and
both sides change together. Nothing is typed twice.

**Order of events.** `site.yml` -> preflight checks -> galera1 bootstraps ->
galera2 joins (SST) -> both wait until `Synced` -> manifest rendered.
Cluster size is **2** at this point. You apply the YAML, garbd joins, and it
becomes **3**. `verify.yml` waits for that and asserts it.

## Requirements

- 2 Ubuntu 24.04 VMs, SSH access, a user with passwordless `sudo`
- Ansible on the machine you run this from (`ansible-core`)
- `kubectl` access to the Kubernetes cluster
- The VMs and the Kubernetes nodes must reach each other:
  - VMs: `3306` from the Kubernetes nodes; `4567` (TCP+UDP), `4568`, `4444` between the VMs
  - garbd (runs on a Kubernetes node's IP): `4567` reachable **both ways** between it and the VMs
- Pods need internet to pull images, and the garbd pod needs apt access on first
  start (or use your own image, see "Production notes")

## Quick start

**1. Passwordless SSH** (from the machine running Ansible):
```bash
ssh-keygen -t ed25519            # press Enter through the prompts (skip if you have a key)
ssh-copy-id ubuntu@<galera1-ip>
ssh-copy-id ubuntu@<galera2-ip>
ssh ubuntu@<galera1-ip> 'sudo -n true && echo sudo-ok'   # must print sudo-ok with no prompt
```

**2. Fill in two files**
- `inventory.ini`: the two VM IPs (replace the `REPLACE_...` values)
- `group_vars/all.yml`: replace every `CHANGE_ME_...` password

The playbook refuses to run while placeholders remain, and tells you which one.

**3. Build the cluster and generate the manifest**
```bash
ansible -m ping galera           # both should answer: pong
ansible-playbook site.yml
```
Expect `wsrep_cluster_size` = **2** in the report. That is correct for now.

**4. Deploy ProxySQL + garbd**
```bash
kubectl apply -f build/proxysql-garbd.yaml
kubectl -n galera-proxy get pods -w      # garbd takes ~1-2 min on first start (installs itself)
```

**5. Confirm it is 3**
```bash
ansible-playbook verify.yml
```
It waits (up to 10 min) for `wsrep_cluster_size` = 3 on both nodes, then asserts
`Primary` and `Synced`. You can also check by hand on either VM:
```bash
sudo mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"        # 3
sudo mysql -e "SHOW STATUS LIKE 'wsrep_cluster_status';"      # Primary
sudo mysql -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"  # lists the members
```

**6. Query through ProxySQL** (throwaway client pod; `@@hostname` shows which node answered):
```bash
kubectl -n galera-proxy run mysql-test --rm -it --restart=Never --image=mariadb:lts -- \
  mariadb -h proxysql -P 6033 -u appuser -p'<app_password>' -e "SELECT @@hostname, 1+1;"
```
Your applications in the cluster connect to `proxysql.galera-proxy.svc.cluster.local:6033`.

## Demo: kill a node and show it survives
```bash
# on galera2:
sudo systemctl stop mariadb
# on galera1: size drops to 2 (galera1 + garbd), but it stays Primary and keeps serving
sudo mysql -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status');"
# re-run the step 6 query: still works, ProxySQL stops sending traffic to galera2
# bring it back:
sudo systemctl start mariadb      # size returns to 3 once it has resynced
```
This is the whole point of the arbitrator. Without garbd, stopping galera2 would
leave galera1 non-Primary and refusing queries.

## If garbd does not join (size stays 2)
1. `kubectl -n galera-proxy logs deploy/garbd`: look for apt errors (no internet) or Galera connection errors.
2. `kubectl -n galera-proxy describe pod -l app=garbd`: a Pod Security rejection means the namespace label was not applied.
3. Check ports `4567` TCP+UDP both directions between the garbd pod's *node IP* and each VM.
4. The group name must match `galera_cluster_name` exactly.

## Production notes
- **Failure domain:** put garbd on a Kubernetes node that does NOT share a physical
  host with a Galera VM (use `garbd_node_selector`). If one hypervisor hosts galera1
  *and* garbd, losing it removes 2 of 3 votes and the cluster stops.
- **garbd image:** by default the pod installs `galera-arbitrator-4` at startup.
  For production, bake an image (`FROM ubuntu:24.04` + `apt-get install -y galera-arbitrator-4`),
  push it to your registry, set `garbd_image`, and set `garbd_install_on_start: false`.
- **Secrets:** `build/proxysql-garbd.yaml` contains passwords (it is git-ignored).
  Move them to Ansible Vault / sealed secrets before this goes beyond a lab.
- **Writes through ProxySQL:** both nodes are in one hostgroup, so writes may land on
  either node. That is safe in Galera, but busy hot rows can hit deadlock retries.
  If your app needs a single writer, ProxySQL's Galera hostgroups feature can enforce it.
- **Re-running** `site.yml` is safe: a node that is already a healthy cluster member is left alone.
- **Version drift:** `mariadb-server` is unpinned, so a rebuild months later may pull a newer 10.11.x.

## What was tested, and what was not
Tested here (offline): playbook syntax; preflight passes with real values and rejects
placeholders/bad passwords with a clear message; the generated YAML passes strict
Kubernetes schema validation (v1.33, v1.34); IPs, cluster name and group are identical
across Galera config, ProxySQL and garbd; the embedded garbd script is valid bash;
`verify.yml`'s assertions pass on healthy output and fail on size 2 / non-Primary;
the garbd flags were checked against the real Ubuntu `galera-arbitrator-4` package
(26.4.16, same Galera 4 family as the VMs); the ProxySQL image tag exists.

**Not tested (no VMs or cluster available where this was built):** a live run of the
Galera role (it reuses the logic verified on EC2; new here are the `appuser` account
and the wait-for-Synced step), garbd actually joining from a pod on your network, and
ProxySQL pods reaching your VMs. Expect to use the troubleshooting list above on the first run.
