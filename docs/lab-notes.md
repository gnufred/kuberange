# KubeRange lab notes

See the [KubeRange README](../README.md) for host prerequisites and a starter cluster. These notes record how the range is provisioned and which layouts have been tested.

## Stable stack validation

On 19 November 2026, the Alpine 3.24.4 box built with the Packer QEMU 1.1.8 plugin and booted under libvirt. UEFI, SSH key replacement, `doas` and the QEMU guest agent worked on `lb0`. The signed package refresh brought HAProxy to 3.4.8 and installed Alpine's OpenSSL and GnuTLS security updates. HAProxy and Unbound passed their configuration checks.

`cp0` and `wo0` bootstrapped Kubernetes 1.37.2 with containerd 2.4.2, runc 1.5.3 and CNI plugins 1.10.0. A Deployment using the cached pause image ran on `wo0`, and a temporary diagnostic Pod on `wo0` resolved the API Service through CoreDNS 1.15.0. The Python 37.0.3 client passed the provisioning tasks for node registration, DNS and RBAC. After a reboot of `wo0`, OpenRC restarted containerd and kubelet, and the DNS probe passed again.

The etcd 3.7.4 snapshot exercise restored a ConfigMap to its earlier value on a single control plane. `cp1` then joined, both etcd endpoints passed health checks, and CoreDNS ran on both control planes. On `nf0`, a static NFSv4.2 PV/PVC let a non-root writer Pod save a file and a replacement reader Pod recover it.

## Provisioning details

### Host services

After a host reboot, run `run/services` to start the `virtqemud`, `virtstoraged`, and `virtnetworkd` systemd sockets used by libvirt. The command asks for sudo when needed and reports each socket's status. Use `run/services status` to check without starting anything. These sockets activate their daemons on demand; they are not enabled for automatic startup by this helper.

The host firewall must still allow traffic on the libvirt management bridge; the bridge name and existing UFW rules depend on the host configuration. This helper does not modify firewall rules or start VMs.

### Provision

Start the validated single-control-plane setup with:

```bash
run/provision lb0
run/provision cp0
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes'
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -n kube-system'
```

This creates a fresh control plane behind the load balancer. Changing the base box takes effect when the VM is recreated. The argument-free `run/provision` builds the entire lab.

### Add the other control planes

After `cp0` is healthy, provision fresh replicas with:

```bash
run/provision cp1
run/provision cp2
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide'
```

The full provision helper also adds control planes sequentially and stops with a nonzero exit status if provisioning or joining fails. This avoids overlapping stacked-etcd membership changes and prevents a background job from hiding a failed join.

Control-plane joins explicitly use the private `eth1` address for `--apiserver-advertise-address`, matching `cp0`. Kubeadm uses this address for the new API server and etcd peer certificates/listeners. Allowing automatic selection chose the management address (`192.168.121.x`) in the first test: etcd then rejected a peer's management source IP because its certificate contained the private IP. The failed learner was removed and `cp1` rejoined using `10.1.2.21`; `cp2` joined directly using `10.1.2.22`.

The join playbook creates the etcd certificate directory before copying the shared CA material, including when retrying after `kubeadm reset`. Shared private keys are installed as root with mode `0600`; public certificates and keys use `0644`.

Before the network-address changes, validation on 4 November 2026 found `cp0`, `cp1` and `cp2` Ready on v1.36.1, with 16/16 system Pods Running. Etcd has three voting members, no learners, one leader and matching applied Raft indexes; health proposals succeeded at every endpoint. Authenticated `/readyz` requests passed on all three private API addresses and the load-balanced endpoint. The controller-manager and scheduler on `cp1` retain a restart count of one from the failed join/reset; retained logs distinguish that attempt from their healthy replacements. Helper checks verified sequential ordering and propagation of both provisioning and join failures.

### Add workers

After the control plane is healthy, provision fresh workers with:

```bash
run/provision wo0
run/provision wo1
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide'
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o custom-columns=NAME:.metadata.name,POD-CIDR:.spec.podCIDR'
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf -n kube-system get pods -o wide'
```

