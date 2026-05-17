# Sustained Appeasement Mod Implementation Plan

> **For Gemini:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transform the instant-payoff infamy mod into a realistic, sustained economic transfer system where the player pays a weekly percentage of Great/Major Power GDPs directly to those AI nations for increased infamy decay.

**Architecture:** A decision triggers an event to select an appeasement tier. This sets a country modifier and initializes variables for tracking the cost. A weekly pulse script (`on_actions`) iterates through relevant AI nations, calculating the cost based on their GDP, deducting it from the player, and adding it to the AI treasuries. A yearly pulse recalculates the target nations.

**Tech Stack:** Victoria 3 Modding (Jomini Script, Events, Decisions, Modifiers, On Actions)

---

### Task 1: Define Static Modifiers for Appeasement Tiers

**Files:**
- Modify: `common/static_modifiers/pp_infamy_base_modifier.txt` (overwrite existing modifiers)

**Step 1: Write the Modifiers**
We need three tiers of modifiers that provide significant infamy decay boosts. Since this mod replaces the old one, we will redefine the existing `modifier_positive_propaganda` entries.

```vic3
modifier_positive_propaganda.1 = {
	icon = gfx/interface/icons/timed_modifier_icons/modifier_documents_positive.dds
	country_infamy_decay_mult = 2.0		# +200% base decay
}

modifier_positive_propaganda.2 = {
	icon = gfx/interface/icons/timed_modifier_icons/modifier_documents_positive.dds
	country_infamy_decay_mult = 9.0		# +900% base decay
}

modifier_positive_propaganda.3 = {
	icon = gfx/interface/icons/timed_modifier_icons/modifier_documents_positive.dds
	country_infamy_decay_mult = 24.0	# +2400% base decay
}
```

**Step 2: Commit**
```bash
git add "common/static_modifiers/pp_infamy_base_modifier.txt"
git commit -m "feat: redefine appeasement modifiers for percentage decay"
```

---

### Task 2: Create the Weekly Transfer Script Logic

**Files:**
- Create: `common/scripted_effects/pp_infamy_effects.txt`

**Step 1: Write the Weekly Transfer Effect**
This script calculates the cost based on the active modifier and performs the transfer to Great and Major Powers.

```vic3
pp_process_weekly_appeasement = {
    # Determine the multiplier based on active modifier
    if = {
        limit = { has_modifier = modifier_positive_propaganda.1 }
        set_variable = { name = pp_appeasement_rate value = 0.001 } # 0.1%
    }
    else_if = {
        limit = { has_modifier = modifier_positive_propaganda.2 }
        set_variable = { name = pp_appeasement_rate value = 0.003 } # 0.3%
    }
    else_if = {
        limit = { has_modifier = modifier_positive_propaganda.3 }
        set_variable = { name = pp_appeasement_rate value = 0.006 } # 0.6%
    }
    else = {
        # Failsafe if no modifier is active but script runs
        set_variable = { name = pp_appeasement_rate value = 0 }
    }

    # Only proceed if there is a rate set
    if = {
        limit = { var:pp_appeasement_rate > 0 }
        
        # Scope to all Great and Major powers (excluding self)
        every_country = {
            limit = {
                NOT = { this = root }
                country_rank >= rank_value:major_power
            }
            
            # Calculate cost for this specific AI
            set_variable = {
                name = temp_appeasement_cost
                value = gdp
            }
            multiply_variable = {
                name = temp_appeasement_cost
                value = root.var:pp_appeasement_rate
            }

            # Transfer logic: Subtract from player (root), Add to AI (this)
            # add_treasury uses negative numbers for subtraction
            root = {
                add_treasury = {
                    value = prev.var:temp_appeasement_cost
                    multiply = -1
                }
            }
            add_treasury = var:temp_appeasement_cost
            
            # Cleanup temp var on AI
            remove_variable = temp_appeasement_cost
        }
    }
}
```

**Step 2: Commit**
```bash
git add "common/scripted_effects/pp_infamy_effects.txt"
git commit -m "feat: create scripted effect for weekly wealth transfer"
```

---

### Task 3: Hook the Script to the Weekly Pulse

