# Le volume d'un pod Elasticsearch est plein

## Contexte

On a rencontré plusieurs fois ce cas de figure où l'on n'augmente pas suffisamment rapidement la taille d'un volume ElasticSearch et qu'il se remplit.
Une fois le volume plein, le pod Elasticsearch peut ne plus démarrer, même après un redimensionnement du PV et du PVC.
Cette documentation explique comment modifier la taille du volume dans ce cas de figure.

## Quand ?

Il faut suivre cette documentation quand :

- le volume d'Elasticsearch est à 100 % dans Grafana.
- la taille des volumes d'Elasticsearch a été augmentée avec un commit dans les values du chart associé.
- le pod d'Elasticsearch dont le volume est plein n'arrive toujours pas à démarrer.

## Quoi faire ?

- Scale à 0 le StateFulSet elasticsearch-es-default
- Créer un pod de debug qui monte le volume du pod Elasticsearch qui n'arrive plus à démarrer. Dans cet exemple, nous prendrons le cas de `elasticsearch-es-default-0`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: es-debug
  namespace: elastic-stack
spec:
  restartPolicy: Never
  containers:
    - name: ubuntu
      image: ubuntu:24.04
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
      volumeMounts:
        - name: elasticsearch-data
          mountPath: /data
  volumes:
    - name: elasticsearch-data
      persistentVolumeClaim:
        claimName: elasticsearch-data-elasticsearch-es-default-0
```

- Faire `kubectl apply -f manifest.yaml` de ce pod pour créer le pod de debug
- Se connecter en remote-shell dans ce container :

```bash
kubectl exec -it -n elastic-stack es-debug -- /bin/bash
```

- Vérifier l'espace disque et redimensionner le filesystem :

```bash
df -Th /data
resize2fs /dev/sdg
df -Th /data
```

- Vous devriez voir des logs qui ressemblent à ceci et constater que la taille du volume a bien été mise à jour.

```log
resize2fs 1.47.x
Filesystem at /dev/sdg is mounted on /data; on-line resizing required
old_desc_blocks = ...
The filesystem on /dev/sdg is now XXXXX blocks long.
````

- Supprimer le pod de debug :

```bash
kubectl delete pod -n elastic-stack es-debug
```

- Scale à 3 le StateFulSet elasticsearch-es-default

## Contrôle

Il y a deux contrôles possibles :

1. Dans le pod de debug, vous devez constater que la taille du volume monté a augmenté entre les deux exécutions de `df -Th /data`.
1. Le pod Elasticsearch qui n'arrivait plus à démarrer doit réussir à démarrer sans souci une fois la taille du volume corrigée.
