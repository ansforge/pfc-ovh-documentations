# Comment rajouter une cible de scrape pour Prometheus afin de collecter les métriques d'une base de données OVH

## Contexte

Les bases de données OVH exposent des métriques via un endpoint HTTP. Pour collecter ces métriques avec Prometheus, il est nécessaire de configurer une cible de scrape dans la configuration de Prometheus.
Les informations pour scraper ces métriques sont disponibles dans la console OVH, dans la section "Métriques / Intégration Prometheus" de votre base de données.
La configuration est gérée par un ExternalSecret templatisé, et les mots de passe sont stockés dans un Vault.

## Prérequis

- Avoir accès à la console OVH et aux informations de scrape de votre base de données.
- Avoir accès au Vault pour stocker le mot de passe de votre base de données.
- Avoir accès au repository GitHub où la configuration de Prometheus est gérée (https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/).
- Avoir accès à ArgoCD avec les permissions pour sync l'application `prometheus-operator-<nom du cluster>-gh`.

## Étapes

1. Récupérer les informations de scrape dans la console OVH. Si besoin, réinitialiser le mot de passe de scraping afin d'en obtenir un nouveau.
2. Dans le Vault, créer un secret ici : https://vault.pfccloudovh.esante.gouv.fr/ui/vault/secrets/amont-tools/kv/list avec un nom identifiable à votre base de données (exemple: `<nom de la db>-db-monitoring`) et un champ `password` contenant le mot de passe pour les métriques de votre bdd.
3. Dans le fichier situé ici : https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/blob/main/applications/operationnel/prometheus-operator/amont/templates/external-secret-scrape-configs.yaml , ajouter une nouvelle entrée pour votre base de données OVH en utilisant les blocs de codes situés ci-dessous, et en remplaçant les valeurs entre `<>` par les valeurs spécifiques à votre base de données.
4. Mettez vos changements sur la branche `main`, puis faites un sync dans ArgoCD.

### Blocs de code à rajouter dans le fichier `external-secret-scrape-configs.yaml`

```yaml
# Bloc à rajouter dans spec / target / template / data / scrape-configs.yaml
          - job_name: postgresql-<nom de la db>-<nom du cluster>
            scrape_interval: 10s
            metrics_path: /metrics
            scheme: https
            basic_auth:
              username: prometheus
              password: "{{ `{{ .<nom de la db>Password }}` }}"
            tls_config:
              insecure_skip_verify: true
            static_configs:
              - targets:
                - <endpoint de scrape de votre base de données OVH>
```

```yaml
# Bloc à rajouter dans spec / data
    - secretKey: <nom de la db>Password
      remoteRef:
        conversionStrategy: Default
        decodingStrategy: None
        key: <nom de la db>-db-monitoring
        metadataPolicy: None
        property: password
```
