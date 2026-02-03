# ha-ressource-monitoring

**Solution complète de Monitoring & Alertes pour Home Assistant.**

Combine une **Carte Dashboard interactive** pour le suivi temps réel et un **Système d'Alertes** proactif (via Blueprint ou IA) pour garantir la santé de votre Hôte et de vos Add-ons.

![Dashboard Preview](assets/dashboard_preview.gif)

## Filtres & Tri Dynamique
- **Host Monitoring** : Affichage des constantes vitales (CPU, RAM, Disque, Température) via des Chips.
- **Top Add-ons** : Classement automatique des services par consommation.
- **Interactivité** : Double-clic pour démarrer/arrêter un Add-on (avec sécurité contre les erreurs).
- **Design Pill** : Bords arrondis 33px et transparence totale avec contours fins.

## Pré-requis

### Cartes Lovelace (via [HACS](https://hacs.xyz/))
- 🍄 [Mushroom Cards](https://github.com/piitaya/lovelace-mushroom)
- 🗂️ [Auto-entities](https://github.com/thomasloven/lovelace-auto-entities)
- 🎨 [Card-mod](https://github.com/thomasloven/lovelace-card-mod)
- 📂 [Fold-entity-row](https://github.com/thomasloven/lovelace-fold-entity-row)

### Packs d'Icônes (via [HACS](https://hacs.xyz/))
Certaines icônes premium utilisées dans la carte nécessitent l'installation de ces packs :
- 📦 [Plug-and-Hyve Icons (phu)](https://github.com/Mariusthvdb/phu-icons)
- 💎 [Material Symbols (m3r, m3rf)](https://github.com/beecho01/material-symbols)

### Intégrations (Sources de Données)
Ce système s'appuie sur plusieurs intégrations pour fournir une vision complète (Hôte + Add-ons) :

*   **Home Assistant Supervisor** :
    *   *Usage* : Monitoring dynamique des Add-ons.
    *   *Données* : CPU %, RAM %, Statut (Running/Stopped), et le switch pour démarrer/arrêter les add-ons.
*   **Glances** (Add-on + Intégration) :
    *   *Usage* : Monitoring global de l'Hôte (Chips du Dashboard + Alertes).
    *   *Données* : RAM Réelle, Espace Disque utilisé, Températures fines.
*   **System Monitor** (Intégration Core) :
    *   *Usage* : Complément pour l'Hôte.
    *   *Données* : Usage Processeur (%) et Température.

> 💡 **Note** : Les "Chips" en haut de la carte et les alertes "Santé Hôte" dépendent directement de **Glances** et **System Monitor**. Assurez-vous qu'ils sont installés.

## 🚨 Système d'Alertes Intelligentes

Ce projet ne se contente pas d'afficher des jauges, il surveille activement votre système via une **Automatisation Unique** (`alertes_ressources.yaml`).

> Un **[Blueprint Universel](blueprints/alertes_ressources.yaml)** est disponible pour une installation facile sans mes scripts IA et notifications discord (voir plus bas).



### Fonctionnalités
*   **Détection proactive de Crash** : Si un Add-on s'arrête (statut `OFF` alors que le CPU existe), vous êtes notifié immédiatement (délai 1 min).
*   **Surveillance Surcharge Add-on** : Alerte si un Add-on dépasse 80% de CPU pendant 5 minutes.
*   **Santé de l'Hôte** :
    *   Surchauffe Processeur (> 75°C)
    *   Surcharge RAM (> 90%)
    *   Disque Plein (> 90%)
    *   Surcharge GPU (> 90%)

### 🤖 L'Intelligence Artificielle (K-2SO)
Les alertes ne sont pas de simples logs. Elles sont traitées par le script **K-2SO** (IA locale ou cloud) qui génère un message contextuel :
*   **Technique** : Il reçoit les stats précises (CPU, RAM, Disque).
*   **Sarcastique** : Il commente la situation avec son "cynisme bienveillant".
*   **Pertinent** : Il filtre les infos inutiles (pas de disque = pas d'info disque).

### 📱 Notifications (Discord)
Le rendu est optimisé pour Discord avec :
*   **Titre Visuel** : `🚨 CRASH : Frigate` ou `⚠️ SURCHARGE`.
*   **Icône Dynamique** : Affiche automatiquement le logo de l'intégration concernée (via `brands.home-assistant.io`) ou une icône générique si inconnu.
*   **Message IA** : Le texte généré par K-2SO.

---

## 🛠️ Installation

### 1. La Carte (Frontend)
1.  Installez les cartes requises via HACS (Mushroom, Auto-entities, etc.).
2.  Créez une nouvelle carte **Manuel** dans votre tableau de bord.
3.  Copiez-collez l'intégralité du contenu du fichier [`cards/monitoring_card.yaml`](cards/monitoring_card.yaml).

### 2. L'Automatisation (Backend)
1.  Allez dans **Paramètres** > **Automatisations et Scènes** > **Créer une automatisation**.
2.  Passez en mode **YAML** (3 petits points en haut à droite).
3.  Copiez-collez le contenu du fichier [`automations/alertes_ressources.yaml`](automations/alertes_ressources.yaml).
4.  Sauvegardez.

### 2b. Alternative : Le Blueprint (Recommandé pour les autres utilisateurs)
Si vous ne souhaitez pas utiliser l'IA ou modifier du code yaml, utilisez le Blueprint :
1.  Copiez [`blueprints/alertes_ressources.yaml`](blueprints/alertes_ressources.yaml) dans `/config/blueprints/automation/perso/`.
2.  Créez une automatisation via **Paramètres** > **Utiliser un plan** > **"Système • Surveillance & Alertes 🛡️"**.
3.  Configurez simplement vos seuils et votre notification !

### 3. (Optionnel) Le Rafraîchissement Turbo
Pour une réactivité à 5 secondes (recommandé pour les jauges), créez une seconde automatisation avec le contenu de [`automations/turbo_refresh.yaml`](automations/turbo_refresh.yaml).

---

## 🧠 Analyse technique (Le "Cerveau" Jinja)

Ce projet repose sur une détection dynamique , partagée par la Carte et l'Automatisation.

### 1. Le Filtrage Intelligent (La base)
Que ce soit pour afficher les jauges ou détecter un crash, nous utilisons le même filtre Jinja2 pour trouver les Add-ons :

```jinja
{% set sensors = states.sensor 
  | selectattr('entity_id', 'search', 'cpu_percent|pourcentage_du_processeur')
  | rejectattr('entity_id', 'search', 'node_|qemu_|pc_debian_')
  | list %}
```
*   **Agnostique** : Fonctionne que votre système soit en Anglais (`cpu_percent`) ou Français (`pourcentage...`).
*   **Propre** : Filtre les processus internes (QEMU, Node, etc.).
*   **Robuste** : Si un Add-on change de nom, il suffit de "Recréer les identifiants" dans Home Assistant pour qu'il soit détecté.

### 2. Logique de la Carte (Frontend)
La carte [`monitoring_card.yaml`](cards/monitoring_card.yaml) pousse la logique plus loin pour l'affichage :

#### A. Reconstruction intelligente des entités
Pour chaque capteur trouvé, le code "devine" les chemins des autres entités (RAM, Binary Sensor) en essayant les suffixes courants (`_running`, `_en_cours_d_execution`).
```jinja
{%- set st_run = "binary_sensor." ~ base ~ "_running" -%}
{%- set st_exec = "binary_sensor." ~ base ~ "_en_cours_d_execution" -%}
{%- set status_ent = st_run if states(st_run) != 'unknown' else st_exec -%}
```
C'est ce qui garantit l'affichage du badge Play/Stop pour tous les services, incluant VS Code ou File Editor.

#### B. Calcul de Priorité (Sticky-Bottom)
Pour savoir quel Add-on doit figurer dans le **Top 5**, on calcule la valeur maximale entre son CPU et sa RAM.
*Astuce* : Si l'Add-on est détecté comme éteint (`off`), on lui donne une priorité de `-1` pour le forcer à descendre tout en bas de la liste.
```jinja
{%- set priority = -1 if states(status_ent) == 'off' else (cpu if cpu > ram else ram) -%}
```

#### C. Répartition "Top 5" vs "Reste"
On divise la liste triée en deux groupes pour garder un dashboard épuré.
```jinja
{% for item in sorted_items %}
  {# ... génération de la carte Mushroom ... #}
  {% if loop.index <= 5 %}
    {# Ajout à la liste visible #}
  {% else %}
    {# Ajout à la liste déroulante (Fold row) #}
  {% endif %}
{% endfor %}
```

### 3. Logique de l'Automatisation (Backend)
L'automatisation [`alertes_ressources.yaml`](automations/alertes_ressources.yaml) reprend exactement le même filtre de base pour ses triggers :
*   **Trigger "Crash"** : Si le sensor CPU existe MAIS que le statut est OFF -> Alerte.
*   **Trigger "Surcharge"** : Si le sensor CPU dépasse 80% pendant 5 min -> Alerte.
*   **Mapping Intelligent** : Un dictionnaire interne corrige les noms exotiques (`frigate_full_access` -> `frigate`) pour afficher la bonne icône dans la notification.

### 4. La Gestion des Exceptions (Le "Fixe")
Bien que le système soit à 90% dynamique, certaines exceptions nécessitent un mapping manuel pour un affichage parfait :

*   **Dans la Carte** : Une liste `icons` définit les icônes spécifiques (ex: *Frigate* -> `phu:frigate`) car Home Assistant ne devine pas toujours le bon logo MDI/PHU.
*   **Dans l'Automatisation** : Une liste `corrections_icones` corrige les "slugs" techniques (ex: `frigate_full_access` -> `frigate`) pour que l'intégration *Home Assistant Brands* trouve le bon logo officiel à afficher dans Discord.

### 5. Monitoring Hôte (Partie Statique)
Contrairement aux Add-ons, les "Chips" (en haut) pointent vers des entités fixes définies en dur dans le template de la carte. Si vos capteurs ont des noms différents, il faudra modifier ces lignes :
*   **CPU** : `sensor.system_monitor_utilisation_du_processeur`
*   **RAM** : `sensor.glances_ha_utilisation_de_la_memoire` (%) et `sensor.glances_ha_memoire_utilisee` (GiB)
*   **Disque** : `sensor.glances_ha_utilisation_disque_data`
*   **Température** : `sensor.system_monitor_temperature_du_processeur`
*   **GPU** : `sensor.frigate_intel_vaapi_gpu_load` (Spécifique à mon N100/N150)

## ❓ Dépannage (FAQ)

### La carte reste vide ou affiche des erreurs
*   Vérifiez que vous avez bien installé **Auto-entities** et **Mushroom** via HACS.
*   Assurez-vous que vos sensors CPU contiennent bien les termes `cpu_percent` ou `pourcentage_du_processeur` (Regardez dans Outils de développement > États).

### J'ai des doublons d'alertes
*   Si vous avez activé le Blueprint ET gardé votre ancienne automatisation, c'est normal. Désactivez l'une des deux.

### Le Blueprint ne me propose pas mon sensor GPU
*   Le sélecteur attend un `sensor`. Si votre intégration fournit un attribut, vous devez d'abord créer un Template Sensor intermédiaire.

### Mes jauges ne bougent pas vite
*   Par défaut, Home Assistant rafraîchit les infos système toutes les 15-60 secondes.
*   Pour avoir du temps réel (5s), installez l'automatisation **Turbo Refresh** (voir section Installation). *Attention à la taille de votre base de données !*
