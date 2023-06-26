# Istio Development Environment in Rootless Podman

This `Dockerfile.ubi9` creates a [Red Hat Universal Base Image (UBI)](https://catalog.redhat.com/software/base-images) for developing on Istio.

## Setup an Alternative Container Engine Podman from a MacOS host

[Podman Installation](https://podman.io/docs/installation)

To install `podman` engine from a MacOS host, run:

```bash
$ brew install podman;
$ podman machine init -v "${HOME}":"${HOME}";
$ podman machine start;
$ podman info;
```

To expose the podman.sock on macOS, run:

```bash
$ podman system connection list
Name                         URI                                                         Identity                                Default
podman-machine-default       ssh://core@127.0.0.1:[SSH_PORT]/run/user/[UID]/podman/podman.sock  /Users/[USER]/.ssh/podman-machine-default  true
podman-machine-default-root  ssh://root@127.0.0.1:[SSH_PORT]/run/podman/podman.sock           /Users/[USER]/.ssh/podman-machine-default  false
# replace the following 501 with your UID above.
# replace the port 50865 with the SSH_PORT above.
$ ssh -fnNT -L/tmp/podman.sock:/run/user/501/podman/podman.sock -i ~/.ssh/podman-machine-default ssh://core@localhost:50865 -o StreamLocalBindUnlink=yes
$ export DOCKER_HOST='unix:///tmp//podman.sock'
```

error : --cache-from: repository must contain neither a tag nor digest

$ DOCKER_SOCKET_MOUNT="-v /var/run/user/501/podman/podman.sock:/var/run/docker.sock" HUB=localhost CONTAINER_CLI=podman BUILD_WITH_CONTAINER=0 make containers-test 

## Image Configuration

- The base Istio development tools and the following additional tools are installed, with Bash completion configured:
    - [Docker CLI](https://docs.docker.com/engine/reference/commandline/cli/)
    - [Google Cloud SDK (gcloud)](https://cloud.google.com/sdk/gcloud/)
    - [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)
    - [Kubernetes IN Docker (KIND)](https://github.com/kubernetes-sigs/kind)
    - [Helm](https://helm.sh/)
- A user with the same name as the local host user is created. That user has full sudo rights without password.
- The following volumes are mounted from the host into the container to share development files and configuration:
    - Go directory: `$(GOPATH)` → `/home/$(USER)/go`
    - Podman socket, to access Podman from within the container:
      - [EXPERIMENTAL] In rootless mode container `/var/run/docker.sock`
        → podman machine VM `$(XDG_RUNTIME_DIR)/podman/podman.sock` i.e. `/var/run/user/${shell id -u}/podman/podman.sock`
        → host `${HOME}/.local/share/containers/podman/machine/qemu/podman.sock`
      - In rootful mode container `/var/run/docker.sock` → podman machine VM `/run/podman/podman.sock`
- The working directory is `/home/$user/go/src/istio.io/istio`.

## Creating The Container

To create your dev container, run:

```bash
$ make dev-shell-ubi9-podman-mac BUILD_WITH_CONTAINER=0
[USER@8c1521fefe4b ~] exit
```

The first time this target it run, a Docker image named `istio/dev:USER` is created, where USER is your local username.
Any subsequent run won't rebuild the image unless the `Dockerfile.ubi9` is modified.

The first time this target is run, a container named `istio-dev` is run with this image, and an interactive shell is executed in the container.
Any subsequent run won't restart the container and will only start an additional interactive shell.

## Kubernetes Cluster Creation Using KIND

In order to run KIND using rootless Podman, you need to have spinned up a virtual machine from `podman machine`. It spawns a virtual machine using `qemu` and connects Podman client to the given machine.

Currently, it's an experimental option for KIND to run in a rootless Podman container.
Meanwhile, you can start creating a KIND cluster in the virtual machine created by `podman machine`.

To copy KIND and kubectl from your dev container to the virtual machine, run:

```bash
$ podman machine ssh;
[core@localhost ~] podman cp istio-dev:/usr/bin/kind ./kind
[core@localhost ~] sudo mv ./kind /usr/local/bin/kind

[core@localhost ~] podman cp istio-dev:/usr/bin/kubectl ./kubectl
[core@localhost ~] sudo mv ./kubectl /usr/local/bin/kubectl
```

A Kubernetes can be created using KIND. For instance, to create a cluster named `blah` with 2 workers, run the following command within the container:

```bash
[core@localhost ~] export CLUSTER_NAME="blah"
[core@localhost ~] kind create cluster --name="$CLUSTER_NAME" --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
kubeadmConfigPatches:
  - |
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: ClusterConfiguration
    metadata:
      name: config
    apiServer:
      extraArgs:
        "service-account-issuer": "kubernetes.default.svc"
        "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
EOF

[core@localhost ~] chmod o+rw .kube/config
[core@localhost ~] podman network connect kind istio-dev
```

KIND was originally intended to run from the host, so KIND rewrites the kubeconfig to redirect the port to kubeadmin.
This rewriting must be undone to allow connecting directly from within the container:

Check that you can access the cluster:

```bash
[USER@8c1521fefe4b ~] kubectl get nodes
```

## Build Istio and Run Tests in Container

```bash
cd ~/go/src/istio.io/istio
```

To build Istio:

```bash
make build BUILD_WITH_CONTAINER=0
```

To run unit tests:

```bash
make test BUILD_WITH_CONTAINER=0
```

When rebuilding, you may find that istio cannot find the go path,
one simple approach is to add the go binary file to system bin/

```bash
cp /usr/local/go/bin/go /bin
```

## Removing The Container

```bash
$ podman stop istio-dev
$ podman rm istio-dev
```

## Troubleshooting

1. docker.io/istio/dev image access denied

```
Resolving "istio/dev" using unqualified-search registries (/etc/containers/registries.conf.d/999-podman-machine.conf)
Trying to pull docker.io/istio/dev:[USER]...
Error: initializing source docker://istio/dev:[USER]: reading manifest [USER] in docker.io/istio/dev: requested access to the resource is denied
```

The above error was caused after rebooting a podman machine instance. The previous built image doesn't allow access from a new podman machine instance uid.

You can run the clean target and then rebuild the image.

```bash
$ make clean-dev-shell-ubi9-podman-mac BUILD_WITH_CONTAINER=0
```