Worker provisioning remains parallel during a full bootstrap, but the helper records each worker PID and checks its exit status. A failed provision or join returns a nonzero status even if a later worker succeeds. A targeted `run/provision wo0` launches only that worker.

Before adding the control-plane replicas and DNS DaemonSet, the tested layout had three Ready nodes (`cp0`, `wo0`, `wo1`) and eight ready system Pods. Temporary non-root diagnostic Pods on all three nodes passed direct Pod connections in every direction and connections through a separate Service per endpoint, including each Pod reaching itself through its Service. Servers observed the original client Pod IP on cross-node connections. Each client resolved cluster and external names, received NXDOMAIN for a nonexistent cluster name, queried CoreDNS over UDP and TCP, and made a service-account-authenticated HTTPS request to the API `/version` with CA verification.

The diagnostic image was built offline from the checksum-verified Alpine 3.24.3 minirootfs and verified cached Python APKs. It is temporary; the regular worker archive still contains only kube-proxy and pause. No network daemon, overlay or new CNI plugin is installed. See the network addressing section for routes, NAT, firewall persistence and the scope of these checks.

### NFS storage

Provision the dedicated NFS server VM with `run/provision nf0`. It serves `/export` at private address `10.1.2.40`, using the cached Alpine NFS packages and the OpenRC `rpcbind` and `nfs` services. `/etc/exports` allows the six worker hostnames (`wo0` through `wo5`) and retains `rw,root_squash,no_subtree_check`. The NFS server stays separate from Kubernetes node registration.

The export directory is owned by `root:vagrant` (GID 1000), with mode `2775`: the shared lab group can write, and newly created subdirectories inherit its group. Client-root writes are still squashed and denied. Applications should use the matching identity/group; the tested Pods used:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  runAsNonRoot: true
```

A root-owned `0755` export mounted successfully but did not let the normal worker user write. Provisioning now sets the tested group and mode explicitly without making the directory world-writable or disabling root squash. Backing directories for static NFS volumes can be created by UID/GID 1000 underneath `/export`.

Before the network-address changes, validation on 4 November 2026 confirmed `wo0` mounted `nf0:/export` over NFSv4.2/TCP, wrote/read data as UID/GID 1000, and failed a client-root write as expected. A temporary static `ReadWriteMany` NFS PV/PVC bound successfully with `nfsvers=4.2`. A non-root writer Pod on `wo0` read worker-created data and wrote its own file. After that Pod was deleted, a replacement reader Pod recovered the same file. Its contents were also verified directly on `nf0` and through the worker mount. Temporary Pods, PV/PVC, test data, mounts and the diagnostic image were cleaned up. Multi-worker sharing has not yet been exercised.

The full provision helper keeps NFS and workers parallel, tracks the NFS job explicitly, and waits for it before returning. NFS VM/playbook failures now return a nonzero status even when workers succeed; worker failures also remain visible when NFS succeeds. The duplicate empty NFS installation play was removed.

### Playbook

Use the Playbook helper to run playbooks, roles, or tags on the Vagrant VM. 

Read `run/playbook --help`.

Vagrant's Ansible provisioners use `run/vagrant-ansible` through the supported `playbook_command` setting. The wrapper replaces Vagrant 2.4.9's deprecated `--inventory-file` argument with `--inventory`, forwards all other arguments, and preserves Ansible's exit status. This avoids the deprecated alias without changing the installed Vagrant package.

Privilege escalation uses `community.general.doas`, matching the Alpine base box. Install the `community.general` collection on the host if it is not supplied by your Ansible package. The box's sudo compatibility shim does not implement all options used by Ansible's default sudo plugin.

The local `ansible.cfg` uses `interpreter_python = auto_silent`: Ansible still discovers the guest Python interpreter, but suppresses its interpreter-selection warning. This applies when running the lab commands from this directory.

For disposable Vagrant VMs, the same configuration uses `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null`. Ansible does not check or store user known-host keys, allowing destroy/recreate cycles that regenerate keys and reuse IP addresses. This deliberately skips SSH host identity verification for Ansible runs from this lab directory; normal SSH configuration and other project configurations are not modified.

### Kubectl

The control plane includes kubectl 1.37.2. Use `vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes'` from this directory. A host client within one minor version can connect using `.kubeconfig`.

