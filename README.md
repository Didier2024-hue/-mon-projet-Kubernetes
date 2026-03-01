# Évaluation Kubernetes — MySQL + FastAPI

**Namespace :** `eval`  
**Image Docker Hub :** `kiemberaid48302/k8s-dst-eval-fastapi`  
**Cluster :** K3s (AWS EC2 Ubuntu)

---

## Architecture déployée

```
Namespace: eval
│
├── StatefulSet: mysql (1 replica)
│   ├── Image : docker.io/mysql:8.4
│   ├── Secret : mysql-root-password  →  MYSQL_ROOT_PASSWORD=admin
│   ├── Secret : mysql-user           →  MYSQL_USER=appuser / MYSQL_PASSWORD=admin / MYSQL_DATABASE=appdb
│   ├── PersistentVolume : mysql-local-data-pv (2Gi, hostPath: /mnt/mysql-data)
│   ├── PVC généré : mysql-storage-mysql-0
│   └── Service ClusterIP : mysql-service → IP fixe 10.43.0.42, port 3307 → 3306
│
├── Pod de test: fastapi (phase développement)
│   ├── Volume hostPath vers eval/test/k8s-eval-fastapi/app (--reload activé)
│   └── Accès Swagger : kubectl port-forward pod/fastapi 8000:8000 -n eval
│
└── Deployment: fastapi-deployment (3 replicas)
    ├── Image : kiemberaid48302/k8s-dst-eval-fastapi:latest
    ├── Secret : mysql-user (envFrom)
    └── Service NodePort : fastapi-service → port 30000
```

---

## Structure des dossiers

```
eval/
├── data/
├── fastapi/
│   ├── fastapi-deployment.yaml
│   └── fastapi-service.yaml
├── mysql/
│   ├── mysql-local-data-folder-pv.yaml
│   ├── mysql-statefulset.yaml
│   └── mysql-service.yaml
└── test/
    └── k8s-eval-fastapi/
        ├── app/
        │   └── main.py
        ├── Dockerfile
        ├── fastapi-pod.yaml
        └── requirements.txt
```

---

## Étape 1 — Initialisation

```bash
mkdir eval && cd eval
mkdir data fastapi mysql test

# Créer et activer le namespace
kubectl create namespace eval
kubectl get namespaces | grep eval
kubectl config set-context --current --namespace=eval
```

---

## Étape 2 — Secrets MySQL

```bash
# Secret 1 : mot de passe root
kubectl create secret generic mysql-root-password \
  --from-literal=MYSQL_ROOT_PASSWORD='admin' \
  -n eval

# Secret 2 : utilisateur applicatif
kubectl create secret generic mysql-user \
  --from-literal=MYSQL_USER='appuser' \
  --from-literal=MYSQL_PASSWORD='admin' \
  --from-literal=MYSQL_DATABASE='appdb' \
  -n eval

# Vérification
kubectl get secrets -n eval
```

---

## Étape 3 — PersistentVolume

Fichier : `eval/mysql/mysql-local-data-folder-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-local-data-pv
spec:
  storageClassName: local-path
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  claimRef:
    namespace: eval
    name: mysql-storage-mysql-0   # Nom généré par le StatefulSet
  hostPath:
    path: /mnt/mysql-data
```

> Le nom du PVC suit le pattern `<volumeClaimTemplate.name>-<statefulset-name>-<index>` → `mysql-storage-mysql-0`

```bash
kubectl apply -f mysql-local-data-folder-pv.yaml
```

---

## Étape 4 — StatefulSet MySQL

Fichier : `eval/mysql/mysql-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: eval
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: docker.io/mysql:8.4
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-root-password
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-user
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-user
                  key: MYSQL_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-user
                  key: MYSQL_DATABASE
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 2Gi
```

```bash
kubectl apply -f mysql-statefulset.yaml
```

---

## Étape 5 — Service MySQL

Fichier : `eval/mysql/mysql-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: eval
spec:
  type: ClusterIP
  clusterIP: 10.43.0.42
  ports:
    - name: mysql
      port: 3307
      targetPort: 3306
  selector:
    app: mysql
```

```bash
kubectl apply -f mysql-service.yaml -n eval
```

### Vérification de l'ensemble MySQL

```bash
kubectl get pv                      # PersistentVolume (ressource cluster)
kubectl get pvc -n eval             # PVC généré : mysql-storage-mysql-0
kubectl get statefulsets -n eval    # StatefulSet mysql
kubectl get pods -n eval            # Pod mysql-0 → STATUS: Running
kubectl get svc -n eval             # mysql-service → 10.43.0.42:3307
```

