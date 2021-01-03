# Kubernetes with IPv6 only (on Ubuntu)

**DRAFT**

Configure a kubernetes cluster with IPv6 only.

IPv6 only infrastructure deployments allow for simpler management and maintenance than dual-stack, with IPv4 access provided via a front end reverse proxy or content distribution network (CDN).

## Install Kubernetes with IPv6 only on Ubuntu 20.04

NOTE: You need to have DNS64 + NAT64 available to your IPv6 only server, as the installation uses several resources (Docker registry, Github) that are still IPv4 only.

There are several main steps required:

1. Set up a container runtime
1. Install Kubernetes components
1. Set up the Kubernetes control plane
1. Deploy a Container Network Interface (CNI)
1. Configure routing
1. Add management components


## Set up a container runtime (Docker CE)

I found Docker CE the easiest to set up, following the instructions at https://kubernetes.io/docs/setup/production-environment/container-runtimes/ although you could try an alternative.

First, enable IPv6 forwarding and IP tables; although not mentioned in the Docker CE install they are used later.

```bash
# Setup required sysctl IPv6 params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.ipv6.conf.all.forwarding        = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

This installs a specific version of Docker CE:

```bash
# Prerequisite packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2

# Add Docker's official GPG key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo apt-key --keyring /etc/apt/trusted.gpg.d/docker.gpg add -

# Add the Docker apt repository:
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

# Install Docker CE
sudo apt-get update
sudo apt-get install -y \
  containerd.io=1.2.13-2 \
  docker-ce=5:19.03.11~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.11~3-0~ubuntu-$(lsb_release -cs)

# Set up the Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Create /etc/systemd/system/docker.service.d
sudo mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# Set docker to start on boot
sudo systemctl enable docker
```


## Installing kubeadm, kubelet and kubectl

Follow the instructions at https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

The iptables configuration is already done above, and both overlay and br_filter should be installed with docker.

The package installation excludes them from automatic updates:

```bash
# Prerequisite packages
sudo apt-get update
sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Set up the Kubernetes control plane

### Prepare a configuration file

We need to use a `kubeadm init` configuration file for IPv6, as per the notes at https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file

You can get a copy of the default configuration, however a lot of the options can be deleted from the file as the default is fine. However we do need to configure the options needed for IPv6 for each component in the Kubernetes stack. Details for this taken from the excellent talk by André Martins, https://www.youtube.com/watch?v=q0J722PCgvY

| Component | Details | IPv6 Options with example |
| --------- | ------- | ------------------------- |
| etcd | 53 CLI options, <br> 5 relevant for IPv6 | --advertise-client-urls https://[fd00::0b]:2379 <br> --initial-advertise-peer-urls https://[fd00::0b]:2380 <br> --initial-cluster master=https://[fd00::0b]:2380 <br> --listen-client-urls https://[fd00::0b]:2379 <br> --listen-peer-urls https://[fd00::0b]:2380 <br> |
| kube-apiserver | 120 CLI options, <br> 5 relevant for IPv6 | --advertise-address=fd00::0b <br> --bind-address '::' <br> --etcd-servers=https://[fd00::0b]:2379 <br> --service-cluster-ip-range=fd03::/112 <br> --insecure-bind-address <br> |
| kube-scheduler | 32 CLI options, <br> 3 relevant for IPv6 | --bind-address '::' <br> --kubeconfig <br> --master <br> |
| kube-controller-manager | 87 CLI options, <br> 5 relevant for IPv6 | --allocate-node-cidrs=true <br> --bind-address '::' <br> --cluster-cidr=fd02::/80 <br> --node-cidr-mask-size 96 <br> --service-cluster-ip-range=fd03::/112 <br> |
| kubelet | 160 options, <br> 3 relevant for IPv6 | --bind-address '::' <br> --cluster-dns=fd03::a <br> --node-ip=fd00:0b <br> |

To set the options you need to use the address of your server and plan the ranges you will use for Services and Pods, and how they will be allocated to Nodes.

You can use either public assigned addresses (recommended if available) or Unique Local Addresses (ULA, the equivalent of private address ranges, will require custom routing or NAT).

The above examples use very short ULAs (e.g. `fd00::0b`) for demonstration purposes; a standard ULA will be a random `fd00::` range, e.g. `fd12:3456:789a::/48`, of which you might use one or more subnets, e.g. `fd12:3456:789a:1::/64`.