## Build an Alpine base box

`run/build-alpine` builds a local Alpine 3.24.4 amd64 UEFI Vagrant/libvirt box from the pinned [rgl recipe](../build/alpine/README.md), adapted for an Arch host. It requires Packer 1.16+, QEMU/KVM and `edk2-ovmf`. Use `run/build-alpine --check` to initialize plugins and validate without booting a VM. Build output stays in `.cache/build-alpine/`. After a successful build, the helper prints the command to register the box as `myk8s/alpine-3.24`; it does not change the lab configuration or register it automatically.

## Offline OS packages

The APK cache targets Alpine **3.24 / x86_64**, using the latest packages available in that stable branch when refreshed. `config/apk-packages.json` lists the requested packages and repositories; `config/apk-packages.lock.json` records every cached filename and SHA-256. Each directory includes its complete recursive dependency set, so provisioning needs no Alpine mirror access. Shared dependencies appear in multiple groups to keep each existing wildcard installation self-contained.

The regular Alpine packages include Python 3.14, OpenRC, UFW, iptables, HAProxy, Unbound, rsyslog and NFS tooling. The guest kernel and libraries follow Alpine 3.24 stable. The iptables package uses the **nf_tables** backend.

After pulling LFS, run `run/cache-apk --check` to verify the APKs without network access. To refresh them online, run `run/cache-apk` on a Linux x86-64 host with Python 3.11+ and `bsdtar`. The helper checks the resolver and release keyring and verifies the downloaded APK signatures before replacing the cache. Provisioning requires the Alpine 3.24 base box.

Validation on 4 November 2026: all 312 cached APK files passed Alpine signature verification. `lb0` and `cp0` were created from the Alpine 3.24.3 base box and provisioned using the local APK and image caches. Kubernetes reports `cp0` Ready on v1.36.1 with containerd 2.3.7; all six system Pods are 1/1 Running with zero restarts. API readiness succeeded directly and through the authenticated host connection to `https://10.1.2.10:9443`; etcd health and DNS queries through CoreDNS and Unbound passed.

## Kubernetes and containerd configuration

`config/kubernetes.json` pins the Kubernetes version and image references used by the Ansible tasks and image-cache helper. The kubeadm template uses `kubeadm.k8s.io/v1beta4` and explicitly sets `kubernetesVersion`, so offline initialization does not query the upstream release channel. It advertises the control plane's private `eth1` address. Kubelet's OpenRC arguments likewise select the private node address.

Kubelet and runc use the `cgroupfs` driver with OpenRC's unified cgroup v2 hierarchy. Provisioning starts/enables the `cgroups` service and checks `/sys/fs/cgroup/cgroup.controllers`. BusyBox `stat -f %T` reports `UNKNOWN` for this filesystem on the tested box, so it is not used as the readiness check.

The release binaries use pinned upstream checksums in [config/binaries.json](../config/binaries.json). `run/cache-binaries --check` verifies them locally, and `run/cache-binaries` downloads any missing pins. Kubernetes uses `gcompat` from Alpine stable for the upstream kubelet binary. The runtime and CLI binaries are under `/usr/local/bin`. CNI plugins are under `/opt/cni/bin`.

OpenRC runs the bundled containerd and kubelet services. `/etc/containerd/config.toml` imports `/etc/containerd/conf.d/20-kubernetes.toml`. This pins the pause image and selects the cached runc binary with the `cgroupfs` driver. Containerd restarts if provisioning changes its runtime, CNI plugins or configuration. Use `vagrant ssh cp0 -c 'doas crictl images'` or `vagrant ssh cp0 -c 'doas ctr --namespace k8s.io images list'` for runtime diagnostics.

The control-plane Python environment at `/opt/kuberange/ansible` uses wheels from `.lfs/python/`. Ansible's Kubernetes modules use this interpreter for API operations. To repopulate the Alpine x86-64 Python 3.14 wheel cache for the committed lockfile, run:

