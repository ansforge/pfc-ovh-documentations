# Documentation publique

## Schémas Excalidraw

Des schémas d'architecture utilisés dans cette documentation sont faits avec Excalidraw. Les schémas sont stockés en AsCode dans le dossier `schemas`.

Les schémas peuvent être édités directement depuis VSCode en utilisant l'extension [Excalidraw](https://marketplace.visualstudio.com/items?itemName=pomdtr.excalidraw-editor).

Une fois l'extension installée, vous pouvez éditer directement les schémas et les sauvegarder comme n'importe quel fichier.

Pour ajouter des schémas dans la documentation, vous pouvez :

1. Sélectionner la partie du schéma que vous souhaitez intégrer
2. Cliquer sur l'icône du menu en haut à gauche, puis sur `Export Image`
3. Si la visualisation vous convient, cliquer sur `Télécharger SVG`
4. Ajouter le fichier SVG téléchargé dans le dossier `src`
5. Ajouter une balise dans votre documentation pour y intégrer l'image SVG que vous venez de créer

```markdown
![Nom amical du schema](./src/nom-du-schema.svg)
```
