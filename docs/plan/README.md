# Plans du bâtiment — méthode

Objectif double, traité dans cet ordre :

1. **Documentation technique** — placer les capteurs, dimensionner le maillage Zigbee, câbler.
2. **Tableau de bord Home Assistant** — un rendu 3D avec les états superposés en temps réel.

Le même relevé sert aux deux. D'où la règle : **on mesure une fois, proprement, et tout le reste
en découle.** Un relevé approximatif se paye deux fois.

---

## Étape 1 — Relevé 2D coté

C'est le seul travail qu'aucun outil ne fera à ta place, et celui dont tout dépend.

### Ce qu'il faut mesurer

Par pièce :

- **Dimensions** au sol (longueur × largeur), en cm.
- **Hauteur sous plafond** — elle change souvent d'une pièce à l'autre dans l'ancien.
- **Épaisseur et nature des murs** : cloison placo, brique, pierre. Déterminant pour le Zigbee —
  un mur en pierre de 60 cm coupe la radio bien plus qu'une cloison.
- **Ouvertures** : portes et fenêtres, position et largeur.
- **Radiateurs** : position, et **type de robinet** (le TRV-ZBT se visse sur un corps standard
  M30×1,5 — à vérifier robinet par robinet, c'est la mauvaise surprise classique).
- **Prises électriques** disponibles : elles conditionnent où poser les S60ZBTPF, donc où passe le
  maillage Zigbee.

Pour le terrain, en plus : distances **bâtiment → électrificateurs de clôture**, et les
dépendances intermédiaires (grange, abri) susceptibles d'accueillir une prise-relais.

### Conventions

- **Unité : le centimètre.** Pas de pieds, pas de mètres à virgule flottante.
- **Origine** : angle nord-ouest du bâtiment principal, axe X vers l'est, Y vers le sud.
- **Nommage des pièces** : identique à celui des entités Home Assistant
  (`salon`, `cuisine`, `chambre_nord`…), en minuscules sans accent. C'est ce qui permettra de
  relier automatiquement une pièce du plan à ses capteurs.

### Où consigner

Remplir [`releve.md`](releve.md) au fur et à mesure. Ce fichier est la **source de vérité** :
le plan et la 3D en sont des rendus, pas l'inverse.

---

## Étape 2 — Plan 2D

Une fois le relevé fait, le plan se dessine vite. Deux options selon ce dont tu as envie :

| Outil | Pour quoi | Remarque |
|---|---|---|
| **Sweet Home 3D** | Recommandé | On dessine en 2D, la 3D se construit toute seule — donc l'étape 3 est déjà à moitié faite. Gratuit, en français, tourne sous Windows. |
| **draw.io / Excalidraw** | Plan schématique rapide | Suffisant pour raisonner sur le maillage Zigbee, mais impasse pour la 3D. |
| **QCAD / LibreCAD** | Plan coté précis | Si tu veux du DXF propre, mais aucune 3D derrière. |

**Recommandation : Sweet Home 3D.** C'est le seul qui ne demande pas de refaire le travail à
l'étape 3. Compter quelques heures pour une maison entière une fois le relevé en main.

Exporter dans `docs/plan/` :

- le fichier source (`.sh3d`) — versionné, c'est lui qui se modifie ;
- un export **PNG** du plan de chaque niveau, pour la lecture directe dans le dépôt.

### Couches à représenter

Reprendre la logique de la référence : des **zones colorées** par usage, et les **réseaux** par
dessus.

- Une couleur par **zone de chauffe** (une zone = un TRV + une sonde).
- Les emplacements des **SNZB-02D** — à mi-hauteur, loin des radiateurs, des fenêtres et des
  sources de chaleur, sinon la régulation par sonde déportée perd tout son intérêt.
- Les emplacements des **S60ZBTPF** : c'est la carte du maillage Zigbee, puisque ce sont les seuls
  routeurs du réseau.
- La position du **ZBBridge-P** et les distances jusqu'aux points les plus éloignés.
- Le passage des **réseaux hydrauliques** depuis la chaufferie.

---

## Étape 3 — 3D

Deux livrables distincts, à ne pas confondre :

### 3a — Volume de la maison

Découle directement du Sweet Home 3D de l'étape 2. Exporter une **vue isométrique** (caméra
orthographique, vue de dessus inclinée) en PNG haute résolution vers `docs/rendus/`.

C'est cette image qui servira de fond au tableau de bord Home Assistant : on y superpose les
entités avec une carte `picture-elements`, en positionnant chaque température sur sa pièce. Le
rendu est figé, seules les valeurs bougent — d'où l'importance d'un export en haute résolution et
d'une caméra dont on **note les réglages** pour pouvoir refaire l'image à l'identique plus tard.

### 3b — Chaufferie

À faire **après** la validation du [schéma hydraulique](../chaufferie/schema-hydraulique.md), pas
avant. La 3D sert au repérage physique — retrouver quelle sonde est où quand on a les mains dans
l'installation — tandis que le schéma porte la logique de régulation. Modéliser la pièce, les deux
chaudières, le tampon, les deux ECS et la tuyauterie principale, puis **annoter chaque point de
mesure avec son repère** (`T_tampon_haut`…).

---

## Structure des fichiers

```
docs/
├── plan/
│   ├── README.md          # ce document
│   ├── releve.md          # relevé des dimensions — source de vérité
│   ├── maison.sh3d        # modèle Sweet Home 3D (à venir)
│   └── niveau-*.png       # exports 2D par niveau (à venir)
├── chaufferie/
│   ├── schema-hydraulique.md
│   └── chaufferie.sh3d    # modèle 3D (à venir)
└── rendus/                # images destinées au tableau de bord HA
```

> 📦 **Fichiers binaires.** Les `.sh3d` et les PNG sont versionnés tels quels : à cette échelle le
> dépôt reste petit, et l'historique d'un plan a de la valeur. Si le volume devient gênant, on
> passera à Git LFS — pas avant.
