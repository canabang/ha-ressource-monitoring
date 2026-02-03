# ha-ressource-monitoring

Carte de monitoring avancée pour Home Assistant permettant de suivre en temps réel les ressources (CPU/RAM) de l'hôte et des Add-ons (**Home Assistant Supervisor**).

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

### Intégrations Requises
- **Home Assistant Supervisor** : Fournit les statistiques CPU/RAM des Add-ons et les entités de contrôle (Start/Stop).
- **Glances** : Crucial pour les métriques de l'Hôte dans les Chips (Utilisation RAM en % et GiB, Utilisation Disque).
- **System Monitor** : Fournit les métriques Host de base (Usage CPU moyen, Température processeur).

## 🚨 Système d'Alertes Intelligentes

Ce projet ne se contente pas d'afficher des jauges, il surveille activement votre système via une **Automatisation Unique** (`alertes_ressources.yaml`).

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

### 3. (Optionnel) Le Rafraîchissement Turbo
Pour une réactivité à 5 secondes (recommandé pour les jauges), créez une seconde automatisation avec le contenu de [`automations/turbo_refresh.yaml`](automations/turbo_refresh.yaml).

---

## 🧠 Analyse technique (Le "Cerveau" Jinja)

Ce projet repose sur une détection dynamique unique, partagée par la Carte et l'Automatisation.

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
La carte [`monitoring_card.yaml`](cards/monitoring_card.yaml) utilise ce filtre pour :
1.  **Reconstruire les entités** : Deviner le `binary_sensor` (Statut) à partir du sensor CPU.
2.  **Trier par urgence** : `CPU > RAM` ? On affiche le plus critique.
3.  **Sticky Bottom** : Les Add-ons éteints (`OFF`) sont forcés en bas de liste.

### 3. Logique de l'Automatisation (Backend)
L'automatisation [`alertes_ressources.yaml`](automations/alertes_ressources.yaml) reprend exactement le même principe pour ses triggers :
*   **Trigger "Crash"** : Si le sensor CPU existe MAIS que le statut est OFF -> Alerte.
*   **Trigger "Surcharge"** : Si le sensor CPU dépasse 80% pendant 5 min -> Alerte.
*   **Mapping Intelligent** : Un dictionnaire interne corrige les noms exotiques (`frigate_full_access` -> `frigate`) pour afficher la bonne icône dans la notification.
