# Recommandation pour la mise en place de la haute disponibilité (HA) sur la PFC 3AZ

Dans le cadre du déploiement de la PFC sur la zone OVH de Paris (3AZ), voici quelques conseils pour bénéficier de la HA.
Cela permettra en particulier d'éviter des downtimes de l'application lors des opérations de maintenance faites régulièrement sur la PFC.

## Prérequis

Dans le but de pouvoir mettre en place ces recommandations, voici les contraintes à remplir :

- Un déploiement StateLess [cf source](https://www.redhat.com/fr/topics/cloud-native-apps/stateful-vs-stateless)
- Avoir au moins deux replicas
- Votre déploiement a un label `app: app-name-env`

## Configuration

Ajouter dans votre déploiement cette configuration de Pod Anti Affinity pour éviter que les pods se déploient sur le même nœud.
Le but étant d'être résilient à la perte d'un nœud Kubernetes.

```yaml
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - app-name-env
            topologyKey: "kubernetes.io/hostname"
```

Ajouter dans votre déploiement cette configuration de Topology Spread Constraint pour que les pods d'un même replicaset se répartissent sur les différentes zones de disponibilité.
Le but étant d'être résilient à la perte d'une zone de disponibilité.
La configuration matchLabelKeys ne doit pas être changée. Sans elle, on peut se retrouver avec une mauvaise répartition des pods sur les zones de disponibilité après un rollout du déploiement.

```yaml
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: app-name-env
        matchLabelKeys:
          - pod-template-hash
```

Ajouter une ressource Pod Disruption Budget pour votre déploiement pour s'assurer que Kubernetes conserve toujours 1 des replicas en état de fonctionnement.
Attention, ne mettre un Pod Disruption Budget que si le nombre de replicas est supérieur ou égal à 2, sinon cela va bloquer les opérations de maintenance automatique en empêchant le redémarrage du container.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-name-env-pdb
  namespace: app-name-env
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: app-name-env
```

## Résultat attendu

Si cette configuration est correctement ajoutée, voici les résultats théoriquement attendus :

- Pas de downtime de l'application lors des mises à jour Kubernetes
- Pas de downtime de l'application lors du redémarrage d'un nœud
- Pas de downtime de l'application lors de l'indisponibilité d'une zone
- Toujours un pod en état de fonctionner lors des nouveaux déploiements

## Pour aller plus loin

Si cette configuration n'est pas suffisante, il faudra sûrement peaufiner la configuration des readiness et liveness probes, et ajuster la valeur de terminationGracePeriodSeconds.
Le but est que les statuts sur lesquels se base Kubernetes soient en adéquation avec le statut réel de l'application.
Si besoin, vous pouvez également créer un ticket pour demander de l'aide à l'équipe d'infogérance de la PFC.
