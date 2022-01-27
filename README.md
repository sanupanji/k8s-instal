# Contents
# 
# 1	Introduction ...............................................................................................................................
# 1.1	Document History................................................................................................................
# 1.2	Target Audience...................................................................................................................
# 
# 2	Architecture
# 
# 3	Prerequisites ......................................................................................................................................................
# 3.1	Gather Details....................................................................................................................................................
# 3.2	Check Network Connectivity ........................................................................................................................
# 3.3	Check and Open Ports.....................................................................................................................................
# 3.4	Set Hostname ....................................................................................................................................................
# 3.5	Disable Swap.....................................................................................................................................................
# 3.6	Enable and Load Kernel modules ...............................................................................................................
# 3.7	Add Kernel settings..........................................................................................................................................
# 
# 4	Pre-Installation Steps..............................................................................................................
# 3.1	Install containerd as runtime..............................................................................................
# 3.2	Install Kubeadm, kubelet and kubectl
# 3.3	Install helm client
# 3.4	Configure keepalived and haproxy 
# 
# 4	Installation Steps......................................................................................................................
# 4.1	Initialize cluster
# 4.2 	Join other master nodes
# 4.3 	Join worker nodes
# 4.4	Deploy Calico network
# 
# 5	Post Installation verification ..........................................................................................
# 5.1	Deploy Hello world application with replicas	
# 
# 6	Post Installation Configuration ..........................................................................................
# 6.1	Install Nginx Ingress Controller.......................................................................
# 6.2	Install LCM Container Registry.............................................................................................






















1	Introduction

1.1                    Document History

Table 1       Document History


1.2                    Target Audience















2	Architecture


 


3	Prerequisite 

3.1	Gather Details

	
Director Server IPs	
10.164.227.143
10.164.227.144

Master Server IPs	
10.164.227.145
10.164.227.146
10.164.227.147

Worker Server IPs	
10.164.227.148
10.164.227.149
10.164.227.150
10.164.227.151

Virtual IP for LB	
 10.164.227.61

POD CIDR	
Kubernetes and its components versions. 
	
	
	
	








3.2	Check Network Connectivity
Full network connectivity among all machines in the cluster. You can use either a public or a private network 
3.3	Check and Open Ports
Below ports should be open 
  
  
3.4	Set hostname 
Like for master 1: 
hostnamectl set-hostname master01.<FQDN>
add to /etc/hosts
 
3.5	Disable swap 
swapoff -a; sed -i '/swap/d' /etc/fstab 
Disable Firewall (should not disable firewall fully) 
  systemctl stop firewalld
  systemctl disable firewalld
3.6	Enable and Load Kernel modules 
cat <<EOF | tee /etc/modules-load.d/containerd.conf 
overlay 
br_netfilter 
EOF 
 
modprobe overlay 
modprobe br_netfilter 
3.7	Add Kernel settings 
sed -i '/net.bridge.bridge-nf-call-ip6tables/d' /etc/sysctl.conf
    sed -i '/net.bridge.bridge-nf-call-iptables/d' /etc/sysctl.conf
    sed -i '/net.ipv4.ip_forward/d' /etc/sysctl.conf
  cat >>/etc/sysctl.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    EOF
sysctl --system
 
4	Pre-Installation Steps

4.1	Installing docker as runtime  
yum install -y yum-utils 
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/rhel/docker-ce.repo
 
yum install docker-ce docker-ce-cli containerd.io 
 
systemctl start docker 
systemctl enable docker 
 
To use the systemd cgroup driver with docker, set  
{ 
  "exec-opts": ["native.cgroupdriver=systemd"], 
  "log-driver": "json-file", 
  "log-opts": { 
    "max-size": "100m" 
  }, 
  "storage-driver": "overlay2" 
} 
 
  
If you apply this change make sure to restart docker again:  
systemctl daemon-reload 
systemctl restart docker  
 
4.2	Installing Kubeadm, kubelet and kubectl (on all nodes, kubectl is optional in worker nodes) 
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
 
# Set SELinux in permissive mode (effectively disabling it) 
sudo setenforce 0 
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config 
 
sudo yum install -y kubelet-1.21.1-00 kubeadm-1.21.1-00 kubectl-1.21.1-00 --disableexcludes=kubernetes 
 
sudo systemctl enable --now kubelet 
 
4.3	 Install helm client (optional in worker nodes) 
 
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 
 
chmod 700 get_helm.sh 
./get_helm.sh
 
4.4	Configure keepalived and haproxy 
Configuration: https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#keepalived-and-haproxy 
Get the interface: 
ip a s #check the interface with host ip 
 
