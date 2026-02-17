# Génération d'un certificat avec cert-manager

## Prérequis

- Avoir un FQDN qui se termine par `esante.gouv.fr`

## Migration vers cert-manager
### Actions à réaliser

Lors du passage à cert-manager, il est nécessaire de faire les actions suivantes:

0. Effectuer un backup du manifest du secret contenant le précédent certificat pour pouvoir le restaurer en cas d'imprévu durant l'opération.

    Exemple avec grafana-tls
    ```bash
    k get secret grafana-tls -n monitoring -o yaml > secret-grafana-tls.backup-test-prod.yaml
    ```

1. Supprimer l'externalSecret qui injecte le certificat (stocké dans le vault) dans le secret et qui est utilisé par l'application.
Pour ce faire, il faut supprimer la ressource de l'externalSecret dans le helm chart de l'application.
2. Ajouter l'annotation suivante sur l'ingress visé
```yaml
annotations:
      cert-manager.io/cluster-issuer: lets-encrypt-gandi
```

Si l'ingress ne dispose pas encore de champ `tls`, il faut l'ajouter comme suit:
```yaml
 tls:
    - secretName: {APP-NAME}-tls
      hosts:
      - {APP-HOST}.esante.gouv.fr
```

3. Une fois l'annotation ajoutée sur l'ingress et le champ `tls` renseigné, cert-manager sera en mesure de générer un certificat et de remplir sa valeur dans le secret indiqué dans la partie `tls` de l'ingress.


## Création d'un certificat pour une nouvelle application

- Ajouter l'annotation suivante sur l'ingress de la nouvelle application
```yaml
annotations:
      cert-manager.io/cluster-issuer: lets-encrypt-gandi
```

Une fois l'annotation ajoutée sur l'ingress, cert-manager sera en mesure de générer un certificat et de remplir sa valeur dans le secret indiqué dans la partie `tls` de l'ingress.
