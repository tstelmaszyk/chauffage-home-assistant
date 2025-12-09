# Home Assistant - Gestion de bout en bout du chauffage

Système complet de gestion du chauffage pour Home Assistant utilisant des blueprints (modèles d'automatisation).

Ce dépôt contient les fichiers associés à l'article du forum HACF : https://hacf.fr/blog/confort-gestion-chauffage/

## Fonctionnalités

- **Thermostat proportionnel TPI** (Time Proportional & Integral) : régulation précise de la température
- **Gestion des ouvertures de fenêtres** : coupure automatique du chauffage
- **Modes de chauffage multiples** : hors-gel, éco, confort
- **Planification horaire** : zones de confort programmables
- **Détection de présence** : adaptation automatique selon votre présence

## Architecture

Le système est composé de deux blueprints complémentaires :

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PILOTAGE CHAUFFAGE (chauffage_pilotage.yaml)         │
│                                                                         │
│  TRIGGERS:                                                              │
│  • Changement état activation_chauffage                                 │
│  • Changement état presence_maison                                      │
│  • Changement état planning_chauffage                                   │
│                                                                         │
│  ACTIONS:                                                               │
│  • Évalue le mode actif (HG / eco / confort) - basé sur les triggers    │
│  • Met à jour entity_consigne avec la température correspondante        │
│                                                                         │
│                              ↓                                          │
│                    [ entity_consigne ]  ← SORTIE                        │
│                              ↓                                          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               │ Température de consigne
                               │ (valeur partagée)
                               │
                               ↓
┌──────────────────────────────┴──────────────────────────────────────────┐
│                                                                         │
│                       [ entity_consigne ]  ← TRIGGER                    │
│                              ↓                                          │
│                    THERMOSTAT TPI (thermostat_tpi.yaml)                 │
│                                                                         │
│  TRIGGERS:                                                              │
│  • Toutes les 10 minutes (time_pattern)                                 │
│  • Changement de entity_consigne  ← LIEN ENTRE LES 2 BLUEPRINTS         │
│  • Changement état fenêtre                                              │
│                                                                         │
│  ACTIONS:                                                               │
│  • Lit: consigne, temp_int, temp_ext, fenêtre                           │
│  • Calcule: puissance = coeff_c×(consigne-temp_int) +                   │
│                         coeff_t×(consigne-temp_ext)                     │
│  • Met à jour entity_puissance (affichage)                              │
│  • Contrôle entity_chauffage:                                           │
│    - Puissance = 0%   → switch OFF                                      │
│    - Puissance = 100% → switch ON continu                               │
│    - Puissance = X%   → switch ON pendant (X × 6) secondes              │
│                                                                         │
│                              ↓                                          │
│                    [ entity_chauffage ]  ← SORTIE                       │
│                              ↓                                          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ↓
                      Contrôle physique du chauffage
```

### 1. Thermostat TPI (`thermostat_tpi.yaml`)

Implémente un thermostat proportionnel qui calcule la puissance de chauffe nécessaire :

**Formule** : `Puissance = coeff_c × (consigne - temp_int) + coeff_t × (consigne - temp_ext)`

- **coeff_c** (recommandé : 0.6) : coefficient pour l'écart de température intérieure
- **coeff_t** (recommandé : 0.01) : coefficient pour l'écart de température extérieure
- **Cycle** : 10 minutes (la puissance calculée en % détermine le temps de chauffe)
- **Sécurité** : arrêt automatique si une fenêtre est ouverte

### 2. Pilotage du chauffage (`chauffage_pilotage.yaml`)

Gère les différents modes de fonctionnement et ajuste automatiquement la température de consigne :

- **Mode Hors-gel** : température minimale (3-10°C) quand le chauffage est désactivé
- **Mode Éco** : température réduite (10-21°C) en cas d'absence ou hors plage horaire
- **Mode Confort** : température optimale (15-28°C) quand présence ET dans la plage horaire

**Point clé** : Le blueprint "Pilotage" modifie `entity_consigne`, ce qui déclenche automatiquement le blueprint "Thermostat TPI" grâce à son trigger sur cette même entité.

## Installation

### Méthode 1 : Import direct depuis GitHub

1. Dans Home Assistant, allez dans **Configuration** > **Blueprints**
2. Cliquez sur **Import Blueprint** (bouton bleu en bas à droite)
3. Collez l'URL du blueprint :
   - Thermostat TPI : `https://github.com/tstelmaszyk/chauffage-home-assistant/blob/main/blueprint/thermostat_tpi.yaml`
   - Pilotage chauffage : `https://github.com/tstelmaszyk/chauffage-home-assistant/blob/main/blueprint/chauffage_pilotage.yaml`

### Méthode 2 : Installation manuelle

1. Téléchargez les fichiers du dossier `blueprint/`
2. Copiez-les dans le dossier `blueprints/automation/` de votre configuration Home Assistant
3. Redémarrez Home Assistant ou rechargez les automatisations

## Configuration requise

Avant de configurer les blueprints, vous devez créer les entités suivantes dans Home Assistant :

### Pour le Thermostat TPI :

```yaml
# Dans configuration.yaml
input_number:
  consigne_temperature:
    name: Température de consigne
    min: 5
    max: 30
    step: 0.5
    unit_of_measurement: "°C"

  puissance_chauffage:
    name: Puissance de chauffe
    min: 0
    max: 100
    step: 1
    unit_of_measurement: "%"
```

Vous aurez également besoin de :
- Un capteur de température intérieure (`sensor.temperature_interieure`)
- Un capteur de température extérieure (`sensor.temperature_exterieure`)
- Un capteur d'ouverture de fenêtre (`binary_sensor.fenetre`)
- Un switch pour contrôler le chauffage (`switch.chauffage`)

### Pour le Pilotage du chauffage :

```yaml
# Dans configuration.yaml
input_boolean:
  activation_chauffage:
    name: Activation du chauffage
    icon: mdi:radiator

  presence_maison:
    name: Présence à la maison
    icon: mdi:home-account

schedule:
  planning_chauffage:
    name: Planning de chauffe
    # Configurez les plages horaires dans l'interface
```

## Utilisation

### Étape 1 : Créer l'automatisation TPI

1. Allez dans **Configuration** > **Automatisations & Scènes**
2. Créez une nouvelle automatisation depuis le blueprint **Thermostat TPI**
3. Configurez :
   - Coefficients C et T (valeurs recommandées : 0.6 et 0.01)
   - Sélectionnez vos entités (consigne, températures, fenêtre, puissance, chauffage)

### Étape 2 : Créer l'automatisation de pilotage

1. Créez une nouvelle automatisation depuis le blueprint **Pilotage chauffage**
2. Configurez :
   - Les températures pour chaque mode (hors-gel, éco, confort)
   - Sélectionnez vos entités (consigne, activation, présence, planning)

### Étape 3 : Utilisation quotidienne

- **Activation saisonnière** : utilisez `input_boolean.activation_chauffage` (ON en hiver, OFF en été)
- **Présence** : activez `input_boolean.presence_maison` quand vous êtes à la maison
- **Planning** : configurez vos plages horaires de confort via `schedule.planning_chauffage`
- La température de consigne s'ajuste automatiquement selon les modes
- Le thermostat TPI régule la chauffe pour atteindre la consigne

## Fonctionnement détaillé

### Hiérarchie des modes (pilotage)

1. **Activation OFF** → Mode Hors-gel (priorité absolue)
2. **Présence OFF** → Mode Éco
3. **Planning ON** → Mode Confort
4. **Planning OFF** → Mode Éco

### Algorithme TPI

Le thermostat calcule la puissance nécessaire toutes les 10 minutes :
- **Puissance = 0%** → chauffage éteint
- **Puissance = 100%** → chauffage allumé en continu
- **Puissance intermédiaire (ex: 60%)** → chauffage allumé 6 minutes puis éteint 4 minutes

Le mode `restart` annule les cycles en cours si les conditions changent.

## Paramètres recommandés

### Coefficients TPI

- **coeff_c** : 0.6 (à ajuster selon l'inertie de votre logement)
  - Augmenter si la température met trop de temps à monter
  - Diminuer si le système oscille trop

- **coeff_t** : 0.01 (à ajuster selon l'isolation)
  - Augmenter pour anticiper davantage les variations extérieures
  - Diminuer si le système réagit trop aux variations extérieures

### Températures

- **Hors-gel** : 7-8°C (protection minimale)
- **Éco** : 17-18°C (nuit ou absence courte)
- **Confort** : 19-21°C (présence)

## Dépannage

**Le chauffage ne démarre pas :**
- Vérifiez que `input_boolean.activation_chauffage` est ON
- Vérifiez que la fenêtre n'est pas ouverte
- Vérifiez que la consigne est supérieure à la température intérieure

**Le chauffage s'allume et s'éteint trop souvent :**
- Diminuez le coefficient C (ex: de 0.6 à 0.4)
- Vérifiez que vos capteurs de température sont précis et stables

**La température n'atteint jamais la consigne :**
- Augmentez le coefficient C (ex: de 0.6 à 0.8)
- Vérifiez que votre système de chauffage est suffisamment puissant

## Liens utiles

- Article complet : https://hacf.fr/blog/confort-gestion-chauffage/
- Forum HACF : https://hacf.fr/
- Documentation Home Assistant :  https://www.home-assistant.io/
                                  https://www.home-assistant.io/integrations/schedule/
                                  https://www.home-assistant.io/integrations/input_boolean