```bash
python3 -m pip download --require-hashes --only-binary :all: \
  --implementation cp --python-version 3.14 --abi cp314 --abi abi3 \
  --platform musllinux_1_2_x86_64 --platform musllinux_1_1_x86_64 \
  --dest .lfs/python -r config/kubernetes-python.lock.txt
```

## Offline Kubernetes images

The Git LFS OCI archives contain verified Linux/amd64 image manifests, configurations and layers from the official Kubernetes registry. `config/kubernetes-images.lock.json` records the index/manifest digests and archive SHA-256 values. Pull LFS after cloning to restore the archives; provisioning never downloads them.

The control-plane archive contains the API server, controller manager, scheduler, kube-proxy, CoreDNS, pause and etcd. The worker archive contains kube-proxy and pause. Both are imported into containerd's `k8s.io` namespace before kubeadm initializes or joins a node. Kubeadm's CoreDNS and etcd tags match the cached versions declared in `config/kubernetes.json`.

Run `run/cache-images` (Python 3.11+) to rebuild the archives and lockfile online. It selects the Linux/amd64 manifest for each image and verifies the manifest and layer digests. For an existing node, `run/playbook -h cp0 -b install -t blobs` imports the cached control-plane archive again.

## Practice etcd snapshot recovery

Use the single-control-plane cluster with `lb0` and `cp0`. A restore rolls back Kubernetes objects to their state at the time of the snapshot.

```bash
run/playbook -h cp0 -b ic -t etcd_backup
run/playbook -h cp0 -b ic -t etcd_restore -x '{"backup_file":"/absolute/path/to/snapshot.db"}'
```

The backup playbook saves the snapshot under `pb/ic/files/`. Use the path printed by Ansible. The restore playbook checks that etcd has one member, restores and verifies the snapshot with `etcdutl`, and stops the API and etcd static Pods while it swaps the data directory. It waits for the restored API to become ready before discarding the previous data.

## Network addressing

`config/network.json` defines shared Service/DNS settings and each Kubernetes node's private address and Pod subnet. Vagrant and the generated `/etc/hosts` entries use the same node address map. The VM private network stays `10.1.2.0/24`; Kubernetes Services use `10.96.0.0/16`, with the API Service at `10.96.0.1` and CoreDNS at `10.96.0.10`. Kubeadm and kubelet use the same DNS address. Changing the Service CIDR requires a fresh cluster; this configuration is not an in-place migration of existing VMs.

Pods query CoreDNS directly. CoreDNS answers `cluster.local` and forwards other queries to Unbound on `lb0` (`10.1.2.10`). Unbound retains its `cluster.local` forwarding zone for host/VM clients, now pointing to the DNS Service IP. Its other queries go to libvirt DNS at `192.168.121.1`, which uses the host resolver. Direct requests from `lb0` to the old public upstream `1.1.1.1` timed out on the test host, while libvirt DNS answered; no host firewall policy was changed. CoreDNS handles cluster names authoritatively, so this split forwarding does not send unresolved cluster names back to Unbound.

Because `lb0` is outside Kubernetes, it needs a route to the Service subnet via `cp0` (`10.1.2.20`). The DNS playbook installs and applies `/etc/network/if-up.d/myk8s-service-route`; Alpine reapplies it when the private interface comes up. This preserves the existing dependency on `cp0` for host-side cluster DNS. No new networking component is installed. The DNS playbook updates the kubeadm-created Service without deleting/recreating it.

Validation on 4 November 2026 (Service/DNS checkpoint): a fresh `lb0`/`cp0` bootstrap produced a Ready control plane and six ready system Pods. API and DNS Services have the new addresses; kubelet uses `10.96.0.10`. CoreDNS and Unbound both resolved the Kubernetes API name to `10.96.0.1`, external DNS resolved, and a missing Service name returned NXDOMAIN. Unbound cluster DNS passed over UDP and TCP after rebooting `lb0`; its static Service route survived. The following Pod-routing checkpoint extends these checks across nodes.

### CoreDNS placement

The DNS playbook manages a `coredns` DaemonSet restricted to Linux control-plane nodes, with the control-plane taint tolerations. It runs one Pod on `cp0` during bootstrap, then adds one each on `cp1` and `cp2` as they join. Its rolling update uses `maxUnavailable: 1` and `maxSurge: 0`, keeping at most one DaemonSet Pod per control plane during replacement. The existing `kube-dns` Service (`10.96.0.10`, UDP/TCP 53) selects all replicas.

