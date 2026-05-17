# Plan: Finalizing UI and Budget Fixes

> **For Gemini:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Permanently fix the tooltip `/w` error and the invisible budget expense by relying strictly on pre-calculated variables and simple script values.

**Architecture:** 
1. The `when_taken` block of the decision will trigger a scripted effect to pre-calculate the exact costs for Tier 1, 2, and 3, saving them as variables on the player country.
2. The UI tooltips (localization) will directly read these variables using `[GetPlayer.MakeScope.Var('name').GetValue|0v]`. This is 100% UI-safe and prevents evaluation errors.
3. The event will lock the chosen cost into `pp_locked_weekly_cost`.
4. The expense modifier will use a simple script value `pp_locked_cost_value` that just reads `root.var:pp_locked_weekly_cost` to ensure the `multiplier` argument parses correctly.

**Tech Stack:** Victoria 3 Modding (Variables, Script Values, Localization, Modifiers)

---

### Task 1: Create Reliable Script Values

**Files:**
- Modify: `common/script_values/pp_infamy_values.txt`

**Step 1: Write the Values**
Replace any existing content with a simple wrapper for the locked cost. This ensures the `add_modifier` command can safely use it as a multiplier.

```vic3
# Simply reads the locked cost variable
pp_locked_cost_value = {
	value = root.var:pp_locked_weekly_cost
}
```

**Step 2: Commit**
```bash
git add "common/script_values/pp_infamy_values.txt"
git commit -m "fix: replace complex script values with a safe wrapper"
```

---

### Task 2: Implement Pre-Calculation Effect

**Files:**
- Modify: `common/scripted_effects/pp_infamy_effects.txt`

**Step 1: Write the Pre-Calculation Logic**
Add `pp_calculate_target_gdp` to calculate the costs and save them as variables *before* the event opens. Keep the `pp_process_weekly_appeasement` effect for wealth transfer.

```vic3
pp_calculate_target_gdp = {
	set_variable = { name = pp_total_target_gdp value = 0 }
	every_country = {
		limit = {
			NOT = { this = root }
			country_rank >= rank_value:major_power
		}
		root = {
			add_to_variable = { name = pp_total_target_gdp value = prev.gdp }
		}
	}
	
	# Pre-calculate Tier 1
	set_variable = { name = pp_cost_tier_1_var value = var:pp_total_target_gdp }
	multiply_variable = { name = pp_cost_tier_1_var value = 0.001 }
	
	# Pre-calculate Tier 2
	set_variable = { name = pp_cost_tier_2_var value = var:pp_total_target_gdp }
	multiply_variable = { name = pp_cost_tier_2_var value = 0.003 }
	
	# Pre-calculate Tier 3
	set_variable = { name = pp_cost_tier_3_var value = var:pp_total_target_gdp }
	multiply_variable = { name = pp_cost_tier_3_var value = 0.006 }
}

pp_process_weekly_appeasement = {
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
	
	if = {
		limit = { var:temp_rate > 0 }
		every_country = {
			limit = {
				NOT = { this = root }
				country_rank >= rank_value:major_power
			}
			# AI receives their share
			add_treasury = {
				value = gdp
				multiply = root.var:temp_rate
			}
		}
	}
}
```

**Step 2: Commit**
```bash
git add "common/scripted_effects/pp_infamy_effects.txt"
git commit -m "feat: pre-calculate costs to variables for safe UI rendering"
```

---

### Task 3: Hook Pre-Calculation into Decision

**Files:**
- Modify: `common/decisions/pp_infamy.txt`

**Step 1: Update the Decision**
Ensure `pp_calculate_target_gdp` runs when the decision is clicked, *before* the event triggers.

```vic3
pp_infamy_decision = {
	name = pp_infamy_decision
	is_shown = {
		is_player = yes
		is_ai = no
	}

	possible = {
        infamy >= 25
	}

	when_taken = {
		pp_calculate_target_gdp = yes
		trigger_event = { id = pp_infamy_event.1 popup = yes }
	}

	ai_chance = {
		base = 0
	}
}
```

**Step 2: Commit**
```bash
git add "common/decisions/pp_infamy.txt"
git commit -m "feat: calculate appeasement costs when decision is taken"
```

---

### Task 4: Update Event Modifier Application

