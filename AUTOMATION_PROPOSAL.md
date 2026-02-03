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

### 3. Les Scénarios & Déclencheurs (Détail)

#### A. "The Crash" (Services Critiques) 🚨
Si un service essentiel s'arrête (état `off` ou `unavailable` pendant 1 min).
*   `binary_sensor.frigate_running` (Frigate)
*   `binary_sensor.zigbee2mqtt_running` (Zigbee2MQTT - *à confirmer si présent*)
*   `binary_sensor.mosquitto_broker_running` (MQTT)
*   `binary_sensor.matter_server_running` (Matter)

#### B. "The Hog" (Surcharge CPU) 🐌
Si un service consomme anormalement pendant plus de 5 minutes.
*   `sensor.frigate_cpu_percent` > 80%
*   `sensor.glances_cpu_percent` > 50%
*   `sensor.home_assistant_core_cpu_percent` > 40%

#### C. "The Heater" (Santé du N100) 🔥
Si le matériel est en souffrance.
*   `sensor.system_monitor_temperature_du_processeur` > 75°C
*   `sensor.system_monitor_utilisation_du_processeur` > 90% (Global)
*   `sensor.glances_ha_utilisation_de_la_memoire` > 90% (RAM saturée)

## 4. Structure Technique (Le Chaînage)
Ton script `k_2so_generateur_de_message` renvoie la variable `generated_message`.

**Exemple de code YAML projeté :**
```yaml
alias: "System - Monitor & Alert"
trigger:
  # --- CRASH DETECTORS ---
  - platform: state
    entity_id: binary_sensor.frigate_running
    to: "off"
    for: "00:01:00"
    id: "crash_frigate"
  - platform: state
    entity_id: binary_sensor.matter_server_running
    to: "off"
    for: "00:01:00"
    id: "crash_matter"
    
  # --- RESOURCE HOGS ---
  - platform: numeric_state
    entity_id: sensor.frigate_cpu_percent
    above: 80
    for: "00:05:00"
    id: "cpu_frigate"

action:
  # 1. BRAIN : K-2SO génère le message
  - action: script.k_2so_generateur_de_message
    data:
      mission: >
        {% if 'crash' in trigger.id %}
          Panne Critique : {{ trigger.to_state.name }}
        {% else %}
          Surcharge CPU : {{ trigger.to_state.name }}
        {% endif %}
      details: "État actuel : {{ trigger.to_state.state }}"
      consigne: "Sois bref, technique et sarcastique."
    response_variable: k2so_output

  # 2. VOICE : Discord (avec le message généré)
  - action: script.notification_discord
    data:
      nom: "Alerte Système"
      description: "{{ k2so_output.generated_message.data }}"
      image_url: "https://brands.home-assistant.io/homeassistant/icon.png"

  # 3. DISPLAY : Awtrix (Message court statique)
  - action: script.awtrix_dynamique_customapp_and_notify
    data:
      message: "{{ trigger.id | upper | replace('_', ' ') }}"
      icone: "alert"
      color: "#FF0000"
      duree: 8
```
