# Palantír

> *« Les pierres de vision permettaient de voir au loin et de communiquer d'une tour à l'autre. »*

Serveur de domotique auto-hébergé pour le pilotage et la supervision de l'ensemble de la
maison et du terrain : chauffage, eau chaude sanitaire, clôtures électriques, énergie
(consommation et production) et potager.

---

## 🎯 Objectifs

- **Superviser** : centraliser toutes les mesures (températures, humidité, énergie, niveaux) dans une base de données temporelle unique.
- **Piloter** : automatiser les décisions (production de chaleur, charge des ballons, ouverture des zones, arrosage) plutôt que de les subir.
- **Optimiser** : faire correspondre les consommations aux périodes de production locale (hydro + solaire) et au tarif réseau.
- **Alerter** : détecter les anomalies (clôture coupée, ballon qui refroidit, production nulle, gel) avant qu'elles ne deviennent des problèmes.

---

## 🏗️ Périmètre fonctionnel

### 1. Chauffage

Installation hybride avec stockage tampon.

| Élément | Rôle | À piloter / mesurer |
|---|---|---|
| Chaudière **fioul** | Appoint / secours | Commande marche-arrêt, temps de fonctionnement, consommation estimée |
| Chaudière **bois** | Source principale | Température de départ, détection de flambée, fin de combustion |
| **Ballon tampon** | Stockage hydraulique | Sondes de stratification (haut / milieu / bas), énergie stockée |
| **2 × ballons ECS** | Eau chaude sanitaire | Température, priorité de charge, appoint électrique |
| **Zones de chauffe** | Distribution | Vannes thermostatiques + sondes T/RH SONOFF par pièce/zone |

**Logique cible :**
- Le bois est prioritaire ; le fioul ne démarre que si le tampon descend sous un seuil et que le bois ne monte pas.
- L'ECS est chargée depuis le tampon quand celui-ci est chaud ; l'appoint électrique privilégie les heures de production solaire/hydro excédentaire.
- Chaque zone régule indépendamment via consigne + programmation horaire, avec prise en compte de l'humidité (confort et point de rosée).

### 2. Clôtures électriques (chevaux)

- Commande marche/arrêt des électrificateurs via **prises connectées SONOFF**.
- Surveillance de la consommation des prises pour détecter une coupure ou un électrificateur hors service.
- Alerte immédiate en cas de perte de clôture (sécurité animale prioritaire).

### 3. Énergie

- **Consommation électrique** : global + sous-comptage par usage (chauffage, ECS, clôtures, maison).
- **Production hydro** : suivi de puissance et d'énergie produite, détection d'arrêt (crue, colmatage, défaut).
- **Production solaire** : puissance instantanée, production journalière, prévision.
- **Bilan** : autoconsommation, surplus, déclenchement de charges opportunistes (ECS, autres) sur excédent.

### 4. Potager

- Sondes de température / humidité (air et sol).
- Arrosage piloté (programmation + conditions météo).
- Suivi saisonnier : gel, cumul de degrés-jours, historique des cultures.

---

## 🧱 Architecture envisagée

```
   Capteurs / actionneurs            Passerelles              Cœur                 Interfaces
 ─────────────────────────      ──────────────────     ─────────────────      ─────────────────
  SONOFF T/RH, prises      ──►   Zigbee2MQTT      ──►                    ──►   Tableaux de bord
  Vannes thermostatiques   ──►   sur ZBBridge-P   ──►   Broker MQTT      ──►   Alertes (mobile)
        [Zigbee]                 (Tasmota, TCP)         Moteur de règles ──►   API
  Sondes ballons (1-Wire)  ──►   Passerelle 1-Wire──►   Base temporelle
  Compteurs élec / onduleur──►   Modbus / P1      ──►
```

