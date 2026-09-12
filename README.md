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
| **Zones de chauffe** | Distribution | Vannes **SONOFF TRV-ZBT** + sondes **SONOFF SNZB-02D** (T/RH) par pièce/zone |

**Logique cible :**
- Le bois est prioritaire ; le fioul ne démarre que si le tampon descend sous un seuil et que le bois ne monte pas.
- L'ECS est chargée depuis le tampon quand celui-ci est chaud ; l'appoint électrique privilégie les heures de production solaire/hydro excédentaire.
- Chaque zone régule indépendamment via consigne + programmation horaire, avec prise en compte de l'humidité (confort et point de rosée).

### 2. Clôtures électriques (chevaux)

- Commande marche/arrêt des électrificateurs via prises **SONOFF S60ZBTPF** (16 A, mesure d'énergie).
- Surveillance de la puissance mesurée par la prise pour détecter une coupure ou un électrificateur hors service.
- `power_on_behavior = on` : retour automatique de la clôture après coupure secteur, sans dépendre du serveur.
- Alerte immédiate en cas de perte de clôture (sécurité animale prioritaire).

### 3. Énergie

- **Consommation électrique** : Linky (TIC) au point de livraison + **Socomec DIRIS Digiware** (M-50 / U-30 / I-35) en Modbus TCP pour le sous-comptage par départ, complétés par les S60ZBTPF.
- **Production hydro** : suivi de puissance et d'énergie produite, détection d'arrêt (crue, colmatage, défaut).
- **Production solaire** : puissance instantanée, production journalière, prévision.
- **Bilan** : soutirage / injection relevés au Linky, autoconsommation, surplus, déclenchement de charges opportunistes (ECS, autres) sur excédent.

### 4. Potager

- Sondes de température / humidité (air et sol).
- Arrosage piloté (programmation + conditions météo).
- Suivi saisonnier : gel, cumul de degrés-jours, historique des cultures.

---

## 🧱 Architecture envisagée

```
   Capteurs / actionneurs            Passerelles              Cœur                 Interfaces
 ─────────────────────────      ──────────────────     ─────────────────      ─────────────────
  SNZB-02D, S60ZBTPF       ──►                                          ──►   Tableaux de bord
  TRV-ZBT                  ──►   Zigbee2MQTT      ──►   Broker MQTT     ──►   Alertes (mobile)
  ZLinky_TIC               ──►   sur ZBBridge-P   ──►                   ──►   API
        [Zigbee]                 (Tasmota, TCP)         Moteur de règles
                                                        Base temporelle
  Socomec Digiware         ──►   M-50 (Modbus TCP)──►
  Sondes ballons           ──►   Passerelle 1-Wire──►
```

**Briques pressenties** :
- **Home Assistant** comme cœur d'intégration et d'automatisation.
- **Mosquitto** (MQTT) comme bus de messages.
- **Zigbee2MQTT** pour le parc radio — SNZB-02D, S60ZBTPF, TRV-ZBT et ZLinky_TIC.
  Un seul réseau Zigbee, un seul coordinateur, aucune dépendance au cloud eWeLink.
- **Modbus TCP** vers le DIRIS Digiware M-50 pour le comptage de tableau.
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

Tout le parc SONOFF (SNZB-02D, S60ZBTPF, TRV-ZBT) est en **Zigbee**, géré par
**Zigbee2MQTT** qui publie sur Mosquitto. Un seul coordinateur pour l'ensemble.

### Comment fonctionne la passerelle

Le Zigbee est un **réseau radio maillé**, sans IP : les équipements se parlent de proche en proche.
Ce réseau a besoin d'un **coordinateur** — un et un seul — qui en est la racine : il fixe le canal,
détient la clé réseau et autorise les nouveaux équipements à rejoindre. Le coordinateur n'est
**qu'une puce radio**, pas un ordinateur : il ne décide rien, il transmet.

L'intelligence, c'est **Zigbee2MQTT**, qui tourne sur le NAS. Il lui faut donc un **lien série**
vers une puce radio. Dans le montage classique, c'est une clé USB : `/dev/ttyUSB0`.

Le ZBBridge-P contient **deux puces** :

| Puce | Rôle |
|---|---|
| **ESP32** | Un petit ordinateur Wi-Fi |
| **CC2652** | La radio Zigbee — l'équivalent exact d'une clé USB |

**Sorti du carton**, l'ESP32 fait tourner le firmware eWeLink : la passerelle est autonome et parle
au cloud SONOFF. Ce n'est pas ce qu'on veut — on perdrait le « local d'abord ».

**Après flashage**, les rôles changent complètement : l'ESP32 ne décide plus rien, il devient un
simple **câble série qui passe par le Wi-Fi**. Il prend les octets que la radio CC2652 produit et
les recopie sur une connexion TCP, et inversement. Le CC2652, lui, redevient exactement ce qu'est
une clé USB — simplement posé ailleurs dans la maison.

D'où la configuration : `tcp://<ip>:8888` **remplace** `/dev/ttyUSB0`. C'est le même montage, avec
un câble plus long.

```
  Capteurs Zigbee              ZBBridge-P                        NAS
 ┌──────────────┐        ┌────────────────────┐        ┌────────────────────┐
 │ SNZB-02D     │        │  CC2652            │        │  Zigbee2MQTT       │
 │ TRV-ZBT      │◄─radio─┤  (la radio)        │        │  (l'intelligence)  │
 │ S60ZBTPF     │ Zigbee │       ▲            │        │        ▲           │
 │ ZLinky_TIC   │        │       │ série      │        │        │           │
 └──────────────┘        │       ▼            │        │        │           │
                         │  ESP32 / Tasmota   │◄──TCP──┤  tcp://ip:8888     │
                         │  (le câble)        │  Wi-Fi │        │           │
                         └────────────────────┘        │        ▼           │
                                                       │  MQTT → Home Assist.│
                                                       └────────────────────┘
```

**Trois conséquences pratiques :**

1. **L'emplacement de la passerelle détermine la portée du réseau**, puisque c'est là qu'est
   l'antenne. Le NAS peut rester dans sa baie ; le ZBBridge-P, non.
2. **Si le Wi-Fi tombe**, Zigbee2MQTT perd sa radio et ne voit plus rien — mais le réseau Zigbee
   lui-même et les appairages sont intacts, et tout revient seul à la reconnexion.
3. **Une IP fixe est obligatoire** (réservation DHCP) : Zigbee2MQTT se connecte à une adresse
   écrite en dur dans `.env`.

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

### Parc d'équipements

| Modèle | Rôle | Alimentation | Rôle réseau |
|---|---|---|---|
| **SONOFF SNZB-02D** | Sonde T/RH avec écran, par zone | Pile CR2450 | Terminal (ne route pas) |
| **SONOFF TRV-ZBT** (TRVZB) | Vanne thermostatique de radiateur | Piles | Terminal (ne route pas) |
| **SONOFF S60ZBTPF** (iPlug S60) | Commande + mesure des électrificateurs et des usages élec | Secteur, 16 A | **Routeur** (étend le maillage) |
| **SONOFF ZBBridge-P** | Coordinateur | Secteur | Coordinateur |

⚠️ **Conséquence importante** : sondes et vannes sont toutes sur pile, donc **terminaux**. Les
seuls routeurs du réseau seront les **prises sur secteur**. Le maillage dépend donc entièrement
d'elles — il faudra en prévoir davantage que le strict nécessaire fonctionnel, uniquement pour
tenir le réseau.

**SNZB-02D** — expose `temperature`, `humidity`, `battery`, plus la calibration
(`temperature_calibration`, `humidity_calibration`), la précision, les seuils de confort
(`comfort_temperature_min/max`, `comfort_humidity_min/max`) et l'unité d'affichage. À savoir :
il faut **réveiller la sonde par le bouton arrière** avant d'écrire les seuils de confort ou
l'unité, sinon l'écriture est perdue. Mise à jour OTA supportée.

**TRV-ZBT / TRVZB** — expose un `climate` complet : modes `off` / `auto` / `heat`, consigne de
4 à 35 °C, `local_temperature` et sa calibration (−12,7 à +12,7 °C), état `running_state`.
Côté avancé : `valve_opening_degree` / `valve_closing_degree` (0–100 %), programmation
hebdomadaire jusqu'à 6 transitions par jour, modes **boost** et **timer**, **détection de fenêtre
ouverte** (coupure si la température chute de plus de 1,5 °C en 4,5 min), **hors-gel** actif même
en mode `off`, et **verrouillage enfant**. Un algorithme **PID** interne est disponible, mais il
désactive le pilotage manuel du pourcentage d'ouverture. Le niveau de batterie peut mettre 24 h à
se stabiliser après l'appairage, et la vanne doit être **correctement montée pour se calibrer**.

**S60ZBTPF** — prise 16 A avec **mesure d'énergie**, protection de surcharge, successeur des S31 ZB
et S40ZBTPB Lite. Expose `switch`, `power`, `voltage`, `current`, `energy`, plus les compteurs
`energy_today` / `energy_yesterday` / `energy_month`, et les réglages `power_on_behavior`,
`overload_protection`, `outlet_control_protect`, `inching_control_set`. Nécessite un
**herdsman-converters ≥ 23.41.0** pour que les données d'énergie soient exposées.

Deux usages dans le projet, et c'est la pièce la plus polyvalente du parc :

- **Clôtures** : commande on/off de l'électrificateur, et surtout `power` comme **témoin de bon
  fonctionnement** — un électrificateur qui ne consomme plus est un électrificateur en panne, bien
  avant que le cheval ne s'en aperçoive.
- **Sous-comptage électrique** : `energy_*` donne la ventilation par usage (appoint ECS, annexes)
  sans poser de compteur supplémentaire.

> 🔒 **`power_on_behavior` est un réglage de sécurité ici.** Sur les prises de clôture, il doit être
> fixé à **`on`** : après une coupure secteur, la clôture doit se ré-alimenter seule, sans attendre
> que le NAS, le Wi-Fi et Zigbee2MQTT soient revenus. C'est l'application concrète du principe de
> dégradation sûre.

### Régulation par sonde déportée

Le point d'architecture clé du chauffage : la sonde interne du TRV mesure la température **au
radiateur**, pas celle de la pièce — elle est donc systématiquement biaisée. Le TRVZB sait
utiliser une sonde externe :

- `temperature_sensor_select` : `internal` | `external` | `external_2` | `external_3`
- `external_temperature_input` : la valeur injectée depuis l'extérieur

En mode externe, **la calibration locale devient inutile**. C'est le bon montage : la **SNZB-02D**
de la zone devient la référence de régulation du **TRV-ZBT** correspondant.

> ⚠️ **Zigbee2MQTT ne fait pas la synchronisation.** Il expose seulement le champ ; c'est à nous de
> republier périodiquement la valeur de la SNZB-02D vers `external_temperature_input` du TRV, via
> une automatisation. Il faut donc gérer explicitement le cas **sonde muette** (pile vide, hors de
> portée) : sans rafraîchissement, le TRV régule sur une valeur figée. Prévoir un repli automatique
> sur `internal` au-delà d'un délai sans mise à jour.

Nécessite **Zigbee2MQTT ≥ 2.1.2**. Si le champ n'apparaît pas, mettre à jour Z2M et le convertisseur
avant de chercher plus loin.

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

## ⚡ Comptage électrique

Deux sources complémentaires, de natures très différentes : le **Linky** au point de livraison
(la frontière avec le réseau) et une **centrale de mesure Socomec DIRIS Digiware** pour la
ventilation interne des départs.

### Socomec DIRIS Digiware — M-50 + U-30 + I-35

C'est le comptage de précision, en tableau :

| Module | Rôle |
|---|---|
| **M-50** | Passerelle de communication — point d'accès unique du système Digiware (alimentation + communication de toute la chaîne) |
| **U-30** | Module de mesure de **tension** |
| **I-35** | Module de mesure de **courant** (capteurs par départ) |

Les modules U/I dialoguent entre eux sur le **bus Digiware**, et le **M-50** expose l'ensemble vers
l'extérieur. Il supporte **Modbus RTU, Modbus TCP, BACnet IP et SNMP**, en RS485 ou Ethernet, et
fait passerelle **Modbus RTU → Modbus TCP**.

**Intégration — trois voies IP possibles, qui ne se valent pas côté Home Assistant :**

| Protocole | Support Home Assistant | Rôle retenu |
|---|---|---|
| **Modbus TCP** | Intégration native et mature, accès à toute la table de mesure | ✅ **Voie principale** — toutes les grandeurs électriques |
| **SNMP** | Plateforme `sensor` native, mais un capteur = un OID (verbeux pour de la mesure) | 🩺 **Supervision de la passerelle** : est-elle vivante, a-t-elle redémarré |
| **BACnet IP** | Pas d'intégration native, nécessite un pont tiers BACnet → MQTT | 🔁 **Repli** si le Modbus pose problème |

Le M-50 est donc raccordé en **Ethernet** et interrogé en **Modbus TCP** : pas de convertisseur
série, et le comptage passe par le réseau IP plutôt que par le Zigbee (sans charger le maillage
radio).

**SNMP garde un usage propre** et complémentaire : il répond à une question que le Modbus traite
mal — *la passerelle est-elle vivante ?* Un timeout Modbus ne dit pas si le problème vient du M-50,
du bus Digiware ou du réseau. Le suivi de `sysUpTime` en SNMP tranche, et explique après coup les
trous dans l'historique. Voir `config/homeassistant/packages/comptage_diris.yaml`.

Si le BACnet devient nécessaire, il faudra ajouter un conteneur passerelle **BACnet → MQTT** à la
pile ; Home Assistant consommera alors du MQTT, comme pour le Zigbee, sans intégration
supplémentaire.

> ⚙️ **Alimentation 24 VDC.** Le M-50 demande du 24 VDC ±10 % (≈ 2,5 VA) et alimente le bus
> Digiware. À prévoir au tableau — ce n'est pas une alimentation à négliger dans le plan.

À faire au moment de l'intégration : relever la **table des registres Modbus** des grandeurs
utiles (P, Q, S, U, I, cos φ, index d'énergie) par départ, et figer une correspondance
départ physique → usage (chauffage, ECS, clôtures, annexes, hydro, solaire). C'est ce mapping,
plus que la lecture Modbus elle-même, qui demande du soin.

### Linky — télé-information client (TIC)

Le Linky donne la vue « point de livraison » : soutirage, **injection** (essentiel avec l'hydro et
le solaire), index par tarif, puissance apparente, tension.

**Solution retenue : LiXee ZLinky_TIC** — un lecteur TIC **en Zigbee**, donc directement intégré au
réseau Zigbee2MQTT existant, sans passerelle supplémentaire ni câblage réseau jusqu'au Linky. Il
est alimenté par le Linky lui-même.

- Expose plus de **80 grandeurs** : puissance active soutirée **et injectée**, index multi-tarifs
  (`EASF01`…`EASF10`), tension, courant et puissance apparente par phase, informations tarifaires,
  registres d'état.
- Réglage **`linky_mode`** : `historique` ou `standard` — laisser sur **`auto`** pour la détection
  automatique. Un redémarrage peut être nécessaire pour la prise en compte.
- Le mode **standard** est nettement plus riche (et seul à remonter proprement l'injection) : si le
  compteur est encore en `historique`, demander le passage en `standard` à Enedis.

### Répartition des rôles

- **Linky / ZLinky** → bilan avec le réseau : soutirage, injection, autoconsommation, tarifs.
  C'est la référence de facturation et le déclencheur des charges opportunistes sur surplus.
- **Socomec Digiware** → ventilation interne par départ, avec une précision et une granularité que
  le Linky ne donne pas.
- Les **S60ZBTPF** → comptage d'appoint là où tirer un TI au tableau n'a pas de sens (prises
  isolées, annexes, électrificateurs).

---

## 🚦 État du projet

🌱 **Phase 0 — Cadrage.** Rien n'est encore déployé. Prochaines étapes :

- [ ] Inventorier le matériel : nombre de SNZB-02D et de TRV-ZBT par zone, S60ZBTPF, compteurs, onduleurs.
- [ ] Cartographier la portée Zigbee jusqu'aux clôtures et placer les S60ZBTPF-relais en conséquence.
- [ ] Relever la table des registres Modbus du DIRIS Digiware et mapper départ physique → usage.
- [ ] Vérifier le mode TIC du Linky (`standard` requis pour l'injection) et poser le ZLinky_TIC.
- [ ] Écrire la synchronisation SNZB-02D → `external_temperature_input` des TRV, avec repli sur sonde interne.
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
├── README.md                       # ce fichier
├── docker-compose.yml              # la pile de services déployée sur le NAS
├── .env.example                    # modèle des variables et secrets (copier vers .env)
├── config/                         # configuration versionnée
│   ├── mosquitto/
│   │   └── mosquitto.conf          # broker MQTT, accès anonyme interdit
│   ├── zigbee2mqtt/
│   │   ├── configuration.yaml      # coordinateur ZBBridge-P en TCP, canal, availability
│   │   ├── devices.yaml            # équipements + convention de nommage
│   │   └── groups.yaml
│   ├── homeassistant/
│   │   ├── configuration.yaml      # socle, découpage en packages
│   │   ├── influxdb.yaml           # historisation longue durée
│   │   ├── modbus.yaml             # DIRIS Digiware (squelette à compléter)
│   │   ├── secrets.yaml.example    # modèle des secrets (copier vers secrets.yaml)
│   │   └── packages/
│   │       ├── chauffage.yaml      # sonde déportée TRV + replis
│   │       ├── clotures.yaml       # surveillance et alertes clôtures
│   │       ├── comptage_diris.yaml # Modbus / SNMP / BACnet, supervision M-50
│   │       ├── energie.yaml        # bilan réseau, surplus, production
│   │       └── potager.yaml        # gel, arrosage (à écrire)
│   └── grafana/provisioning/       # source de données et tableaux de bord
├── data/                           # volumes d'exécution — NON versionné
└── docs/                           # schémas, inventaire matériel (à venir)
```

**Ce qui n'est pas versionné**, et pourquoi : `.env`, `secrets.yaml` et `config/mosquitto/passwd`
contiennent des secrets ; `data/` contient les volumes d'exécution ; la base d'appairage
Zigbee2MQTT et le `coordinator_backup.json` contiennent la **clé réseau Zigbee**. Ces derniers
doivent malgré tout être **sauvegardés** (Hyper Backup) : sans eux, tout le parc est à ré-appairer,
capteurs inaccessibles compris.

---

## ▶️ Premier démarrage

```bash
# 1. Secrets
cp .env.example .env                                    # puis renseigner
cp config/homeassistant/secrets.yaml.example \
   config/homeassistant/secrets.yaml                    # puis renseigner
openssl rand -hex 32                                    # pour chaque jeton

# 2. Utilisateur MQTT (le mot de passe doit correspondre à MQTT_PASSWORD)
docker run --rm -it -v "$PWD/config/mosquitto:/m" eclipse-mosquitto:2 \
  mosquitto_passwd -c -b /m/passwd palantir '<mot-de-passe>'

# 3. Démarrage
docker compose up -d
docker compose logs -f zigbee2mqtt
```

**Prérequis avant de lancer la pile :**

- Le **ZBBridge-P est flashé** (Tasmota + firmware CC2652) et basculé en mode `Sonoff ZBPro TCP`.
- Le ZBBridge-P **et** le DIRIS M-50 ont une **IP fixe** (réservation DHCP) : elles sont inscrites
  en dur dans `.env`.
- Le **canal Zigbee est figé** dans `configuration.yaml` **avant le premier appairage** — en
  changer ensuite impose de tout ré-appairer.

**Accès :** Home Assistant `:8123` · Zigbee2MQTT `:8081` · Grafana `:3000` · InfluxDB `:8086`.
Le port 8081 est retenu pour Zigbee2MQTT car 8080 est fréquemment déjà pris sur un NAS Synology.

---

## 📜 Licence

Projet personnel. Aucune licence définie pour l'instant.