Run as static pod in Kubernetes: https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#option-2-run-the-services-as-static-pods 
 
4	Installation Steps
4.1	Initialize cluster 
 
Run below command from master 1: 
 
 
 

kubeadm init --control-plane-endpoint="{{VIP}}:6443" --upload-certs --apiserver-advertise-address={{MASTER1_NODE_IP}} --pod-network-cidr={{POD_CIDR}}
 
 
  
 4.4      get kube config
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
    echo 'alias k=kubectl' >>~/.bashrc
    echo 'complete -F __start_kubectl k' >>~/.bashrc

4.4	Deploy Calico network (the CNI) 
kubectl --kubeconfig=/etc/kubernetes/admin.conf apply -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml

4.2 	Join other master nodes Use control plane join command for other master node and worker node join command for workers. 
Add --apiserver-advertise-address={{MASTER_NODE_IP}} for master node join 

4.3 	Join worker nodes Use control plane join command for other master node and worker node join command for workers. 
 
 5	Post Installation verification
5.1	Deploy Hello world application with replicas	



6	Post Installation Configuration


6.1	Install nginx ingress controller 
Metallb:
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb
helm install metallb metallb/metallb --create-namespace --namespace metallb-system -f values.yaml
values.yaml looks like (replace {{ ingress_external_vip }} with ip assigned for external communication):
configInline:
  address-pools:
   - name: default
     protocol: layer2
     addresses:
     - {{ingress_external_vip}}/32

Get ingress helm chart

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm pull ingress-nginx/ingress-nginx

ingres.conf (replace jinja2 variable)
[ req ]
default_bits  = 2048
distinguished_name = req_distinguished_name
req_extensions  = req_ext
prompt = no

[ req_distinguished_name ]
countryName = SE
stateOrProvinceName = Stockholm
localityName = Stockholm
organizationName = Ericsson
organizationalUnitName = IT SERVICES
commonName = {{cdd_dns_name}}

[ req_ext ]
subjectAltName = @alt_names

[alt_names]
DNS.1 = {{cdd_dns_name}}
DNS.2 = *.{{cdd_dns_name}}





openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ingress.key -out ingress.crt -config ingress.conf
kubectl create secret  tls ingress-tls  -n kube-system  --key="ingress.key" --cert="ingress.crt"

helm install ingress-nginx ingress-nginx-4.0.13.tgz --create-namespace --namespace ingress-nginx  \
--set controller.extraArgs.default-ssl-certificate=kube-system/ingress-tls \
--set controller.extraArgs.annotations-prefix=nginx.ingress.kubernetes.io \
--set defaultBackend.enabled=true \
--set controller.replicaCount=2 \
--set controller.name="" \
--set controller.extraArgs.enable-ssl-passthrough= \
--set controller.admissionWebhooks.enabled=false


Apply tcp and udp service
Tcp-services.yaml:
apiVersion: v1
data:
  "22": cdd-app/eric-cm-yang-provider:22
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"name":"tcp-services","namespace":"ingress-nginx"}}
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2021-10-19T15:20:46Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:22: {}
    manager: kubectl
    operation: Update
    time: "2021-10-20T14:04:50Z"
  name: tcp-services
  namespace: ingress-nginx



udp-services.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"name":"udp-services","namespace":"ingress-nginx"}}
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
    manager: kubectl-client-side-apply
    operation: Update
  name: udp-services
  namespace: ingress-nginx


apply:
kubectl apply -f tcp-services.yaml -n ingress-nginx
kubectl apply -f udp-services.yaml -n ingress-nginx


6.2	Install lcm container registry 

Need to crate NFS server if readymade nfs server not available
Add nfs storage class and make it default

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --create-namespace --namespace nfs-external-provisioner --set nfs.server=10.164.227.145 --set nfs.path=/data/repositories --set storageClass.name=network-block
kubectl patch storageclass network-block -n nfs-external-provisioner  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'


htpasswd -cBb htpasswd admin ericsson123
kubectl create secret generic  cr-registry-credentials --from-file=htpasswd=./htpasswd -n kube-system

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -config registry.conf
kubectl create secret  tls cr-registry-tls  -n kube-system  --key="tls.key" --cert="tls.crt"

mkdir -p /etc/docker/certs.d/k8s-registry.<service FQDN>
vi /etc/docker/certs.d/k8s-registry.<service FQDN> /tls.crt
Copy the tls.crt created in Section 2.2.3 Step 2
service docker restart
 
 
Useful links: 
https://dev.to/itminds/high-availability-kubernetes-cluster-on-bare-metal-part-2-46j9 
https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing 
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd 
https://www.youtube.com/watch?v=SueeqeioyKY 
 


