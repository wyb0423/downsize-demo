# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Victoria 3 mod that solves late-game private investment AI overbuilding.

**For detailed Chinese documentation aimed at beginners, see `README.md`.**

**v2 features:**
- Interactive Journal Entry with enable/disable and weekly/monthly frequency toggle
- Gradual level reduction (tiered by building level and severity)
- 6-month cooldown per building to prevent repeated downsizing
- Statistics tracking (buildings affected count)

## Mod Installation (for testing)

```bash
cp -r v3-auto-downsize ~/Documents/Paradox\ Interactive/Victoria\ 3/mod/
```

Enable in Paradox Launcher playset.

## Game Installation Paths

- Game files: `/Users/wyb/Library/Application Support/Steam/steamapps/common/Victoria 3/game/`
- User data: `~/Documents/Paradox Interactive/Victoria 3/`
- Error logs: `~/Documents/Paradox Interactive/Victoria 3/logs/error.log`

## Architecture

```
v3-auto-downsize/
├── .metadata/metadata.json
├── common/
│   ├── defines/auto_downsize_defines.txt            # AI construction constraints (prevention)
│   ├── journal_entries/auto_downsize_je.txt    # JE: UI + pulse hooks
│   ├── scripted_buttons/auto_downsize_buttons.txt  # Toggle & frequency buttons
│   ├── scripted_effects/auto_downsize_effects.txt  # Core downsize logic (treatment)
│   └── on_actions/auto_downsize_on_actions.txt     # AI half-yearly hook
└── localization/
    ├── english/auto_downsize_l_english.yml
    └── simp_chinese/auto_downsize_l_simp_chinese.yml
```

**Flow:** JE pulses (weekly/monthly) → condition check (enabled + frequency) → scripted_effect (iterate buildings, filter, reduce levels, track stats)

JE pulses are used instead of global `on_actions` because they are scoped to the player's country only and support `on_weekly_pulse` / `on_monthly_pulse`.

## Key Scripting API (confirmed from game files)

### Journal Entry Pulses
- `on_weekly_pulse = { effect = { ... } }` — fires every week per active JE
- `on_monthly_pulse = { effect = { ... } }` — fires every month per active JE
- JE scope: root = country that owns the JE

### Scripted Buttons
```text
button_key = {
    name = "LOC_KEY"        # Button label
    desc = "LOC_KEY_DESC"   # Tooltip text
    visible = { ... }       # When button appears
    possible = { ... }      # When button is clickable
    selected = { ... }      # When highlighted
    effect = { ... }        # On click
}
```

### Building Iteration
- `every_scope_building = { limit = { ... } ... }` — iterates ALL buildings in a country (country scope, confirmed functional)

### Building Triggers (building scope)
- `occupancy` — staffing level (0.0–1.0)
- `weekly_profit` — weekly profit in £
- `cash_reserves_ratio` — cash reserves fullness (0.0–1.0)
- `is_building_group = bg_xxx` — check building group membership
- `is_subsistence_building = yes` — check if subsistence building
- `is_building_type = building_xxx` — check exact building type
- `level` — current building level
- `has_variable = <name>` — check if variable exists on scope

### Building Effects (state scope)
- `add_building_level = { building = <building_type_key> level = <int> }` — change building level (negative = reduce)
- `remove_building = <building_type_key>` — removes a building type from state

### Variables
- Country (ROOT) scope: `set_variable = { name = xxx }`, `change_variable = { name = xxx add = N }`
- Building scope: `set_variable = { name = xxx months = N }` (cooldown pattern)
- Variable existence: `has_variable = xxx`

### Scope References
- `ROOT` — original scope (country in JE context)
- `PREV` — previous scope in chain
- `THIS` — current scope
- `state = { ... }` — enter state scope (from building or country)

## Key Building Groups (for filtering)

Excluded from downsizing: `bg_government`, `bg_military`, `bg_public_infrastructure`, subsistence buildings, `bg_owner_buildings`, `bg_financial_districts`, `bg_manor_houses`, `bg_company_headquarter`, `bg_company_regional_headquarter`, `bg_trade`

## Downsize Tiers (v2)

| Building Level | Severe (occ<25%, cash<10%) | Moderate (occ<50%, cash<25%) |
|----------------|---------------------------|------------------------------|
| 1-10           | -1 level                  | -1 level                     |
| 11-25          | -3 levels                 | -2 levels                    |
| 26-50          | -5 levels                 | -3 levels                    |
| 51-100         | -8 levels                 | -5 levels                    |
| 100+           | -12 levels                | -8 levels                    |

Cooldown: 6 months per building after downsizing.

## Risk: `add_building_level = { building = PREV ... }` syntax

The `PREV` reference in the `building` parameter may not resolve correctly (building type key vs scope). If error.log shows errors about this syntax, the fallback is hardcoded building type checks. The game code uses `PREV` as parameter values elsewhere (e.g. `transfer_character = PREV`), so this is likely supported.

## Testing

1. Copy mod to `~/Documents/Paradox Interactive/Victoria 3/mod/`
2. Enable in launcher playset
3. Load late-game save (1880+), open Journal panel
4. Click the toggle button to enable
5. Advance 1+ months, check building browser for level reductions
6. Check error.log:
   ```bash
   tail -f ~/Documents/Paradox\ Interactive/Victoria\ 3/logs/error.log
   ```

## Files to NOT modify

- `.metadata/metadata.json` — launcher metadata
- Localization files must use UTF-8 BOM encoding; if game doesn't show text, re-save with BOM
