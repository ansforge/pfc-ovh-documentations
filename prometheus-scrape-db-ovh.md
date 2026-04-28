# Comment rajouter une cible de scrape pour Prometheus afin de collecter les métriques d'une base de données OVH

## Contexte

Les bases de données OVH exposent des métriques via un endpoint HTTP. Pour collecter ces métriques avec Prometheus, il est nécessaire de configurer une cible de scrape dans la configuration de Prometheus.
Les informations pour scraper ces métriques est disponible dans la console OVH, dans la section "Métriques / Intégration Prometheus" de votre base de données.
La configuration est gérée par du code, et les mots de passe sont stockés dans un vault.

## Prérequis

- Avoir accès à la console OVH et aux informations de scrape de votre base de données.
- Avoir accès au vault pour stocker le mot de passe de votre base de données.
- Avoir accès au repository GitHub où la configuration de Prometheus est gérée (https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/).
- Avoir accès à ArgoCD avec les permissions pour sync.

## Étapes

1. Récupérer les informations de scrape dans la console OVH. Si besoin, réinitialiser le mot de passe afin d'en obtenir un nouveau.
2. Dans le vault, créer un secret ici : https://vault.pfccloudovh.esante.gouv.fr/ui/vault/secrets/amont-tools/kv/list avec un nom identifiable à votre base de données (exemple: `<nom de la bdd>-db-monitoring`) et un champ `password` contenant le mot de passe pour les métriques de votre bdd.
3. Dans le fichier situé ici : https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/blob/main/applications/operationnel/prometheus-operator/amont/templates/external-secret-scrape-configs.yaml , ajouter une nouvelle entrée pour votre base de données OVH:
    - Rajoutez une entrée dans la section `spec / target / template / data / scrape-configs.yaml` en modifiant le `job_name`, la référence `password`, et en spécifiant les valeurs récupérées à l'étape 1.
    - Rajoutez une entrée dans la section `spec / data` pour créer une référence vers le mot de passe que vous avez stocké dans le vault.
4. Mettez vos changements sur la branche `main`, puis faites un sync dans ArgoCD.