For public addresses if you have been assigned a single `/64`, e.g. `2001:db8:1234:5678::/64`, you can use a plan like the following. This allows up to 65,000 Services, with 65,000 Nodes, and 250 Pods on each Node, and only uses a small fraction of the `/64` available.

| Plan Range | Description |
| ----- | ----------- |
| `2001:db8:1234:5678::1`          | Main address of the master node |
| `2001:db8:1234:5678:8:3::/112`   | Service CIDR. Range allocated for Services, `/112` allows 65,000 Services. Maximum size in Kubernetes is `/108`. |
| `2001:db8:1234:5678:8:2::/104`   | Node CIDR. Range used to allocate subnets to Nodes. Maximum difference in Kubernates between Node range and block size (below) is 16 (Calico allows any size). |
| `2001:db8:1234:5678:8:2:xx:xx00/120` | Allocate a `/120` CIDR from the Node CIDR range to each node; each Pod gets an address from the range. This allows 65,000 Nodes, with 250 Pods on each. Calico block range is 116-128 with default `/122` for 64 Pods per Node (Kubernetes allows larger subnets). |

We want to customise the control plane with the IPv6 configuration settings from above, and also configure the cgroup driver. https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/

This gives us the following configuration file template, where you can insert your address ranges and host name (see below for how to download and do this):

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 2001:db8:1234:5678::1
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: node0001
  kubeletExtraArgs:
    cluster-dns: 2001:db8:1234:5678:8:3:0:a
    node-ip: 2001:db8:1234:5678::1
---
apiServer:
  extraArgs:
    advertise-address: 2001:db8:1234:5678::1
    bind-address: '::'
    etcd-servers: https://[2001:db8:1234:5678::1]:2379
    service-cluster-ip-range: 2001:db8:1234:5678:8:3::/112
apiVersion: kubeadm.k8s.io/v1beta2
controllerManager:
  extraArgs:
    allocate-node-cidrs: 'true'
    bind-address: '::'
    cluster-cidr: 2001:db8:1234:5678:8:2::/104
    node-cidr-mask-size: '120'
    service-cluster-ip-range: 2001:db8:1234:5678:8:3::/112
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      advertise-client-urls: https://[2001:db8:1234:5678::1]:2379
      initial-advertise-peer-urls: https://[2001:db8:1234:5678::1]:2380
      initial-cluster: __HOSTNAME__=https://[2001:db8:1234:5678::1]:2380
      listen-client-urls: https://[2001:db8:1234:5678::1]:2379
      listen-peer-urls: https://[2001:db8:1234:5678::1]:2380
kind: ClusterConfiguration
networking:
  serviceSubnet: 2001:db8:1234:5678:8:3::/112
scheduler:
  extraArgs:
    bind-address: '::'
---
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
clusterDNS:
- 2001:db8:1234:5678:8:3:0:a
healthzBindAddress: ::1
kind: KubeletConfiguration
```

You can also download the template and do a replace for your host name (node name), allocated `/64`, and master node address. Use the following, but insert your /64 for `xxxx` and your master node for `yyyy`. Include the last segment of your `/64` when replacing the master node to avoid conflict with the `::1` used for `healthzBindAddress`.

Note that the `initial-cluster` parameter (the value replaced) needs to match the generated node name, which defaults to the host name, so we insert the host name into the config file (you can check this in the dry run ClusterStatus apiEndpoints, or in the mark-control-plane section, in case the node name differs).

```bash
curl -O https://raw.githubusercontent.com/sgryphon/kubernetes-ipv6/main/init-config-ipv6.yaml
sed -i "s/__HOSTNAME__/$HOSTNAME/g" init-config-ipv6.yaml

sed -i 's/2001:db8:1234:5678/xxxx:xxxx:xxxx:xxxx/g' init-config-ipv6.yaml
sed -i 's/xxxx::1/xxxx::yyyy/g' init-config-ipv6.yaml
```

We can do a dry run to check the config file and expected output:

```bash
sudo kubeadm init --config=init-config-ipv6.yaml --dry-run | more
```

### Initialise the cluster control plane

Once the configuration is ready, use it to initialise the Kubernetes control plane:

```bash
sudo kubeadm init --config=init-config-ipv6.yaml
.
.
.
"Your Kubernetes control-plane has initialized successfully!"
```

**IMPORTANT:** Take note of the token to use for joining worker nodes.

You need to follow the instructions to get kubectl to work as a non-root user, either creating the `.kube/config` file or exporting `admin.conf`.

For a single node cluster you also want to be able to schedule Pods on the master, so need to un-taint it.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl taint nodes --all node-role.kubernetes.io/master-
```

