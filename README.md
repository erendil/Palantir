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
  Vannes thermostatiques   ──►   / Tasmota        ──►   Broker MQTT      ──►   Alertes (mobile)
  Sondes ballons (1-Wire)  ──►   Passerelle 1-Wire──►   Moteur de règles ──►   API
  Compteurs élec / onduleur──►   Modbus / P1      ──►   Base temporelle
```

**Briques pressenties** (à trancher) :
- **Home Assistant** comme cœur d'intégration et d'automatisation.
- **Mosquitto** (MQTT) comme bus de messages.
- **Zigbee2MQTT** ou **Tasmota** pour les périphériques SONOFF (selon le modèle : Zigbee ou Wi-Fi).
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
- **Passerelles radio** : la clé Zigbee/RF doit être branchée en USB sur le NAS et exposée au
  conteneur (`devices:`), ou déportée via une passerelle réseau (Zigbee-over-IP / ser2net) si le
  NAS n'est pas idéalement placé dans la maison.
- **Réseau** : `network_mode: host` ou macvlan pour la découverte mDNS/UPnP des équipements
  SONOFF et des onduleurs.
- **Volumes persistants** : configuration et base temporelle sur un volume du NAS, inclus dans
  Hyper Backup.
- **Disponibilité** : le NAS héberge des fonctions touchant à la sécurité (clôtures), d'où la
  règle de dégradation sûre ci-dessous — redémarrage automatique des conteneurs et repli local
  des équipements.

---

## 🚦 État du projet

🌱 **Phase 0 — Cadrage.** Rien n'est encore déployé. Prochaines étapes :

- [ ] Inventorier précisément le matériel existant (modèles SONOFF, type de sondes, compteurs, onduleurs).
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
