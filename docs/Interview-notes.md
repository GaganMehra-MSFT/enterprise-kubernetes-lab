Q: Why does every node need containerd or a container runtime?
A: Because every node that runs Pods requires a container runtime to pull images and start containers. kubelet manages Pods but delegates container creation to the container runtime.
