# Recommandation de configuration des ressources CPU et RAM

Voici quelques conseils sur les bonnes pratiques recommandées pour déployer votre application sur la PFC.

## Rappels

- Toujours configurer des requests de ressources. Ne pas en configurer sur un pod est un risque d'instabilité : risque de ne pas être schedulé ou de se faire OOMKilled ou Evicted.
- Request : les ressources qui sont garanties aux containers (CPU, memory). Il faut impérativement que ces ressources soient disponibles sur un nœud, sinon le pod ne peut pas être schedulé.
- Limit : un pod peut utiliser plus de ressources que ses requests s'il en a besoin et que ces ressources sont disponibles sur le même nœud et non attribuées aux requests d'un autre pod. Ces phases s'appellent le burst, les limites permettent de limiter ce burst.

## Éviter le gaspillage

Pour éviter le gaspillage de ressources sur la PFC et réduire l'empreinte environnementale et financière, il est recommandé de ne pas surévaluer les besoins de vos applications.
Il est également recommandé d'allouer moins de ressources pour les environnements hors-production, à l'exception d'un environnement "iso" à la production pour faire des tests de performances.
Les projets sont facturés proportionnellement à leurs usages pour éviter les dérives.

## Phénomène de throttling

Dans Kubernetes, on peut partager des cœurs de CPU entre plusieurs containers. Le principe sera alors de répartir les opérations de chaque container concurrent sur des fenêtres de 100ms.
Pour les applications nécessitant une faible latence, attribuer des fractions de CPU peut provoquer une augmentation du temps de traitement appelée throttling CPU.
Notre recommandation pour éviter le throttling est d'allouer des cœurs entiers de CPU (valeur entière et pas en millièmes de CPU). Ces usages doivent se limiter uniquement aux applications qui le nécessitent.

## Configuration

Nos convictions sur la configuration des ressources :

- Pour le CPU, configurer le CPU moyen utilisé lors des périodes de charge afin de garantir les performances lors des pics d'utilisation.
- Pour le CPU, ne pas mettre de limit pour conserver la possibilité de burst et donc de répondre plus rapidement si la charge sur le nœud le permet.
- Pour la RAM, mettre request = limit, on ne veut pas qu'une partie de la mémoire à utiliser soit incertaine pour éviter les phénomènes de OOMKilled (Out Of Memory).
- Pour la RAM, configurer une valeur qui n'est jamais atteinte lors des pics de charge (prendre ~30% de marge) et si possible qui est une puissance de 2.

Exemple :

```yaml
    resources:
      limits:
        memory: 512Mi
      requests:
        cpu: 250m
        memory: 512Mi
```

## Résultat attendu

Si cette configuration est correctement ajoutée, voici les résultats théoriquement attendus :

- Maîtriser les coûts et réduire le gaspillage de ressources.
- Possibilité de burst en CPU, donc améliorations des performances de temps de réponse.
- Pas de OOMKilled si la RAM est bien sizée, sinon augmenter la mémoire à la fois en request et en limit.

## Pour aller plus loin

Bien dimensionner cette configuration est important pour bien configurer l'Horizontal Pod Autoscaling. Vous pouvez créer un ticket Jira si vous souhaitez être accompagné sur ce sujet par l'équipe d'infogérance de la PFC.