**Briques pressenties** :
- **Home Assistant** comme cœur d'intégration et d'automatisation.
- **Mosquitto** (MQTT) comme bus de messages.
- **Zigbee2MQTT** pour tout le parc SONOFF — sondes T/RH, prises de clôture, vannes
  thermostatiques. Un seul réseau Zigbee, un seul coordinateur, aucune dépendance au cloud eWeLink.
- **InfluxDB / TimescaleDB + Grafana** pour l'historique long terme et les graphiques énergie.
- Déploiement **Docker Compose** sur le **NAS Synology** (192.168.0.5).

---

## 🚀 Déploiement

Le serveur tourne en **Docker sur le NAS Synology** (`192.168.0.5`), comme les autres projets
auto-hébergés. Tous les services (cœur domotique, broker MQTT, base temporelle, Grafana) sont
décrits dans un unique `docker-compose.yml` versionné ici.

```bash
# Dépôt distant sur le NAS (hook post-receive → redéploiement automatique)
git remote add origin ssh://erendil@192.168.0.5/volume1/git/palantir.git
git push origin master

# Vérifier les conteneurs
ssh -i ~/.ssh/id_ed25519 erendil@192.168.0.5 'docker ps'
```

**Points d'attention propres au NAS :**
- **Coordinateur Zigbee** : déporté sur une passerelle **SONOFF ZBBridge-P** (Tasmota, mode TCP).
- **Réseau** : `network_mode: host` ou macvlan pour la découverte des onduleurs et compteurs IP
  (le Zigbee, lui, ne dépend pas du réseau IP).
- **Volumes persistants** : configuration et base temporelle sur un volume du NAS, inclus dans
  Hyper Backup.
- **Disponibilité** : le NAS héberge des fonctions touchant à la sécurité (clôtures), d'où la
  règle de dégradation sûre ci-dessous — redémarrage automatique des conteneurs et repli local
  des équipements.

---

## 📡 Réseau Zigbee

Tout le parc SONOFF (sondes T/RH, prises de clôture, vannes thermostatiques) est en **Zigbee**,
géré par **Zigbee2MQTT** qui publie sur Mosquitto. Un seul coordinateur pour l'ensemble.

### Coordinateur : SONOFF ZBBridge-P

Matériel déjà disponible : une **passerelle SONOFF Zigbee Bridge Pro (ZBBridge-P)**. Elle sert de
coordinateur **déporté**, relié à Zigbee2MQTT par TCP sur le Wi-Fi — le NAS n'a donc besoin
d'aucun dongle USB, ce qui écarte d'emblée le problème des pilotes série DSM et celui du placement
radio en baie.

Le ZBBridge-P embarque un **ESP32** + une puce radio **CC2652** (protocole **ZNP**, à ne pas
confondre avec le ZBBridge non-Pro qui est en EZSP). Mise en service, dans l'ordre :

1. **Flasher Tasmota** sur l'ESP32 — build `Tasmota32 Sonoff-ZigbeeBridgePro`, qui contient déjà
   les fichiers et le système de fichiers attendus. Pas de soudure nécessaire.
2. **Flasher le firmware coordinateur** sur la puce CC2652, depuis Tasmota.
3. **Basculer en mode TCP** : interface web → *Configuration* → *Auto-configuration* →
   `Sonoff ZBPro TCP`. Tasmota ne fait alors plus que relayer le flux série ; les commandes Zigbee
   de la console Tasmota ne sont plus disponibles.
4. **Pointer Zigbee2MQTT** vers la passerelle :

```yaml
serial:
  port: tcp://<ip-du-zbbridge>:8888   # port défini côté Tasmota
  adapter: zstack                     # CC2652 / ZNP (et non ezsp, réservé au ZBBridge non-Pro)
```

> ⚠️ **Limite connue.** La documentation Zigbee2MQTT déconseille les ponts série **en Wi-Fi** :
> tolérance insuffisante aux pertes de paquets et à la latence. C'est acceptable pour démarrer
> sans rien acheter, mais à surveiller — si le réseau s'avère instable, deux replis : fixer l'IP
> et soigner la couverture Wi-Fi du ZBBridge, ou passer à un dongle USB dédié (auquel cas il
> faudra régler la question des pilotes série DSM).

