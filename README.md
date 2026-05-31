# devops-k8s-hpa-pdb

Template **HPA + PodDisruptionBudget** — autoscaling horizontal couplé à un budget de disruption pour maintenir la disponibilité pendant les mises à jour, drains de nœuds et maintenances.

---

## Démarrage rapide / Quick start

```bash
# Prérequis : metrics-server installé sur le cluster
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Déployer l'app, le HPA et le PDB
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml
kubectl apply -f pdb.yaml

# Vérifier
kubectl get hpa app-hpa -w
kubectl get pdb app-pdb
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
# Kubernetes respectera le PDB — il attendra que les pods soient Readyailleurs
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
kubectl run load-generator --image=busybox --rm -it -- \
  /bin/sh -c "while true; do wget -q -O- http://app; done"

# Observer les événements de scaling
kubectl describe hpa app-hpa | grep -A 20 "Events:"
```

---

*Auteur : [Djihadi Ahamdy](https://djihadi.dev) — partie de la [boîte à outils DevOps](https://djihadi.dev)*