To migrate the kubeadm-created Deployment, the playbook creates the DaemonSet and waits for it to become ready before deleting the Deployment. Both can briefly serve queries on `cp0` during this migration; the playbook waits for the old Deployment Pods to terminate before returning. Subsequent runs are idempotent. Kubeadm normally manages CoreDNS as a Deployment, so its addon behavior must be accounted for during future upgrades; reapply this playbook if kubeadm recreates that Deployment.

Inspect or reapply it with:

```bash
run/playbook -h cp0 -b dns -t coredns,kubedns
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf -n kube-system get daemonset coredns'
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf -n kube-system get pods -l k8s-app=kube-dns -o wide'
```

Validation on 4 November 2026 (DaemonSet bootstrap checkpoint): with only `cp0` eligible, the DaemonSet has one desired/current/ready Pod on `cp0`; the legacy Deployment is absent. Cluster DNS through the Service and cluster/external DNS through Unbound passed. Reapplying the migration reported zero changes. Additional control-plane validation is recorded below. The Service route from `lb0` still uses `cp0`; replica placement alone does not provide VM-side DNS routing failover.

When the Service selects a CoreDNS Pod on another control plane, kube-proxy can masquerade node/VM clients to the forwarding node's private IP. Initial `lb0` probes succeeded only 10/20 times with two DNS replicas: `cp1` logged UFW drops from `10.1.2.20` to its DNS Pod at `10.88.1.2:53`. The installation playbook now permits UDP/TCP 53 from `lab_node_subnet` (`10.1.2.0/24`) to the local Pod subnet on control planes only. Existing Pod-to-Pod rules and the routed deny default are retained.

Validation on 4 November 2026 (three-control-plane DNS checkpoint): fresh `cp1` and `cp2` joined sequentially with private addresses and the configured static Pod CIDRs. All five Kubernetes nodes are Ready, all 20 system Pods are ready, and etcd has three healthy voting members. The DaemonSet has exactly one ready Pod on each of `cp0`, `cp1`, and `cp2`; no CoreDNS Deployment remains, and the shared Service has three ready DNS endpoints.

Temporary non-root clients on `wo0`/`wo1` queried every CoreDNS Pod IP, a separate diagnostic Service per backend, and the shared Service over UDP and TCP. `lb0` queried all backend Services, the shared Service, and Unbound. Each path returned the API Service A record, NXDOMAIN for fresh nonexistent cluster names, and external A records: 114 queries passed per full matrix. The shared Service also passed six queries from each of the five node VMs. With `cp0` halted, both worker Pod clients passed all six shared-Service queries after Kubernetes removed the offline DNS endpoint; the API readiness check through `cp1` passed. `cp0` was restored with `--no-provision`, all three DNS endpoints returned, and the full 114-query matrix passed again.

The final DNS playbook and Pod-network rules reapplied with zero changes. Diagnostic namespaces, Pods, backend Services, labels, image records and uploaded archives were removed. `lb0`, `cp0`, `cp1`, `cp2`, `wo0` and `wo1` remain running. This tests Pod-side DNS node redundancy; Unbound's cluster-DNS route still depends on `cp0`.

### Static Pod routing

The existing `bridge`, `host-local` and `portmap` plugins are retained. The old IPv6 pool is removed; kubeadm and the Pod network are IPv4-only. The Pod range is `10.88.0.0/16`, split into fixed, disjoint `/24` subnets:

| Node | Private IP / route gateway | Pod subnet |
| --- | --- | --- |
| cp0 | 10.1.2.20 | 10.88.0.0/24 |
| cp1 | 10.1.2.21 | 10.88.1.0/24 |
| cp2 | 10.1.2.22 | 10.88.2.0/24 |
| wo0 | 10.1.2.30 | 10.88.10.0/24 |
| wo1 | 10.1.2.31 | 10.88.11.0/24 |
| wo2 | 10.1.2.32 | 10.88.12.0/24 |
| wo3 | 10.1.2.33 | 10.88.13.0/24 |
| wo4 | 10.1.2.34 | 10.88.14.0/24 |
| wo5 | 10.1.2.35 | 10.88.15.0/24 |