Conséquence pratique : **placer le ZBBridge-P au centre de gravité du parc**, pas à côté du NAS —
au plus près du bâti et de la direction des extérieurs, avec une bonne couverture Wi-Fi. Son
emplacement détermine toute la portée du réseau Zigbee.

### Portée et maillage

Le point sensible du projet, ce sont les **clôtures pour chevaux** : elles sont dehors, loin, et
c'est la fonction la plus critique.

- Les **prises SONOFF sur secteur sont des routeurs Zigbee** : elles étendent le maillage. Les
  sondes T/RH sur pile, elles, sont des terminaux et ne relaient rien.
- Prévoir des **prises-relais intermédiaires** (grange, dépendance, abri) pour construire une
  chaîne jusqu'aux électrificateurs.
- Si la distance reste hors de portée, ne pas forcer le Zigbee : envisager du **LoRa / 868 MHz**
  ou un second réseau pour les points éloignés.

### Bonnes pratiques

- **Canal Zigbee** choisi pour ne pas recouvrir le Wi-Fi 2,4 GHz (typiquement canal 15, 20 ou 25).
- **Canal Wi-Fi du ZBBridge-P** choisi lui aussi pour ne pas se recouvrir avec le canal Zigbee : la passerelle émet en Wi-Fi et en Zigbee sur la même bande 2,4 GHz.
- **Sauvegarder la clé réseau et le coordinator backup** de Zigbee2MQTT — sans eux, ré-appairer
  tout le parc, y compris les capteurs hors d'atteinte.
- **Noms stables** : `friendly_name` explicites (`cloture.pre_nord`, `zone.salon`), jamais l'adresse IEEE dans les automatisations.
- **Batteries** : alerte sur niveau faible et sur capteur muet depuis X heures (un capteur qui se
  tait est plus dangereux qu'un capteur qui remonte une valeur fausse).

---

## 🚦 État du projet

🌱 **Phase 0 — Cadrage.** Rien n'est encore déployé. Prochaines étapes :

- [ ] Inventorier précisément le matériel existant (modèles SONOFF Zigbee, type de sondes, compteurs, onduleurs).
- [ ] Flasher le ZBBridge-P (Tasmota + firmware CC2652) et le basculer en mode TCP.
- [ ] Choisir le cœur domotique et la stratégie de persistance.
- [ ] Schématiser l'installation hydraulique (chaudières, tampon, ECS, zones) et placer les sondes.
- [ ] Mettre en place le socle sur le NAS (Docker Compose) : broker MQTT + cœur domotique + base temporelle.
- [ ] Intégrer par lots : (1) clôtures, (2) énergie, (3) chauffage, (4) potager.
- [ ] Écrire les automatisations et les alertes.
- [ ] Sauvegardes et plan de reprise (une panne du serveur ne doit pas couper le chauffage ni les clôtures).

---

## ⚠️ Principes de conception

1. **Dégradation sûre.** Si Palantír tombe, le chauffage et les clôtures doivent continuer de fonctionner en mode autonome. Aucune automatisation ne doit être un point de défaillance unique.
2. **Local d'abord.** Pas de dépendance au cloud pour les fonctions critiques.
3. **Tout mesurer, automatiser ensuite.** On instrumente et on observe avant d'écrire des règles.
4. **Infrastructure versionnée.** Configuration et automatisations en Git, déploiement reproductible.

---

## 📁 Structure du dépôt

```
palantir/
├── README.md              # ce fichier
├── docker-compose.yml     # pile de services déployée sur le NAS (à venir)
├── config/                # configuration des services, versionnée (à venir)
└── docs/                  # schémas, inventaire matériel, notes d'installation (à venir)
```

---

## 📜 Licence

Projet personnel. Aucune licence définie pour l'instant.
