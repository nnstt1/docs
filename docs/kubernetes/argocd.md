# ArgoCD

https://argoproj.github.io/argo-cd/getting_started/

```bash
# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# CLI
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
sudo curl -SL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd

kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
argocd login 192.168.2.243

Username: admin
Password: 
'admin' logged in successfully
Context '192.168.2.243' updated

argocd account update-password
*** Enter current password: 
*** Enter new password: 
*** Confirm new password: 
Password updated
Context '192.168.2.243' updated
```
