# TP 33 : Déploiement d'une application Spring Boot sur Kubernetes

## Description
Projet de démonstration pour déployer une application Spring Boot sur Kubernetes (Minikube).

L'application expose un endpoint REST `/api/hello` qui retourne un message JSON.

## Structure du Projet

```
tp33/
├── src/
│   └── main/
│       ├── java/
│       │   └── com/example/demok8s/
│       │       ├── DemoK8sApplication.java
│       │       └── api/
│       │           └── HelloController.java
│       └── resources/
│           └── application.properties
├── k8s-deployment.yaml
├── k8s-service.yaml
├── k8s-configmap.yaml
├── Dockerfile
└── pom.xml
```

## Pré-requis

- Java 17 ou 21
- Maven
- Docker
- Minikube
- kubectl

## Étape 1 : Test Local de l'Application

### 1.1 Compiler et exécuter localement

```powershell
# Compiler le projet
mvn clean package -DskipTests

# Exécuter l'application
mvn spring-boot:run
```

### 1.2 Tester l'endpoint

```powershell
# Dans un autre terminal
curl http://localhost:8080/api/hello

# Ou avec PowerShell
Invoke-WebRequest -Uri http://localhost:8080/api/hello | Select-Object -Expand Content
```

Réponse attendue :
```json
{
  "message": "Hello from Spring Boot on Kubernetes",
  "status": "OK"
}
```

## Étape 2 : Conteneurisation avec Docker

### 2.1 Construire le JAR

```powershell
mvn clean package -DskipTests
```

### 2.2 Construire l'image Docker

```powershell
docker build -t demo-k8s:1.0.0 .
```

### 2.3 Tester l'image Docker localement (optionnel)

```powershell
# Lancer le conteneur
docker run -p 8080:8080 demo-k8s:1.0.0

# Tester
curl http://localhost:8080/api/hello
```

## Étape 3 : Préparation de Minikube

### 3.1 Démarrer Minikube

```powershell
minikube start
```

### 3.2 Configurer Docker pour utiliser l'environnement Minikube

```powershell
# PowerShell
minikube docker-env | Invoke-Expression

# Reconstruire l'image dans l'environnement Docker de Minikube
docker build -t demo-k8s:1.0.0 .
```

### 3.3 Vérifier que Minikube fonctionne

```powershell
kubectl cluster-info
kubectl get nodes
```

## Étape 4 : Créer le Namespace

```powershell
kubectl create namespace lab-k8s

# Vérifier
kubectl get namespaces
```

## Étape 5 : Déployer sur Kubernetes

### 5.1 Appliquer la ConfigMap

```powershell
kubectl apply -f k8s-configmap.yaml
```

### 5.2 Déployer l'application

```powershell
kubectl apply -f k8s-deployment.yaml
```

### 5.3 Créer le Service

```powershell
kubectl apply -f k8s-service.yaml
```

### 5.4 Vérifier le déploiement

```powershell
# Vérifier les pods
kubectl get pods -n lab-k8s

# Vérifier les services
kubectl get svc -n lab-k8s

# Vérifier les détails du déploiement
kubectl describe deployment demo-k8s-deployment -n lab-k8s

# Vérifier la ConfigMap
kubectl get configmap -n lab-k8s
kubectl describe configmap demo-k8s-config -n lab-k8s
```

## Étape 6 : Tester l'Application sur Kubernetes

### 6.1 Obtenir l'URL du service

```powershell
# Méthode 1 : Obtenir l'IP de Minikube et utiliser le NodePort
minikube ip
# Supposons que l'IP est 192.168.49.2
# L'application est accessible sur : http://192.168.49.2:30080/api/hello

# Méthode 2 : Utiliser minikube service (recommandé)
minikube service demo-k8s-service -n lab-k8s
# Cette commande ouvre automatiquement l'URL dans le navigateur

# Méthode 3 : Obtenir l'URL
minikube service demo-k8s-service -n lab-k8s --url
```

### 6.2 Tester l'endpoint

```powershell
# Remplacer <MINIKUBE_IP> par l'IP obtenue
curl http://<MINIKUBE_IP>:30080/api/hello

# Exemple avec PowerShell
$minikubeIp = minikube ip
Invoke-WebRequest -Uri "http://${minikubeIp}:30080/api/hello" | Select-Object -Expand Content
```

Réponse attendue :
```json
{
  "message": "Hello from ConfigMap in Kubernetes",
  "status": "OK"
}
```

### 6.3 Tester l'endpoint Actuator

```powershell
curl http://<MINIKUBE_IP>:30080/actuator/health
```

## Étape 7 : Observation et Diagnostic

### 7.1 Consulter les logs

```powershell
# Lister les pods
kubectl get pods -n lab-k8s

# Consulter les logs d'un pod spécifique
kubectl logs <POD_NAME> -n lab-k8s

# Suivre les logs en temps réel
kubectl logs -f <POD_NAME> -n lab-k8s

# Logs de tous les pods du déploiement
kubectl logs -l app=demo-k8s -n lab-k8s
```

