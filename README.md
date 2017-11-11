
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
        
        ansible-playbook -i inventory/inventory.cfg upload_standalone_packages.yml --become --become-method=sudo --private-key=<PRIVATE_KEY> -u <SSH_USER> -vvv

    Example:
    
        ansible-playbook -i inventory/inventory.cfg upload_standalone_packages.yml --become --become-method=sudo --private-key=~/.ssh/id_rsa -u jerrypeng -vvv

This might take a while to run.

3. Run deployment script to deploy a Kubernetes Cluster

        ansible-playbook -i inventory/inventory.cfg cluster.yml --become --private-key=<PRIVATE_KEY> -u <SSH_USER> -vvv
        
    Example:

        ansible-playbook -i inventory/inventory.cfg cluster.yml --become --private-key=~/.ssh/id_rsa -u jerrypeng -vvv
        
4. Wait for scripts to run to completion.
5. Go to the UI. SSH into one of the nodes and run:

        kubectl describe pod kubernetes-dashboard --namespace=kube-system
    Look for "Node" field this will tell you what node the Kubernetes UI is scheduled on.
    
    Run:
    
        kubectl describe service kubernetes-dashboard --namespace=kube-system
        
    Look for the "NodePort" field. This will tell you what port the UI is using
    
    Now you have the node and port that the UI is running on

6.  To deploy Streamlio/Heron related components to Kubernetes Cluster
 
        ansible-playbook -i inventory/inventory.cfg heron-deploy.yml --private-key=<PRIVATE_KEY> -u <SSH_USER> -vvv

    Example:
        
        ansible-playbook -i inventory/inventory.cfg heron-deploy.yml --private-key=~/.ssh/id_rsa -u jerrypeng -vvv
        
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
systemctl start kubelet
systemctl start docker

