# Inception of Things

A Kubernetes infrastructure lab progressing from K3s virtual machines to a local K3d cluster with Argo CD. The bonus adds a self-hosted GitLab deployment backed by PostgreSQL and Redis.

## Project layout

| Directory | Contents |
| --- | --- |
| [`p1/`](p1/) | Two Debian VMs: a K3s server and an agent |
| [`p2/`](p2/) | One K3s VM with three web applications and host-based ingress |
| [`p3/`](p3/) | A K3d cluster, Argo CD, and an automatically synchronized application |
| [`bonus/`](bonus/) | GitLab, PostgreSQL, Redis, persistent storage, and an optional Argo CD overlay |

Run each part separately. Parts 1 and 2 reuse VM names and addresses; part 3 and the bonus use the same default K3d cluster name and host ports.

## Requirements

For parts 1 and 2, use a host with KVM/libvirt, Vagrant, the `vagrant-libvirt` provider, and networking that can reach `192.168.56.0/24`. Part 2 uses rsync for its shared folder. Both Vagrantfiles use the `generic/debian12` box.

For part 3 and the bonus, install Docker, K3d, kubectl with Kustomize support, and Make. The default cluster publishes host ports 80 and 443, so those ports must be free. Image pulls, the K3s installer, and the remote Argo CD manifests require internet access.

Provisioning downloads K3s without pinning a version. Argo CD uses the moving `stable` manifests, and GitLab uses a `latest` image. Record or pin versions when you need reproducible environments.

## Part 1: K3s server and agent

The Vagrantfile reads a shared cluster token from `p1/node-token`, which is ignored by Git. Create it before starting the VMs:

```sh
cd p1
umask 077
openssl rand -hex 32 > node-token
vagrant up --provider=libvirt
```

Keep the same token while using this cluster; regenerate it only when rebuilding both nodes together.

| VM | Role | Address | Configured resources |
| --- | --- | --- | --- |
| `zkotbiS` | K3s server | `192.168.56.110` | 1 CPU, 1 GiB RAM |
| `zkotbiSW` | K3s agent | `192.168.56.111` | 1 CPU, 1 GiB RAM |

The provisioning scripts select `eth1` for Flannel. If your provider uses another interface name, update both scripts before provisioning.

Check the nodes from the server VM:

```sh
vagrant ssh zkotbiS -c 'sudo kubectl get nodes -o wide'
```

From `p1/`, use `vagrant halt` to stop the VMs or `vagrant destroy` to remove them. Destroy the part 1 VMs before starting part 2, which reuses `zkotbiS` and `192.168.56.110`. Halting alone can leave conflicting libvirt VM names behind.

## Part 2: applications and ingress

From the repository root:

```sh
cd p2
vagrant up --provider=libvirt
vagrant ssh zkotbiS -c 'sudo kubectl get deployments,pods,services,ingress'
```

Provisioning installs K3s, waits for a ready node, and applies the Kustomize resources synced to `/vagrant`. The VM forwards guest port 80 to host port 8080.

| Host header | Destination | Replicas |
| --- | --- | --- |
| `app1.com` | `app1-service` | 1 |
| `app2.com` | `app2-service` | 3 |
| Any other host | `app3-service` | 1 |

After the pods are ready, test routing from the host without editing its hosts file:

```sh
curl -H 'Host: app1.com' http://127.0.0.1:8080/
curl -H 'Host: app2.com' http://127.0.0.1:8080/
curl -H 'Host: other.local' http://127.0.0.1:8080/
```

The deployments pull `zkotbi/app1:v1`, `zkotbi/app2:v1`, and `zkotbi/app3:v1`. Source Dockerfiles and pages are under `p2/apps/`; provisioning does not build or publish those images. If the referenced images are unavailable, publish your own and update the deployment image references.

After editing local manifests, run the following from `p2/` to sync them and apply the changes:

```sh
vagrant rsync
vagrant ssh zkotbiS -c 'sudo kubectl apply -k /vagrant'
```

The installer makes the VM's kubeconfig world-readable with `chmod 644`. Keep this lab isolated, or restrict that file's permissions and use `sudo kubectl`.

## Part 3: Argo CD and GitOps

Review [`p3/application.yaml`](p3/application.yaml) first. Its current source is the external repository `https://github.com/zkotbi/zkotbi-iot-p3.git`, branch `main`, path `app`. Point it at a repository you control if you want to change the deployed application. A private source also needs Argo CD repository credentials, which this project does not configure.

From the repository root:

```sh
make -C p3
kubectl get nodes
kubectl get applications -n argocd
kubectl get pods -n dev
```

Run Make without `-j`: the aggregate target lists provisioning steps but does not encode dependencies between them for parallel execution.

The Makefile creates `dev-cluster` with two agents, installs Argo CD, waits for its deployments, applies the application, and prints the initial admin credentials. Application synchronization is asynchronous; a successful Make run does not guarantee the application is healthy yet.

The application targets the `dev` namespace with automatic synchronization, pruning, self-healing, and namespace creation enabled. Review that policy before pointing it at your own manifests.

To open the Argo CD UI, run this in a separate terminal:

```sh
kubectl port-forward -n argocd svc/argocd-server 8081:80
```

Visit `http://localhost:8081` and sign in as `admin` using the password printed by Make. The overlay sets `server.insecure: "true"`, so this port-forward uses HTTP. An ingress for `argocd.worklet.dev` is also included, but hostname resolution and TLS are not configured by this repository.

`CLUSTER_NAME` can be overridden, for example `make -C p3 CLUSTER_NAME=iot-p3`. Keep the `argocd` namespace unless you also update the manifests: the Makefile's `NAMESPACE` variable alone does not rewrite their namespace declarations.

## Bonus: GitLab

[`bonus/gitlab/`](bonus/gitlab/) contains GitLab, PostgreSQL, Redis, services, an ingress for `gitlab.worklet.dev`, and persistent volume claims requesting 16 GiB in total. PostgreSQL explicitly uses the `local-path` storage class.

Before applying the manifests, replace the checked-in example passwords in `gitlab-config-map.yaml`, including the duplicate database and Redis values embedded in `GITLAB_OMNIBUS_CONFIG`. Those values are stored in a ConfigMap; move credentials to Secrets before using real accounts. Rotate them if they have been reused outside this lab.

For a standalone bonus cluster, after stopping any cluster that already binds host ports 80 and 443:

```sh
make -C bonus
kubectl get pods,pvc -n gitlab
kubectl logs -n gitlab deployment/gitlab
```

Alternatively, to install GitLab into an existing part 3 cluster, first confirm that kubectl points to the intended cluster, then run only:

```sh
make -C bonus install-gitlab
```

The bonus Makefile applies GitLab but does not wait for it to become ready. It also does not install the optional overlay in `bonus/argocd/` or create an Argo CD Application pointing to GitLab; the part 3 Application continues to use its configured GitHub source.

The GitLab configuration declares an HTTPS external URL, but its ingress has no TLS configuration. Complete the hostname, TLS, and GitLab Nginx configuration before relying on browser access. The image and database version combination has not been validated here.

## Cleanup

Run cleanup only for the environment you intend to remove:

```sh
# VM environments: run from p1/ or p2/
vagrant destroy

# K3d environments: run from the repository root
make -C p3 clean
# Or, for a standalone bonus cluster:
make -C bonus clean
```

K3d cleanup deletes the named cluster and its local cluster data. If you overrode `CLUSTER_NAME`, supply the same value to the cleanup target. The default part 3 and bonus cleanup targets both refer to `dev-cluster`.
