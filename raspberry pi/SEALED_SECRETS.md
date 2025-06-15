# 🛡️ Sealed Secrets Cheatsheet (Kubernetes)

## 📦 1. Installer `kubeseal` (CLI)

```bash
cd ~/k3s
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.30.0/kubeseal-0.30.0-linux-arm64.tar.gz"
tar -xvzf kubeseal-0.30.0-linux-arm64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
rm kubeseal-0.30.0-linux-arm64.tar.gz
```

---

## 🚀 2. Déployer le contrôleur Sealed Secrets sur le cluster

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
```

🔍 Vérifier que le service est bien déployé :

```bash
kubectl get svc -A | grep sealed
```

---

## 🔐 3. Créer un Secret classique (en clair, temporaire)

```bash
kubectl create secret generic kubeseal-secret \
  --from-literal=password='Password' \
  --dry-run=client -o yaml > my-secret.yaml
```

---

## 🔏 4. Générer le Sealed Secret (encrypté)

```bash
kubeseal \
  --controller-namespace kube-system \
  --namespace default \                     # <- namespace cible du secret
  --format yaml < my-secret.yaml > my-sealed-secret.yaml
```

---

## 📥 5. Appliquer le Sealed Secret dans le cluster

```bash
kubectl apply -f my-sealed-secret.yaml -n default
```

---

## 🧪 6. Vérifier le Secret déchiffré par le contrôleur

```bash
kubectl get secrets -n default
kubectl get secret kubeseal-secret -n default -o yaml
```

---

## 🧠 Notes

- Le namespace passé à `--namespace` dans `kubeseal` doit être **celui où tu veux utiliser le secret**.
- Le contrôleur déchiffre automatiquement les SealedSecrets et crée les vrais `Secret` K8s.
- Le `SealedSecret` est **sûr à commit dans Git**, contrairement à un `Secret` normal.

---

✅ Tu es maintenant prêt à versionner tes secrets de façon sécurisée en GitOps 🚀
