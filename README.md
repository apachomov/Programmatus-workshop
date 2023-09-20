# KUBERNETES
## MAIN KUBERNETES COMPONENTS
Explore [Kubernetes basic architecture](https://kubernetes.io/docs/concepts/overview/components/).

## KUBERNETES INSTALLATION 
General boostrap information is available [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/). There are more then one way to install `Kubernetes`, in this workshop we will use `kubeadm` method.
1. Prepare your hosts with supported and up to date OS, and perform few more steps
Set SELinux in permissive mode (effectively disabling it)
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
Install helping packages
```bash
yum install -y wget yum-utils tar
```

In the control plane, we will utilize the [Helm package manager](https://helm.sh/docs/intro/install/) for installing software into the Kubernetes cluster later on.
```bash
wget https://get.helm.sh/helm-v3.12.3-linux-amd64.tar.gz
tar -zxvf helm-v3.12.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```
# Disable firewall 
```bash
systemctl disable --now firewalld.service

# Or configure it - Configuration requirements may vary based on your choice of CNI and other services. These are the default opening examples.
## For Control Plane node
firewall-cmd --add-port=6443/tcp --add-port=2379-2380/tcp --add-port=10250/tcp --add-port=10259/tcp --add-port=10257/tcp --add-port=30000-32767/tcp --permanent
firewall-cmd --reload
## For Worker node:
firewall-cmd --add-port=10250/tcp --add-port=30000-32767/tcp --permanent
firewall-cmd --reload
```
Add Docker packages repo
```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
Add Kubernetes packages repo
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
2. Install the [Container Runtime Interface](https://kubernetes.io/docs/setup/production-environment/container-runtimes/). We are using Docker in our workshop.
```bash
yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
```
Additional [installation steps](https://github.com/Mirantis/cri-dockerd) are required to install `cri-dockerd` because Docker does not have a default CRI.
```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd-0.3.4-3.el8.x86_64.rpm
yum localinstall -y cri-dockerd-0.3.4-3.el8.x86_64.rpm 
systemctl enable --now cri-docker.service
```

3. Install [kubeadm, kubectl and kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```
4. [Initialize cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
Bootstrap a cluster and make note of the `join` command, which will be prompted at the end of a successful installation, resembling this: `kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>`
```bash
kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16
```
Set up `kubectl` access to the API and include a few additional tuning steps for enhanced kubectl ease of use.
```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

5. Join a worker node. You can repeat this process on as many nodes as needed at any time.
```bash
ssh root@pws-wn
# Repeat steps 1-3
kubeadm join 192.168.62.56:6443 --token p2olf3.3j6q9bbx78agugod --discovery-token-ca-cert-hash sha256:8e83af9e1ddf1ab160a1de7f73fd027ff4c7050662538dc96a4a7c50901cfdfa --cri-socket unix:///var/run/cri-dockerd.sock
```

6. Install [Container Network Interface](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy). In our workshop, we are using `flannel`. 
Install the `flannel` plugins on both the Control Plane node and Worker nodes.
```bash
mkdir -p /opt/cni/bin
curl -O -L https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.2.0.tgz
```
Install CNI 
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## GENERAL CLUSTER OVERVIEW
```bash
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
kubectl get all -n kube-system
kubectl get pods -A
```

## DECLARATIVE vs IMPERATIVE
In Kubernetes, you can achieve many tasks using imperative commands, but in the GitOps operational model, a declarative approach is essential. This means that any object in Kubernetes can be described using either `yaml` or `json` format.

To obtain information in these formats, append the appropriate output formatting option to commands:
```bash
kubectl get nodes -o yaml
kubectl get namespaces -o json
```

When you want to create a resource defined in declarative files, you can use the following commands
```bash
kubectl create -f <path_to_file>
kubectl apply -f <path_to_file>

# This would apply all files found in specified directory 
kubectl apply -f <directory_name>/.
```

### Dry run option
If you need to generate a `yaml` file quickly, one of the fastest methods is to utilize imperative commands with the `--dry-run=client -o yaml` option:
```bash
kubectl create deployment my-nginx-app --image nginx --dry-run=client -o yaml
kubectl create service nodeport my-nginx-app --tcp 80 --dry-run=client -o yaml
```

## SERVICE
In Kubernetes, a `Service` is a method for exposing a network application that is running as one or more `Pods` in your cluster.
- Unlike pods, a `Service` has a permanent IP and directs to `Pods`.
- The lifecycles of `Pods` and `Services` are separate; even if a `pod` dies, the `Service` IP remains unchanged.

### Service types:
* ClusterIP: Used for internal communication within the cluster; this is the default type.
* LoadBalancer: Exposes service availability to external sources by assigning a unique service IP:
    * For on-premises setups, it relies on DHCP or manual IP specification. DHCP setups often use [MetallLB](https://metallb.universe.tf/)
    * On cloud infrastructures, creating such objects usually involves creating cloud load balancers. Cloud-specific documentation should be consulted, like in this ([AWS Example](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html)).
* NodePort: Exposes a service on a specific port on nodes, enabling access via `<NodeIP>:<SpecifiedPort>`. Often used for testing but not for production use.

### Example: Creating Service
1. First, prepare to deploy two `pods`.
```bash
kubectl run svc-exmpl-pod01 --image nginx --dry-run=client -o yaml > pod.yaml
echo "---" >> pod.yaml
kubectl run svc-exmpl-pod02 --image nginx --dry-run=client -o yaml >> pod.yaml
```
2. Modify the metadata of `pods` to have the same label, such as `example-service-01`.
```bash
vi pod.yaml
```
3. Create `pods` and verify their information.
```bash
kubectl apply -f pod.yaml 
kubectl get pods -o wide --show-labels
```
4. Deploy a `service`.
```bash
kubectl expose --help
kubectl expose --port=80 --target-port=80 --name=example-service-01 pod svc-exmpl-pod01 --dry-run=client -o yaml > service.yaml
cat service.yaml
kubectl apply -f service.yaml

kubectl get svc
kubectl describe svc example-service-01

kubectl get endpoints

kubectl describe endpoints example-service-01
```

3. Test accessing the `service` via the `service` name.
```bash
kubectl run --image nginx curler
kubectl exec -it curler -- curl http://example-service-01
```
## DEPLOYMENTS
A `Deployment` is a crucial Kubernetes object that ensures declared applications are consistently operational, maintaining the specified number of instances. In the event of an application crash, the `Deployment` automatically orchestrates its restart to ensure seamless operation.

### Example: Creating a `Deployment`
```bash
kubectl create deployment --help
kubectl create deployment app-04 --image nginx --replicas 3 --dry-run=client -o yaml > deployment.yaml

cat deployment.yaml

kubectl apply -f deployment.yaml
kubectl get pods --watch

kubectl delete pod <pod_name>
```

### Adjusting a Deployment
Edit the deployment configuration - first try to change image and replicas, then only replicas
```bash
vi deployment.yaml 
```
Update deployment and observe
```bash
kubectl apply -f deployment.yaml
kubectl get pods --watch
```
Create Service for deployment and check application accessabillity
```bash
kubectl expose deployment app-04 --name app-04 --port 80
kubectl exec curler -- curl http://app-04
```

Good to know - there are other widely used components such as `StatefulSet` and `DaemonSet`. 
- **StatefulSet**: A `StatefulSet` is akin to a `Deployment` with distinct characteristics. It maintains two key differences: it ensures that the names of pods persist across rescheduling, enabling stable network identities, and it offers dynamic volume provisioning and management. These attributes make it ideal for stateful applications like databases, where consistent naming and persistent storage are crucial.

- **DaemonSet**: Much like a `Deployment` in structure, a `DaemonSet` stands apart by omitting the concept of a replica count. Instead, it deploys one pod on every available node within the cluster. This mechanism makes it particularly useful for tasks that need to run on every node, like log collection or monitoring agents.

## LIVELINESS AND READINESS PROBES

Liveliness probes - Many applications running for long periods of time eventually transition to broken states, and cannot recover except by being restarted. Kubernetes provides liveness probes to detect and remedy such situations.

Readiness probes - Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. In such cases, you don't want to kill the application, but you don't want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. 

Study [examples](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

## KUBERNETES DASHBOARD
Install [kubernetes-dashboard](https://github.com/kubernetes/dashboard) via [helm]((https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard)). 
```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
kubectl get pods -n kubernetes-dashboard
kubectl expose -n kubernetes-dashboard deployment kubernetes-dashboard  --name k8s-dashboard --type NodePort
```
Generate a long-lived API token that will be required for logging in.
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-dashboard-token
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: kubernetes-dashboard 
type: kubernetes.io/service-account-token
EOF
```
Grant the missing permissions, extract the token, and then verify access to the service at https://192.168.62.57:31775/.
```bash
kubectl create clusterrolebinding --clusterrole admin --serviceaccount kubernetes-dashboard:kubernetes-dashboard kubernetes-dashboard-admiin-binding
kubectl get secrets -n kubernetes-dashboard kubernetes-dashboard-token -o jsonpath={'.data.token'} | base64 -d
kubectl get svc -n kubernetes-dashboard 
```

## TROUBLESHOOTING
Troubleshooting in Kubernetes usually involves two main aspects: `application` and `infrastructure` levels.

When troubleshooting applications, consulting logs is a common approach:
```bash
# General syntax 
kubectl logs -f <object>
# A cool way to gather from multiple pods
kubectl logs -f --selector labelkey:value 
```
Example of previously deployed app-04
```bash
kubectl get pods --show-labels
kubectl logs -f  --selector app=app-04
```
Open another terminal for the demo, attempt to access different pages of the application, and review the logs.
```bash
kubectl exec curler -- curl http://app-04
kubectl exec curler -- curl http://app-04/thispagedoesntexist
```

For infrastructure issues, consider these steps using:
```bash
# Investigate a problematic pod
kubectl describe <pod>

# Check runtime events
kubectl get events
```

Here are few more examples to explore.
Example 1
```bash
kubectl create deployment --image mysql mysql-deploy
kubectl get pods --watch
kubectl describe pod mysql-deploy-5bdb9d659-zwgzq
kubectl logs -f  mysql-deploy-5bdb9d659-zwgzq
kubectl get deploy mysql-deploy -o yaml > mysql-deploy.yaml
vi mysql-deploy.yaml
# Add
# env:
# - name: MYSQL_ROOT_PASSWORD
#   value: hello

kubectl delete deployments.apps mysql-deploy
kubectl apply -f mysql-deploy.yaml
kubectl get pods --watch
kubectl exec -it mysql-deploy-7886f447b9-4qkqx -- bash
mysql -p
```
Example 2
```bash
kubectl run ubuntu-pod --image ubuntnu
kubectl get pods --watch
kubectl describe pod ubuntu-pod

kubectl delete pod ubuntu-pod 
kubectl run ubuntu-pod --image ubuntu
kubectl get pods --watch
kubectl describe pod ubuntu-pod 

kubectl delete pod ubuntu-pod 
kubectl run ubuntu-pod --image ubuntu -- slepp 3600
kubectl get pods --watch
kubectl describe pod ubuntu-pod 

kubectl delete pod ubuntu-pod 
kubectl run ubuntu-pod --image ubuntu -- sleep 3600
kubectl get pods --watch
kubectl exec -it ubuntu-pod -- ps afux
```


Sometimes, you may encounter performance-related situations where a quick overview using [metrics-server](https://github.com/kubernetes-sigs/metrics-server) can be helpful. However, in production environments, it's essential to have a comprehensive statistical overview. For this purpose, the most widely used solution is `Prometheus` with `Grafana` for visualization.

To install `metrics-server`
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl edit deployments.apps -n kube-system metrics-server 
# Add option
# - --kubelet-insecure-tls

kubectl top pods
kubectl top nodes
```