At this point you can checking the status. The etcd, apiserver, controller manager, and scheduler should be running, although CoreDNS requires a networking add on before it will start, as mentioned in the init output.

```bash
kubectl get all --all-namespaces
```

### Aside: Manual modifications to the kubeadm config file

You can get a copy of the default configuration, including Kubelet, from kubeadm:

```bash
kubeadm config print init-defaults --component-configs KubeletConfiguration \
  | tee init-defaults-config.yaml
```

The IPv6 changes required are described above, and you can make any other changes you need for your cluster configuration.

A good approach when making changes is to make a small change at a time and then do a dry-run to ensure the configuration is still valid.

## Deploy a Container Network Interface (CNI)

The `kubeadm init` output instructs that you must also deploy a pod network, a Container Network Interface (CNI), to get things working, and the output of init provides a link to a list. https://kubernetes.io/docs/concepts/cluster-administration/addons/

Base Kubernetes is tested with Calico, and it also supports IPv6, with instructions for the needed setup changes. https://docs.projectcalico.org/networking/ipv6

A modified configuration file with the IPv6 changes already applied is available for download. (Alternatively you can make the modifications yourself, see below.)

```bash
curl -O https://raw.githubusercontent.com/sgryphon/kubernetes-ipv6/main/calico-ipv6.yaml

sed -i 's/2001:db8:1234:5678/xxxx:xxxx:xxxx:xxxx/g' calico-ipv6.yaml
```

Then apply the configuration, which will set up all the custom object definitions and the calico service:

```bash
kubectl apply -f calico-ipv6.yaml
```

After this you can check everything is running. If successful, then there should be some calico pods, and the coredns pods should now be running.

```bash
kubectl get all --all-namespaces
```

### Installing calicoctl

For future administration of calico, you can also install calicoctl. https://docs.projectcalico.org/getting-started/clis/calicoctl/install

For example, to install calicoctl as a Pod:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml

# Save an alias in bash config, and then load it
cat <<EOF | tee -a ~/.bash_aliases
alias calicoctl="kubectl exec -i -n kube-system calicoctl -- /calicoctl"
EOF
. ~/.bash_aliases

calicoctl get profiles -o wide
```

### Aside: Manual modifications to the calico install file

An already modified version is available for download as above, otherwise you can make the changes yourself (e.g. if calico updates the main manifest).

Following the main install guide first, download the config file for "Install Calico with Kubernetes API datastore, 50 nodes or less".

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

Edit as per the IPv6 instructions and disable encapsulation (not needed for IPv6).

For step 3, modify the section at the top for "# Source: calico/templates/calico-config.yaml" in data > cni_network_config, and set assign_ipv6.

```json
 "ipam": {
        "type": "calico-ipam",
        "assign_ipv4": "false",
        "assign_ipv6": "true"
    },
```

For steps 4, 5, and the optional bit to turn off IPv4, you need to go past all the CustomResourceDefinition sections and find the section "# Source: calico/templates/calico-node.yaml".

Scroll down a bit more to find spec > template > spec > containers > name: calico-node > env, and set the environment variables. Enable IPv6 support, disable IPv4, set the router ID method. Also disable IPIP, as encapsulation is not needed with IPv6.

```yaml
- name: IP6
  value: "autodetect"
- name: FELIX_IPV6SUPPORT
  value: "true"
- name: IP
  value: "none"
- name: CALICO_ROUTER_ID
  value: "hash"
- name: CALICO_IPV4POOL_IPIP
  value: "Never"
- name: CALICO_IPV6POOL_CIDR
  value: "2001:db8:1234:5678:8:2::/104"
- name: CALICO_IPV6POOL_BLOCK_SIZE
  value: "120"
