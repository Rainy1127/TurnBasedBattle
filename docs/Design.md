# Combat Design

Words only, no C++. I will choose the form (classes, structs) in January, when it is clear what I have already learned.

## 1. Entities

### Fighter
Hero and enemy are the same entity. They differ by the `side` field. Decisions are made by the player or by a simple AI.

- **Stores:** name, health, armor, magic resistance, initiative, side, action points (maximum), current action points, list of abilities, list of active effects
- **Can:** start the turn, use an ability, take damage, receive an effect, end the turn

### Ability
A recipe. It is stored in JSON and does not change during the battle.

- **Stores:** name, damage, damage type (physical or magical), cost in action points, which effect it applies, for how many turns, effect power
- **Can:** check if the fighter has enough current action points to use it; apply itself to a target

### Effect
It exists only during the battle and is attached to one specific fighter. Each fighter has their own counter. It is created when an ability is used on a target: kind, turns and power are copied from the ability. Changes in the JSON do not change an effect that already exists.

- **Stores:** kind (burn, stun), turns left, power
- **Can:**
  - trigger at the start of the turn of the fighter it is attached to
  - decrease the turns counter by 1
  - remove itself at 0
- **When it triggers:**
  - burn: the fighter's health decreases by the effect's power
  - stun: the fighter's current action points decrease by the effect's power

### Battle

- **Stores:** list of fighters, whose turn it is, round number, battle status
- **Can:** decide whose turn is next; pass the turn; check if the battle is over

## 2. Turn

Order at the start of a fighter's turn:

1. Current action points are set to the maximum.
2. Effects trigger: burn reduces health, stun reduces **current** action points.
3. The fighter chooses an ability and acts.

Why: if stun reduced the maximum, the maximum would have to be restored when the effect ends, and that is a source of silent bugs. If it reduces the current points, the maximum stays constant and nothing needs to be restored.

Example: maximum 5, stun removes 2. At step 3 the fighter has 3 points. On the next turn (the stun is already gone) they have 5 again.

## 3. Battle Rules

- Battle status: **in progress**, **victory**, **defeat**.
- The battle is over when all fighters of one side have 0 health.
- No draws: only the fighter whose turn it is deals damage, and burn only hurts the fighter it is attached to. If an ability that also hits its user is added, draws will have to come back.
- Ability check: the fighter's current action points are compared with the ability's cost. The maximum is not used in this check.
- A fighter with 0 health skips every turn until the end of the battle.
- End-of-battle check: it happens after effects and after an ability. If it happened only after an ability, the hero would get an extra turn in an already won battle (the last enemy burned to death).
- If `effect` is `none`, the values of `turns` and `power` are ignored, and the ability counts as having no effect, whatever those values are.
- Reapplying effects: a fighter cannot have two effects of the same kind at once. Applying it again fully replaces the current effect with the new values from the ability, so a weak but long burn completely replaces a strong one.
- The turn queue is built once at the start of the battle: fighters go in descending order of initiative. If an effect that changes initiative appears, the queue must be rebuilt every round, from the living fighters.
- Equal initiative: the order between such fighters is random, decided once when the queue is built.
- Defense: armor and magic resistance are percentages and reduce damage by that percent. Armor reduces physical damage, magic resistance reduces magical damage, including burn.
- Burn damage is calculated when the effect triggers, using the resistance the fighter has on that turn.
- Rounding: after defense is applied, the fractional part is dropped (round down). Damage can become 0, and that is allowed. Known consequence: on small numbers defense is very noticeable: damage 2 with 10% resistance becomes 1, and damage 1 is fully absorbed by any non-zero defense.
- End of turn: the player ends the turn manually, at any moment, even with points left. The turn of an AI-controlled fighter ends when its current points are not enough for any of its abilities.

## 4. Not in This Version

- Levels
- Items
- Equipment

Reason: each of these pulls in a separate system, and the project has only 4-8 weeks. They are not part of the project goals.

## 5. Data

### Abilities, first version

```json
[
  {
    "name": "Fire Ball",
    "damage": 5,
    "type": "magical",
    "cost": 4,
    "effect": "burn",
    "turns": 2,
    "power": 2
  },
  {
    "name": "Hand Strike",
    "damage": 2,
    "type": "physical",
    "cost": 2,
    "effect": "none",
    "turns": 0,
    "power": 0
  },
  {
    "name": "Stunning Blow",
    "damage": 2,
    "type": "physical",
    "cost": 3,
    "effect": "stun",
    "turns": 2,
    "power": 2
  }
]
```

## 6. Enemy AI

- Target: the hero (for now there is only one fighter on the player's side).
- For every ability it has enough current points for, the AI calculates the damage after the target's defense, using the same formula as in the battle.
- It picks the ability with the highest damage.
- On equal damage, it prefers the one that applies an effect.
- If that is equal too, it picks the cheaper one: more points are left for the next ability.
- If the cost is equal too, it picks the one that comes first in the fighter's list of abilities.
- An effect counts as a plus even when the target already has that effect: applying it again extends it.

Known consequences:
- The AI picks the best option right now and does not plan the whole turn. Fire Ball (3 damage, 1 point left) is weaker in total than Hand Strike plus Stunning Blow (1 + 1 damage and a stun). This is acceptable for the first version.
- With several enemies, a second stun in the same round changes nothing, but the AI will still prefer it.

## Open Questions

- [x] A burn kills a fighter at the start of the turn (step 2). Should they reach step 3?
- [x] When should the battle check if it is over: only after abilities, or after effects too?
- [x] How damage is calculated: formula with armor and magic resistance
- [x] Abilities block: JSON format, list of abilities for the first version
- [x] Effects block: exact rules for burn and stun
- [x] Simple enemy AI: how it chooses an ability and a target
- [x] Do armor and magic resistance affect burn damage
- [x] How many abilities a fighter can use per turn and when the turn ends
- [ ] How to test the queue if the order is random for equal initiative