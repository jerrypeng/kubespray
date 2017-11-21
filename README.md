
# Deploying Kubernetes Cluster Offline Mode

This directory contains scripts to deploy a Kubernetes cluster

## Prerequisites for deploy machine
1. python pip
2. Ansible 
    - sudo pip install ansible
3. python-netaddr
    - sudo pip install netaddr
4. ssh access to all nodes


## Deployment Steps

1. Update inventory/inventory to have to correct nodes and roles for the cluster
2. Run package uploader script to load and install all images and rpm dependencies
        
        ansible-playbook -i inventory/inventory.cfg upload_standalone_packages.yml --become --become-method=sudo --private-key=<PRIVATE_KEY> -u <SSH_USER> -vvv --extra-vars "ansible_sudo_pass=<PASSWORD>"

    Example:
    
        ansible-playbook -i inventory/inventory.cfg upload_standalone_packages.yml --become --become-method=sudo --private-key=~/.ssh/id_rsa -u jerrypeng -vvv --extra-vars "ansible_sudo_pass=<PASSWORD>"

This might take a while to run.

3. Run deployment script to deploy a Kubernetes Cluster

        ansible-playbook -i inventory/inventory.cfg cluster.yml --become --private-key=<PRIVATE_KEY> -u <SSH_USER> -vvv --extra-vars "ansible_sudo_pass=<PASSWORD>"
        
    Example:

        ansible-playbook -i inventory/inventory.cfg cluster.yml --become --private-key=~/.ssh/id_rsa -u jerrypeng -vvv
        
4. Wait for scripts to run to completion.

5.  To deploy Streamlio/Heron related components to Kubernetes Cluster
 
        ansible-playbook -i inventory/inventory.cfg heron-deploy.yml --private-key=<PRIVATE_KEY> -u <SSH_USER> -vvv

    Example:
        
        ansible-playbook -i inventory/inventory.cfg heron-deploy.yml --private-key=~/.ssh/id_rsa -u jerrypeng -vvv

## Running Heron Topologies on Kubernetes

After you have followed the previous instruction on install a kubernetes cluster and have start the neccessary components on kubernetes for Heron to run, please to the following:

1. Install Heron CLI on the node that you wish to submit Heron topologies from.  You must also have your topology JARS on this node as well
2. SSH to a master node and run 'kubectl proxy'
3. Setup port forwording for port 8001 on the master node to port 8001 on localhost.  This can be done with the following command:

        ssh -L 8001:localhost:8001 <USER>@<MASTER_NODE>
4. Submit topology to K8s cluster

        heron submit --service-url=http://localhost:8001/api/v1/proxy/namespaces/default/services/heron-apiserver:9000 --verbose kubernetes <TOPOLOGY_JAR_PATH> <TOPOLOGY_CLASSPATH> <NAME>
        
    For example:
        
        heron submit --service-url=http://localhost:8001/api/v1/proxy/namespaces/default/services/heron-apiserver:9000 --verbose kubernetes ~/workspace/heron/bazel-bin/examples/src/java/api-examples-unshaded_deploy.jar com.twitter.heron.examples.api.AckingTopology test-3
        
## Dashboards and Heron UI

After you have followed the previous instruction on install a kubernetes cluster and have start the neccessary components on kubernetes for Heron to run, please setup proxy on the master node as described above with the command "kubectl proxy".  Then you can access the kubernetes dashboard and Heron UI via the following URLs:

Kubernetes Dashboard URL:

http://127.0.0.1:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard


Heron UI URL:

http://127.0.0.1:8001/api/v1/proxy/namespaces/default/services/heron-ui:8889

# Install and configure Kubectl on other nodes

ansible-playbook -i inventory/inventory.cfg setup_local_kubectl.yml --become --private-key=~/.ssh/id_rsa -u root -vvv

# How it works

1. Package up all dependencies
    - Download all need docker images as a tar file
    
            ansible-playbook -i inventory/inventory.cfg create_standalone_package.yml -vvv

    - Download all dependencies (e.g. docker etc) as rpms.  Also download all dependencies of dependencies
            
            bash roles/standalone/create-package/download-rpms.sh
            
2. Upload dependencies and setup local yum repo 
    - upload all the docker images and rpms to nodes.  Setup local yum repo that point to the rpms. 
3. Run deploy cluster script

# If kube-dns crashes

sysctl net.ipv4.conf.all.forwarding=1
iptables -P FORWARD ACCEPT
systemctl stop kubelet
systemctl stop docker
iptables --flush
iptables -tnat --flush
systemctl start docker
systemctl start kubelet

ansible-playbook -i inventory/inventory.cfg reset.yml --become --private-key=<PRIVATE_KEY> -u <USER> -vvv --extra-vars "ansible_sudo_pass=<PASSWORD>"


[node1/master]
kubectl drain --force --ignore-daemonsets --delete-local-data jerry-2
kubectl delete node jerry-2
kubectl drain --force --ignore-daemonsets --delete-local-data jerry-1
kubectl delete node jerry-1

systemctl stop kubelet
systemctl stop docker

rm -rf /var/lib/kubelet/
rm -rf /var/lib/dnsmasq/
rm -rf /var/lib/docker
rm -rf /var/lib/etcd/
rm -rf /etc/kubernetes/

rm -rf /etc/systemd/system/docker.service*
rm -rf /etc/systemd/system/kubelet.service
rm -rf /etc/systemd/system/etcd.service

yum remove docker-engine-1.13.1-1.el7.centos.x86_64
yum remove docker-engine-selinux-17.05.0.ce-1.el7.centos.noarch
rm /usr/local/bin/kubectl


[Node2]
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/kubelet/
rm -rf /var/lib/dnsmasq/
rm -rf /var/lib/docker
rm -rf /var/lib/etcd/
rm -rf /etc/kubernetes/

rm -rf /etc/systemd/system/docker.service*
rm -rf /etc/systemd/system/kubelet.service
rm -rf /etc/systemd/system/etcd.service

yum remove docker-engine-1.13.1-1.el7.centos.x86_64
yum remove docker-engine-selinux-17.05.0.ce-1.el7.centos.noarch