```

The documents say that Calico will pick up the IP pool from kubeadm, but it didn't work for me and I got the default CIDR `fda0:95bd:f195::/48`(a random `/48`) with the default block size `/122` (which is 64 pods per node), as per https://docs.projectcalico.org/reference/node/configuration


## Configure routing

Once the networking is configured you should have all the cluster components talking to each other.

You can test this by installing a Pod running a basic terminal instance (the Kubernetes examples use `busybox`, but also `dnsutils` and full `alpine`).

```bash
kubectl apply -f https://k8s.io/examples/admin/dns/busybox.yaml
kubectl exec -ti busybox -- nslookup kubernetes.default
```

You can check communication (ping) between the host and Pods, and between Pods:

```bash
kubectl get pods --all-namespaces -o \
  custom-columns='NAMESPACE:metadata.namespace','NAME:metadata.name','STATUS:status.phase','IP:status.podIP' 
ping -c 2 `kubectl get pods busybox -o=custom-columns='IP:status.podIP' --no-headers`
kubectl exec -ti busybox -- ping -c 2 `hostname -i`
kubectl exec -ti busybox -- ping -c 2 \
  `kubectl get pods -n kube-system -o=custom-columns='NAME:metadata.name','IP:status.podIP' | grep -m 1 coredns | awk '{print $2}'`
```

However communication with the rest of the Internet won't work unless incoming routes have been configured.

Your hosting provider may route the entire /64 to your machine, or may simply make the range available to you and be expecting your server to respond to Neighbor Discovery Protocol (NDP) solicitation messages with advertisement of the addresses it can handle.

Simply adding the address to the host would advertise it, but also handle the packets directly (INPUT), rather than forwarding to the cluster. To advertise the cluster addresses in response to solicitation you need to use a Neighbor Discovery Protocol proxy.

Linux does have some limited ability to proxy NDP, but it is limited to individual addresses; a better solution is to use `ndppd`, the Neighbor Discovery Protocol Proxy Daemon, which supports proxying of entire ranges of addresses, https://github.com/DanielAdolfsson/ndppd

This will respond to neighbor solicitation requests with advertisement of the Pod and Service addresses, allowing the upstream router to know to send those packets to the host. Use your addresses ranges in the configuration, advertising the Pod `/104` range automatically based on known routes, which includes individual pods (from `ip -6 route show`), and the entire `/112` Service range (as a static range).

```bash
# Install
sudo apt update
sudo apt install -y ndppd

# Create config
cat <<EOF | sudo tee /etc/ndppd.conf
proxy eth0 {
    rule 2001:db8:1234:5678:8:2::/104 {
        auto
    }
    rule 2001:db8:1234:5678:8:3::/112 {
        static
    }
}
EOF

# Restart
sudo systemctl daemon-reload
sudo systemctl restart ndppd

# Set to run on boot
sudo systemctl enable ndppd
```

Once running you can test that communication works between Pods and the Internet:

```bash
kubectl exec -ti busybox -- ping -c 2 2001:4860:4860::8888
```

### Routing alternatives

#### Basic NDP proxy

Linux directly supports NDP proxying for individual addresses, although the functionality is considered deprecated:

```
sysctl -w net.ipv6.conf.eth0.proxy_ndp=1
ip -6 neigh add proxy 2001:db8:1234:5678:8:2:42:1710 dev eth0
```

#### BGP

Calico support BGP, and runs full mesh BGP between nodes in the cluster. See https://docs.projectcalico.org/networking/bgp

For dynamic communication of routes to upstream routers, you can peer Calico using BGP (Note: based on documentation; not tested). Replace the values with those for your hosting provider, if they allow it.

```bash
cat <<EOF | tee calico-bgppeer.yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: global-peer
spec:
  peerIP: 2001:db1:80::3
  asNumber: 64123
EOF

calicoctl create -f calico-bgppeer.yaml

