# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Home Assistant heating management system implemented entirely using blueprints (automation templates). The system provides complete heating control including:
- Proportional thermostat with TPI (Time Proportional & Integral) algorithm
- Window opening detection and management
- Multiple heating modes (hors-gel/frost protection, eco, comfort)
- Schedule-based temperature control
- Presence detection

Documentation: https://hacf.fr/blog/confort-gestion-chauffage/

## Architecture

The system consists of two main blueprints in `/blueprint/`:

### 1. thermostat_tpi.yaml - TPI Thermostat Controller
Implements a proportional thermostat that calculates heating power based on:
- **Coefficients**: C (interior temp difference) and T (exterior temp difference)
- **Formula**: Power = coeff_c × (setpoint - temp_int) + coeff_t × (setpoint - temp_ext)
- **Triggers**: Every 10 minutes, on setpoint change, or window state change
- **Control logic**: Cycles heating switch on/off based on calculated power percentage (0-100%)
- **Safety**: Automatically turns off heating when window is open

Key entities required:
- `entity_consigne`: Input number for temperature setpoint
- `entity_temp_int/ext`: Temperature sensors (interior/exterior)
- `entity_fenetre`: Binary sensor for window opening
- `entity_puissance`: Input number to display current heating power
- `entity_chauffage`: Switch to control the actual heating system

### 2. chauffage_pilotage.yaml - Heating Mode Controller
Manages the setpoint temperature based on heating modes:
- **Hors-gel mode**: When heating activation is off (summer/long absence)
- **Eco mode**: When presence is off OR when schedule is off
- **Confort mode**: When presence is on AND schedule is on

The controller sets the `entity_consigne` value which is then used by the TPI thermostat.

Key entities required:
- `entity_consigne`: Same input number as used by TPI thermostat
- `entity_activation`: Input boolean for heating activation
- `entity_precense`: Input boolean for presence detection
- `entity_planning`: Schedule entity for time-based control

## System Integration

The two blueprints work together:
1. `chauffage_pilotage.yaml` determines the target temperature and updates `entity_consigne`
2. `thermostat_tpi.yaml` monitors `entity_consigne` and controls the heating switch to maintain that temperature

## Working with YAML Files

- All YAML files are Home Assistant configuration format
- VS Code is configured to treat `*.yaml` files as Home Assistant files (see `.vscode/settings.json`)
- Blueprints use Home Assistant's blueprint schema with inputs, variables, triggers, and actions
- Templates use Jinja2 syntax (`{{ }}` for expressions, `{% %}` for statements)

## Testing and Validation

Since this is a Home Assistant blueprint project:
- No build commands needed
- Testing requires importing blueprints into a Home Assistant instance
- Validation can be done using Home Assistant's configuration check tools
- The blueprints are designed to be imported via the `source_url` field pointing to the GitHub repository

## Important Notes

- The TPI algorithm uses a 10-minute cycle time (600 seconds) where heating power is converted to on-time (power × 6 = seconds)
- Mode is set to `restart` in TPI thermostat to cancel previous heating cycles when conditions change
- Temperature values should respect physical constraints (e.g., hors-gel: 3-10°C, eco: 10-21°C, confort: 15-28°C)
