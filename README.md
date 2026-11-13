# devops-k8s-hpa-pdb

Template **HPA + PodDisruptionBudget + Service + ConfigMap + Ingress** — autoscaling horizontal couplé à un budget de disruption pour maintenir la disponibilité pendant les mises à jour, drains de nœuds et maintenances, exposé en interne via un Service et publiquement via un Ingress TLS.

---

## Structure

```
devops-k8s-hpa-pdb/
├── deployment.yaml   # Deployment nginx (2 réplicas, requests pour le HPA)
├── service.yaml      # Service ClusterIP `app` (port 80 → targetPort 80)
├── configmap.yaml    # ConfigMap d'environnement (consommée via envFrom)
├── hpa.yaml          # HorizontalPodAutoscaler (CPU 70% / mémoire 80%)
├── pdb.yaml          # PodDisruptionBudget (minAvailable: 1)
└── ingress.yaml      # Ingress nginx + TLS (cert-manager)
```

---

## Démarrage rapide / Quick start

```bash
# Prérequis : metrics-server installé sur le cluster
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Déployer la ConfigMap, l'app, le Service, le HPA et le PDB
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl apply -f pdb.yaml

# (Optionnel) Exposer l'app publiquement via l'Ingress TLS
# Prérequis : ingress-nginx + cert-manager installés, et un ClusterIssuer
# `letsencrypt-prod` déclaré (ou adapter l'annotation dans ingress.yaml).
kubectl apply -f ingress.yaml

# Vérifier
kubectl get hpa app-hpa -w
kubectl get pdb app-pdb
kubectl get service app
```

> **Ordre important :** `configmap.yaml` doit être appliqué **avant** `deployment.yaml`,
> car le Deployment référence `app-config` via `envFrom`.

---

## Service, ConfigMap et Ingress

### Service (`service.yaml`)

Service `ClusterIP` nommé `app`, aligné sur le sélecteur du Deployment (`app: app`),
qui route le trafic interne vers le `containerPort` 80 des pods.

```yaml
spec:
  type: ClusterIP
  selector:
    app: app
  ports:
    - port: 80
      targetPort: 80
```

C'est ce Service que le test de charge atteint via `wget http://app` (résolution DNS interne).

### ConfigMap (`configmap.yaml`)

ConfigMap d'environnement minimale, consommée par le Deployment via `envFrom`.
Le pattern est repris de la stack source, avec des clés adaptées à l'app nginx.

```yaml
data:
  APP_ENV: production
  NGINX_PORT: "80"
```

### Ingress (`ingress.yaml`)

Ingress `nginx` avec TLS automatique via cert-manager (annotation
`cert-manager.io/cluster-issuer`). Les hosts sont des exemples à adapter
(`app.example.com`) ; le secret TLS `app-tls` est provisionné par cert-manager.

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

---

## Prérequis HPA

Deux conditions sont nécessaires pour que le HPA fonctionne :

1. **metrics-server installé** : collecte les métriques CPU/mémoire des pods
2. **resource requests définis dans le Deployment** : le HPA calcule l'utilisation en pourcentage des requests, pas en valeur absolue

```yaml
resources:
  requests:
    cpu: 100m      # Base de calcul : 70% de 100m = scale up à 70m de CPU
    memory: 128Mi
```

Sans `requests`, le HPA affiche `<unknown>` et ne scale jamais.

---

## Comportement du HPA

```
CPU pods > 70% des requests → scale up (jusqu'à 6 réplicas)
CPU pods < 70% pendant 5 min → scale down (jusqu'à 2 réplicas)
```

Le `behavior` configure la prudence :

```yaml
scaleUp:
  stabilizationWindowSeconds: 0      # Scale up immédiat en cas de pic
scaleDown:
  stabilizationWindowSeconds: 300    # Attendre 5 min de stabilité avant de réduire
```

Sans `stabilizationWindowSeconds`, le cluster peut "flamber" (scale up / scale down en boucle).

---

## Rôle du PodDisruptionBudget

Le PDB répond à la question : "combien de pods peuvent être indisponibles simultanément lors d'une opération volontaire ?"

Opérations volontaires (respectent le PDB) : `kubectl drain`, rolling update, suppression manuelle de pod, mise à jour de nœud.

Opérations involontaires (ignorent le PDB) : crash du process, panne matérielle, OOMKill.

```yaml
minAvailable: 1   # Au moins 1 pod doit rester disponible lors d'un drain
```

| Réplicas | minAvailable | Pods drainables simultanément |
|---|---|---|
| 2 | 1 | 1 |
| 3 | 2 | 1 |
| 3 | 1 | 2 |
| 2 | 2 | 0 (drain bloqué) |

---

## Simuler un drain de nœud

```bash
# Cordonner le nœud (empêche de nouveaux pods d'y être schedulés)
kubectl cordon <node-name>

# Drainer le nœud (déplace les pods vers d'autres nœuds)
# Kubernetes respectera le PDB — il attendra que les pods soient Ready ailleurs
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Vérifier que l'app reste disponible pendant le drain
kubectl get pods -w

# Réintégrer le nœud après maintenance
kubectl uncordon <node-name>
```

---

## Observer le scaling

```bash
# Surveiller le HPA en temps réel (TARGETS = CPU actuel / CPU cible)
kubectl get hpa app-hpa -w

# Générer de la charge pour déclencher un scale up
# Le Service `app` route la charge vers les pods sélectionnés.
kubectl run load-generator --image=busybox --rm -it -- \
  /bin/sh -c "while true; do wget -q -O- http://app; done"

# Observer les événements de scaling
kubectl describe hpa app-hpa | grep -A 20 "Events:"
```

---

*Auteur : [Djihadi Ahamdy](https://djihadi.dev) — partie de la [boîte à outils DevOps](https://djihadi.dev)*
