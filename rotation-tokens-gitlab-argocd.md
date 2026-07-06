# Contexte

ArgoCD a besoin de tokens gitlab pour se connecter aux repos contenant les apps.

Ces tokens ont une date d’expiration et doivent donc être rotate régulièrement.

## Pourquoi ces tokens/ces scopes

Les tokens ne doivent pas être liés à un compte, donc nous choisissons des *Group Access Tokens* (et non des *Personal Access Tokens*).
Les repos sont situés dans différents folders gitlab, et selon le principe de moindre privilège, les tokens générés ne doivent avoir accès qu'aux repos dans ces folders.
Le scope des tokens leur permet de lire les repos gitlab sans aucun droit d'écriture.

# Comment technique

1. Pour chaque folder gitlab nécessaire ([gitops](https://gitlab.esante.gouv.fr/ans/transverse/pfc-ovh/pfc-ovh-argocd-configs/gitops) et [helmchart](https://gitlab.esante.gouv.fr/ans/deploiement/helmchart)), créer un nouveau *Group Access Token* gitlab:
   - Il faut le scope `read_api, read_repository, read_registry` 
   - Noter la date d'expiration
2. Changer le token dans les Vault (champs "token-folder-gitops"/"token-folder-helmchart"):
   - 3AZ: https://vault.pfccloud.esante.gouv.fr/ui/vault/secrets/outils-devops/kv/argocd-gitlab-credentials/details
   - 1AZ: https://vault.pfccloudovh.esante.gouv.fr/ui/vault/secrets/forge-tools/kv/argocd/details
   - 1AZ test: https://vault.test.pfccloudovh.esante.gouv.fr/ui/vault/secrets/forge-tools/kv/argocd/details
3. Attendre (environ 3 minutes) pour que externalsecrets+argocd voient le changement de tokens
4. Vérifier que les tokens marchent bien en connectant un repo dans la console argocd: https://argocd.pfccloud.esante.gouv.fr/settings/repos?addRepo=true (via https, ne pas spécifier de credentials pour qu’il utilise bien le template)
5. Mettre à jour la date d'expiration dans le document Excel https://esantegouv.sharepoint.com/:x:/r/sites/GED-Calypso/espace-projets/_layouts/15/doc2.aspx?sourcedoc=%7B072F03DE-4526-43A3-8D69-A9C2D6B58293%7D&file=SuiviJetonVault.xlsx&action=default&mobileredirect=true
