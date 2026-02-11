# Changer les node pools d'un cluster Kubernetes de la PFC

Dans le cadre de notre mission, Theodo Cloud a été amené à changer le type de nœuds utilisés sur les clusters Kubernetes. Voici la stratégie qui a été utilisée.

## Prérequis

Voici la liste des prérequis nécessaires aux changements de node pool.

- Avoir accès aux VMs pour utiliser le code Terraform
    test : 57.128.106.129
    amont : 51.68.85.205
    prod : 188.165.71.190

- Avoir accès aux repositories GitLab Terraform
- Avoir accès aux clusters Kubernetes
- Pour les clusters `forge` uniquement : avoir les clés pour « unseal » le vault.

## Stratégie globale

Le but de cette section est de donner une vue d'ensemble de la procédure suivie.

1. Création d'un nouveau node pool `extra` avec un autre nom via Terraform
2. Migration de tous les pods sur le nouveau node pool (cordon puis drain des anciens nœuds)
3. Déverrouiller le vault si nécessaire à la suite du redémarrage des pods.
4. Modification du node pool d'origine pour avoir les propriétés souhaitées (dans notre cas le type de machine)
5. Migration de tous les pods sur le node pool d'origine (cordon puis drain des nœuds `extra`)
6. Déverrouiller le vault si nécessaire à la suite du redémarrage des pods.
7. Suppression du node pool `extra`

## Stratégie détaillée

### Création d'un nouveau node pool `extra` avec un autre nom via Terraform

- Se connecter à la VM pour faire du Terraform sur le cluster souhaité
- Aller dans le dossier `/data/pfc-ovh-terraform-sources/infra`
- Charger les variables d'environnement suivantes :

```bash
        # Alias
        alias tf="terraform"

        # Module
        export MODULE="infra"
        export SOURCES_PATH="/data/pfc-ovh-terraform-sources/"

        # S3
        export TF_VAR_s3_bucket_name="ovh-ans-pfctest-tf-state"
        export AWS_REGION="gra"
        export AWS_S3_ENDPOINT="s3.$AWS_REGION.io.cloud.ovh.net"
        export AWS_ACCESS_KEY_ID="****"
        export AWS_SECRET_ACCESS_KEY="*****"

        # ovh
        export OVH_ENDPOINT=ovh-eu
        export OVH_APPLICATION_KEY="****"
        export OVH_APPLICATION_SECRET="****"
        export OVH_CONSUMER_KEY="****"
        export OVH_CLOUD_PROJECT_SERVICE="****"

        # Openstack - test
        export OS_TENANT_ID="****"
        export OS_USERNAME="****"
        export OS_PASSWORD="****"
```

- Créer les variables localement : (choisir le workspace approprié)

```bash
        export WORKSPACE="amont"
        export WORKSPACE="prod"
        export WORKSPACE="forge"
        cat /data/pfc-ovh-terraform-test/$WORKSPACE/infra/terraform.properties > env.auto.tfvars
        tf workspace select $WORKSPACE
```

- Modifier le fichier kubernetes.tf pour y ajouter le nouveau node pool

```terraform
        resource "ovh_cloud_project_kube_nodepool" "extra_pool" {
        kube_id        = ovh_cloud_project_kube.cluster.id
        name           = "${terraform.workspace}-instance"
        service_name   = "8092ef6a7c454c1394f1473f7a167001"
        flavor_name    = var.k8s_extra_nodepool_flavor
        monthly_billed = var.k8s_extra_nodepool_monthly_billed
        min_nodes      = var.k8s_extra_nodepool_min_nodes
        max_nodes      = var.k8s_extra_nodepool_max_nodes
        desired_nodes  = var.k8s_extra_nodepool_desired_nodes
        autoscale      = var.k8s_extra_nodepool_autoscale

        timeouts {
                create = "30m"
                update = "30m"
                delete = "30m"
        }

        depends_on = [ovh_cloud_project_kube_iprestrictions.allowed_ips]
        lifecycle {
                ignore_changes = [desired_nodes]
        }
        }
```

- Ajouter ces nouvelles variables avec les valeurs appropriées dans `env.auto.tfvars` pour le nouveau node pool (valeurs temporaires)

```terraform
        k8s_extra_nodepool_flavor="b3-64"
        k8s_extra_nodepool_monthly_billed="false"
        k8s_extra_nodepool_min_nodes=1
        k8s_extra_nodepool_max_nodes=10
        k8s_extra_nodepool_desired_nodes=6
        k8s_extra_nodepool_autoscale="false"
```

- Créer le nouveau node pool `tf apply -target ovh_cloud_project_kube_nodepool.extra_pool`

### Migration de tous les pods sur le nouveau node pool (cordon puis drain des anciens nœuds) \*

- Dans le cluster Kubernetes, cordonner les nœuds de l'ancien node pool
    Les nœuds ne vont plus accepter de nouveaux pods.
- Puis drainer les nœuds :
    Les pods vont se faire déplacer sur d'autres nœuds.

```bash
        kubectl cordon <node-name>
        kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### Déverrouiller le vault si nécessaire à la suite du redémarrage des pods. \*\*

- Se connecter dans un pod qui est en 0/1 : `kubectl exec -it <pod-name> -- bash`
- Utiliser les "unseal keys" pour unseal le vault, à faire 3 fois

```bash
vault operator unseal <unseal-key1>
vault operator unseal <unseal-key2>
vault operator unseal <unseal-key3>
exit
```

- Le pod du vault doit passer en Ready 1/1

### Modification du node pool d'origine pour avoir les propriétés souhaitées (dans notre cas le type de machine)

Quand plus aucun pod ne fonctionne sur l'ancien node pool.

- Modifier les valeurs dans le fichier `env.auto.tfvars` (valeurs finales)

```terraform
        k8s_nodepool_flavor="b3-64"
        k8s_nodepool_monthly_billed="false"
        k8s_nodepool_min_nodes=1
        k8s_nodepool_max_nodes=10
        k8s_nodepool_desired_nodes=6
        k8s_nodepool_autoscale="false"
```

- Modifier le node pool existant `tf apply -target ovh_cloud_project_kube_nodepool.pool` (dans notre cas, une suppression/recréation était nécessaire)
- Aller dans le repository approprié à l'environnement sur lequel vous travaillez et mettre à jour les valeurs dans le fichier (par exemple pour amont)
    `/data/pfc-ovh-terraform-amont/amont/infra/terraform.properties`
- Faire une Merge Request

### Migration de tous les pods sur le node pool d'origine (cordon puis drain des nœuds `extra`)

Voir l'étape \*.

### Déverrouiller le vault si nécessaire à la suite du redémarrage des pods

Voir l'étape \*\*.

### Suppression du node pool `extra`

Quand plus aucun pod ne fonctionne sur le node pool `extra`.

- Supprimer l'ancien node pool `tf destroy -target ovh_cloud_project_kube_nodepool.extra_pool`
- Dans le fichier `kubernetes.tf`, supprimer la ressource : `ressource "ovh_cloud_project_kube_nodepool" "extra_pool"`
- Supprimer le fichier `env.auto.tfvars`

## Remarques

Cette méthode est perfectible et nécessite de drainer 2 fois un node pool, ce qui veut dire 2 interruptions de service.
La mise en place d'autres mécanismes Kubernetes comme les pdb et des pods antiaffinity peut permettre d'éviter le downtime sur les services lors de ces opérations.
Si l'on ne souhaite pas que le nom du node pool reste inchangé à tout prix, on peut utiliser les commandes `terraform state rm` et `terraform import` pour n'avoir à créer qu'un seul node pool.
