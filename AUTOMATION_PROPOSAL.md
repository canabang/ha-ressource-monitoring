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

#### Option A : La "Liste Blanche" (Ce que j'avais fait)
On liste explicitement chaque add-on.
*   ✅ **Avantage** : On choisit précisément ce qu'on surveille (pas d'alerte pour un truc de test).
*   ❌ **Inconvénient** : Si tu installes un nouvel Add-on, il faut modifier l'automatisation.

#### Option B : Le "Scanner Dynamique" (Reco) 🧠
On utilise un `Template Trigger` qui surveille **tous** les capteurs finissant par `_running` ou `_cpu_percent`.
*   ✅ **Avantage** : 100% Automatique. Tu installes un Add-on, il est surveillé direct.
*   ❌ **Inconvénient** : Peut être bavard si tu as des services instables que tu ne veux PAS surveiller.

**Exemple de code pour l'Option B (Scanner) :**
```yaml
trigger:
  - platform: template
    # Déclenche si N'IMPORTE QUEL add-on passe à OFF
    value_template: >
      {{ states.binary_sensor 
         | selectattr('entity_id', 'search', '_running$') 
         | selectattr('state', 'eq', 'off') 
         | list | count > 0 }}
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
