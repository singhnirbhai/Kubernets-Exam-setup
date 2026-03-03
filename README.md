# Kubernets- Exam Environmet setup 


```
exam/
 ├── setup.sh
 ├── manifest/
 ├── solutions
 └── evaluate.sh
```
```bash
mkdir exam
```
```bash
touch setup.sh evaluate.sh
```
```bash
mkdir -p /exam/manifest
```

## Shell script deploy kind cluster on machine 
```bash
vi kind-cluster.sh
```
```bash
#!/bin/bash

apt update -y
apt install -y docker.io
systemctl enable docker
systemctl start docker

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl


curl -Lo /usr/local/bin/kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x /usr/local/bin/kind
```

```bash
chmod +x kind-cluster.sh
```
# Create cluster
```bash
vi kind-config.yaml
```
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: exam-cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-config.yaml
```
```bash
vim setup.sh
```
```
#!/bin/bash

echo "Creating namespaces..."
kubectl create ns production
kubectl create ns dev-team
kubectl create ns limited-ns

echo "Applying ResourceQuota..."
kubectl apply -f manifest/quota.yaml

echo "Applying LimitRange..."
kubectl apply -f manifest/limitrange.yaml

echo "Deploying broken applications..."
kubectl apply -f manifest/web-app.yaml
kubectl apply -f manifest/backend.yaml
kubectl apply -f manifest/imagepull.yaml
kubectl apply -f manifest/crashloop.yaml
kubectl apply -f manifest/pvc.yaml
kubectl apply -f manifest/configmap.yaml
kubectl apply -f manifest/node-label.yaml
kubectl apply -f manifest/limit-deployment.yaml
kubectl apply -f manifest/exam-pvc.yaml
kubectl apply -f manifest/exam-deployment.yaml
kubectl label nodes exam-cluster-worker disktype=sdd
kubectl label nodes exam-cluster-worker2 disktype=sdd
kubectl cordon exam-cluster-worker2 

echo "Environment Ready. Chaos Loaded."
```
## Questions yml file
## Question 1
```bash
cat <<EOF > quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.memory: "100Mi"
    limits.memory: "200Mi"
    pods: "2"
EOF
```
## Question 2
```bash
cat <<EOF > web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            memory: "300Mi"
          limits:
            memory: "400Mi"
EOF
```
## Question 3
```bash
cat <<EOF> backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-back
  template:
    metadata:
      labels:
        app: web-back
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  selector:
    app: wrong-label
  ports:
  - port: 80
    targetPort: 80
EOF
```
## Question 4
```bash
cat <<EOF> imagepull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image
  template:
    metadata:
      labels:
        app: image
    spec:
      containers:
      - name: nginx
        image: nginx:wrongtag
EOF
```
## Question 5
```bash
cat <<EOF>  crashloop.yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
spec:
  containers:
  - name: busy
    image: busybox
    command: ["sleep"]
EOF
```
## Question 6
```bash
cat <<EOF> pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```
## Question 7
```bash

cat <<EOF> limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit
  namespace: limited-ns
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
    defaultRequest:
      cpu: "100m"
EOF
```
## Question 8
```bash
cat <<EOF> configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: site-config
data:
  message: Old Message
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config
  template:
    metadata:
      labels:
        app: config
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```
## Question 9
```bash
cat <<EOF> node-label.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: node-test
  template:
    metadata:
      labels:
        app: node-test
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: nginx
        image: nginx
EOF
```
## Question 10
```bash
cat <<EOF> limit-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: limited-app
  namespace: limited-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: limited
  template:
    metadata:
      labels:
        app: limited
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```
## Question 11
```bash

cat <<EOF> exam-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exam-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
EOF
```
## Question 12
```bash
cat <<EOF> exam-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exam-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: exam
  template:
    metadata:
      labels:
        app: exam
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: exam-storage
      volumes:
      - name: exam-storage
        persistentVolumeClaim:
          claimName: exam-pvc
EOF
```
```bash
vi exam-check.sh
```
```
#!/bin/bash

TOTAL=0

echo "-----------------------------"
echo "KUBERNETES EXAM EVALUATION"
echo "-----------------------------"

#!/bin/bash

TOTAL=0

echo "-----------------------------"
echo "KUBERNETES EXAM EVALUATION"
echo "-----------------------------"