**Files:**
- Modify: `events/pp_infamy_event.txt`

**Step 1: Fix the `add_modifier` multiplier**
Change `multiplier = var:pp_locked_weekly_cost` to `multiplier = pp_locked_cost_value` to ensure it parses correctly.

*(In `events/pp_infamy_event.txt`, for all 3 options, change the `add_modifier` block for `modifier_pp_appeasement_expense`)*:
```vic3
		add_modifier = {
			name = modifier_pp_appeasement_expense
			multiplier = pp_locked_cost_value
			years = 1
		}
```

**Step 2: Commit**
```bash
git add "events/pp_infamy_event.txt"
git commit -m "fix: use script value for modifier multiplier"
```

---

### Task 5: Update Localization to Read Variables

**Files:**
- Modify: `localization/english/pp_infamy_l_english.yml`
- Modify: `localization/simp_chinese/pp_infamy_l_simp_chinese.yml`

**Step 1: Update the UI Strings**
Replace `[GetPlayer.GetScriptValue('...')]` with `[GetPlayer.MakeScope.Var('...').GetValue|0v]`.

*English:*
```yaml
 pp_infamy.1.n: "Scholarly Dialogues (@money![GetPlayer.MakeScope.Var('pp_cost_tier_1_var').GetValue|0v]/w)"
 pp_infamy.2.n: "Media Partnerships (@money![GetPlayer.MakeScope.Var('pp_cost_tier_2_var').GetValue|0v]/w)"
 pp_infamy.3.n: "Diplomatic Visits (@money![GetPlayer.MakeScope.Var('pp_cost_tier_3_var').GetValue|0v]/w)"
 pp_infamy_decision_cost_tt: "Total Weekly Cost: @money![GetPlayer.MakeScope.Var('pp_cost_tier_1_var').GetValue|0v] - @money![GetPlayer.MakeScope.Var('pp_cost_tier_3_var').GetValue|0v]\n(Based on total GDP of Great and Major Powers)"
 pp_infamy_tier_1_cost_tt: "Weekly Cost: @money![GetPlayer.MakeScope.Var('pp_cost_tier_1_var').GetValue|0v]"
 pp_infamy_tier_2_cost_tt: "Weekly Cost: @money![GetPlayer.MakeScope.Var('pp_cost_tier_2_var').GetValue|0v]"
 pp_infamy_tier_3_cost_tt: "Weekly Cost: @money![GetPlayer.MakeScope.Var('pp_cost_tier_3_var').GetValue|0v]"
```

*Simplified Chinese:*
```yaml
 pp_infamy.1.n: "举行国际学术交流活动 (@money![GetPlayer.MakeScope.Var('pp_cost_tier_1_var').GetValue|0v]/周)"
 pp_infamy.2.n: "与海外媒体建立长期的稳定合作 (@money![GetPlayer.MakeScope.Var('pp_cost_tier_2_var').GetValue|0v]/周)"
 pp_infamy.3.n: "派遣高级官员出访邻国 (@money![GetPlayer.MakeScope.Var('pp_cost_tier_3_var').GetValue|0v]/周)"
 pp_infamy_decision_cost_tt: "总每周成本: @money![GetPlayer.MakeScope.Var('pp_cost_tier_1_var').GetValue|0v] - @money![GetPlayer.MakeScope.Var('pp_cost_tier_3_var').GetValue|0v]\n(基于列强和主要国家的总GDP)"
 pp_infamy_tier_1_cost_tt: "每周成本: @money![GetPlayer.MakeScope.Var('pp_cost_tier_1_var').GetValue|0v]"
 pp_infamy_tier_2_cost_tt: "每周成本: @money![GetPlayer.MakeScope.Var('pp_cost_tier_2_var').GetValue|0v]"
 pp_infamy_tier_3_cost_tt: "每周成本: @money![GetPlayer.MakeScope.Var('pp_cost_tier_3_var').GetValue|0v]"
```
*(Make sure to maintain UTF-8 BOM encoding when writing!)*

**Step 2: Commit**
```bash
git add "localization/english/pp_infamy_l_english.yml" "localization/simp_chinese/pp_infamy_l_simp_chinese.yml"
git commit -m "fix: switch tooltips to read pre-calculated variables safely"
```
