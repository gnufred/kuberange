# KubeRange

KubeRange is a disposable Kubernetes practice range for a Linux workstation. It uses Vagrant, libvirt and Ansible to build a kubeadm cluster on Alpine Linux. Start with one control plane and one worker to practice deployments and troubleshooting. Add control-plane replicas for stacked etcd and API availability exercises, or an NFS server for persistent volumes.

The current configuration uses Alpine 3.24.4, Kubernetes 1.37.2 and containerd 2.4.2. The base box, APK packages and Kubernetes system images are cached in the repository with Git LFS. Once you have the host tools and those assets, cluster provisioning can run offline.

## Host prerequisites

- An x86-64 Linux host with hardware virtualization, working QEMU/KVM and access to the system libvirt instance. The bundled box uses UEFI and expects the Arch `edk2-ovmf` firmware file at `/usr/share/edk2/x64/OVMF.4m.fd`.
- Git and Git LFS, Vagrant with the `vagrant-libvirt` provider, Ansible, Python 3, and `rsync`. The Ansible playbooks use the `community.general` and `kubernetes.core` collections.
- Enough free RAM and disk for the guests. The three-VM starter assigns about 4.6 GB of guest RAM. All eleven VMs together assign about 18.2 GB, plus host overhead.
- An isolated lab network and a host firewall that allows traffic over the libvirt management bridge. The range uses `10.1.2.0/24` for private VM addresses. Avoid running two copies that use this same subnet.

## Start a small cluster

Run the host commands from the repository root, which contains `Vagrantfile`.

```bash
ansible-galaxy collection install community.general kubernetes.core
```

If you cloned from a remote, hydrate the LFS files before provisioning. For a local checkout that already has the payloads, go straight to the cache check.

```bash
git lfs install --local
git lfs pull
```

Verify the APK cache, start the systemd libvirt sockets, and bring up the load balancer, primary control plane and first worker in order.

```bash
run/cache-apk --check
run/services
export VAGRANT_DEFAULT_PROVIDER=libvirt
run/provision lb0
run/provision cp0
run/provision wo0
```

`run/services` may ask for `sudo`. `run/provision` registers the versioned local Alpine box if needed, then runs the VM setup and the playbooks for that node. Use this helper for the first boot of each new VM so the node also receives its Kubernetes role.

### Verify the cluster

Use the matching `kubectl` on `cp0` from the host. Give the system Pods time to initialize. `cp0` and `wo0` should reach `Ready`, and CoreDNS should be running on `cp0`.

```bash
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide'
vagrant ssh cp0 -c 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -n kube-system -o wide'
```

With a compatible host `kubectl`, you can also use the checked-in kubeconfig. Provisioning `cp0` generates the certificates it references in `.certs/`. This connection goes through HAProxy at `10.1.2.10:9443`.

```bash
kubectl --kubeconfig .kubeconfig get --raw=/readyz
kubectl --kubeconfig .kubeconfig get nodes
```

## Practice deployment recovery offline

Connect to `cp0` with `vagrant ssh cp0`, then run the commands below in the guest. The `pause` image is already imported on the worker, so this exercise needs no registry access.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl create namespace practice
kubectl -n practice create deployment hold --image=registry.k8s.io/pause:3.10.2
kubectl -n practice scale deployment hold --replicas=2
kubectl -n practice rollout status deployment/hold --timeout=120s
kubectl -n practice get pods -o wide
kubectl -n practice delete pods -l app=hold
kubectl -n practice get pods -w
```

Compare the Pod names before and after deletion. The Deployment replaces the deleted Pods and restores the replica count. Press `Ctrl+C` once the replacements are `Running`, then run `kubectl delete namespace practice` and `exit`.

## Expand the range

| Guests | Role | Private addresses |
| --- | --- | --- |
| `lb0` | HAProxy API load balancer and Unbound DNS forwarder | `10.1.2.10` |
| `cp0`–`cp2` | kubeadm control planes with stacked etcd and one CoreDNS Pod per node | `10.1.2.20`–`10.1.2.22` |
| `wo0`–`wo5` | Workers for scheduling and cross-node networking exercises | `10.1.2.30`–`10.1.2.35` |
| `nf0` | NFS server exporting `/export` for static persistent volumes | `10.1.2.40` |

On the host, add replicas one at a time after `cp0` is healthy. You can add a second worker and NFS independently.

```bash
run/provision cp1
run/provision cp2
run/provision wo1
run/provision nf0
```

For a fresh full deployment, run `run/provision` without a hostname. It brings up every VM, adding control planes sequentially and provisioning workers and NFS in parallel. The current network layout has been tested with `lb0`, all three control planes, and `wo0` and `wo1`. The NFS persistence test used the earlier network layout. The four other workers and NFS still need a run on the current layout. The API gateway is a single VM, and host-side cluster DNS routes through `cp0`.

## Stop, resume and reset

For the three-VM starter, halt and resume the same guests to keep their disks and cluster state. Include any guests you added when operating on a larger range.

```bash
vagrant halt wo0 cp0 lb0
run/services
vagrant up --provider=libvirt --no-provision lb0 cp0 wo0
```

Use `vagrant destroy wo0 cp0 lb0` when you want a fresh start. It asks for confirmation before deleting the named VM disks and their data. Destroying `nf0` also deletes the data stored in its export. Rebuild with the provisioning commands above.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Missing box, APKs or image archives | Run `git lfs pull` for a remote checkout, `git lfs fsck HEAD`, and `run/cache-apk --check`. |
| Vagrant cannot start or reach the guests | Run `run/services status` and `vagrant status`. Check access to libvirt and `/dev/kvm`, the OVMF file, and firewall rules on the libvirt bridge. |
| `cp0` fails to initialize | Read `run/log cat -h cp0 -n kubeadm` or follow it with `run/log watch -h cp0 -n kubeadm`. |
| A node or system Pod stays unready | Check nodes and Pods with the commands above. Inspect guest logs with `run/log cat -h wo0 -n kubelet` or list the local container images with `vagrant ssh wo0 -c 'doas crictl images'`. |
| A practice workload reports `ImagePullBackOff` | The worker cache contains only pause and kube-proxy. Use a cached image, import the workload image on its node, or give the guest access to its registry. |

Shared settings and cache lockfiles live in [config/](config/). See [lab notes](docs/lab-notes.md) for playbooks, network and DNS design, storage permissions, asset refresh commands and past validation. See the [Alpine box recipe](build/alpine/README.md) to rebuild the base box.
