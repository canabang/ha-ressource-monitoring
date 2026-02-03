# Proposition : Système d'Alertes Intelligentes (K-2SO)

L'objectif est de ne plus subir les pannes mais d'être prévenu proactivement par tes assistants (Discord & Awtrix), avec la touche "personnalité" de K-2SO.

## 1. La Philosophie : "Silence is Golden"
On ne veut pas être spammé. Une alerte ne doit partir que si c'est **important** et **confirmé**.
- **Pas d'alerte** pour un pic CPU de 3 secondes.
- **Alerte** si un Add-on critique (Frigate, Z2M) est DOWN ou consomme 100% CPU depuis 5 minutes.

## 2. Le Cerveau : K-2SO
On utilisera ton script `k2so` existant pour générer le message.
- **Input** : `consigne` (ex: "Alerte, Frigate consomme trop de RAM"), `titre` (ex: "Surcharge Système").
- **Output** : Un message sarcastique ou informatif sur Discord + une notif courte sur Awtrix.

## 3. Les Scénarios Proposés

### 3. Les Options de Déclencheurs (Trigger)

Il y a deux écoles pour surveiller tes Add-ons :

### 3. Les Déclencheurs (Stratégie Dynamique)

On surveille **Tout** ce qui est sur la carte (Chips + Liste) sans rien nommer en dur.

#### A. Les Chips (La Santé Complète de l'Hôte) 🏥
On surveille les 5 points vitaux présents sur tes Chips :
1.  **CPU Global** : `sensor.system_monitor_utilisation_du_processeur` > 90%
2.  **RAM** : `sensor.glances_ha_utilisation_de_la_memoire` > 90%
3.  **Température** : `sensor.system_monitor_temperature_du_processeur` > 75°C
4.  **Disque (Data)** : `sensor.glances_ha_utilisation_disque_data` > 90% (Nouveau)
5.  **GPU (Frigate)** : `sensor.frigate_intel_vaapi_gpu_load` > 90% (Nouveau)

#### B. La Liste des Add-ons (Le Scanner Intelligent) 🕵️‍♂️
*Pourquoi passer par le CPU ?*
C'est une astuce technique : Home Assistant n'a pas de "groupe" officiel listant tous les Add-ons.
Par contre, Glances crée systématiquement un capteur `_cpu_percent` pour chaque conteneur actif.
-> C'est donc le moyen le plus fiable de **découvrir** dynamiquement ce qui tourne, pour ensuite aller vérifier son statut `_running`.

*   **Trigger 1 (Crash)** : On liste les services via leur CPU, et on vérifie si leur bouton `_running` est OFF.
*   **Trigger 2 (Surcharge)** : Si n'importe quel `sensor` finissant par `_cpu_percent` dépasse 80%.

**Code YAML Final pour l'Automatisation :**
```yaml
trigger:
  # 1. HOST HEALTH (Les Chips - Complet)
  - platform: numeric_state
    entity_id: sensor.system_monitor_temperature_du_processeur
    above: 75
    id: "host_heat"
  - platform: numeric_state
    entity_id: sensor.glances_ha_utilisation_de_la_memoire
    above: 90
    id: "host_ram"
  - platform: numeric_state
    entity_id: sensor.glances_ha_utilisation_disque_data
    above: 90
    id: "host_disk"
  - platform: numeric_state
    entity_id: sensor.frigate_intel_vaapi_gpu_load
    above: 90
    id: "host_gpu"
  
  # 2. ADD-ONS HEALTH (Scanner Dynamique)
  - platform: template
    value_template: >
      {{ states.binary_sensor 
         | selectattr('entity_id', 'search', '_running$') 
         | selectattr('state', 'eq', 'off') 
         | list | count > 0 }}
    id: "addon_crash"

  # 3. ADD-ONS HOGS (Scanner Surcharge)
  - platform: template
    value_template: >
      {{ states.sensor 
         | selectattr('entity_id', 'search', '_cpu_percent$') 
         | map(attribute='state') 
         | map('float', 0) 
         | select('gt', 80) 
         | list | count > 0 }}
    for: "00:05:00"
    id: "cpu_hog"
```

### 4. Les Options d'Actions

Une fois l'alerte levée, qu'est-ce qu'on fait ?

*   **1. Notification Riche (Discord)** :
    *   Texte généré par K-2SO ("Panne détectée...").
    *   Image dynamique de l'intégration.
    *   Bouton "Redémarrer" directement dans Discord ?
*   **2. Affichage (Awtrix / Dashboard)** :
    *   Faire clignoter l'icône de l'Add-on en rouge sur l'Awtrix.
    *   Envoyer une notif persistante sur le Dashboard HA.
*   **3. Self-Healing (Auto-réparation)** :
    *   Tenter de redémarrer l'Add-on automatiquement via `hassio.addon_restart` ? (Risqué si c'est une panne de config).

Quelle profondeur d'automatisation veux-tu ? (Juste prévenir ? Ou tenter de réparer ?)

## 4. Structure Technique (Le Chaînage)
Ton script `k_2so_generateur_de_message` renvoie la variable `generated_message`.

**Exemple de code YAML projeté :**
```yaml
alias: "System - Monitor & Alert"
