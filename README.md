
# Manual setup steps
- Set up a k3s cluster
- Set up add-ons
- Set up Gateway
- Set up Flux

## Setting up a k3s cluster
We are setting up a cluster with 1 server node (control plane) and 4 agent nodes (kubelet, etc.)
### Create cluster
```
k3d cluster create k8s-demo \
  --servers 1 \
  --agents 3 \
  --port "8080:80@loadbalancer" \
  --k3s-arg "--disable=traefik@server:0"
```

### Current pods
```
kubectl get nodes
kubectl get pods -A
```
Output
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

### Check contexts
```
kubectl config get-contexts
kubectl config current-context
```
Output
```
CURRENT   NAME           CLUSTER        AUTHINFO             NAMESPACE
*         k3d-k8s-demo   k3d-k8s-demo   admin@k3d-k8s-demo   
```

## Setting up Add-ons
We are setting up Envoy Gateway, monitoring (Prometheus + Grafana), logging (Grafana + Loki/Alloy) and metrics-server here. 

### Setting up Helm
```
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Creating namespaces
```
kubectl create namespace monitoring   --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace logging      --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace demo         --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace gateway      --dry-run=client -o yaml | kubectl apply -f -
```

### Install metrics-server
Note: This step is for reference. In my k3s setup metrics-server was already installed.
```
# Install
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system

# Verify
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -A | head
```

### Install Prometheus + Grafana
```
# Install
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# Verify
kubectl -n monitoring get pods

# Port forward (forward tunnel, only for testing)
kubectl -n monitoring port-forward --address 0.0.0.0 svc/monitoring-grafana 3000:80

# Get admin password
kubectl -n monitoring get secret monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

### Install Loki + log collector
```
# Install
helm install alloy grafana/alloy --namespace logging
helm upgrade --install loki grafana/loki -n logging -f infra/logging/loki/values.yaml
helm upgrade --install alloy grafana/alloy -n logging -f infra/logging/alloy/values.yaml
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring \
  -f infra/monitoring/kube-prometheus-stack/loki-source.yaml
  
# Verify
kubectl -n logging get pods,svc,pvc

# Confirms Alloy is running; after a minute you should see logs being shipped.
kubectl -n logging get pods
kubectl -n logging logs deploy/alloy --tail=50
```

### Install Envoy Gateway
```
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.2 \
  -n envoy-gateway-system \
  --create-namespace

# Wait until ready
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

# Apply default routing
kubectl apply -f https://github.com/envoyproxy/gateway/releases/download/v1.6.2/quickstart.yaml -n default

# Set up proxy
kubectl apply -f infra/gateway/envoyproxy/nodeport.yaml
kubectl apply -f infra/gateway/envoy-gateway/gatewayclass.yaml


# Verify
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
```

## Setting up Gateway
### Fetch Gateway class name
```
# Note: Gateway class is needed in the `infra/gateway/envoy-gateway/gateway.yaml`
GWC_NAME="$(kubectl get gatewayclass -o jsonpath='{.items[0].metadata.name}')"
echo "Using GatewayClass: $GWC_NAME"
```

### Apply gateway and routing config
```
# Apply
kubectl apply -f infra/gateway/envoy-gateway/gateway.yaml  # Update `gatewayClassName` based on above
kubectl apply -f infra/gateway/routes/grafana.yaml

# Add port forwarding
GATEWAY_SERVICE=$(kubectl -n envoy-gateway-system get svc \
  -l gateway.envoyproxy.io/owning-gateway-name=main-gw,gateway.envoyproxy.io/owning-gateway-namespace=gateway \
  -o jsonpath='{.items[0].metadata.name}')
NODEPORT=$(kubectl -n envoy-gateway-system get svc "$GATEWAY_SERVICE" \
  -o jsonpath='{.spec.ports[0].nodePort}')
echo "Envoy NodePort: $NODEPORT"
k3d cluster edit k8s-demo --port-add "8081:${NODEPORT}@loadbalancer"

# Verify
curl -I -H "Host: grafana.local" http://127.0.0.1:8081/

kubectl -n gateway get gateway main-gw -o wide
kubectl -n gateway describe gateway main-gw | sed -n '/Status:/,$p'

kubectl -n monitoring get httproute grafana-route -o wide
kubectl -n monitoring describe httproute grafana-route | sed -n '/Status:/,$p'
```

## Setting up Flux
### Bootstrapping Flux
```
export GITHUB_TOKEN="<your_pat>"
export GITHUB_OWNER="robert-fekete"
export GITHUB_REPO="KubernetesDemo"

flux bootstrap github \
  --owner="$GITHUB_OWNER" \
  --repository="$GITHUB_REPO" \
  --branch=main \
  --path=clusters/dev \
  --personal \
  --token-auth \
  --components-extra=image-reflector-controller,image-automation-controller

# Verify
flux check
kubectl -n flux-system get pods
```

