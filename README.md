## Install MicroK8s (Local Testing)

```bash
sudo snap install microk8s --classic
microk8s status --wait-ready
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
microk8s status
sudo snap alias microk8s.kubectl kubectl
kubectl get nodes
microk8s enable dns hostpath-storage ingress
```

Expose ingress via NodePort:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-microk8s-controller
  namespace: ingress
spec:
  type: NodePort
  selector:
    name: nginx-ingress-microk8s
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
```
## Install helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```
## Copy kubeconfig
```bash
mkdir ~/.kube ;  microk8s config > ~/.kube/config
```

## Install Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm repo ls
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd
helm -n argocd ls
```

## Install Argo CD CLI on Ubuntu

```bash
# Download the latest Argo CD CLI binary
wget https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 -O argocd

# Make it executable
chmod +x argocd

# Move it into your PATH
sudo mv argocd /usr/local/bin/

# Verify installation
argocd version
```

## Run Argo CD on nodeport
```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "nodePort": 30081}]}}'
```

## Login to Argo CD

Once the CLI is installed, you can log in to your Argo CD server:

```bash
argocd login <ARGOCD_SERVER> \
  --username admin \
  --password <initial-password>
```

- `<ARGOCD_SERVER>` → the external IP or DNS name of your Argo CD API server (e.g., `localhost:8080` if port‑forwarded, or nodeport or the LoadBalancer DNS on EKS).  
- `<initial-password>` → by default, it’s the name of the Argo CD server pod:
  ```bash
  kubectl -n argocd get pods
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

## Output for referene
```bash
prayag@devops-vm:~$ argocd login 10.10.0.2:30081 \
  --username admin \
  --password <changeme> \
  --insecure
'admin:login' logged in successfully
Context '10.10.0.2:30081' updated
prayag@devops-vm:~$
```

## Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Install plugin:
```bash
brew install argoproj/tap/kubectl-argo-rollouts
```

## Manifests for Blue‑Green Rollout

### `rollout.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo-rollout
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=Hello from GREEN"
        - "-listen=:80"
        ports:
        - containerPort: 80
  strategy:
    blueGreen:
      activeService: demo-service
      previewService: demo-preview
      autoPromotionEnabled: false
```

### `demo-service.yaml` (Active)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
```

### `demo-preview.yaml` (Preview)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-preview
spec:
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
```

### `demo-ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-service
            port:
              number: 80
  - host: preview.demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-preview
            port:
              number: 80
```

## Apply and Test

```bash
kubectl apply -f rollout.yaml
kubectl apply -f demo-service.yaml
kubectl apply -f demo-preview.yaml
kubectl apply -f demo-ingress.yaml
```

Add host entries:
```bash
echo "10.10.0.2 demo.local" | sudo tee -a /etc/hosts
echo "10.10.0.2 preview.demo.local" | sudo tee -a /etc/hosts
```

Test:
```bash
curl http://demo.local:30080
Hello from BLUE   # active version

curl http://preview.demo.local:30080
Hello from GREEN  # preview version
```
## Promote / Rollback

Promote Green:
```bash
kubectl argo rollouts promote demo-rollout
curl http://demo.local:30080
Hello from GREEN
```

Rollback:
```bash
kubectl argo rollouts undo demo-rollout
curl http://demo.local:30080
Hello from BLUE
```
