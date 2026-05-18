# Gestion des permissions de l'utilisateur admin Postgresql d’une db managée par ovh

## Pourquoi ?
Les équipes applicatives ont besoin d’un utilisateur `admin` avec des permissions élevées pour pouvoir gérer les bases de données utilisées par leurs applications (par exemple, pour créer des utilisateurs, créer des bases de données, ou encore voir toutes les lignes des tables en passant outre la sécurité des RLS).

OVH Cloud crée par défaut un utilisateur `advadmin` avec toutes les permissions, y compris des permissions uniquement utiles pour la maintenance. C'est pourquoi il est mieux de créer un utilisateur `admin` avec les permissions nécessaires pour les équipes applicatives, et de laisser l'utilisateur `avnadmin` pour les opérations de maintenance.

L'utilisateur `admin` est créé par terraform (par exemple avec [le module `ansforge/terraform-module-postgres`](https://github.com/ansforge/terraform-module-postgres)), mais OVH ne permet pas de gérer les permissions des utilisateurs de la base de données avec terraform. Il est donc nécessaire de faire cette configuration manuellement, en suivant les étapes de cette documentation.

### Prérequis
Il y a 2 prérequis pour pouvoir suivre cette documentation :
- Avoir les credentials de l'utilisateur `avnadmin` pour la base de données managée par OVH:
  - *Si besoin, vous pouvez réinitialiser les credentials de l'utilisateur `avnadmin` depuis la console OVH Cloud.*
- Pouvoir déployer des pods k8s sur le cluster de destination de l'application/environnement qui va utiliser la base de données (cluster amont ou cluster de production)

## Quels droits donner ?
Pour avoir un utilisateur `admin` avec les droits nécessaires pour les équipes applicatives, il est nécessaire de lui donner les droits suivants :
- Le droit de créer d'autres utilisateurs/rôles
- Le droit de créer des bases de données
- Le droit de voir toutes les lignes des tables, en passant outre la sécurité des RLS (Row Level Security)

## Comment ?
- Déployer une image contenant une console PSQL, de préférence dans le namespace default : `k run --image stangirard/alpine-powerhouse --command -- operationpod sleep infinity`
- Se connecter à la db depuis cette image avec PSQL: `k exec -it operationpod -- psql <connection string>`, en remplaçant `<connection string>` par la chaîne de connexion à la base de données, en utilisant les credentials de l'utilisateur `avnadmin` (par exemple `postgresql://avnadmin:<AVNADMIN_PASSWORD>@<DB_HOST>:<DB_PORT>/<DB_NAME>?sslmode=require`)
- vérifier que les permissions actuelles de l’utilisateur admin sont restreintes, avec la commande psql `\du admin` (vous pouvez comparer à `\du avnadmin` pour voir la différence de permissions)
- faire la commande `ALTER ROLE admin CREATEROLE CREATEDB BYPASSRLS;` pour donner les permissions supplémentaires au user admin
- vérifier que l’utilisateur admin a bien les nouvelles permissions avec la commande psql `\du admin` (il ne devrait pas avoir la permission `Replication` qui est présente pour avnadmin, mais il devrait avoir les permissions `Create role`, `Create DB` et `Bypass RLS` qui n’étaient pas présentes avant)
- supprimer le pod de debug : `k delete pod operationpod`
