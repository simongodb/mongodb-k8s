# mongodb-k8s
How to deploy a MongoDB cluster and Ops Manager on Kubernetes leveraging Minikube, MongoDB Kubernetes Operator, Helm and a single AWS EC2 instance.


1. Prerequisites

Launch a AWS EC2 t3.xlarge instance on Ubuntu 22.04 with a 30GB root volume.

Edit the associated Security Group to allow all inbound traffic to ease the exercise.

Connect to instance with SSH

    ssh -i "<pem-filename>.pem" ubuntu@ec2-XX-XXX-XXX-XXX.eu-west-3.compute.amazonaws.com

Install kubectl

    curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl

Install docker

    sudo apt-get update && \
    sudo apt-get install docker.io -y
    
Install Minikube

    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
    minikube version
    
 Install conntrack
 
     sudo apt install conntrack
     
 Install cri-dockerd
 
    wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.0/cri-dockerd-v0.2.0-linux-amd64.tar.gz
    tar xvf cri-dockerd-v0.2.0-linux-amd64.tar.gz
    sudo mv ./cri-dockerd /usr/local/bin/ 
    cri-dockerd --help
    
    wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
    wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
    sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
    sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
    
    sudo systemctl daemon-reload
    sudo systemctl enable cri-docker.service
    sudo systemctl enable --now cri-docker.socket
    sudo systemctl status cri-docker.socket
    
    sudo sysctl fs.protected_regular=0

Install crictl

    VERSION="v1.24.1"
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
    sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
    rm -f crictl-$VERSION-linux-amd64.tar.gz
    
Start Minikube

    sudo -i
    minikube start --vm-driver=none
    minikube status
    
 Install HELM
 
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    sudo apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm
    
Install mongosh

    wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
    echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
    sudo apt-get update
    sudo apt-get install -y mongodb-mongosh

Start Kubernetes Dashboard

    minikube dashboard --url

Open another terminal an create SSH tunnel

    ssh -i [private key.pem] -L [local port]:localhost:[remote port of minikube dashboard] ubuntu@[public ip]
    
Browse from your local machine

    http://127.0.0.1:8081/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/error?namespace=_all
    
2. Deploy MongoDB Kubernetes Enterprise Operator

Install with HELM

    helm repo add mongodb https://mongodb.github.io/helm-charts
    helm install enterprise-operator mongodb/enterprise-operator --namespace mongodb --create-namespace
    kubectl config set-context $(kubectl config current-context) --namespace=mongodb