# Scaling Modifiers for Budget Transparency Implementation Plan

> **For Gemini:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Display the dynamic weekly appeasement costs as a clear "National Expenses" line item in the budget tooltip using a scaling modifier, while preserving the background wealth transfer to AI nations.

**Architecture:** We will create a static modifier that applies a fixed expense (e.g., £1). The weekly pulse script will calculate the total cost (GDP sum * tier rate) and apply this modifier to the player with a multiplier equal to the total cost. This causes the budget UI to display "Appeasement Payments: -£[Total Cost]".

**Tech Stack:** Victoria 3 Modding (Static Modifiers, Scripted Effects, Script Values)

---

### Task 1: Define the Base Expense Modifier

**Files:**
- Modify: `common/static_modifiers/pp_infamy_base_modifier.txt`
- Modify: `localization/english/pp_infamy_l_english.yml`
- Modify: `localization/simp_chinese/pp_infamy_l_simp_chinese.yml`

**Step 1: Write the Modifier**
We define a modifier that represents a single unit of cost.

*In `common/static_modifiers/pp_infamy_base_modifier.txt` (Append to file):*
```vic3
# Base cost modifier for budget UI
modifier_pp_appeasement_expense = {
	icon = gfx/interface/icons/timed_modifier_icons/modifier_documents_negative.dds
	country_expenses_fixed_cost_add = 1.0 # 1 Pound per multiplier level
}
```

**Step 2: Add Localization**
*In `localization/english/pp_infamy_l_english.yml` (Append):*
```yaml
 modifier_pp_appeasement_expense: "International Appeasement Payments"
```

*In `localization/simp_chinese/pp_infamy_l_simp_chinese.yml` (Append):*
```yaml
 modifier_pp_appeasement_expense: "国际绥靖支出"
```

**Step 3: Commit**
```bash
git add "common/static_modifiers/pp_infamy_base_modifier.txt" "localization/english/pp_infamy_l_english.yml" "localization/simp_chinese/pp_infamy_l_simp_chinese.yml"
git commit -m "feat: define base expense modifier for budget UI"
```

---

### Task 2: Refactor Scripted Effect to Apply Scaling Modifier

**Files:**
- Modify: `common/scripted_effects/pp_infamy_effects.txt`

**Step 1: Update the Logic**
The weekly pulse must now calculate the total cost, apply the scaling modifier to the player, and then distribute the wealth to the AIs using `add_treasury`.

*In `common/scripted_effects/pp_infamy_effects.txt` (Replace entire contents):*
```vic3
pp_process_weekly_appeasement = {
	# 1. Determine active tier rate
	if = {
		limit = { has_modifier = modifier_positive_propaganda.1 }
		set_variable = { name = temp_rate value = 0.001 }
	}
	else_if = {
		limit = { has_modifier = modifier_positive_propaganda.2 }
		set_variable = { name = temp_rate value = 0.003 }
	}
	else_if = {
		limit = { has_modifier = modifier_positive_propaganda.3 }
		set_variable = { name = temp_rate value = 0.006 }
	}
	else = {
		set_variable = { name = temp_rate value = 0 }
	}
	
	# 2. Process payments if a tier is active
	if = {
		limit = { var:temp_rate > 0 }
		
		# Calculate total cost for player UI
		set_variable = { name = pp_total_weekly_cost value = 0 }
		
		# Distribute to AIs and sum total cost
		every_country = {
			limit = {
				NOT = { this = root }
				country_rank >= rank_value:major_power
			}
			# Calculate AI share
			set_variable = { name = ai_share value = gdp }
			multiply_variable = { name = ai_share value = root.var:temp_rate }
			
			# Give money to AI
			add_treasury = var:ai_share
			
			# Add to player's total cost counter
			root = {
				add_to_variable = { name = pp_total_weekly_cost value = prev.var:ai_share }
			}
			
			remove_variable = ai_share
		}
		
		# 3. Apply the scaling modifier to the Player for Budget UI
		# First remove any existing instance of the modifier
		if = {
			limit = { has_modifier = modifier_pp_appeasement_expense }
			remove_modifier = modifier_pp_appeasement_expense
		}
		
		# Apply fresh modifier with multiplier equal to total cost
		# country_expenses_fixed_cost_add = 1.0 * multiplier
		add_modifier = {
			name = modifier_pp_appeasement_expense
			multiplier = var:pp_total_weekly_cost
			days = 8 # Lasts just over a week, refreshed by the pulse
		}
	}
	else = {
		# Cleanup if no tier is active
		if = {
			limit = { has_modifier = modifier_pp_appeasement_expense }
			remove_modifier = modifier_pp_appeasement_expense
		}
	}
}
```

**Step 2: Commit**
```bash
git add "common/scripted_effects/pp_infamy_effects.txt"
git commit -m "feat: implement scaling modifier logic for budget transparency"
```
