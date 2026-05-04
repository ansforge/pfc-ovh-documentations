# Comment configurer une nouvelle application ou un nouvel environnement client sur la PFC (version 3AZ)

## Créer les KV Vault et les ServiceAccounts

- Dans le repository [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s/), dans les dossiers `layers/outils/vault/config-infra-transverse/3az-operationnel`/`<amont|production>`, ajouter un dossier avec le nom de l'application, et copier dans ce dossier le contenu d'un dossier d'une application déjà configurée (par exemple `mcps`).
- Ajuster les paramètres dans les fichiers `inputs.hcl` (notamment les environnements de l'application) si besoin.
- Faire un `terragrunt apply` pour créer les ressources dans Vault.
- Une fois les changements effectués, les mettre sur main (de préférence avec une PR).

## Créer les repository GitLab

- Dans le repository [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s/), dans le dossier `layers/outils/gitlab-3az`, ajouter un dossier avec le nom de l'application, et copier dans ce dossier le contenu d'un dossier d'une application déjà configurée (par exemple `mcps`).
- Ajuster les paramètres dans les fichiers `inputs.hcl` (notamment les environnements de l'application) si besoin.
- Faire un `terragrunt apply` pour créer les ressources dans Vault.
- Une fois les changements effectués, les mettre sur main (de préférence avec une PR).

## Créer un ApplicationSet pour les applications avec les SecretStores

- Aller dans la console ArgoCD de la PFC 3AZ (https://argocd.pfccloud.esante.gouv.fr/)
- Aller dans `Applications` > `New App` puis remplir les champs suivants en remplaçant les valeurs `<>` par les bonnes valeurs :
  - Application Name : `<nom de l'app>-scm`
  - Project : `outils`
  - Sync Policy : `Manual`
  - Repository URL : `https://gitlab.esante.gouv.fr/ans/transverse/pfc-ovh/pfc-ovh-argocd-configs/gitops/applications-metiers/<nom de l'app>.git`
  - Revision : `main`
  - Path : `app/`
  - Destination Cluster : `https://kubernetes.default.svc`
  - Destination Namespace : `default`

## Donner les accès aux utilisateurs

La configuration des rôles et des permissions Keycloak pour pouvoir se connecter à Vault et ArgoCD est automatisée as code via Crossplane.
- Dans le fichier https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/blob/main/components/operationnel/keycloak-config/outils/values/users.yaml, ajouter les utilisateurs avec les rôles et les applications auxquels ils doivent avoir accès. Par exemple:
```yaml
      - username: marie.blachere
        firstName: Marie
        lastName: Blachère
        email: marie.blachere@exemple.fr
        generateInitialPassword: true
        groups:
          - <nom de l'app>-[re7 | developer | devops]
```
- Dans le fichier https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/blob/main/components/operationnel/keycloak-config/outils/values/groups.yaml, ajouter les rôles suivants:
```yaml
     - name: <nom de l'app>-devops
       realmRoles:
         - argocd-access
         - argocdcli-access
         - grafana-access
         - vault-access
       clientRoles:
         vault:
           - <nom de l'app>-devops
     - name: <nom de l'app>-developer
       realmRoles:
         - argocd-access
         - grafana-access
         - vault-access
       clientRoles:
         vault:
           - <nom de l'app>-developer
     - name: <nom de l'app>-re7
       realmRoles:
         - vault-access
       clientRoles:
         vault:
           - <nom de l'app>-developer
```
- Dans le fichier https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/blob/main/components/operationnel/argo-cd/outils/values.yaml, ajouter une entrée de `appsetOptions / additioalValuesFiles` pour un nouveau fichier à créer selon le modèle des fichiers déjà présents, avec comme nom `<nom de l'app>.yaml`. Le code est à adapter en fonction des environnements sur lesquels est déployée l'application.

Une fois le compte créé, envoyer le mail suivant avec Christian en copie (ne pas oublier de remplacer toutes les valeurs `$`)

```txt
TO: $EMAIL
CC: Christian.CRIMETZ@esante.gouv.fr
Subject: ANS-PFC Accès à $APPLICATION
Content:
Bonjour $PRENOM,

On m'a demandé de vous créer un compte développeur pour l'application $APPLICATION.

Voici les liens des outils :
• ArgoCD: https://argocd.pfccloud.esante.gouv.fr/ (cliquer sur « Login with Keycloak »)
• Vault: https://vault.pfccloud.esante.gouv.fr/ (choisir comme méthode de connexion « OIDC », puis cliquer sur « Sign in with OIDC Provider »)

Lors de la première connexion :
• Identifiant : $PRENOM.$NOM
• Mot de passe (à changer en cliquant sur mot de passe oublié)
• Il faudra configurer la double authentification (OTP)

Cordialement,
```
