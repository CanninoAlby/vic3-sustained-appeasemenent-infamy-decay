# Superpowers Brainstorm

## Goal
Answer the user's question regarding whether the in-game money spent on appeasing infamy mechanically goes to the AI countries or simply disappears from the player's treasury.

## Constraints
* Must provide an accurate answer based strictly on the current mod script files (e.g., `pp_infamy_effects.txt`).

## Known context
* In `common/scripted_effects/pp_infamy_effects.txt`, the scripted effect `pp_process_weekly_appeasement` loops through AI countries and distributes money.
* It selects all countries where `NOT = { this = root }` (not the player) and `country_rank >= rank_value:major_power` (Major Powers and above).
* It uses the `add_treasury` command with a value of `gdp` multiplied by `root.var:temp_rate`.

## Risks
* Players might assume their money goes to *all* countries, whereas it currently only goes to Major Powers or higher. This could cause confusion if they check a minor power's treasury.

## Options (2–4)
1. **Explain the current behavior:** Simply state that the money *does* go to AI countries, but specify that it only targets Major Powers and above.
2. **Explain the exact math:** Clarify that the AI receives money proportional to their own GDP (`gdp * temp_rate`) rather than an exact split of the player's total expenditure.
3. **Propose an expansion (Optional):** Discuss whether the logic should be changed to include Minor Powers or Unrecognized powers to better reflect the global distribution of funds.

## Recommendation
Clearly confirm to the user that **yes, the money goes to the AI**. However, clarify two crucial details:
1. It is only given to **Major Powers and above**.
2. The amount they receive is based on *their own GDP* and the current appeasement rate tier (`temp_rate`), rather than a direct 1:1 transfer of the player's exact spent coins.

## Acceptance criteria
* The user's question is directly and accurately answered based on the script's `add_treasury` logic.
* The exact conditions (Major Power rank) and calculations (`gdp * temp_rate`) are communicated.
