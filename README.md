# Lab Infra cloud conteneur 

## Deployment de Cluster Kubernetes sur GCP à l'aide de Terraform

### Crétion de Compte GCP
A l'aide d'un compte google, souscrivez à l'offre de GCP vous octroyant 300$ de credit.

### Déploiement de GKE à l'aide de terraform
A l'aide de Terraform, vous allez déployer un cluster Kubernetes avec les caractéristiques suivantes:
* création d'un VPC dédié dans le Cloud
  * Le nom du VPC sera comme tel : `<Nom_du_projet_Google>-<Votre_prenom>`
  * Création d'un sous réseaux dédié dans ce VPC dont la plage d'adressage du sous réseaux est : `"10.10.0.0/24"`
  * Le nom du sous réseaux sera : `<Nom_du_projet_Google>-<Votre_prenom>-subnet`
* La création du Pool d'instances Worker Kubernetes:
  * Nombre de noeuds : `2`
  * Supprimer le Pool par default
  * Ce pool sera dans le sous réseaux créé dans le VPC
  * Type d'instance Google : `"n1-standard-1"`

### Accès au cluster Kubernetes créé
Sur votre machine locale ou à partir d'une machine d'administration, installez l'utilitaire `kubectl` et configurer les identifiants pour se rassurer qu'on a bien accès aux ressources Kubernetes

## Déploiement d'une application de microservices

le répertoire `voting-app` contient le code source d'une application composé de microservices

Nous nous mettons dans le contexte où nous avons développé une application de vote `voting-app`
Pour rappel, [example-voting-app](https://github.com/dockersamples/example-voting-app) est une application maintenu par la communauté Docker qui est très souvent utilisée pour faire des démo de microservices.
Il est concu en microservices avec l'architecture suivante :

![architecture](https://user-images.githubusercontent.com/59444079/202094328-30a5cb07-f33b-40bf-828a-7dc5de405bb7.png)

La stack est composée de 5 services:
* un service front `voting-app` développé en Python : Les utilisateurs peuvent y accéder pour voter en cliquant
* un service front `result-app` développé en Node.js : Les utilisateurs peuvent y accéder pour voir les résultats du vote
* un service backend `worker` qui traites les requètes et les persistent dans une base de donnée PostgreSQL
* un service de Base de données `db` PostgreSQL qui persiste la donnée
* un service de cache `redis` utilisé par le microservice `voting-app`

L'objectif de cette partie du lab est de :
1. Dockeriser cette stack
2. Créer des fichier de déploiement, services nécessaires pour avoir ce microservice déployé dans le Cluster K8s précédement créé
3. Vérifiez qu'il est bien accessible depuis une IP publique et que toutes les fonctionnalités de l'application sont opérationnelles
## GitOps

L'objectif de cette partie du lab est d'automatiser le Déploiement de la stack de microservice suivant le shéma workflow suivant :
La mise à jour des fichiers de déploiement du microservice dans le dépôt GitHub ==> déclenche le déploiement de cette nouvelle mise à jour dans le cluster Kubernetes GKE.
Pour ce faire, nous utiliserons ArgoCD comme outil de Continuous Delivery (CD)
### Installer ArgoCD

De manière natif K8s de permet pas de faire du CD ==> Solution on va étendre son API avec l'installation de ArgoCD pour ajouter cette fonctionalité

```bash
kubectl api-resources | grep -i argo
```

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Patcher le service

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Récupérer le mot de pass Admin
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Via l'IP du loadbalancer, vérifiez que vous avez bien accès la console d'administration et de gestion de ArgoCD

On utilisera `ArgoCD CLI` pour certaines commandes d'administration; piur ce faire vous devez l'installer sur votre machine d'administration.

Exemple d'installation sur MacOS
```bash
brew install argocd
```

Se connecter à ArgoCD

```bash
kubectl get service argocd-server -n argocd --output=jsonpath='{.status.loadBalancer.ingress[0].ip}'
# argocd login <IP LoadBalancer> --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo) --insecure

argocd login $(kubectl get service argocd-server -n argocd --output=jsonpath='{.status.loadBalancer.ingress[0].ip}') --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo) --insecure

```

Ajout du repos Git

Créer un Repo Github et ajoutez le dans ArgoCD comme Repo à partir duquel ils déploiera l'application `voting-app`

```bash
argocd repo add git@github.com:mpakoupete/gitops-test-argocd.git --ssh-private-key-path ~/.ssh/id_rsa
```

Aller dans l'application (interface Web de ArgoCD) poour les config
- Application

Configurer un déploiement continu de l'application dans le namespace dev-voting-app

```bash
kubectl create ns dev-voting-app
```

