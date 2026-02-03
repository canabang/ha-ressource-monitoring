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

#### A. Les Chips (Santé Hôte : Sources Mixtes) 🏥
On utilise **exactement** les mêmes capteurs que ceux affichés en haut de ta carte :

1.  **CPU Global** : `sensor.system_monitor_utilisation_du_processeur` (Source: **System Monitor**)
2.  **RAM** : `sensor.glances_ha_utilisation_de_la_memoire` (Source: **Glances**)
3.  **Température** : `sensor.system_monitor_temperature_du_processeur` (Source: **System Monitor**)
4.  **Disque (Data)** : `sensor.glances_ha_utilisation_disque_data` (Source: **Glances**)
5.  **GPU** : `sensor.frigate_intel_vaapi_gpu_load` (Source: **Frigate**)

#### B. La Liste des Add-ons (Le Scanner) 🕵️‍♂️
*Pourquoi on filtre par le CPU ?*
C'est pour isoler les "Machines" parmi les milliers d'entités de ton HA.
Les services comme *Supervisor* ou *Glances* créent un capteur `_cpu_percent` pour chaque conteneur actif. C'est notre "balise" pour identifier ce qui tourne.

*   **Trigger 1 (Crash)** : On détecte l'Add-on via son CPU, puis on vérifie son binary_sensor `_running` associé.
*   **Trigger 2 (Surcharge)** : On vérifie si ce même capteur CPU dépasse 80%.

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

  # --------------------------------------------------------------------------
  # B. BRAIN : K-2SO GÉNÈRE LE MESSAGE 🧠
  # --------------------------------------------------------------------------
  - action: script.k_2so_generateur_de_message
    data:
      mission: >
        {% set info = enquete.resultat %}
        {% if info.etat == 'OFF' %}
          Panne Critique : {{ info.nom }} ne répond plus.
        {% elif 'SURCHARGE' in info.etat %}
          Surcharge Système : {{ info.nom }} est en souffrance.
        {% elif 'DISQUE' in info.etat %}
          Espace Disque Critique : {{ info.nom }} est saturé.
        {% else %}
          Alerte Ressources : {{ info.nom }} est en zone rouge.
        {% endif %}
      
      details: >
        - CPU : {{ enquete.resultat.cpu }}%
        - RAM : {{ enquete.resultat.ram }}%
        - État : {{ enquete.resultat.etat }}
      
      consigne: "Sois technique, sarcastique et bref."
    response_variable: generated_message

  # --------------------------------------------------------------------------
  # C. PROPOSITIONS DISCORD (À CHOISIR) 📢
  # --------------------------------------------------------------------------
  
  # OPTION 1 : La "Classique" (Propre et efficace)
  # - Affiche l'icône de l'Add-on (si dispo) ou HA.
  # - Nom de l'alerte = "Monitor System"
  # - Texte = Message de K-2SO.
    
  - action: script.notification_discord
    data:
      nom: "Monitor System"
      description: "{{ generated_message.generated_message.data }}"
      image_url: >
        {% if enquete.resultat.etat == 'OFF' %}
          https://brands.home-assistant.io/hassio/icon.png
        {% else %}
          https://brands.home-assistant.io/homeassistant/icon.png
        {% endif %}

  # OPTION 2 : La "Visuelle" (Code couleur dans le titre)
  # - Change le nom de l'envoyeur pour attirer l'oeil "🚨 ALERTE CRITIQUE" vs "🐌 SURCHARGE".
  
  - action: script.notification_discord
    data:
      nom: >
        {% if enquete.resultat.etat == 'OFF' %}
          🚨 ALERTE CRASH
        {% else %}
          ⚠️ ALERTE RESSOURCES
        {% endif %}
      description: "{{ generated_message.generated_message.data }}"
      image_url: "https://brands.home-assistant.io/glances/icon.png"

  # OPTION 3 : La "Mentions" (Ping tout le monde si critique)
  # - Ajoute @everyone uniquement si c'est un CRASH.
  
  - action: script.notification_discord
    data:
      nom: "Monitor System"
      description: >
        {% if enquete.resultat.etat == 'OFF' %}
          @everyone 
        {% endif %}
        {{ generated_message.generated_message.data }}
      image_url: ...
