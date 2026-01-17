# Kubernetes Demo
## Steps
- Set up a k3s cluster
- 

## Setting up a k3s cluster
We are setting up a cluster with 1 server node (control plane) and 4 agent nodes (kubelet, etc.)
```
k3d cluster create k8s-demo \
  --servers 1 \
  --agents 3 \
  --port "8080:80@loadbalancer" \
  --k3s-arg "--disable=traefik@server:0"
```
Verify
```
kubectl get nodes
kubectl get pods -A
```
Current pods
```
NAME                    STATUS   ROLES                  AGE     VERSION
k3d-k8s-demo-agent-0    Ready    <none>                 3m37s   v1.31.5+k3s1    <------
k3d-k8s-demo-agent-1    Ready    <none>                 3m37s   v1.31.5+k3s1    <------
k3d-k8s-demo-agent-2    Ready    <none>                 3m37s   v1.31.5+k3s1    <------
k3d-k8s-demo-server-0   Ready    control-plane,master   3m43s   v1.31.5+k3s1    <------
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-ccb96694c-4zx2d                   1/1     Running   0          3m32s
kube-system   local-path-provisioner-5cf85fd84d-bf6r2   1/1     Running   0          3m32s
kube-system   metrics-server-5985cbc9d7-j8tqd           1/1     Running   0          3m32s
```
Check contexts
```
kubectl config get-contexts
kubectl config current-context
```
Current contexts
```
CURRENT   NAME           CLUSTER        AUTHINFO             NAMESPACE
*         k3d-k8s-demo   k3d-k8s-demo   admin@k3d-k8s-demo   
```

## Appendix
### Dependencies
Docker
```
sudo apt-get remove -y docker docker-engine docker.io containerd runc || true # optional

# Update registry
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

# Install
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
sudo docker run --rm hello-world
```

Kubernetes
```
# Update registry
sudo apt-get update
sudo apt-get install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 0644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# Install
sudo apt-get install -y kubectl

# Verify
kubectl version --client
```

K3s
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```