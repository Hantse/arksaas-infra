# arksaas-infra

Squelette GitOps/Kubernetes pour deployer les services backend de `ark-saas-manager` sur un cluster K3s/Kubernetes avec ArgoCD et Kustomize.

## Ce repo gere

- la gateway publique et les microservices backend ArkSaas
- les placeholders de configuration non sensible et sensible
- l'entree ArgoCD (`AppProject`, `Application`, `ImageUpdater`)
- le pattern de migration via `initContainers` pour les services SQL principaux

## Ce repo ne gere pas

- le `local-agent` Windows et son flux MSI/GitHub Release
- un frontend web separe
- les dependances externes SQL Server, MongoDB, Redis, Ollama, Seq ou Jaeger
- le TLS/cert-manager de production

## Structure

- `argocd/`
  manifests ArgoCD a appliquer dans le namespace `argocd`
- `clusters/k3s/prod/`
  environnement de production Kustomize
- `clusters/k3s/prod/configs/`
  configuration non sensible
- `clusters/k3s/prod/secrets/`
  placeholders a remplacer avant tout deploiement reel
- `clusters/k3s/prod/services/`
  workloads et services Kubernetes

## Hypotheses prises

- les noms DNS internes suivent les `appsettings.Container.json` du repo applicatif, par exemple `identity-service:8081`
- la gateway reste l'unique entree publique
- `agent-service` reste a `1` replica par defaut, meme si Redis est deja prevu
- les services suivants migrent leur base via `initContainers` :
  `identity`, `admin`, `billing`, `notification`, `permission`, `server-provisioning`
- `agent-service` n'a pas ete force dans le meme pattern de migration pour rester plus proche du comportement courant de l'application

## Remplacements obligatoires avant usage

1. Remplacer `hantse` dans :
   - `argocd/arksaas-project.yaml`
   - `argocd/arksaas-prod-app.yaml`
   - `argocd/arksaas-image-updater.yaml`
   - `clusters/k3s/prod/kustomization.yaml`
2. Remplacer les valeurs placeholder dans :
   - `clusters/k3s/prod/configs/arksaas-public-config.yaml`
   - `clusters/k3s/prod/secrets/runtime-secrets.yaml`
   - `clusters/k3s/prod/secrets/firebase-admin-secret.yaml`
3. Adapter le host d'ingress dans `clusters/k3s/prod/ingress.yaml`
4. Verifier que le workflow applicatif pousse bien les images GHCR attendues

## Deploiement type

1. Pousser ce repo sur GitHub
2. Remplacer les placeholders
3. Appliquer les manifests ArgoCD :

```powershell
kubectl apply -f .\argocd\arksaas-project.yaml -n argocd
kubectl apply -f .\argocd\arksaas-prod-app.yaml -n argocd
kubectl apply -f .\argocd\arksaas-image-updater.yaml -n argocd
```

4. Verifier le rendu Kustomize localement :

```powershell
kubectl kustomize .\clusters\k3s\prod
```

5. Appliquer l'environnement avec Kustomize :

```powershell
kubectl apply -k .\clusters\k3s\prod
```

Ne pas utiliser `kubectl apply -f .\clusters\k3s\prod` sur ce repo.
Cette commande n'applique pas correctement toute l'arborescence Kustomize et peut laisser de cote des ressources comme les `ConfigMap`, `Secret` ou workloads ranges dans les sous-dossiers.

Le namespace `arksaas-prod` n'est pas versionne comme ressource Kubernetes dans ce repo.
On s'appuie sur l'option ArgoCD `CreateNamespace=true` pour eviter les erreurs de permission sur les ressources cluster-scoped de type `Namespace`.

## Authentification GHCR

Si les images GHCR restent privees, le cluster doit disposer d'un secret Docker valide avec un token GitHub qui a au minimum le scope `read:packages`.

Le secret `ghcr-pull` n'est plus gere par ce repo GitOps pour eviter qu'un placeholder n'ecrase un secret cree manuellement ou via un autre systeme de secrets.

Le plus simple est de recreer le secret directement :

```powershell
kubectl delete secret ghcr-pull -n arksaas-prod --ignore-not-found
kubectl create secret docker-registry ghcr-pull `
  --namespace arksaas-prod `
  --docker-server=ghcr.io `
  --docker-username=VOTRE_LOGIN_GITHUB `
  --docker-password=VOTRE_PAT_GITHUB `
  --docker-email=unused@example.com
```

Puis verifier :

```powershell
kubectl get secret ghcr-pull -n arksaas-prod
kubectl rollout restart deploy -n arksaas-prod
```

Notes :

- si tu utilises un PAT GitHub classique, il doit avoir `read:packages`
- si le package GHCR appartient a une organisation, il faut aussi que ce token ait acces a cette organisation
- si tu preferes eviter un secret prive, tu peux rendre les packages GHCR publics

## Notes utiles

- Les images sont taggees au SHA Git dans le repo applicatif, donc l'`ImageUpdater` attend des tags de 40 caracteres hexadecimaux.
- Les services gRPC exposent `8081`; les services HTTP exposent `8080`.
- Les secrets sont laisses en clair uniquement comme placeholders de scaffold. Avant usage reel, il vaut mieux passer sur Sealed Secrets, External Secrets ou un autre mecanisme similaire.
