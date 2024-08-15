# Home Lab set up


## Environment
- mini pc with Virtual Box installed
- OS: Ubuntu 22.04
- Kubernetes: rke2
- rke2:password
- IP address: 192.168.71.50

  st. joseph server
- IP adress:192.168.71.51
- rke2
- password
- 

## Setup
-Create the vm with 4 cores/8GB RAM/25GB storage
-install Ubuntu 22.04
-assign IP/set static IP on network to avoid the IP changing
- For RKE2 to run, need to disable the firewall to all rke to install/setup
``sudo systemctl disable --now ufw``
may need to disable app-armor but foudn this could cause problems. If you need to disable app armor, use:
``sudo systemctl disable --now apparmor.service``

- Follow RKE2 quickstart guide at: https://docs.rke2.io/install/quickstart
- ``curl -sfL https://get.rke2.io | sh -``

``systemctl enable rke2-server.service``

``systemctl start rke2-server.service``

``journalctl -u rke2-server -f``

(This tutorial is very helpful too https://www.youtube.com/watch?v=ipmDnk97hbA&t=242s)

Pre-Req For RKE2 Server
1) Stop firewall
systemctl disable --now ufw
systemctl disable --now apparmor

2) Run updates and upgrade and install packages
apt update
apt install nfs-common open-iscsi -y
apt upgrade -y

3) Automatic removal of unnecessary packages.
apt autoremove -y

RKE2 Server Installation
1) Install RKE2 server
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -
For specific RKE2 Server version
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.26 INSTALL_RKE2_TYPE=server sh -

2) Enable and start RKE2 Server service
systemctl enable rke2-server.service
systemctl start rke2-server.service
Alternatively, you can do this, which does the same thing
systemctl enable --now rke2-server.service

3) Check is RKE2 Server is running
systemctl status rke2-server.service


Setup Kubectl (on RKE2 Server)
1) Simlink all the things - kubectl
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl
 Alternatively, you can do this, which download the binary rather than using simlink. But note that you are downloading specific version of kubectl. Getting it from the RKE2 Server is a better option (aka simlink)
 curl -L# https://dl.k8s.io/release/v1.24.0/bin... -o /usr/local/bin/kubectl
 chomd 755 /usr/local/bin/kubectl
2) Add kubectl conf
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml 
 Permanent
 nano .bashrc
 export PATH=$PATH:/etc/rancher/rke2/rke2.yaml
3) Check if kubectl works
kubectl get node -o wide
To watch as you start the RKE2 agent use this command.
kubectl get node -o wide -w

Setup local Kube config file to work with local k8s with out setting the context
1) edit local rke2.yaml with correct IP address
   vi /etc/rancher/rke2/rke2.yaml
   edit the IP address found here to one mathcing local machine
2) copy the file locally
    cat /etc/rancher/rke2/rke2.yaml
3) export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

## Optional if not setting up multiple nodes
Get RKE2 Server IP address And Token (on RKE2 Server)
1) Get server IP address and copy it somewhere
ip addr | grep inet
2) Get server token and copy it somewhere
cat /var/lib/rancher/rke2/server/node-token

Pre-Req For RKE2 Agents
1) Stop the software firewall
systemctl disable --now ufw
2) Run updates and upgrade and install nfs
apt update
apt install nfs-common -y open-iscsi
apt upgrade -y
This is require for Longhorn
iscsiadm --version
3) Remove downloaded update
apt autoremove -y

RKE2 Agent Installation
1) Install RKE2 agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -
For specific RKE2 Server version
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.26 INSTALL_RKE2_TYPE=agent sh -
2) Create config directory
mkdir -p /etc/rancher/rke2/
3) Create config.yaml file for RKE2 agent
nano /etc/rancher/rke2/config.yaml
Enter text shown below.
server: https://#RKE2_SERVER_IP_ADDRESS#:9345
token: #TOKEN_FROM_RKE2_SERVER#
The #RANCHER_SERVER_IP_ADDRESS# is the ip address of RKE2 Server Install - Step 5
The #TOKEN_FROM_RANCHER_SERVER# is the ip address of RKE2 Server Install - Step 6
4) Enable and start RKE2 Agent service
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
 Alternatively, you can do this, which does the same thing
 systemctl enable --now rke2-agent.service
systemctl status rke2-agent.service
5) Check is RKE2 Agent is running
systemctl status rke2-agent.service
Run this command in RKE2 Server to check if RKE2 Agents are communicating with the RKE2 Server.
kubectl get node -o wide
Run this command in RKE2 Server to check all pods status.
kubectl get pod -A

** NOTE **
Repeat "RKE2 Agent Installation", if you have multiple RKE2 agents.


### Resume install here
Get/Change FQDN
1) Get/Change Current hostname
nano /etc/hostname
2) Add FQDN here
nano /etc/hosts
Add following line
#RANCHER_SERVER_IP_ADDRESS# rke2.pg.local

Rancher Install Using Helm (on RKE2 Server)
1) Install Helm
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3| bash
2) Add Rancher to repo
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
3) Add Jetstack (cert mgr) to repo
helm repo add jetstack https://charts.jetstack.io
4) Create cert-manager CRD
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml --kubeconfig=stjoseph.yaml
1) Helm install Jetstack
helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
1) Helm install Rancher
helm upgrade -i rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=demo.pg.local --set bootstrapPassword=hippo --set replicas=1 --set global.cattle.psp.enabled=false
1) Check if Rancher and Jetstack (CertManager) containers are running
kubectl get pod -A
curl -k https://rke2.pg.local

Lognhorn Install Using Helm (on RKE2 Server)
1) Add Longhorn to repo
helm repo add longhorn https://charts.longhorn.io
2) Update
helm repo update
3) Helm install Longhorn chart
helm upgrade -i longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
4) Check if Longhorn container is running
kubectl get pod -A

- Note: kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml --kubeconfig=rke2.yaml


copy kube config file from ``/etc/rancher/rke2/rke2.yaml``, edit the IP addreess
-- I copied it to it's own kube config file and saved it my home lab folder. But I need to add ``--kubeconfig=rke2.yaml`` after every ``kubectl`` command




Next I was able to follow the steps on https://www.crunchydata.com/developers/get-started/postgres-operator/install

first set up namespaces

``kubectl apply -k kustomize/install/namespace`` (I created 2 names spaces, ``postgres-operator`` and ``demo``)

next I had to create my registry credential in the cluster to pull images from the crunchy access portal. See ``regcred_template.sh`` for details

I could then create the operator

``kubectl apply --server-side -k kustomize/install/default``

``kubectl apply -k kustomize/postgres``

Next steps:

-Connect to DB
-evaluate AI's to start experimenting with. 