calicoctl node status
```

## Troubleshooting

If the initial installation using kubeadm fails, you may need to look at the container runtime to make sure there are no problems (I had issues when the initial cluster name didn't match the host name).

```bash
docker ps -a
docker logs CONTAINERID
```

If `kubectl` is working (a partially working cluster), you can also examine the different elements, e.g.

```bash
kubectl get all --all-namespaces
kubectl describe service kube-dns -n kube-system
kubectl describe pod coredns-74ff55c5b-hj2gm -n kube-system
```

You can look at the logs of individual components:

```bash
kubectl logs --namespace=kube-system kube-controller-manager-hostname.com
```

For troubleshooting from within the cluster, deploy a Pod running terminal software (see above for details).

To troubleshoot network routing issues you can config ip6tables tracing for ping messages in combination with tcpdump to check connections (see below for cleanup). For information on ip6tables debugging see https://backreference.org/2010/06/11/iptables-debugging/

```bash
# Enable tracing for ping (echo) messages
ip6tables -t raw -A PREROUTING -p icmpv6 --icmpv6-type echo-request -j TRACE
ip6tables -t raw -A PREROUTING -p icmpv6 --icmpv6-type echo-reply -j TRACE
ip6tables -t raw -A OUTPUT -p icmpv6 --icmpv6-type echo-request -j TRACE
ip6tables -t raw -A OUTPUT -p icmpv6 --icmpv6-type echo-reply -j TRACE
```

On an external server (e.g. home machine if you have IPv6), run `tcpdump` with a filter for the cluster addresses:

```bash
sudo tcpdump 'ip6 and icmp6 and net 2001:db8:1234:5678::/64'
```

Then do a ping from the busybox Pod to the external server (replace the address with your external server, running tcpdump, address):

```bash
kubectl exec -ti busybox -- ping -c 1 2001:db8:9abc:def0::1001
```

To examine the ip6tables tracing, view the kernel log on the Kubernetes server:

```bash
tail -50 /var/log/kern.log
```

There will be a lot of rows of output, one for each ip6tables rule that the packets passed through (which is why we only send one ping).

The kernel log should show the echo-request packets (`PROTO=ICMPv6 TYPE=128`) being forwarded from the source address of the Pod to the destination address of the external server.

The `tcpdump` on the external server should show when the packet was received and the reply sent.

The log should then show the echo-reply packets (`PROTO=ICMPv6 TYPE=128`) being forwarded from the source address of the external server to the destination address of the Pod.

This should allow you to determine where the failure is. For example, before configuring routing using `ndppd` (see above), the network was not working, but I could see the outgoing forwarded packets, the reply on the external server, but not being received on the Kubernetes server (for forwarding to the Pod) -- the issue was the server was not advertising the address so the upstream router didn't know where to send the replies.

For more details on Kubernetes troubleshooting, see https://kubernetes.io/docs/tasks/debug-application-cluster/

### ip6tables tracing cleanup

Tracing generates a lot of log messages, so make sure to clean up afterwards. Entries are deleted by row number, so check the values are correct before deleting.

```
ip6tables -t raw -L PREROUTING --line-numbers
# Check the line numbers are correct before deleted; delete in reverse order
ip6tables -t raw -D PREROUTING 3
ip6tables -t raw -D PREROUTING 2

ip6tables -t raw -L OUTPUT --line-numbers
# Check the line numbers are correct before deleted; delete in reverse order
ip6tables -t raw -D OUTPUT 3
ip6tables -t raw -D OUTPUT 2
```

## Add management components

At this point you should be able reach components such a the API service via a browser `https://[2001:db8:1234:5678::1]:6443/`, although without permissions it will only return an error, although at least that shows it is up and running.


### Helm (package manager)

Helm will also be useful for installing other components. https://helm.sh/docs/intro/install/

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Ingress controller

NGINX is a popular web server front end / reverse proxy that can be used as an ingress controller. https://kubernetes.github.io/ingress-nginx/deploy/#using-helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

NGINX ingress will install with mapped ports (type LoadBalancer). You can check the mapped ports allocated to the service:

```bash
kubectl get service --all-namespaces
```

You can then use a browser to check the mapped HTTPS (443) port, e.g. `https://[2001:db8:1234:5678::1]:31429`, although you will just get a nginx 404 page until you have services to expose.

With IPv6, the deployment can be checked by directly accessing the address of the service:

```
kubectl get services
curl -k https://[`kubectl get service ingress-nginx-controller --no-headers | awk '{print $3}'`]
```

If routing is set up correctly, you should also be able to access the Service address directly from the Internet, e.g. `https://[2001:db8:1234:5678:8:3:0:d3ef]/`, with the same nginx 404 page.


### Certificate manager (automatic HTTPS)

HTTPS certificates are now readily available for free, and all communication these days should be HTTPS. A certificate manager is needed to supply certificates to the ingress controller, such as cert-manager for Let's Encrypt. https://cert-manager.io/docs/installation/kubernetes/

