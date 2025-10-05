

---

````markdown
# Kubernetes / Minikube Quick Reference & Demo Commands

---

## Prerequisites
- `kubectl` configured to your Minikube cluster  
- `minikube` running (`minikube start`)  
- For Windows PowerShell: prefer single-line commands or proper quoting (examples below)

---

## Cluster & Node Health
Show pods, nodes, and extended info:
```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods --all-namespaces

kubectl get node
kubectl get node -o wide
````

---

## Deployments / Services (nginx example)

Expose nginx Deployment via NodePort or LoadBalancer:

```bash
# Expose as NodePort (explicit node-port 30080)
kubectl expose deployment nginx-deployment \
  --type=NodePort --name=nginx-nodeport --port=80 --target-port=80 --node-port=30080

# Expose as LoadBalancer (Minikube will map this via minikube tunnel or minikube service)
kubectl expose deployment nginx-deployment --type=LoadBalancer --port=80
```

View service details and open in browser (Minikube):

```bash
kubectl get svc
kubectl describe svc nginx-nodeport

# Get the minikube mapped URL (NodePort / LoadBalancer)
minikube service nginx-nodeport --url
minikube service nginx-deployment --url
```

To route LoadBalancer services on Windows (Minikube):

```bash
# Run in Admin PowerShell (keeps an active tunnel)
minikube tunnel
```

---

## Manage nginx content (ConfigMap + rollout)

Create a ConfigMap from a local `index.html`, mount via Deployment, then restart:

```bash
# Create ConfigMap from a local file
kubectl create configmap nginx-index --from-file=index.html

# Restart nginx Deployment to pick up new ConfigMap-mounted content
kubectl rollout restart deployment nginx-deployment
```

Extract the rendered index from a pod (ensure local outputs directory exists):

```bash
mkdir -p outputs      # Git Bash / Linux
mkdir outputs         # PowerShell / Windows

kubectl exec web-0 -- cat /usr/share/nginx/html/index.html > outputs/web-0-index.txt
```

List pods by label:

```bash
kubectl get pods -l app=web
```

---

## Debugging & Inspection

Describe or inspect services/pods:

```bash
kubectl describe svc nginx-nodeport
kubectl describe pod <pod-name>
kubectl logs <pod-name>                      # container logs
kubectl exec -it <pod-name> -- /bin/sh       # get a shell inside a pod
```

---

## Test Connectivity Inside the Cluster

Run a temporary BusyBox pod to test DNS / HTTP:

```bash
kubectl run -it testpod --rm --image=busybox:1.28 --restart=Never -- /bin/sh
# Then inside the pod:
nslookup php-apache
wget -S --header="User-Agent: Mozilla/5.0" -O- http://php-apache
```

> Note: Using a custom User-Agent avoids Apache 403 responses from simple clients.

---

## Deploy / Manage PHP-Apache + HPA

Apply your YAML (deployment, service, HPA):

```bash
kubectl apply -f 03-php-hpa.yaml
kubectl get deploy
kubectl get svc
kubectl get hpa
```

Inspect pods and logs if stuck:

```bash
kubectl get pods -l app=php-apache
kubectl describe pod -l app=php-apache
kubectl logs -l app=php-apache

# Delete a stuck pod — Deployment will recreate it
kubectl delete pod -l app=php-apache
```

Delete php-apache resources:

```bash
kubectl delete deploy php-apache
kubectl delete svc php-apache
kubectl delete hpa php-apache-hpa

# Or delete everything defined in a file
kubectl delete -f 03-php-hpa.yaml
```

---

## Load Generation (Inside Cluster)

**Interactive method** (recommended):

```bash
kubectl run loadgen --image=busybox:1.28 --restart=Never -- sleep 3600
kubectl exec -it loadgen -- /bin/sh

# inside the pod:
while true; do
  wget -q --header="User-Agent: Mozilla/5.0" -O- http://php-apache >/dev/null && echo "[SUCCESS]" || echo "[FAIL]"
  sleep 1
done
```

**One-liner for PowerShell**:

```powershell
kubectl run -it loadgen --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q --header='User-Agent: Mozilla/5.0' -O- http://php-apache >/dev/null && echo '[SUCCESS] php-apache responded' || echo '[FAIL] php-apache not reachable'; sleep 1; done"
```

**One-liner for Git Bash / Linux shell**:

```bash
kubectl run -it loadgen --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c 'while true; do wget -q --header="User-Agent: Mozilla/5.0" -O- http://php-apache >/dev/null && echo "[SUCCESS]" || echo "[FAIL]"; sleep 1; done'
```

> Tip: On Windows/Git Bash, if you see `ContainerCannotRun` errors, use the create-then-exec approach to avoid path/quoting issues.

---

## Watch HPA / Autoscaling

Run loadgen and watch HPA and pods:

```bash
kubectl get hpa -w
kubectl get pods -l app=php-apache -w
```

Stop load generator when done:

```bash
Ctrl+C         # inside interactive session
kubectl delete pod loadgen
```

---

## Cleanup / Reset

Delete resources when finished:

```bash
kubectl delete -f 03-php-hpa.yaml
kubectl delete deployment nginx-deployment
kubectl delete svc nginx-nodeport nginx-deployment
kubectl delete pod loadgen testpod* --ignore-not-found
```

---

## Helpful Tips / Gotchas

* **Use Service DNS** inside cluster (`php-apache`) instead of Pod IPs. Pod IPs change frequently.
* If you see `ImagePullBackOff`, make sure to use a valid image tag (`php:7.4-apache` or `php:8.2-apache`).
* On Windows PowerShell, handle quoting carefully; single quotes `'...'` or the create-then-exec method are safer for complex commands.
* `wget -S --header="User-Agent: Mozilla/5.0" -O- http://service` avoids Apache 403 errors for simple clients.
* Always create output directories before redirecting pod data to host files: `mkdir outputs`.

##Author: Abhishek-Mishra