# Q1 - web-app Running (Quota fixed)
RUNNING=$(kubectl get pods -n production -l app=web --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
if [ "$RUNNING" -ge 2 ]; then
  echo "Q1 PASS - web-app running"
  TOTAL=$((TOTAL+10))
else
  echo "Q1 FAIL - web-app not fixed"
fi

# Q2 - Secret created
SECRET=$(kubectl get secret site-secret -n production --no-headers 2>/dev/null | wc -l)
if [ "$SECRET" -eq 1 ]; then
  echo "Q2 PASS - secret exists"
  TOTAL=$((TOTAL+10))
else
  echo "Q2 FAIL - secret missing"
fi

# Q3 - backend service endpoints exist
ENDPOINTS=$(kubectl get endpoints backend-service -n production -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)
if [ -n "$ENDPOINTS" ]; then
  echo "Q3 PASS - service fixed"
  TOTAL=$((TOTAL+10))
else
  echo "Q3 FAIL - service not routing"
fi

# Q4 - ResourceQuota in dev-team
QUOTA=$(kubectl get resourcequota -n dev-team --no-headers 2>/dev/null | wc -l)
if [ "$QUOTA" -ge 1 ]; then
  echo "Q4 PASS - ResourceQuota created"
  TOTAL=$((TOTAL+10))
else
  echo "Q4 FAIL - ResourceQuota missing"
fi

# Q5 - PVC Bound
PVC=$(kubectl get pvc data-pvc -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$PVC" = "Bound" ]; then
  echo "Q5 PASS - PVC bound"
  TOTAL=$((TOTAL+10))
else
  echo "Q5 FAIL - PVC not bound"
fi

# Q6 - ConfigMap updated
CONFIG=$(kubectl get configmap site-config -o jsonpath='{.data.message}' 2>/dev/null)
if [[ "$CONFIG" == *"Kubernetes"* ]]; then
  echo "Q6 PASS - ConfigMap updated"
  TOTAL=$((TOTAL+10))
else
  echo "Q6 FAIL - ConfigMap not updated"
fi

# Q7 - CrashLoop fixed
CRASH=$(kubectl get pod broken-app -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$CRASH" = "Running" ]; then
  echo "Q7 PASS - CrashLoop fixed"
  TOTAL=$((TOTAL+10))
else
  echo "Q7 FAIL - CrashLoop not resolved"
fi

# Q8 - ImagePullBackOff fixed
IMAGE=$(kubectl get pods -n production -l app=image-test --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
if [ "$IMAGE" -ge 1 ]; then
  echo "Q8 PASS - Image issue fixed"
  TOTAL=$((TOTAL+10))
else
  echo "Q8 FAIL - Image issue not resolved"
fi

# Q9 - limited-ns deployment running
LIMITED=$(kubectl get pods -n limited-ns --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
if [ "$LIMITED" -ge 1 ]; then
  echo "Q9 PASS - LimitRange handled"
  TOTAL=$((TOTAL+10))
else
  echo "Q9 FAIL - LimitRange issue unresolved"
fi

# Q10 - exam-app running with PVC
EXAM=$(kubectl get pods -n default -l app=exam-app --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
if [ "$EXAM" -ge 2 ]; then
  echo "Q10 PASS - PV/PVC attached"
  TOTAL=$((TOTAL+10))
else
  echo "Q10 FAIL - Storage scenario incomplete"
fi

# Q11 - Node uncordoned
CORDONED=$(kubectl get nodes | grep SchedulingDisabled | wc -l)
if [ "$CORDONED" -eq 0 ]; then
  echo "Q11 PASS - Node schedulable"
  TOTAL=$((TOTAL+10))
else
  echo "Q11 FAIL - Node still cordoned"
fi

# Q12 - NodeSelector issue fixed
NODE_APP=$(kubectl get pods -n production -l app=node-test --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
if [ "$NODE_APP" -ge 2 ]; then
  echo "Q12 PASS - Node selector issue resolved"
  TOTAL=$((TOTAL+10))
else
  echo "Q12 FAIL - Node selector not fixed"
fi

echo "-----------------------------"
echo "TOTAL SCORE: $TOTAL / 120"
echo "-----------------------------"
```

