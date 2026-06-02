# Comment deployer et configurer Loki sur la PFC (3AZ)

## Contexte

Loki est un système qui est souvent utilisé pour collecter, stocker et analyser les logs d'applications et d'infrastructures.

## Prérequis - Infra
Loki est déployé et configuré en mode distribué, il utilise 3 buckets S3 pour stocker les données (pfc-loki-chunks, pfc-loki-admin, pfc-loki-rulers)

#TODO
