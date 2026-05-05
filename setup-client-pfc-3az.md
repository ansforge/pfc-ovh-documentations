# Comment configurer une nouvelle application ou un nouvel environnement client sur la PFC (version 3AZ)

## Contexte

Cette documentation couvre la configuration d'une nouvelle application sur la PFC dans sa version **3AZ**. Pour la version 1AZ, se référer à [cette documentation](./setup-client-pfc-1az.md).

L'équipe Theodo a travaillé pour automatiser des parties de la configuration d'une application sur la PFC.

À ce jour, Terragrunt permet de :

- Initialiser un KeyVault Vault avec les groupes et policies developer, devops et re7 associées.
- Créer les ServiceAccounts Kubernetes nécessaires.
- Initialiser les repositories GitLab pour les applications ArgoCD et les Helm Charts pour une nouvelle application.

Crossplane permet de :

- Configurer les rôles et les permissions Keycloak pour pouvoir se connecter à Vault, ArgoCD, et Grafana
- Assigner un utilisateur à un groupe Keycloak

## Prérequis pour suivre la documentation

- Avoir un compte Keycloak admin sur le realm PFC [lien](https://keycloak.pfccloud.esante.gouv.fr/)
- Avoir accès au repository Terragrunt [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s)
- Avoir accès au repository Crossplane [pfc-ovh-argocd-config-infrastructure](https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure)
- Avoir accès via `kubectl` au cluster d'outils de la PFC 3AZ
- Installer Terraform et Terragrunt (utilisation de tgswitch et tfswitch)

## Prérequis des demandes

Pour pouvoir traiter les demandes efficacement, voici la liste des éléments dont on a besoin :

- Nom de l'application
- Nom des environnements
- Cluster de destination pour chaque environnement
- Liste des utilisateurs

Pour chaque utilisateur, nous avons besoin de :

- Nom
- Prénom
- Email
- Rôle: DevOps, Developer ou Recette (re7)
- Applications auxquelles il doit accéder

## Créer les KV Vault et les ServiceAccounts

- Dans le repository [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s/), dans les dossiers `layers/outils/vault/config-infra-transverse/3az-operationnel`/`<amont|production>`, ajouter un dossier avec le nom de l'application, et copier dans ce dossier le contenu d'un dossier d'une application déjà configurée (par exemple `mcps`).
- Ajuster les paramètres dans les fichiers `inputs.hcl` (notamment les environnements de l'application) si besoin.
- Faire un `terragrunt apply` pour créer les ressources dans Vault.
- Une fois les changements effectués, ouvrir une PullRequest.

## Créer les repository GitLab

- Dans le repository [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s/), dans le dossier `layers/outils/gitlab-3az`, ajouter un dossier avec le nom de l'application, et copier dans ce dossier le contenu d'un dossier d'une application déjà configurée (par exemple `mcps`).
- Ajuster les paramètres dans les fichiers `inputs.hcl` (notamment les environnements de l'application) si besoin.
- Faire un `terragrunt apply` pour créer les ressources dans Vault.
- Une fois les changements effectués, les mettre sur main (de préférence avec une PR).

## Créer un ApplicationSet pour les applications avec les SecretStores

- Créer un fichier `bootstrap-app-argocd.yaml` avec le contenu suivant (remplacer `<nom de l'app>` par le nom de l'application):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <nom de l'app>-scm
  namespace: argo-cd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: outils
  source:
    path: app/
    repoURL: https://gitlab.esante.gouv.fr/ans/transverse/pfc-ovh/pfc-ovh-argocd-configs/gitops/applications-metiers/<nom de l'app>.git
    targetRevision: main
  syncPolicy:
    automated:
      enabled: true
```
- Faire un `kubectl apply -f bootstrap-app-argocd.yaml` pour créer l'Application dans ArgoCD.
- Le fichier peut ensuite être supprimé.

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
- Dans le fichier https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure/blob/main/components/operationnel/argo-cd/outils/values.yaml, ajouter une entrée de `appsetOptions / additioalValuesFiles` pour un nouveau fichier à créer selon le modèle des fichiers déjà présents, avec comme nom `<nom de l'app>.yaml`. Le contenu du fichier `<nom de l'app>.yaml` doit être le suivant (en adaptant les permissions selon les environnements de l'application) :
```yaml
argo-cd:
  configs:
    rbac:
      policy.<nom de l'app>.csv: |
        # scm
        p, proj:outils:<nom de l'app>-devops, applications, *, outils/<nom de l'app>-scm, allow
        p, proj:outils:<nom de l'app>-devops, exec, create, outils/<nom de l'app>-scm, allow

        # dev
        p, proj:amont:<nom de l'app>-developer, applications, *, amont/<nom de l'app>-dev, allow
        p, proj:amont:<nom de l'app>-developer, exec, create, amont/<nom de l'app>-dev, allow

        # integ
        p, proj:amont:<nom de l'app>-developer, applications, *, amont/<nom de l'app>-integ, allow
        p, proj:amont:<nom de l'app>-developer, exec, create, amont/<nom de l'app>-integ, allow

        # prod
        p, proj:production:<nom de l'app>-devops, applications, *, production/<nom de l'app>-prod, allow
        p, proj:production:<nom de l'app>-devops, exec, create, production/<nom de l'app>-prod, allow

        # group mappings
        g, <nom de l'app>-devops, proj:amont:<nom de l'app>-developer
        g, <nom de l'app>-devops, proj:outils:<nom de l'app>-devops
        g, <nom de l'app>-devops, proj:production:<nom de l'app>-devops

        g, <nom de l'app>-developer, proj:amont:<nom de l'app>-developer
```

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
