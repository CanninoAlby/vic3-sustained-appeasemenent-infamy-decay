## Goal
Solve the UI bug and the mathematical bug elegantly by pre-calculating all three costs in the Decision and assigning them to three separate variables *before* the event opens. This allows the engine's native modifier tooltips to perfectly display the exact, non-scaling costs.

## Assumptions
* Victoria 3's engine evaluates `add_modifier` multipliers for UI tooltips based on the state of variables *at the exact moment the event window opens*.
* By setting `pp_locked_cost_1`, `pp_locked_cost_2`, and `pp_locked_cost_3` inside the decision's `when_taken` block, the variables will be accurately populated before the UI needs to render the event tooltips.
* We can use three separate script values (`pp_locked_val_1`, etc.) to pass these three distinct variables to the `multiplier` argument in the three options.

## Plan

**Step 1: Update Decision to Pre-calculate Costs**
*   **Files:** `common/decisions/pp_infamy.txt`
*   **Change:** In the `when_taken` block, before `trigger_event`, use `add_to_variable` to calculate and store the three tier costs into `pp_locked_cost_1`, `pp_locked_cost_2`, and `pp_locked_cost_3`.

**Step 2: Create Three Locked Script Values**
*   **Files:** `common/script_values/pp_infamy_values.txt`
*   **Change:** Delete `pp_locked_cost_value`. Create `pp_locked_val_1`, `pp_locked_val_2`, and `pp_locked_val_3` which simply return their respective `root.var:pp_locked_cost_X`.

**Step 3: Update Event Modifiers and Remove UI Hacks**
*   **Files:** `events/pp_infamy_event.txt`
*   **Change:** Remove the `custom_tooltip` blocks and `add_to_variable` blocks we added. Let the event naturally add `modifier_pp_appeasement_expense` but using `multiplier = pp_locked_val_1` for Option 1, `val_2` for Option 2, etc.

**Step 4: Revert Localization Hacks**
*   **Files:** `localization/english/pp_infamy_l_english.yml`, `localization/simp_chinese/pp_infamy_l_simp_chinese.yml`
*   **Change:** Revert the `pp_infamy_tier_X_cost_tt` strings back to their simple "Weekly Cost" format, since the native modifier tooltip will now correctly display the government expenses!

## Risks & mitigations
*   **Risk:** `add_to_variable` might not evaluate the script values if the scope isn't correctly set in the decision.
*   **Mitigation:** The decision `when_taken` block is in the country scope, which perfectly matches the required scope for `pp_cost_tier_X`.

## Rollback plan
*   Revert all modified files via git or manual backup.
