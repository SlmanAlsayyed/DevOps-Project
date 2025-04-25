RKE2 Kubernetes Cluster Setup Guide
This guide provides detailed instructions for setting up an RKE2 Kubernetes cluster, including server and worker node installation, K9s for cluster management, MetalLB for load balancing, Uptime Kuma for monitoring, NGINX Ingress for routing, and deploying a Flask demo application using a Helm chart. It also covers pushing a Docker image to Docker Hub and resetting the cluster.
For additional resources, visit the GitHub repository.
Table of Contents

Install RKE2 Server
Install RKE2 Worker Node
Install K9s
Install MetalLB
Install Uptime Kuma
Install NGINX Ingress
Push Docker Image to Docker Hub
Deploy Flask Demo App with Helm
Reset RKE2 Cluster


Install RKE2 Server
1. Download and Install RKE2
curl -sfL https://get.rke2.io | sh -

2. Enable and Start RKE2 Server
systemctl enable rke2-server.service
systemctl start rke2-server.service

3. Monitor Logs
journalctl -u rke2-server -f

4. Configure Kubeconfig Permissions
sudo chown -h slman:slman ~/.kube/config

5. Retrieve Node Token
sudo cat /var/lib/rancher/rke2/server/node-token

6. Configure kubectl
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
source ~/.bashrc

7. Verify Cluster
kubectl get nodes


Install RKE2 Worker Node
1. Install RKE2 Agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service

2. Configure Agent
Create and edit the configuration file:
mkdir -p /etc/rancher/rke2/
nano /etc/rancher/rke2/config.yaml

Add the following to config.yaml:
server: https://<MASTER_NODE_IP>:9345
token: <YOUR_NODE_TOKEN>

3. Start RKE2 Agent
systemctl start rke2-agent.service


Install K9s
1. Download K9s
wget https://github.com/derailed/k9s/releases/download/v0.32.5/k9s_Linux_amd64.tar.gz

2. Extract and Install
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/
sudo chmod +x /usr/local/bin/k9s

3. Verify Installation
k9s version

4. Run K9s
KUBECONFIG=/etc/rancher/rke2/rke2.yaml k9s

To persist the kubeconfig:
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc
source ~/.bashrc


Install MetalLB
1. Apply MetalLB Manifest
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

2. Verify Pods
kubectl get pods -n metallb-system

3. Configure IPAddressPool
Create metallb-ip-pool.yaml:
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.220-192.168.1.235
  autoAssign: true
  avoidBuggyIPs: false

Apply it:
kubectl apply -f metallb-ip-pool.yaml

4. Configure L2Advertisement
Create metallb-l2.yaml:
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
  nodeSelectors:
  - matchLabels:
      kubernetes.io/hostname: slman11-c36bde5a

Apply it:
kubectl apply -f metallb-l2.yaml

5. Verify Configuration
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system


Install Uptime Kuma
1. Prepare Storage
sudo mkdir -p /var/lib/rancher/rke2/storage
sudo chmod -R 777 /var/lib/rancher/rke2/storage
sudo chown -R nobody:nogroup /var/lib/rancher/rke2/storage

2. Apply Uptime Kuma Manifest
Create uptime-kuma.yaml:
apiVersion: v1
kind: Namespace
metadata:
  name: uptime-kuma
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: uptime-kuma-pv
  namespace: uptime-kuma
spec:
  capacity:
    storage: 512Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  local:
    path: /var/lib/rancher/rke2/storage
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - slman11-c36bde5a
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uptime-kuma-pvc
  namespace: uptime-kuma
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 512Mi
  storageClassName: manual
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
      - name: uptime-kuma
        image: louislam/uptime-kuma:latest
        ports:
        - containerPort: 3001
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: uptime-kuma-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  selector:
    app: uptime-kuma
  ports:
  - protocol: TCP
    port: 3001
    targetPort: 3001
  type: LoadBalancer

Apply it:
kubectl apply -f uptime-kuma.yaml

