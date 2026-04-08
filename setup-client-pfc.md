# Comment configurer une nouvelle application ou un nouvel environnement client sur la PFC

## Contexte

Historiquement, cette documentation était [ici](https://gitlab.pfccloudovh.proxy.prod.forge.esante.gouv.fr/ans/transverse/pfc-ovh/pfc-ovh-documentations/-/blob/main/ajout-application.md?ref_type=heads).

L'équipe Theodo a travaillé pour automatiser des parties de la configuration d'une application sur la PFC.

À ce jour, Terragrunt permet de :

- Initialiser un keyvault Vault avec les groupes et policies developer et devops associées.
- Initialiser le namespace et générer un token Vault.
- Renouveler les tokens Vault de toutes les applications après expiration.
- Initialiser les repositories GitLab pour une nouvelle application.

Crossplane permet de :

- Configurer les rôles et les permissions Keycloak pour pouvoir se connecter à Vault et ArgoCD
- Assigner un utilisateur à un groupe Keycloak

## Prérequis pour suivre la documentation

- Avoir un compte Keycloak admin sur le realm PFC [lien](https://keycloak.pfccloudovh.esante.gouv.fr/)
- Avoir accès au repository GitHub Terragrunt [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s)
- Avoir accès au repository Crossplane [pfc-ovh-argocd-config-infrastructure](https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure)
- Installer Terraform et Terragrunt (utilisation de tgswitch et tfswitch)

## Prérequis des demandes

Pour pouvoir traiter les demandes efficacement, voici la liste des éléments dont on a besoin :

- Nom de l'application
- Nom des environnements
- Cluster de destination pour chaque environnement
- Besoin de créer les repositories ? oui/non
- Création du rôle developer ? oui/non
- Création du rôle de recette ? oui/non
- Liste des utilisateurs DevOps
- Liste des utilisateurs Developer
- Liste des utilisateurs Recette

Pour chaque utilisateur, nous avons besoin de :

- Nom
- Prénom
- Email
- Rôles
- Applications auxquelles il doit accéder

### Inviter un nouvel utilisateur

Si l'utilisateur n'a pas de compte, il va falloir l'inviter. Pour l'instant, cette partie est encore manuelle.
Suivre les instructions suivantes :

- Se connecter à Keycloak avec un compte administrateur sur le realm PFC et aller [ici](https://keycloak.pfccloudovh.esante.gouv.fr/admin/master/console/#/PFC/users).
- Vérifier que l'utilisateur n'a pas encore de compte dans le realm PFC
- Cliquer sur `Add User`
- Ajouter `Configure OTP` et `Update password` comme `Required user actions`
- Mettre en username `$prenom.$name` tout en minuscules
- Remplir l'email, le First name et le Last name
- Envoyer le mail suivant avec Christian en copie (ne pas oublier de remplacer toutes les valeurs `$`)

```txt
TO: $EMAIL
CC: Christian.CRIMETZ@esante.gouv.fr
Subject: ANS-PFC Accès à $APPLICATION
Content:
Bonjour $PRENOM,

On m'a demandé de vous créer un compte développeur pour l'application $APPLICATION.

Voici les liens des outils :
• https://argocd.pfccloudovh.esante.gouv.fr/ (cliquer sur « Login with Keycloak »)
• https://vault.pfccloudovh.esante.gouv.fr/ (choisir comme méthode de connexion « OIDC », puis cliquer sur « Sign in with OIDC Provider »)

Lors de la première connexion :
• Identifiant : $prenom.$name
• Mot de passe (à changer en cliquant sur mot de passe oublié)
• Il faudra configurer la double authentification (OTP)

Je reste à votre disposition si vous avez des questions.
Cordialement,
```

### Initialiser un keyvault Vault

- Cloner le repository GitHub [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s)
- Dans le dossier `layers/vault/applications`, créer un nouveau dossier `$applicationName`
- Dans le dossier `layers/vault/applications/$applicationName`, créer 2 fichiers `inputs.hcl` et `terragrunt.hcl`
- Dans le fichier `terragrunt.hcl`, mettre :

```hcl
include "root" {
  path           = find_in_parent_folders("root.hcl")
  merge_strategy = "deep"
}

include "module" {
  path           = find_in_parent_folders("module.hcl")
  merge_strategy = "deep"
}

include "inputs" {
  path = "./inputs.hcl"
}
```

- Dans le fichier `inputs.hcl`, mettre :

```hcl
locals {
  application = basename(get_parent_terragrunt_dir())
}

inputs = {
  application = local.application
  environments = {
    "$ENVNAME" = {
      name         = "$ENVNAME"
      create_token = true
    }
  }
}
```

- Exporter les variables permettant de faire du Terraform :

```bash
export TF_VAR_s3_bucket_name="cc1668915-ovh-pfcprod-tf-state"
export AWS_REGION="gra"
export AWS_S3_ENDPOINT="s3.$AWS_REGION.io.cloud.ovh.net"
export AWS_ACCESS_KEY_ID="****"
export AWS_SECRET_ACCESS_KEY="****"
export VAULT_TOKEN="hvs.****"
```

- Dans un terminal, dans ce dossier, lancer les commandes suivantes :

```bash
tgswitch 0.99.1
tfswitch 1.14.5
terragrunt apply
```

- Faire `yes` pour valider la création
- Vérifier que le keyvault a bien été créé dans Vault
- Faire un commit avec les 3 fichiers `terragrunt.hcl`, `inputs.hcl` et `.terraform.lock.hcl`

### Initialiser un token Vault et un namespace

- Cloner le repository GitHub [terraform-ovh-platform-k8s](https://github.com/ansforge/terraform-ovh-platform-k8s)
- Dans le dossier `layers/vault/renew-token/operationnel`, dans le dossier correspondant au bon cluster (par exemple `production`)
- Dans le fichier `inputs.hcl`, ajouter la configuration du nouveau namespace et du nouveau token :

```hcl
inputs {
  applications = {
....
    "$application$-prod" = {
      k8s_secret_name      = "$application-prod-token"
      k8s_namespace        = "$application-prod"
      k8s_create_namespace = true
      vault_policy_name    = "$application-prod"
    }
  }
}
```

- Dans un terminal, dans le dossier `layers/vault/renew-token/operationnel/production` (dans cet exemple), exécuter les commandes suivantes :

```bash
tgswitch 0.99.1
tfswitch 1.14.5
terragrunt apply
```

- Vérifier que le namespace a bien été créé dans le bon cluster et que le token existe dans un secret Kubernetes
- Faire un commit avec le fichier `inputs.hcl`

### Configurer Keycloak avec Crossplane

- Cloner le repository GitHub [pfc-ovh-argocd-config-infrastructure](https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure)
- Dans le dossier `components/operationnel/keycloak-clients-config/outils`, dans le fichier `values.yaml`
- Ajouter la configuration suivante :

```yaml
applications:
    ....

  - name: $ApplicationName
    roles:
      - name: $ApplicationName-devops
        groups:
          - name: $ApplicationName-devops

groups:
    ....

  - name: $ApplicationName-devops
    users:
      - name: $prenom1.$name1
        id: $id1
      - name: $prenom2.$name2
        id: $id2
```

- Faire un commit : ArgoCD va s'occuper de créer les ressources Crossplane, et Crossplane s'occupera de créer la configuration Keycloak

### Configurer les droits ArgoCD

- Dans le repository GitHub [pfc-ovh-argocd-config-infrastructure](https://github.com/ansforge/pfc-ovh-argocd-config-infrastructure)
- Dans le fichier `/applications/operationnel/argocd-helm/forge/values.yaml`
- Ajouter une policy RBAC (ne garder que la configuration des environnements qui vous intéressent) :

```yaml
argo-cd:
  configs:
    rbac:
      policy.$applicationName.csv: |
        # scm
        p, proj:forge:$applicationName-devops, applications, *, forge/$applicationName-scm, allow
        p, proj:forge:$applicationName-devops, exec, create, forge/$applicationName-scm, allow

        # amont
        p, proj:amont:$applicationName-devops, applications, *, amont/$applicationName-amont, allow
        p, proj:amont:$applicationName-devops, exec, create, amont/$applicationName-amont, allow

        # prod
        p, proj:prod:$applicationName-devops, applications, *, prod/$applicationName-prod, allow
        p, proj:prod:$applicationName-devops, exec, create, prod/$applicationName-prod, allow

        # group mappings
        g, $applicationName-devops, proj:amont:$applicationName-devops
        g, $applicationName-devops, proj:forge:$applicationName-devops
        g, $applicationName-devops, proj:prod:$applicationName-devops
```

- Faire un commit et ArgoCD va déployer les changements automatiquement

### Initialiser un repository GitLab

// TODO
