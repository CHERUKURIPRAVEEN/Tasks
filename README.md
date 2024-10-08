# Tasks
1. Create a GCP Account
If you don't have one already, go to GCP Console and create a Google Cloud account.

After account creation:

Set up billing.
Enable APIs (Compute Engine, Kubernetes Engine, Cloud DNS, etc.).

2. Set up GCloud SDK on Your Laptop
Install gcloud SDK.
Authenticate and configure the GCP project:
bash
Copy code
gcloud auth login
gcloud config set project [PROJECT_ID]

3. Bring Up a GKE Cluster Using Terraform
Create a Terraform configuration file to provision a GKE cluster:

#### Create a directory for your Terraform files.
* Inside the directory, create main.tf and variables.tf.

main.tf
```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_container_cluster" "gke_cluster" {
  name     = "my-gke-cluster"
  location = var.region

  node_config {
    machine_type = "e2-medium"
  }

  initial_node_count = 3
}

resource "google_container_node_pool" "primary_nodes" {
  cluster    = google_container_cluster.gke_cluster.name
  location   = google_container_cluster.gke_cluster.location
  node_count = 3

  node_config {
    machine_type = "e2-medium"
  }
}
````

variable.tf
```hcl
variable "project_id" {
  type = string
}

variable "region" {
  type    = string
  default = "us-central1"
}
```

#### Run Terraform commands:
terraform init
terraform plan
terraform apply -auto-approve

```hcl
gcloud container clusters get-credentials my-gke-cluster --region us-central1
```

#### Install Nginx Ingress Controller on the GKE Cluster
* Use Helm to install the Nginx ingress controller on GKE:

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx

#### Create a VM on GCP and Install Nginx as Reverse Proxy

```gcp
gcloud compute instances create nginx-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --tags=http-server,https-server
```
```bash
gcloud compute ssh nginx-vm
sudo apt update
sudo apt install nginx
```
#### Configure Nginx as a reverse proxy:

```bash
sudo vi /etc/nginx/sites-available/default
```
#### Replace the content with the following:
```bash
server {
    listen 80;
    listen [::]:80;

    server_name <your_domain>;
        
    location / {
        proxy_pass http://<nginx-ingress-controller-ip>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Deploy Hello World Application on GKE
#### Create a Hello World Deployment and Service:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30001
```
#### Create Ingress Resource to Route Traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
spec:
  rules:
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80

```
#### Deploy application by using below commands
```
kubectl apply -f hello-world.yaml
kubectl apply -f ingress.yaml
```

Route Traffic via Nginx Reverse Proxy to GKE

```bash
curl http://<vm-ip>
```