**Files:**
- Create: `common/on_actions/pp_infamy_on_actions.txt`

**Step 1: Write the On Action**
We need to trigger the effect every week for any country that has one of the modifiers.

```vic3
on_weekly_pulse_country = {
	events = {
        # Intentionally left blank, we use effect block
	}
    effect = {
        if = {
            limit = {
                OR = {
                    has_modifier = modifier_positive_propaganda.1
                    has_modifier = modifier_positive_propaganda.2
                    has_modifier = modifier_positive_propaganda.3
                }
            }
            pp_process_weekly_appeasement = yes
        }
    }
}
```

**Step 2: Commit**
```bash
git add "common/on_actions/pp_infamy_on_actions.txt"
git commit -m "feat: hook appeasement logic to weekly pulse"
```

---

### Task 4: Update the Decision and Event Logic

**Files:**
- Modify: `common/decisions/pp_infamy.txt`
- Modify: `events/pp_infamy_event.txt`

**Step 1: Modify the Decision**
Ensure the decision is repeatable or acts as a toggle, and only triggerable if infamy is high enough.

*In `common/decisions/pp_infamy.txt`*:
```vic3
pp_infamy_decision = {
	is_shown = {
		is_player = yes
		is_ai = no
	}

	possible = {
        infamy >= 25
	}

	when_taken = {
		trigger_event = { id = pp_infamy_event.1 popup = yes }
	}

	ai_chance = {
		base = 0
	}
}
```

**Step 2: Modify the Event**
Remove the old one-time `add_treasury` and `change_infamy` mechanics. The options should *only* apply the timed modifier.

*In `events/pp_infamy_event.txt`* (Replace entire contents):
```vic3
namespace = pp_infamy_event

pp_infamy_event.1 = {
	type = country_event
	title = pp_infamy.0.t	
	desc = pp_infamy.0.d
	flavor = pp_infamy.0.f
	icon = "gfx/interface/icons/event_icons/event_newspaper.dds"
	duration = 3
	on_created_soundeffect = "event:/SFX/UI/Alerts/event_appear"
	
	event_image = {
		video = "unspecific_signed_contract"
	}
	
	option = {	# Tier 1
		name = pp_infamy.1.n
		custom_tooltip = pp_infamy.1.tt
		trigger = {
			not = { has_modifier = modifier_positive_propaganda.1 }
		}
		add_modifier = {
			name = modifier_positive_propaganda.1
			years = 1
		}
        # Ensure mutually exclusive
        remove_modifier = modifier_positive_propaganda.2
        remove_modifier = modifier_positive_propaganda.3
	}
		
	option = {	# Tier 2
		name = pp_infamy.2.n
		custom_tooltip = pp_infamy.2.tt
		trigger = {
			not = { has_modifier = modifier_positive_propaganda.2 }
		}
		add_modifier = {
			name = modifier_positive_propaganda.2
			years = 1
		}
        remove_modifier = modifier_positive_propaganda.1
        remove_modifier = modifier_positive_propaganda.3
	}

	option = {	# Tier 3
		name = pp_infamy.3.n
		custom_tooltip = pp_infamy.3.tt
		trigger = {
			not = { has_modifier = modifier_positive_propaganda.3 }
		}
		add_modifier = {
			name = modifier_positive_propaganda.3
			years = 1
		}
        remove_modifier = modifier_positive_propaganda.1
        remove_modifier = modifier_positive_propaganda.2
	}
}
```

**Step 3: Commit**
```bash
git add "common/decisions/pp_infamy.txt" "events/pp_infamy_event.txt"
git commit -m "feat: update events and decisions to apply sustained modifiers"
```

---

### Task 5: Clean Up Old Script Values

**Files:**
- Modify/Delete: `common/script_values/pp_infamy_values.txt`

**Step 1: Remove obsolete logic**
The old complex math for one-time payments and instant infamy reduction is no longer needed. We can delete or empty this file to prevent conflicts and improve load times.

*Action:* Replace contents with empty file or a comment indicating it's deprecated.

**Step 2: Commit**
```bash
git add "common/script_values/pp_infamy_values.txt"
git commit -m "refactor: remove deprecated instant infamy math"
```