### Configuring Flux + Kustomization
```
# Set up kustomization.yaml and helmrelease.yaml files

# Commit to Git
git add -A
git commit -m "Configuring Flux + Kustomization"
git push origin main

# Reconcile Flux
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization platform -n flux-system

# Verify
flux get kustomizations -A         # Check if READY=True
flux check                         # Check if all checks passed
kubectl -n flux-system get pods    # Check if all Flex related pods are running
flux get helmreleases -A           # Check if READY=True
flux get sources all -A            # Check if READY=True
kubectl get pods -A                # Check if all pods are running

# Troubleshoot if (when?) needed
flux get all -A
kubectl -n flux-system get events --sort-by=.lastTimestamp | tail -n 50
kubectl -n flux-system logs deploy/kustomize-controller --tail=200
kubectl -n flux-system logs deploy/helm-controller --tail=200
kubectl -n flux-system logs deploy/source-controller --tail=200
```

# Flux managed operations

## Health check
```
flux check                   # Check if all check passes
flux get kustomizations -A   # Check if READY=True
flux get helmreleases -A     # Check if READY=True
```

## Diff agains local manifests
```
flux diff kustomization platform \
  -n flux-system \
  --path "$(pwd)/infra"
```

## Make a change
```
# Change the values files under `./infa`

# Commit and push
git add -A
git commit -m "..."
git push origin main

# Reconcile (optional, flux will eventually reconcile the changes)
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization platform -n flux-system

# Get info about running reconciliation
flux get kustomization platform -n flux-system

# Verify relevant kubectl commands
```


## Bootstrap a new cluster
```
# Ensure kubectl current-context is correct
kubectl config current-context
kubectl get nodes

# Bootstrap flux
export GITHUB_TOKEN="<your_pat>"
export GITHUB_OWNER="robert-fekete"
export GITHUB_REPO="KubernetesDemo"

flux bootstrap github \
  --owner="$GITHUB_OWNER" \
  --repository="$GITHUB_REPO" \
  --branch=main \
  --path=clusters/dev \
  --personal \
  --token-auth \
  --components-extra=image-reflector-controller,image-automation-controller

# Verify
flux check                   # Check if all check passes
flux get kustomizations -A   # Check if READY=True
flux get helmreleases -A     # Check if READY=True
```

## Detect and fix drifts
```
# Detect
flux diff kustomization platform -n flux-system --path ./infra

# Fix
# Make git match what you want, commit, push, reconcile (manually or automatically)
# You might need to check the HelmRelease status as well
```

## Debugging
```
# Identify what is failing
flux get kustomizations -A
flux get helmreleases -A
flux get sources all -A

# Check events
kubectl -n flux-system get events --sort-by=.lastTimestamp | tail -n 80
kubectl -n java-demo get events --sort-by=.lastTimestamp | tail -n 80 # Useful if reconciliation is incomplete

# Get Helm manifest
helm -n java-demo get manifest java-demo-api
# Render locally
helm template java-demo-api ./charts/java-demo-api -n java-demo

# Check rollout logs
kubectl -n flux-system logs deploy/source-controller --tail=200
kubectl -n flux-system logs deploy/kustomize-controller --tail=200
kubectl -n flux-system logs deploy/helm-controller --tail=200
```

## Detecting resources not managed by Flux
```
# Inventory
kubectl -n flux-system get kustomization platform -o jsonpath='{range .status.inventory.entries[*]}{.id}{"\n"}{end}' | sort

# Flux topology
flux tree kustomization platform -n flux-system

# List all e.g. http routes with kustomize label
kubectl get httproutes -A -l kustomize.toolkit.fluxcd.io/name=platform

```


# Next steps
- What’s next after these installs?
- Create your Java API Helm chart + a GitHub Actions workflow to build/push the image to GHCR.
- Delete www.example.com route
- Flux image automation (optional but nice) to update the image tag in Git automatically.
- HPA?


# Useful commands
#### Current config context
`kubectl config current-context`

#### Listing all nodes
`kubectl get nodes -o wide`

#### Lists all pods
`kubectl get pods --all-namespace`

#### `top` for K8s
`kubectl top nodes`

#### Opening a shell in a pod
`kubectl -n <pod namespace> exec -it <pod-name> -- sh`

#### Read logs from pod
`kubectl -n logging logs <pod> -c <container> --tail=100`

# Appendix
## Dependencies
### Docker
```
sudo apt-get remove -y docker docker-engine docker.io containerd runc || true # optional

# Update sources
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

### Kubernetes
```
# Update sources
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

### K3s
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### Helm
```
# Download install script
sudo apt-get update
sudo apt-get install -y curl

curl -fsSL -o /tmp/get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 /tmp/get_helm.sh

# Optional but recommended: review what you’re about to run
sed -n '1,160p' /tmp/get_helm.sh

# Install
sudo /tmp/get_helm.sh

# Verify
helm version
```

### Flux
```
# Download install script
curl -s -o /tmp/install_flux.sh https://fluxcd.io/install.sh
chmod 700 /tmp/install_flux.sh

# Review
sed -n '1,160p' /tmp/install_flux.sh

# Install
sudo /tmp/install_flux.sh

# Verify
flux version

# Pre-flight checks
flux check --pre
