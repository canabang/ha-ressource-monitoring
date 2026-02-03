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

## Comment ça fonctionne ? (Explications techniques)

Le cœur de cette carte repose sur un template Jinja complexe qui automatise la découverte de vos services. Voici les points clés :

### 1. La Découverte Dynamique & Filtrage
Le code scanne automatiquement tous les capteurs de votre système pour isoler les Add-ons. Il utilise une logique de filtrage par motifs (Regex) :
```jinja
| selectattr('entity_id', 'search', 'cpu_percent|pourcentage_du_processeur')
| rejectattr('entity_id', 'search', 'node_|qemu_|pc_debian_')
```
- **Inclusion** : On capture les entités se terminant par `cpu_percent` (Supervisor anglais) ou `pourcentage_du_processeur` (Supervisor français).
- **Exclusion** : On élimine les entités parasites liées à la virtualisation (Proxmox, QEMU, nœuds réseau) pour ne garder que les véritables Add-ons.
- **Nettoyage** : Seules les entités ayant un état valide (`ne unknown/unavailable`) sont traitées.

### 2. Reconstruction des Contrôles
Pour chaque Add-on trouvé, le code "devine" ses entités de statut et de switch :
```jinja
{%- set base = s.entity_id | replace('sensor.', '') | replace('_cpu_percent', '') ... -%}
{%- set status_ent = "binary_sensor." ~ base ~ "_en_cours_d_execution" -%}
{%- set switch_ent = "switch." ~ base -%}
```
C'est ce qui permet d'afficher le badge Play/Stop et d'autoriser le Double-Tap sans que vous ayez à configurer chaque service manuellement.

### 3. Le Tri par Priorité
Au lieu de trier simplement par nom, le code calcule une "priorité" pour chaque Add-on :
```jinja
{%- set priority = cpu if cpu > ram else ram -%}
```
Cela permet de faire remonter en haut de liste l'élément qui consomme le plus, que ce soit en processeur ou en mémoire vive.

### 4. Les Barres de Progression Dynamiques
Les barres ne sont pas des images mais des `linear-gradient` générés en temps réel via CSS (`card-mod`) :
```css
background: linear-gradient(to right, {{ c_col }} {{ cpu }}%, transparent {{ cpu }}%) no-repeat bottom 8px center;
```
Le code calcule les couleurs selon des seuils définis, offrant un retour visuel immédiat.

### 5. Interactivité Sécurisée
Pour éviter d'éteindre un service critique par erreur, l'action est liée au **Double-Tap** :
```yaml
double_tap_action:
  action: call-service
  service: switch.toggle
  target:
    entity_id: item.sw_ent
```
L'entité `sw_ent` est reconstruite dynamiquement pour chaque Add-on à partir de son nom de capteur CPU.

### 4. Attribution Automatique des Icônes
Une liste de correspondances (`icons`) au début du code permet d'associer des icônes spécifiques aux noms des Add-ons (ex: *Frigate*, *ESPHome*, etc.). Si un Add-on n'est pas dans la liste, une icône par défaut est utilisée.

## Remarques
Les icônes sont attribuées dynamiquement par mots-clés dans le template de la carte. Si un Add-on n'est pas reconnu, il utilisera une icône par défaut ou vous pouvez l'ajouter dans la liste `icons` au début du code YAML.

## Licence
Ce projet est sous licence MIT. N'hésitez pas à l'adapter et à le partager !
