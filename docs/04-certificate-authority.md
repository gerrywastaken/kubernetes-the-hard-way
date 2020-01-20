# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using the popular openssl tool, then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.

# Where to do these?

We need a machine with `openssl` on it and you should be able to copy the generated files to the provisioned VMs. Or just do these from one of the master nodes.

In our case we do it on the master-1 node, as we have set it up to be the administrative client.

# Stop all the copy and paste madness

By default `openssl` commands involve a lot of cruft that gets in the way if you are trying to learn about kubernetes. You may have noticed this if you tried looking at other forks of this tutorial where there are a lot of commands but most of them are just copy and pasted with small relevant changes hiden somewhere in the middle.

However the process is essentiall the same in very case:
1. Generate a certificate `.crt`
2. Generate a certifcate signing request `.csr`
3. Use the `.csr` to generate a private key `.key`

For this reason we are instead going to use an `openssl` wrapper that allows to see the parts that are relevant to learning kubernetes. However you should still look at the `openssl` commands that are being run under the hood via `-v` (verbose) flag.

https://github.com/gerrywastaken/gencert


## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Create a CA certificate, then generate a Certificate Signing Request and use it to create a private key:

```
gencert ca /CN=KUBERNETES-CA
```

Reference : https://kubernetes.io/docs/concepts/cluster-administration/certificates/#openssl

The ca.crt is the Kubernetes Certificate Authority certificate and ca.key is the Kubernetes Certificate Authority private key.
You will use the ca.crt file in many places, so it will be copied to many places.
The ca.key is used by the CA for signing certificates. And it should be securely stored. In this case our master node(s) is our CA server as well, so we will store it on master node(s). There is not need to copy this file to elsewhere.

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Kubelet Client Certificates

We are going to skip certificate configuration for Worker Nodes for now. We will deal with them when we configure the workers.
For now let's just focus on the control plane components.

### Simple components

Generate the `kube-controller-manager`, `kube-proxy`, `kube-scheduler`, `service-account` client certificate and private key:

```
gencert admin /CN=admin/O=system:masters
gencert kube-controller-manager /CN=system:kube-controller-manager
gencert kube-proxy /CN=system:kube-proxy
gencert kube-scheduler /CN=system:kube-scheduler
gencert service-account /CN=service-accounts
```

Note that the admin user is part of the **system:masters** group. This is how we are able to perform any administrative operations on Kubernetes cluster using kubectl utility.

The admin.crt and admin.key file gives you administrative access. We will configure these to be used with the kubectl tool to perform administrative functions on kubernetes.

The Kubernetes Controller Manager leverages service-account key pair to generate and sign service account tokens as describe in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

### kube-apiserver

The kube-apiserver certificate needs to be valid for all IPs and domains that can be used to reach the kube-apiserver. For example, the master servers IP addresses, the load balancers IP address, the kube-api service IP address and domains.

The `openssl` command cannot take alternate names as command line parameter. It wants you to generate bloated `conf` :cry:. However `gencert` does support this and automatically generates this file. If you are curious just look for the `.conf` file, but really it's a lot of cruft just to specify alternative names which is all we really care about. You can compare with the `etcd-server.conf` (generated below). All that's changing the [alt_names] section at the bottom of the file.

```
gencert kube-apiserver /CN=kube-apiserver \
  --dns kubernetes \
  --dns kubernetes.default \
  --dns kubernetes.default.svc \
  --dns kubernetes.default.svc.cluster.local \
  --ip 10.96.0.1 \
  --ip 192.168.5.11 \
  --ip 192.168.5.12 \
  --ip 192.168.5.30 \
  --ip 127.0.0.1
```

### etcd

The ETCD cert needs to be valid for all ETCD server IPs that are in the cluster

```
gencert etcd-server /CN=etcd-server \
  --ip 192.168.5.11 \
  --ip 192.168.5.12 \
  --ip 127.0.0.1
```

## Distribute the Certificates

Copy the appropriate certificates and private keys to each controller instance:

```
for instance in master-1 master-2; do
  scp ca.crt ca.key kube-apiserver.key kube-apiserver.crt \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    ${instance}:~/
done
```

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab. These certificates will be embedded into the client authentication configuration files. We will then copy those configuration files to the other master nodes.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