3. Verify Installation
kubectl get pods -n uptime-kuma
kubectl get svc -n uptime-kuma
kubectl get pv,pvc -n uptime-kuma


Install NGINX Ingress
1. Apply NGINX Ingress Manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml

2. Verify Pods
kubectl get pods -n ingress-nginx

3. Expose with MetalLB
Edit the service to use LoadBalancer:
export EDITOR=nano
kubectl edit svc ingress-nginx-controller -n ingress-nginx

Verify:
kubectl get svc -n ingress-nginx

4. Deploy Sample Applications
Create nginx-app.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80

Create apache-app.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: apache-service
spec:
  selector:
    app: apache
  ports:
  - port: 80
    targetPort: 80

Apply them:
kubectl apply -f nginx-app.yaml
kubectl apply -f apache-app.yaml

5. Create Ingress
Create ingress.yaml:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: domain.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: apache-service
            port:
              number: 80
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80

Apply it:
kubectl apply -f ingress.yaml

6. Test the Setup
Add the Ingress IP to /etc/hosts:
sudo echo "<INGRESS_EXTERNAL_IP> domain.com" >> /etc/hosts

Test routing:
curl http://domain.com/foo
curl http://domain.com/bar


Push Docker Image to Docker Hub
1. Clone and Build Flask App
git clone https://github.com/nexgtech/flask-demo-app.git
cd flask-demo-app
docker build -t slmann/flask-demo-app:latest .

2. Verify Image
docker images

3. Test Locally
docker run -d -p 5000:8080 --name flask-demo slmann/flask-demo-app:latest
curl http://localhost:5000/

4. Push to Docker Hub
docker login
docker push slmann/flask-demo-app:latest


Deploy Flask Demo App with Helm
1. Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
helm version

2. Create Helm Chart
helm create flask-demo-app

3. Modify Helm Files
Update the following in flask-demo-app/:
Chart.yaml:
apiVersion: v2
name: flask-demo-app
description: A Helm chart for deploying a simple Flask demo app on Kubernetes
version: 0.1.0
appVersion: "1.0.0"

values.yaml:
replicaCount: 1

image:
  repository: slmann/flask-demo-app
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  limits:
    cpu: "0.5"
    memory: "512Mi"
  requests:
    cpu: "0.2"
    memory: "256Mi"

serviceAccount:
  create: true
  annotations: {}
  automount: true
  name: ""

ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: "domain.com"
      paths:
        - path: /foo
          pathType: Prefix
          serviceName: apache-service
          servicePort: 80
        - path: /login
          pathType: Prefix
          serviceName: flask-demo-app-service
          servicePort: 80
        - path: /bar
          pathType: Prefix
          serviceName: nginx-service
          servicePort: 80
  tls: []

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

templates/deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}

templates/service.yaml:
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-service
  labels:
    app: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    protocol: TCP
  selector:
    app: {{ .Chart.Name }}

templates/ingress.yaml:
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "flask-demo-app.fullname" . }}
  labels:
    {{- include "flask-demo-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- with .Values.ingress.className }}
  ingressClassName: {{ . }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - {{ if .host }}host: {{ .host | quote }}{{ else }}{{- end }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- with .pathType }}
            pathType: {{ . }}
            {{- end }}
            backend:
              service:
                name: {{ .serviceName }}
                port:
                  number: {{ .servicePort }}
          {{- end }}
    {{- end }}
{{- end }}

4. Deploy Helm Chart
helm install flask-demo-app .

5. Verify Deployment
kubectl get pods
kubectl get svc
curl <CLUSTER_IP>

6. Use LoadBalancer
Update values.yaml to set service.type: LoadBalancer, then:
helm upgrade flask-demo-app .

Verify:
kubectl get svc
curl <EXTERNAL_IP>


Reset RKE2 Cluster
sudo rke2 server --cluster-reset
sudo systemctl restart rke2-server.service


References

RKE2 Documentation
MetalLB Documentation
NGINX Ingress Controller
Helm Documentation
GitHub Repository

