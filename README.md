# ha-ressource-monitoring

Carte de monitoring avancée pour Home Assistant permettant de suivre en temps réel les ressources (CPU/RAM) de l'hôte et des Add-ons (**Home Assistant Supervisor**).

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

## Installation

### 1. La Carte (Frontend)
Créez une carte **Manuel** et collez le contenu de [monitoring_card.yaml](cards/monitoring_card.yaml).

### 2. Le Rafraîchissement Turbo (Backend)
Pour une réactivité à 5 secondes (recommandé si vos capteurs sont exclus du recorder), importez l'automation [turbo_refresh.yaml](automations/turbo_refresh.yaml).

## Analyse détaillée du code

Le code de la carte (YAML) utilise le moteur de template Jinja2 de Home Assistant pour automatiser la gestion des entités. Voici le découpage par étapes :

### 1. Collecte et Filtrage initial
On commence par isoler tous les capteurs CPU des Add-ons en excluant les entités parasites.
```jinja
{% set sensors = states.sensor 
  | selectattr('entity_id', 'search', 'cpu_percent|pourcentage_du_processeur')
  | rejectattr('entity_id', 'search', 'node_|qemu_|pc_debian_')
  | selectattr('state', 'ne', 'unavailable')
  | selectattr('state', 'ne', 'unknown')
  | list %}
```
- `selectattr(...)` : On ne garde que les capteurs liés aux processeurs (français et anglais).
- `rejectattr(...)` : On exclue explicitement les entités de virtualisation (Proxmox/QEMU) qui pollueraient la liste.
- `unavailable/unknown` : On ignore les services qui ne renvoient pas de données (ex: Add-on arrêté sans capteur actif).

### 2. Traitement et Construction des entités
Pour chaque capteur trouvé, le code "devine" les chemins des autres entités liées (RAM, Switch et Statut).
```jinja
{% for s in sensors %}
  {%- set cpu = s.state | float(0) -%}
  {%- set name = state_attr(s.entity_id, 'friendly_name') 
     | replace(' CPU Percent', '') | replace(' Pourcentage du processeur', '') | trim -%}
  
  {# Calcul de la base pour retrouver switch et statut #}
  {%- set base = s.entity_id | replace('sensor.', '') | replace('_cpu_percent', '') | replace('_pourcentage_du_processeur', '') -%}
  {%- set status_ent = "binary_sensor." ~ base ~ "_en_cours_d_execution" -%}
  {%- set switch_ent = "switch." ~ base -%}

  {# Détection automatique de l'entité RAM correspondante #}
  {%- if '_cpu_percent' in s.entity_id -%}
    {%- set ram_ent = s.entity_id | replace('_cpu_percent', '_memory_percent') -%}
  {%- else -%}
    {%- set r_t = s.entity_id | replace('_pourcentage_du_processeur', '_pourcentage_de_mémoire') -%}
    {%- set ram_ent = r_t if states(r_t) != 'unknown' else s.entity_id | replace('_pourcentage_du_processeur', '_pourcentage_de_memoire') -%}
  {%- endif -%}
{% endfor %}
```
Le code utilise des filtres `replace` en cascade pour nettoyer le nom affiché et reconstruire les `entity_id` du switch Supervisor et du `binary_sensor` de statut.

### 3. Calcul de Priorité et Tri
Pour savoir quel Add-on doit figurer dans le **Top 5**, on calcule la valeur maximale entre son CPU et sa RAM.
```jinja
{%- set ram = states(ram_ent) | float(0) -%}
{%- set priority = cpu if cpu > ram else ram -%}
{%- set item = {"name": name, "priority": priority, "cpu_ent": cpu_ent, "ram_ent": ram_ent, "sw_ent": switch_ent, "st_ent": status_ent} -%}
{%- set add_ons.items = add_ons.items + [item] -%}
```
Puis on trie la liste finale par cette `priority` :
```jinja
{% set sorted_items = add_ons.items | sort(attribute='priority', reverse=true) %}
```

### 4. Répartition Top 5 et Reste
On divise la liste en deux groupes pour garder un dashboard propre.
```jinja
{% set final_cards = namespace(top=[], rest=[]) %}
{% for item in sorted_items %}
  {# ... génération de la carte Mushroom ... #}
  {% if loop.index <= 5 %}
    {% set final_cards.top = final_cards.top + [card] %}
  {% else %}
    {% set final_cards.rest = final_cards.rest + [card] %}
  {% endif %}
{% endfor %}
```

### 5. Assemblage final
On fusionne le Top 5 permanent avec un bloc `custom:fold-entity-row` (en bas) qui contient tout le reste de la liste, camouflé derrière un menu déroulant.
L'entité `sw_ent` est reconstruite dynamiquement pour chaque Add-on à partir de son nom de capteur CPU.

### 4. Attribution Automatique des Icônes
Une liste de correspondances (`icons`) au début du code permet d'associer des icônes spécifiques aux noms des Add-ons (ex: *Frigate*, *ESPHome*, etc.). Si un Add-on n'est pas dans la liste, une icône par défaut est utilisée.

## Remarques
Les icônes sont attribuées dynamiquement par mots-clés dans le template de la carte. Si un Add-on n'est pas reconnu, il utilisera une icône par défaut ou vous pouvez l'ajouter dans la liste `icons` au début du code YAML.

## Licence
Ce projet est sous licence MIT. N'hésitez pas à l'adapter et à le partager !
