# Relevé — source de vérité

Toutes les dimensions en **centimètres**. Nommage des pièces identique aux entités Home Assistant
(minuscules, sans accent, `_` comme séparateur).

> À remplir sur site, mètre en main. Le plan 2D et la 3D seront des rendus de ce tableau.

## Niveaux

| Niveau | Nom | Hauteur sous plafond | Remarques |
|---|---|---|---|
| 0 | rez_de_chaussee | | |
| 1 | etage | | |
| -1 | sous_sol | | Chaufferie |

## Pièces

| Pièce | Niveau | L × l | Hauteur | Murs (nature / ép.) | Radiateurs | Prises dispo. | Zone de chauffe |
|---|---|---|---|---|---|---|---|
| salon | 0 | × | | | | | zone_salon |
| cuisine | 0 | × | | | | | |
| | | × | | | | | |

## Radiateurs et robinets

Le TRV-ZBT se visse sur un corps de robinet **M30×1,5**. Les corps anciens (Comap, Danfoss RA,
Giacomini…) demandent un adaptateur, parfois introuvable — à vérifier robinet par robinet **avant**
de commander les vannes.

| Radiateur | Pièce | Filetage constaté | Adaptateur nécessaire | TRV posé |
|---|---|---|---|---|
| | | | | ☐ |

## Équipements Zigbee

| Équipement | Modèle | Pièce / lieu | Alimentation | Rôle réseau |
|---|---|---|---|---|
| | SNZB-02D | | pile CR2450 | terminal |
| | TRV-ZBT | | piles | terminal |
| | S60ZBTPF | | secteur | **routeur** |
| coordinateur | ZBBridge-P | | secteur | coordinateur |

## Distances radio

Le maillage ne tient que par les S60ZBTPF sur secteur : sondes et vannes sur pile ne relaient
rien. Relever les distances réelles et la nature des obstacles.

| De | Vers | Distance | Obstacles traversés | Relais prévu |
|---|---|---|---|---|
| ZBBridge-P | | | | |
| ZBBridge-P | electrificateur_pre_nord | | | |

## Tableau électrique

| Départ | Usage | Module Digiware | Esclave Modbus |
|---|---|---|---|
| 1 | | I-35 | 1 |
