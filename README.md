# cgroupv2 domain controllers repro

Reproduces the runc error:

```
Error response from daemon: failed to create task for container: failed to create shim task:
OCI runtime create failed: runc create failed: unable to start container process:
unable to apply cgroup configuration: cannot enter cgroupv2 "/sys/fs/cgroup/docker"
with domain controllers -- it is in an invalid state
```

(In some runs the tail reads `it is in threaded mode` instead of `an invalid state`;
they are the same underlying failure.)

## Minimal repro

With a dockerd running on a host whose root cgroup is `domain threaded`
(see `Cause` below), any container that requests a domain-only controller
such as the memory controller will fail to start.

```bash
# Fails — memory controller requires a domain cgroup parent:
docker run --rm --memory=256m alpine echo ok

# Works — no domain-only controller is requested:
docker run --rm alpine echo ok

# Fails — docker compose up with the provided compose file (uses mem_limit):
docker compose up --detach
```

## Cause

On this host the cgroupv2 hierarchy is in a state where:

```text
$ cat /sys/fs/cgroup/cgroup.type
domain threaded

$ cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids

$ cat /sys/fs/cgroup/cgroup.subtree_control
cpuset cpu pids            # <-- note: memory/io/hugetlb NOT delegated

$ cat /sys/fs/cgroup/docker/cgroup.type
threaded

$ cat /sys/fs/cgroup/docker/cgroup.controllers
cpuset cpu pids
```

Because the root cgroup is a `domain threaded` node (the root of a threaded
subtree) and only threaded-capable controllers (`cpuset`, `cpu`, `pids`) are
enabled in `cgroup.subtree_control`, `/sys/fs/cgroup/docker` is created as a
`threaded` cgroup. Requesting a container resource limit that depends on a
**domain-only** controller — `memory` (i.e. `--memory` / `mem_limit`),
`io`, or `hugetlb` — forces runc to try to enter `/sys/fs/cgroup/docker`
"with domain controllers", which is rejected because that cgroup is threaded.

The `docker-compose.yml` in this repo is the minimal form of the original
report: a single `alpine` service with `mem_limit: 6m` (the smallest value
the Docker daemon accepts) is enough to reproduce. The original compose
file's report nondeterminism about which service fails first is just a
consequence of every service having a `mem_limit`.

## Why this host is in a threaded state

The agent/VM host was booted with a cgroup namespace whose root is
`domain threaded`. This is typical of nested-container environments where an
outer orchestrator placed the inner environment inside a threaded subtree
without delegating the `memory`/`io` controllers. Docker itself doesn't put
the root into this state — it inherits it from the parent environment.

## Fixes / workarounds

In order of preference:

1. **Fix the host's cgroup setup** so the cgroup seen by dockerd is a plain
   `domain` cgroup with the `memory` (and ideally `io`, `hugetlb`)
   controllers available in `cgroup.subtree_control`. For systemd-managed
   hosts this usually means enabling the unified hierarchy
   (`systemd.unified_cgroup_hierarchy=1`) and not running dockerd from a
   threaded service slice. For nested environments (LXC, Kubernetes
   nodes, cloud dev VMs) it means the outer platform needs to give the inner
   VM a domain cgroup with the memory controller delegated.

2. **Run dockerd in its own cgroup namespace / unshare the cgroup** so it
   creates a fresh `domain` root. For example, launching dockerd under
   `systemd-run --scope -p Delegate=yes` on a host with a proper unified
   hierarchy, or starting it inside a user cgroup whose
   `cgroup.subtree_control` includes `memory io`.

3. **Remove domain-only resource limits from the compose file** as a
   workaround when you cannot change the host. Dropping every `mem_limit:`
   (and not using `--memory`, `--blkio-weight`, `--device-read-bps`, etc.)
   lets containers start, at the cost of losing those limits. This is only
   a bandage — on a properly configured host you want the limits back.

4. **Use `--cgroup-parent`** on `docker run` (or
   `cgroup_parent:` in compose) to point at an existing `domain` cgroup
   that has the needed controllers enabled, if one exists on the host.

## Files in this repro

- `docker-compose.yml` — minimal compose file that triggers the error:
  one `alpine` service with `mem_limit: 6m`.