---

## Étape 6 — Développement de l'application FastAPI

### Téléchargement des sources

```bash
cd eval/test
wget https://dst-de.s3.eu-west-3.amazonaws.com/kubernetes_fr/eval/k8s-eval-fastapi.tar
tar -xvf k8s-eval-fastapi.tar
cd k8s-eval-fastapi
```

### Application FastAPI — `app/main.py`

Le fichier `main.py` se connecte à MySQL via SQLAlchemy en lisant les variables d'environnement injectées par le secret `mysql-user`. La connexion est construite dynamiquement :

- `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DATABASE` → depuis le secret `mysql-user`
- `MYSQL_SERVICE_SERVICE_HOST` → résolu automatiquement par Kubernetes (= `10.43.0.42`)
- `MYSQL_SERVICE_SERVICE_PORT` → port du service MySQL (= `3307`)

**Routes disponibles :**

| Méthode | Route | Description |
|---------|-------|-------------|
| GET | `/` | Vérifie que l'API est en ligne |
| GET | `/tables` | Liste les tables de la base MySQL |
| PUT | `/table` | Crée une nouvelle table |

### Build et push de l'image Docker

```bash
# Build
docker build -t kiemberaid48302/k8s-dst-eval-fastapi:latest .

# Login Docker Hub
docker login -u kiemberaid48302
# (saisir le token d'accès personnel)

# Push
docker push kiemberaid48302/k8s-dst-eval-fastapi:latest
```

---

## Étape 7 — Pod de test FastAPI (avec --reload)

Fichier : `eval/test/k8s-eval-fastapi/fastapi-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fastapi
  namespace: eval
spec:
  containers:
    - name: fastapi
      image: kiemberaid48302/k8s-dst-eval-fastapi:latest
      ports:
        - containerPort: 8000
      envFrom:
        - secretRef:
            name: mysql-user
      volumeMounts:
        - name: app-volume
          mountPath: /app
  volumes:
    - name: app-volume
      hostPath:
        path: /home/ubuntu/exam_JOSEPHINE/kubernetes/eval/test/k8s-eval-fastapi/app
        type: Directory
```

```bash
kubectl apply -f fastapi-pod.yaml -n eval

# Suivi des logs (affiche les variables de connexion au démarrage)
kubectl logs -f fastapi -n eval

# Accès au Swagger UI via tunnel
kubectl port-forward pod/fastapi 8000:8000 -n eval
# → ouvrir http://localhost:8000/docs
```

**Logs observés au démarrage :**
```
Connexion à la base MySQL :
Utilisateur : appuser
Mot de passe : admin
Hôte : 10.43.0.42
Port : 3307
Base de données : appdb
INFO: Uvicorn running on http://0.0.0.0:8000
INFO: Application startup complete.
```

---

## Étape 8 — Déploiement en production (3 replicas)

Avant de déployer, supprimer `--reload` du Dockerfile et reconstruire l'image :

```bash
# Modifier le Dockerfile : retirer --reload de la commande CMD
# CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

docker build -t kiemberaid48302/k8s-dst-eval-fastapi:latest .
docker push kiemberaid48302/k8s-dst-eval-fastapi:latest
```

### Deployment — `eval/fastapi/fastapi-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
  namespace: eval
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
        - name: fastapi
          image: kiemberaid48302/k8s-dst-eval-fastapi:latest
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: mysql-user
```

### Service NodePort — `eval/fastapi/fastapi-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
  namespace: eval
spec:
  type: NodePort
  selector:
    app: fastapi
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30000
```

```bash
kubectl apply -f fastapi-deployment.yaml
kubectl apply -f fastapi-service.yaml

# Vérification : 3 pods Running
kubectl get pods -n eval -l app=fastapi
```

**Résultat obtenu :**
```
NAME                                  READY   STATUS    RESTARTS   AGE
fastapi-deployment-656597cdc5-llr64   1/1     Running   0          7m
fastapi-deployment-656597cdc5-2p2q6   1/1     Running   0          7m
fastapi-deployment-656597cdc5-w9msq   1/1     Running   0          7m
```

L'API est accessible depuis l'extérieur via `http://<IP_NODE>:30000`

---

## Commandes de vérification globale

```bash
kubectl get all -n eval                        # Vue d'ensemble du namespace
kubectl describe pod <nom-pod> -n eval         # Détail d'un pod
kubectl logs -f <nom-pod> -n eval              # Logs en temps réel
kubectl exec -it <nom-pod> -n eval -- bash     # Shell dans un pod
```

---


