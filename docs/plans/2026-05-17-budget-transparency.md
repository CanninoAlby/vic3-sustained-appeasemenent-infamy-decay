# Budget Transparency for Sustained Appeasement

> **For Gemini:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Make the weekly appeasement payments visible in the Victoria 3 budget UI by utilizing custom Diplomatic Pacts instead of raw `add_treasury` calls.

**Architecture:** We will define a custom `diplomatic_pact_type`. When the player activates a tier, the mod will establish these pacts with all Great and Major Powers. The pacts will handle the economic transfer, ensuring it appears as a "Diplomatic Pact" expense in the budget tooltip.

**Tech Stack:** Victoria 3 Modding (Diplomatic Pacts, Scripted Effects, On Actions)

---

### Task 1: Define the Custom Diplomatic Pact Type

**Files:**
- Create: `common/diplomatic_pact_types/pp_appeasement_pacts.txt`

**Step 1: Write the Pact Definition**
This pact will be "one-way" (player pays AI) and its cost will be calculated dynamically.

```vic3
pp_appeasement_pact = {
	is_read_only = yes # Cannot be broken manually in the diplo screen
	is_subsidies = yes # Marks it as an expense in the budget
	
	selectable_in_diplomatic_panel = no
	
	cost = {
		value = second_country.gdp
		multiply = root.var:pp_appeasement_rate
	}

    # The AI receives what the player pays
    income = {
        value = root.gdp
        multiply = root.var:pp_appeasement_rate
    }
}
```

**Step 2: Commit**
```bash
git add "common/diplomatic_pact_types/pp_appeasement_pacts.txt"
git commit -m "feat: define custom diplomatic pact for budget transparency"
```

---

### Task 2: Refactor Scripted Effect to Use Pacts

**Files:**
- Modify: `common/scripted_effects/pp_infamy_effects.txt`

**Step 1: Update the Logic**
Instead of manually adding/subtracting treasury, we will `establish_diplomatic_pact` and `break_diplomatic_pact`.

```vic3
pp_process_weekly_appeasement = {
    # 1. Update the Rate Variable on Root (Player)
    if = {
        limit = { has_modifier = modifier_positive_propaganda.1 }
        set_variable = { name = pp_appeasement_rate value = 0.001 }
    }
    else_if = {
        limit = { has_modifier = modifier_positive_propaganda.2 }
        set_variable = { name = pp_appeasement_rate value = 0.003 }
    }
    else_if = {
        limit = { has_modifier = modifier_positive_propaganda.3 }
        set_variable = { name = pp_appeasement_rate value = 0.006 }
    }
    else = {
        set_variable = { name = pp_appeasement_rate value = 0 }
    }

    # 2. Manage Pacts
    if = {
        limit = { var:pp_appeasement_rate > 0 }
        every_country = {
            limit = {
                NOT = { this = root }
                country_rank >= rank_value:major_power
            }
            # Establish if not exists
            if = {
                limit = {
                    NOT = { 
                        has_diplomatic_pact = { 
                            who = root 
                            type = pp_appeasement_pact 
                        } 
                    }
                }
                establish_diplomatic_pact = { 
                    type = pp_appeasement_pact 
                    target = root 
                }
            }
        }
        # Break pacts with countries that are no longer Major Powers
        every_country = {
            limit = {
                has_diplomatic_pact = { 
                    who = root 
                    type = pp_appeasement_pact 
                }
                country_rank < rank_value:major_power
            }
            break_diplomatic_pact = { 
                type = pp_appeasement_pact 
                target = root 
            }
        }
    }
    else = {
        # Break all pacts if no tier is active
        every_country = {
            limit = {
                has_diplomatic_pact = { 
                    who = root 
                    type = pp_appeasement_pact 
                }
            }
            break_diplomatic_pact = { 
                type = pp_appeasement_pact 
                target = root 
            }
        }
    }
}
```

**Step 2: Commit**
```bash
git add "common/scripted_effects/pp_infamy_effects.txt"
git commit -m "feat: refactor logic to use diplomatic pacts instead of direct treasury calls"
```

---

### Task 3: Add Localization for the Pact

**Files:**
- Modify: `localization/english/pp_infamy_l_english.yml`

**Step 1: Add Strings**
We need a name for the pact so it looks correct in the budget menu.

```yaml
 pp_appeasement_pact: "International Appeasement Payments"
 pp_appeasement_pact_desc: "We are providing sustained financial support to ensure our international interests are viewed favorably."
```

**Step 2: Commit**
```bash
git add "localization/english/pp_infamy_l_english.yml"
git commit -m "feat: add localization for appeasement pact"
```