Note that this also means that all sites will require a domain name, either supplied by your hosting provider or your own domain name records.

Install the certificate manager:

```bash
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true
```

To configure the certificate manager we need to create an issuer, but should use the Let's Encrypt staging service first to ensure things are working. Make sure to replace the email with your email address.

```bash
cat <<EOF | sudo tee letsencrypt-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-staging
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

kubectl create -f letsencrypt-staging.yaml
```

You can check the cluster issue has been deployed correctly and the account registered:

```bash
kubectl describe clusterissuer letsencrypt-staging
```

There is also a cert-manager tutorial for securing nginx-ingress with ACME (Let's Encrypt), https://cert-manager.io/docs/tutorials/acme/ingress/.

### Dashboard

To manage the Kubernetes cluster install Dashboard, via Helm. https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```

With IPv6, the deployment can be checked by directly accessing the address of the service:

```
kubectl get services
curl -k https://[`kubectl get services | grep kubernetes-dashboard | awk '{print $3}'`]
```

To log in you will need to create a user, following the instructions at https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

```bash
cat <<EOF | tee admin-user-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
EOF

kubectl apply -f admin-user-account.yaml

cat <<EOF | tee admin-user-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: default
EOF

kubectl apply -f admin-user-binding.yaml
```

Then, to get a login token:

```bash
kubectl describe secret `kubectl get secret | grep admin-user | awk '{print $1}'`
```

## Putting it all together

Note that the initial setup uses a staging certificate from Let's Encrypt, as the production servers have tight quotas and if there is a problem you don't want to be mixing up configuration issues with quota limit issues.

To use HTTPS (which you should be) you will need to assign a domain name for the dashboard, e.g. `https://kube001-dashboard.example.com`

You can then configure an ingress specification, which cert-manager will use to get a certificate, allowing nginx to reverse proxy and serve the dashboard pages.

First get the IPv6 address of your ingress controller:

```bash
kubectl get service ingress-nginx-controller
```

Then set up a DNS host name for the dashboard pointing to the service address, and check it resolves (this example has the dashboard as a CNAME that points to ingress that has the AAAA record):

```bash
nslookup kube001-dashboard.example.com

Non-authoritative answer:
kube001-dashboard.example.com	canonical name = kube001-ingress.example.com.
Name:	kube001-ingress.example.com
Address: 2001:db8:1234:5678:8:3:0:d3ef
```

Create a staging ingress for the dashboard, with annotations to use the cert-manager issuer we created, and that the back end protocol is HTTPS. There are many other annotations you can use to configure the nginx ingress, https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/.

```bash
cat <<EOF | sudo tee dashboard-ingress-staging.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  name: dashboard-ingress
spec:
  rules:
  - host: kube001-dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port: 
              number: 443
  tls: 
  - hosts:
    - kube001-dashboard.example.com
    secretName: dashboard-ingress-cert
EOF

kubectl apply -f dashboard-ingress-staging.yaml
```

Checking the created ingress, it should have a notification from cert-manager that the certificate was correctly created:

```bash
kubectl describe ingress dashboard-ingress
```

If everything has worked correctly you should be able to visit `https://kube001-dashboard.example.com`, although there will be a certificate warning you need to bypass as it is only a Staging certificate at this point. Use the login token from above and you can access the kubernetes dashboard.

### Production certificate

Once you have the staging certificate from Let's Encrypt working, you can change the configuration over to use a production certificate.

Create a production issuer:

```bash
cat <<EOF | sudo tee letsencrypt-production.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-production
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

kubectl create -f letsencrypt-production.yaml

kubectl describe clusterissuer letsencrypt-production
```

Change the dashboard ingress from using the staging issuer to the production issuer:

```bash
cat <<EOF | sudo tee dashboard-ingress-production.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  name: dashboard-ingress
spec:
  rules:
  - host: kube001-dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port: 
              number: 443
  tls: 
  - hosts:
    - kube001-dashboard.example.com
    secretName: dashboard-ingress-cert
EOF

kubectl apply -f dashboard-ingress-production.yaml

kubectl describe ingress dashboard-ingress
```

Close and reopen the dashboard page, `https://kube001-dashboard.example.com`, and you should now have a valid working certificate.