For example, `wo0` reaches `10.88.11.0/24` through `10.1.2.31` on `eth1`; `wo1` reaches `10.88.10.0/24` through `10.1.2.30`. Each node has routes for all configured peers, including peers not yet provisioned. Its own Pod subnet is connected through `cni0`, created when its first non-host-network Pod starts. `/etc/network/if-up.d/myk8s-pod-routes` reapplies peer routes after the private interface comes up.

Kubeadm sets the aggregate `podSubnet` for kube-proxy's cluster-traffic detection. The controller-manager's automatic CIDR allocator is disabled: fixed allocations must not depend on join order. After initialization/join, Ansible records the configured subnet as the Node's `spec.podCIDR`/`spec.podCIDRs`, rejecting a conflicting existing allocation. Recreate old-layout nodes; changing their CNI subnet underneath existing Pods is not supported.

The bridge does not install its per-Pod masquerading rules. `/etc/ufw/after.init` instead maintains one idempotent egress NAT rule per node: traffic from its local Pod subnet to destinations outside `10.88.0.0/16` is masqueraded. Cross-node Pod traffic retains its original source IP. Kube-proxy still handles Service translation and hairpin NAT; the bridge enables hairpin mode. Containerd restarts when its CNI template or runtime drop-in changes.

Incoming/outgoing VM access retains the existing permissive training policy. The default routed policy is explicitly **deny**, checked after IP forwarding is enabled. Scoped forwarding rules permit local-to-remote Pod traffic, remote-to-local Pod traffic, Pod DNS to Unbound, Pod API requests to the control planes, Unbound queries reaching CoreDNS after Service DNAT, and node/VM-proxied DNS queries reaching a control plane's local Pod subnet over UDP/TCP port 53. `run/playbook -h cp0,wo0,wo1 -b install -t pod-network` reapplies the networking configuration. The DNS playbook waits for CoreDNS availability, so failed Pod startup cannot be reported as successful provisioning.

This is a static lab network: adding a node requires extending `config/network.json` and applying peer routes on existing nodes. NetworkPolicy enforcement, routing failover, load-balancer HA, arbitrary Pod Internet egress and multi-worker NFS sharing are outside the current validation. No additional networking component is required.

Validation on 4 November 2026 (static routing checkpoint): `cp0`, `wo0` and `wo1` are Ready on v1.36.1 with eight ready system Pods. All three diagnostic Pods passed the direct/Service/DNS/API matrix above, then passed it again after all three Kubernetes VMs were rebooted with `--no-provision`. After reboot, each node had IP forwarding enabled, all eight peer routes, exactly one Pod egress NAT rule, and UFW's routed deny policy. Two UFW reloads on `wo1` also preserved exactly one NAT rule. Reapplying the final Pod-network playbook reported zero changes on all three nodes; reapplying CoreDNS and its availability check reported zero changes. Diagnostic Pods, Services, namespace, image records and uploaded archives were removed. The four requested VMs (`lb0`, `cp0`, `wo0`, `wo1`) remain running. Other nodes were not started for this checkpoint.

## Kubeadm initialization logs

When the initialization task runs, Ansible launches `kubeadm init` with `--v=5` and writes both standard output and standard error to `/var/log/kubeadm-init.log` on `cp0`. Each attempt replaces the previous log. The log is owned by `root:vagrant` with mode `0640`, allowing the Vagrant user to read it. Shell redirection preserves kubeadm's exit status, so Ansible still reports initialization failures.

Follow it from another host terminal with `run/log watch -h cp0 -n kubeadm`, or read it with `run/log cat -h cp0 -n kubeadm`. Watch mode retries while waiting for the file to appear and follows the filename if the file is replaced. Ansible continues to own the initialization process; the helper only reads its log over SSH.

The existing `/etc/kubernetes/admin.conf` guard still controls whether initialization runs. That file may exist after a partially failed initialization; its existence alone does not establish a healthy control plane. Logging applies to the next task invocation and does not capture a command already running.
