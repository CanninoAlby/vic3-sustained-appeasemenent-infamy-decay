# Cost Preview and Tooltip Enhancement Implementation Plan

> **For Gemini:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Provide a dynamic weekly cost preview in the Decision and Event tooltips so the player knows exactly how much they will pay before activating a tier.

**Architecture:** We will implement `script_values` that calculate the total weekly cost by summing the GDP of all Great and Major Powers and applying the tier multipliers. These values will then be referenced in the localization via `GetScriptValue`.

**Tech Stack:** Victoria 3 Modding (Script Values, Localization, Decisions, Events)

---

### Task 1: Implement Cost Calculation Script Values

**Files:**
- Modify: `common/script_values/pp_infamy_values.txt`

**Step 1: Write the Calculation Logic**
We need a way to sum the GDP of all targets and apply the 0.1%, 0.3%, and 0.6% multipliers.

```vic3
# Total GDP of all Great and Major Powers (excluding player)
pp_total_target_gdp = {
	value = 0
	every_country = {
		limit = {
			NOT = { this = root }
			country_rank >= rank_value:major_power
		}
		add = gdp
	}
}

# Tier 1 Weekly Cost (0.1%)
pp_cost_tier_1 = {
	value = pp_total_target_gdp
	multiply = 0.001
}

# Tier 2 Weekly Cost (0.3%)
pp_cost_tier_2 = {
	value = pp_total_target_gdp
	multiply = 0.003
}

# Tier 3 Weekly Cost (0.6%)
pp_cost_tier_3 = {
	value = pp_total_target_gdp
	multiply = 0.006
}
```

**Step 2: Commit**
```bash
git add "common/script_values/pp_infamy_values.txt"
git commit -m "feat: add scripted values for dynamic cost calculation"
```

---

### Task 2: Update Localization with Dynamic Tooltips

**Files:**
- Modify: `localization/english/pp_infamy_l_english.yml`

**Step 1: Add Tooltip Strings**
We will add strings that use the `GetScriptValue` tag to display the numbers.

```yaml
 pp_infamy_decision_cost_tt: "Total Weekly Cost: @money![GetPlayer.GetScriptValue('pp_cost_tier_1')|v] - @money![GetPlayer.GetScriptValue('pp_cost_tier_3')|v]\n(Based on total GDP of Great and Major Powers)"
 pp_infamy_tier_1_cost_tt: "Weekly Cost: @money![GetPlayer.GetScriptValue('pp_cost_tier_1')|v]"
 pp_infamy_tier_2_cost_tt: "Weekly Cost: @money![GetPlayer.GetScriptValue('pp_cost_tier_2')|v]"
 pp_infamy_tier_3_cost_tt: "Weekly Cost: @money![GetPlayer.GetScriptValue('pp_cost_tier_3')|v]"
```

**Step 2: Commit**
```bash
git add "localization/english/pp_infamy_l_english.yml"
git commit -m "feat: add localization for dynamic cost tooltips"
```

---

### Task 3: Attach Tooltips to Decision and Event Options

**Files:**
- Modify: `common/decisions/pp_infamy.txt`
- Modify: `events/pp_infamy_event.txt`

**Step 1: Update the Decision**
Add a `custom_tooltip` to the decision so the range of costs is visible in the Journal/Decision tab.

```vic3
# In common/decisions/pp_infamy.txt
when_taken = {
    custom_tooltip = pp_infamy_decision_cost_tt
    trigger_event = { id = pp_infamy_event.1 popup = yes }
}
```

**Step 2: Update the Event Options**
Add specific cost previews to each event option.

```vic3
# In events/pp_infamy_event.txt
option = { # Tier 1
    name = pp_infamy.1.n
    custom_tooltip = pp_infamy_tier_1_cost_tt
    # ... rest of code
}

option = { # Tier 2
    name = pp_infamy.2.n
    custom_tooltip = pp_infamy_tier_2_cost_tt
    # ... rest of code
}

option = { # Tier 3
    name = pp_infamy.3.n
    custom_tooltip = pp_infamy_tier_3_cost_tt
    # ... rest of code
}
```

**Step 3: Commit**
```bash
git add "common/decisions/pp_infamy.txt" "events/pp_infamy_event.txt"
git commit -m "feat: attach cost tooltips to decision and event options"
```
