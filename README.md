# Kubernetes Cluster using ec2 instances as nodes 

## Cloudformation commands to create/update aws resource stack

### Create stack
```
aws cloudformation create-stack --stack-name ec2-cluster-k8s --template-body file://cf-template.yml  --profile <AWS_PROFILE_NAME> --parameters ParameterKey=MasterUserData,ParameterValue=$(base64 -w0 master-node-setup.sh) ParameterKey=WorkerUserData,ParameterValue=$(base64 -w0 worker-node-setup.sh)
```

### Update Stack
```
aws cloudformation update-stack --stack-name ec2-cluster-k8s --template-body file://cf-template.yml  --profile <AWS_PROFILE_NAME> --parameters ParameterKey=MasterUserData,ParameterValue=$(base64 -w0 master-node-setup.sh) ParameterKey=WorkerUserData,ParameterValue=$(base64 -w0 worker-node-setup.sh)
```

### Delete Stack
```
aws cloudformation delete-stack --stack-name ec2-cluster-k8s --profile <AWS_PROFILE_NAME>
```


## Set up master node:


1. Install curl and apt-transport-https
```
sudo apt install apt-transport-https curl
```
2. Add Kubernetes signing key
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```

3. Add Kubernetes repository
```
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" >> ~/kubernetes.list
sudo mv ~/kubernetes.list /etc/apt/sources.list.d
```

4. Update the servers
```
sudo apt update
```

5. Install kubeadm, kubelet, kubectl, and kubernetes-cni
```
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
Verify installation:
kubectl version --client && kubeadm version
```

6. Disable Swap Memory
```
sudo swapoff -a
sudo nano /etc/fstab # comment out swapfile line (if any)
```

7. Setup unique hostname
```
sudo hostnamectl set-hostname kube-master
```

8. Enable Bridge Traffic in IP Tables
```
sudo modprobe br_netfilter
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

9. Install Docker Runtime and run docker with systemd. refer `docker-set-up.sh` or run
```
sh docker-set-up.sh
```

10. Install cri-dockerd and run with systemd. refer `cri-dockerd-set-up.sh` or run
```
sh cri-dockerd-set-up.sh
```
Reference: https://www.mirantis.com/blog/how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim

11. Initialize Kubernetes Master Node
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```
you will get the joining command as output of this command. save it at a safe place for later use.
output example:
```
sudo kubeadm join 10.0.1.207:6443 --token 5dhogn.cboi55mdxh42hhjd --discovery-token-ca-cert-hash sha256:a1dd325ab9e6d4269f9b2549a9a2de2c4dfd00b9b4f02386c5e7147be6e6e421
```

12. Create Kubernetes Config as advised
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

13. Verify if the master node is setup correctly.
```
kubectl get node
```

## Set-up Pod Network
A pod network facilitates communication between servers and it’s necessary for the proper functioning of the Kubernetes cluster.
We will be using the Flannel pod network for this tutorial. Flannel is a simple overlay network that satisfies the Kubernetes
requirements.

1. Allow firewall rule to create exceptions for port 6443 (default port for Kubernetes)
```
sudo ufw allow 6443
sudo ufw allow 6443/tcp
```

2. Create required resources
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

## Set-up Worker nodes

1. Follow step 1 to 10 from Master Node setup ( till cri-dockerd container run tie setup )

2. Run joining command from the step 11 of Master node setup. (append `--cri-socket=unix:///var/run/cri-dockerd.sock` after join command and run with sudo)
example:
```
sudo kubeadm join 10.0.1.207:6443 --token 5dhogn.cboi55mdxh42hhjd --discovery-token-ca-cert-hash sha256:a1dd325ab9e6d4269f9b2549a9a2de2c4dfd00b9b4f02386c5e7147be6e6e421 --cri-socket=unix:///var/run/cri-dockerd.sock
```

## Run a simple nginx app for testing cluster setup

1. Create nginx Deployment
```
kubectl create deployment nginx --image=nginx
```

2. Make Nginx Accessible
```
kubectl create service nodeport nginx --tcp=81:80
```

3. Get accessible link
```
kubectl get svc
```

4. Verify working or not
```
curl localhost:host_port
```


