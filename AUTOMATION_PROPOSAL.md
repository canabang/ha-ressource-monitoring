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

#### A. Les Chips (La Santé de l'Hôte) 🏥
Ce sont tes indicateurs globaux (CPU, RAM, Température).
*   **Trigger** : Si `sensor.system_monitor_*` ou `sensor.glances_*` dépasse un seuil critique.
    *   Température > 75°C
    *   RAM > 90%
    *   CPU > 90%

#### B. La Liste des Add-ons (Le Scanner) 🕵️‍♂️
On utilise un **Template Trigger** pour détecter tout Add-on qui flanche.
*   **Trigger** : Si n'importe quel `binary_sensor` finissant par `_running` passe à `off`.
*   **Trigger** : Si n'importe quel `sensor` finissant par `_cpu_percent` dépasse 80%.

**Code YAML Final pour l'Automatisation :**
```yaml
trigger:
  # 1. HOST HEALTH (Les Chips)
  - platform: numeric_state
    entity_id: sensor.system_monitor_temperature_du_processeur
    above: 75
    id: "host_heat"
  - platform: numeric_state
    entity_id: sensor.glances_ha_utilisation_de_la_memoire
    above: 90
    id: "host_ram"
  
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