### 7.2 Accéder à un pod

```powershell
# Se connecter à un pod
kubectl exec -it <POD_NAME> -n lab-k8s -- sh

# Dans le pod, tester l'endpoint
apk add curl
curl http://localhost:8080/api/hello
```

### 7.3 Tester depuis l'intérieur du cluster

```powershell
# Créer un pod temporaire pour tester
kubectl run curl-pod -n lab-k8s --image=alpine/curl -it --rm -- sh

# Dans le pod :
curl http://demo-k8s-service:8080/api/hello
exit
```

### 7.4 Vérifier les événements

```powershell
kubectl get events -n lab-k8s --sort-by='.lastTimestamp'
```

## Étape 8 : Mise à l'échelle

### 8.1 Scaler le déploiement

```powershell
# Augmenter le nombre de replicas
kubectl scale deployment demo-k8s-deployment --replicas=3 -n lab-k8s

# Vérifier
kubectl get pods -n lab-k8s

# Réduire le nombre de replicas
kubectl scale deployment demo-k8s-deployment --replicas=2 -n lab-k8s
```

## Étape 9 : Modifier la ConfigMap

### 9.1 Modifier le message

```powershell
# Éditer la ConfigMap
kubectl edit configmap demo-k8s-config -n lab-k8s

# Ou modifier k8s-configmap.yaml et réappliquer
kubectl apply -f k8s-configmap.yaml
```

### 9.2 Redémarrer les pods pour prendre en compte le changement

```powershell
kubectl rollout restart deployment demo-k8s-deployment -n lab-k8s

# Vérifier le statut du rollout
kubectl rollout status deployment demo-k8s-deployment -n lab-k8s
```

## Étape 10 : Nettoyage

### 10.1 Supprimer les ressources

```powershell
# Supprimer le service
kubectl delete -f k8s-service.yaml

# Supprimer le déploiement
kubectl delete -f k8s-deployment.yaml

# Supprimer la ConfigMap
kubectl delete -f k8s-configmap.yaml

# Supprimer le namespace (supprime tout)
kubectl delete namespace lab-k8s
```

### 10.2 Arrêter Minikube

```powershell
minikube stop

# Ou supprimer complètement le cluster
minikube delete
```

## Commandes Utiles

### Informations sur le cluster

```powershell
# État du cluster
kubectl cluster-info

# Noeuds du cluster
kubectl get nodes

# Tous les namespaces
kubectl get namespaces

# Toutes les ressources dans un namespace
kubectl get all -n lab-k8s
```

### Dashboard Kubernetes

```powershell
# Ouvrir le dashboard Minikube
minikube dashboard
```

### Port Forwarding (alternative au NodePort)

```powershell
# Faire un port-forward vers un pod spécifique
kubectl port-forward <POD_NAME> 8080:8080 -n lab-k8s

# Faire un port-forward vers le service
kubectl port-forward svc/demo-k8s-service 8080:8080 -n lab-k8s

# Tester
curl http://localhost:8080/api/hello
```

## Troubleshooting

### Les pods ne démarrent pas

```powershell
# Vérifier l'état des pods
kubectl get pods -n lab-k8s

# Décrire le pod pour voir les événements
kubectl describe pod <POD_NAME> -n lab-k8s

# Vérifier les logs
kubectl logs <POD_NAME> -n lab-k8s
```

### L'image n'est pas trouvée

```powershell
# Vérifier que vous êtes dans l'environnement Docker de Minikube
minikube docker-env

# Lister les images disponibles
docker images | grep demo-k8s

# Reconstruire l'image si nécessaire
eval $(minikube docker-env)  # Linux/Mac
minikube docker-env | Invoke-Expression  # PowerShell
docker build -t demo-k8s:1.0.0 .
```

### Le service n'est pas accessible

```powershell
# Vérifier que le service existe
kubectl get svc -n lab-k8s

# Obtenir les détails du service
kubectl describe svc demo-k8s-service -n lab-k8s

# Utiliser minikube service pour obtenir l'URL
minikube service demo-k8s-service -n lab-k8s --url
```

## Extensions Possibles

1. **Ingress** : Exposer l'application avec un nom de domaine
2. **Helm** : Packager l'application avec Helm
3. **CI/CD** : Intégrer dans un pipeline GitHub Actions ou GitLab CI
4. **Monitoring** : Ajouter Prometheus et Grafana
5. **Multiple Services** : Créer un second microservice et tester la communication
6. **Persistent Storage** : Ajouter un volume persistant
7. **Secrets** : Utiliser des Secrets Kubernetes pour les données sensibles
8. **Resource Limits** : Définir des limites de ressources (CPU, mémoire)

## Ressources

- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Docker Documentation](https://docs.docker.com/)
