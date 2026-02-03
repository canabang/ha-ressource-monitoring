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

### A. "The Crash" (Priorité Haute) ✨
Surveillance des Add-ons critiques. Si l'un d'eux passe à `off` ou `unavailable`.
- **Déclencheur** : `binary_sensor.frigate_running` passe à `off` pendant 1 min.
- **Action** : Relance automatique (optionnel) + Notif Discord "Frigate est tombé au combat."
- **Awtrix** : Icône rouge clignotante.

### B. "The Hog" (Alerte Ressources) 🐷
Si un Add-on monopolise le CPU ou la RAM.
- **Déclencheur** : `sensor.frigate_cpu_percent` > 50% pendant 10 min.
- **Action** : Notif Discord "Frigate abuse des ressources (CPU > 50%)."

### C. "The Heater" (Santé Hôte) 🔥
Si le N100 chauffe trop ou sature sa RAM globale.
- **Déclencheur** : Température > 75°C OU RAM > 90%.
- **Action** : Notif Discord "Je brûle ! (Temp > 75°C)".

## 4. Structure Technique (Le Chaînage)
Ton script `k_2so_generateur_de_message` ne fait que **générer** le texte. Il faut donc orchestrer l'appel en 3 temps dans l'automatisation :

1.  **Génération** : Appel de `script.k_2so_generateur_de_message` qui renvoie une `response_variable`.
2.  **Discord** : Appel de `script.notification_discord` en injectant la réponse de K-2SO.
3.  **Awtrix** : Appel de `script.awtrix_dynamique_customapp_and_notify` avec une version courte.

**Exemple de code YAML projeté :**
```yaml
alias: "System - Monitor & Alert"
trigger:
  - platform: state
    entity_id: binary_sensor.frigate_running
    to: "off"
    for: "00:01:00"
    id: "crash_frigate"
action:
  # 1. On demande à K-2SO de parler
  - action: script.k_2so_generateur_de_message
    data:
      mission: "maintenance"
      details: "Frigate a cessé de fonctionner"
      consigne: "Sois inquiet et sarcastique sur la fiabilité"
    response_variable: k2so_msg

  # 2. On notifie sur Discord avec son message
  - action: script.notification_discord
    data:
      nom: "Alerte Crash"
      description: "{{ k2so_msg.data }}"
      image_url: "https://brands.home-assistant.io/frigate/icon.png"

  # 3. On alerte sur Awtrix (Message court statique ou dynamique)
  - action: script.awtrix_dynamique_customapp_and_notify
    data:
      message: "Frigate CRASH"
      icone: "frigate"
      color: "#FF0000"
      duree: 10
```
