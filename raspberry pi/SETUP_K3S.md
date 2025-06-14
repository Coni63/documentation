# Setting Up K3S on the PI

## Prerequisites

Ensure your Raspberry Pi has:

- SSH access

## Step 1: Update the PI

```sh
sudo apt-get update
sudo apt-get upgrade
```

## Step 2: Install dependancies

```sh
sudo apt install -y curl wget
```

## Step 3: Allow cgroups

```sh
echo "$(cat filename) cgroup_memory=1 cgroup_enable=memory" > /boot/firmware/cmdline.txt

sudo reboot
```

## Step 4: Install K3S

```sh
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s
```

## Step 5: Check that it's ok

```sh
sudo kubectl get nodes
```

## Step 6: If you have a custom registry:

Edit `/etc/rancher/k3s/registries.yaml` to add your registry and then restart the service `sudo systemctl restart k3s`

```txt
# /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.1.19:5000": # Remplacez par l'IP RÉELLE de votre machine Windows
    endpoint:
      - "http://192.168.1.19:5000"
configs:
  "192.168.1.19:5000":
    tls:
      insecure_skip_verify: true # Très important pour un registre HTTP non sécurisé
```

To run a local registry use:

```bash
docker run -d -p 5000:5000 --name registry registry:3
```

## Step 7: If you have issues to user **kubectl** without sudo

```sh
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chmod 644 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

## INFO: Applications

- Config of apps will be set in `~/K3S/`
- Important to build image for linux/arm64 using `docker buildx build --platform linux/arm64 -t 192.168.1.19:5000/app_name:latest .`

# Useful `kubectl` Commands for Deploying `demo-deployment.yaml`

## 🚀 Deploy or Update the Deployment

```bash
kubectl apply -f /home/pi/demo-deployment.yaml
```

## 🔄 Restart the Deployment (e.g., to reload environment variables)

```bash
kubectl rollout restart deployment demo-service-deployment
```

## 📦 Get Pods for the Deployment

```bash
kubectl get pods -l app=demo-service
```

## 📄 View Logs of the Current Pod

```bash
kubectl logs <pod-name>
```

## 💥 View Logs of the Previous Container (if it crashed)

```bash
kubectl logs -p <pod-name>
```

## 🕵️‍♂️ Inspect Pod Details and Events (e.g., image pull errors)

```bash
kubectl describe pod <pod-name>
```

## 📈 Monitor Deployment Status

```bash
kubectl rollout status deployment demo-service-deployment
```

## ❌ Delete the Deployment

```bash
kubectl delete -f /home/pi/demo-deployment.yaml
```

## 🧪 Port Forward to Access Service Locally

```bash
kubectl port-forward pod/<pod-name> 8080:80
```

## 🧼 Clean Up Completed or Evicted Pods

```bash
kubectl delete pod $(kubectl get pods --field-selector=status.phase==Succeeded -o name)
kubectl delete pod $(kubectl get pods --field-selector=status.phase==Failed -o name)
```
