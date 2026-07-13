# Contexte

ArgoCD a besoin d'un token github pour se connecter au repo contenant les apps et la configuration d'infra ([pfc-ovh-argocd-config-infrastructure](https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure)).

Ce token a une date d'expiration et doit donc être rotate régulièrement.

## Pourquoi ce token/ces scopes

Le token est généré par un administrateur de l'organisation github (par exemple, Christian Crimetz).
Selon le principe de moindre privilège, le token généré ne doit avoir accès qu'au repo nécessaire.
Le scope du token lui permet de lire le repo github sans aucun droit d'écriture.

# Comment technique

1. Demander à un administrateur de l'organisation github (par exemple, Christian Crimetz) de générer un nouveau token pour le repo [pfc-ovh-argocd-config-infrastructure](https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure):
   - Il faut un accès en lecture seule au repo
   - Noter la date d'expiration
2. Changer le token dans les Vault (champ "token"):
   - 3AZ: https://vault.pfccloud.esante.gouv.fr/ui/vault/secrets/outils-devops/kv/argocd-config-infra-github-credentials
   - 1AZ: https://vault.pfccloudovh.esante.gouv.fr/ui/vault/secrets/forge-tools/kv/argocd-config-infra-github-credentials/details
   - 1AZ test: https://vault.test.pfccloudovh.esante.gouv.fr/ui/vault/secrets/forge-tools/kv/argocd-config-infra-github-credentials
3. Attendre (environ 3 minutes) pour que externalsecrets+argocd voient le changement de token
4. Vérifier que le token marche bien en connectant un repo dans la console argocd: https://argocd.pfccloud.esante.gouv.fr/settings/repos?addRepo=true (via https, ne pas spécifier de credentials pour qu'il utilise bien le template)
5. Mettre à jour la date d'expiration dans le document Excel https://esantegouv.sharepoint.com/:x:/r/sites/GED-Calypso/espace-projets/_layouts/15/doc2.aspx?sourcedoc=%7B072F03DE-4526-43A3-8D69-A9C2D6B58293%7D&file=SuiviJetonVault.xlsx&action=default&mobileredirect=true
