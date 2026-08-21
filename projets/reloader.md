# Redémarrer automatiquement mes applications

## Reloader

Certains projets ont fait la demande de pouvoir redémarrer leurs déploiements ou leurs StatefulSets lors d'un changement de valeurs dans leurs configmaps ou leurs secrets.
Pour ce faire, nous avons déployé [Reloader](https://github.com/stakater/reloader) sur les clusters de la PFC 3AZ pour répondre à ce besoin.

## Configuration

Pour redémarrer un déploiement lorsqu'une configmap spécifique change, ajouter cette annotation en changeant le nom de la configmap :

```yaml
kind: Deployment
metadata:
    annotations:
        configmap.reloader.stakater.com/reload: "my-config"
```

Pour redémarrer un déploiement lorsqu'un secret spécifique change, ajouter cette annotation en changeant le nom du secret :

```yaml
kind: Deployment
metadata:
    annotations:
        secret.reloader.stakater.com/reload: "my-secret"
```

Pour redémarrer un déploiement pour tout changement de valeur concernant un secret ou une configmap configuré sur ce déploiement utiliser l'annotation automatique :

```yaml
kind: Deployment
metadata:
    annotations:
        reloader.stakater.com/auto: "true"
```

## Résultat attendu

Si cette configuration est correctement ajoutée, voici les résultats théoriquement attendus :

1. La valeur du secret/configmap change
2. Le controller de Reloader va lire les annotations du déploiement et lancer un redémarrage des pods associés en fonction du comportement demandé par les annotations

## Pour aller plus loin

De nombreux cas particuliers peuvent être traités, merci de lire la documentation de l'outil pour voir les options disponibles. Si besoin, vous pouvez également créer un ticket pour demander de l'aide à l'équipe d'infogérance de la PFC.
