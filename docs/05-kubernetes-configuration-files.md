# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user. The reason for using Kubectl for this is so that the certificates are written to the file in a way that Kubernetes can understand.

Much like with generating certs it involves running the same set of commands for each component and replacing the same set of values, so to make it less repetitive there is a tool just like `gencert` called `genkubeconf`. It essentially just takes a $name, $ip and a $system_name and runs the following:

```bash
# Add cluster
kubectl config set-cluster kubernetes-the-hard-way \\
  --certificate-authority=ca.crt \\
  --embed-certs=true \\
  --server=https://$ip:6443 \\
  --kubeconfig=$name.kubeconfig

# Add user
kubectl config set-credentials $system_name \\
  --client-certificate=$name.crt \\
  --client-key=$name.key \\
  --embed-certs=true \\
  --kubeconfig=$name.kubeconfig

# Add default context
kubectl config set-context default \\
  --cluster=kubernetes-the-hard-way \\
  --user=$system_name \\
  --kubeconfig=$name.kubeconfig

# Set current context to default
kubectl config use-context default --kubeconfig=$name.kubeconfig
```

https://github.com/gerrywastaken/genkubeconf

### Kubernetes Public IP Address

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the load balancer will be used. In our case it is `192.168.5.30`

```bash
# For masters
local_ip='127.0.0.1'
genkubeconf kube-controller-manager $local_ip system:kube-controller-manager
genkubeconf kube-scheduler $local_ip system:kube-scheduler
genkubeconf admin $local_ip admin

# For workers
loadbalancer_ip=192.168.5.30
genkubeconf kube-proxy $loadbalancer_ip system:kube-proxy
```

And that's it. You should now have a .kubeconf file for each.

Reference docs:
* [kube-proxy ](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
* [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
* [kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
* [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

## Distribute the Kubernetes Configuration Files

Copy the appropriate `kube-proxy` kubeconfig files to each worker instance:

```
for instance in worker-1 worker-2; do
  scp kube-proxy.kubeconfig ${instance}:~/
done
```

Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```
for instance in master-1 master-2; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
