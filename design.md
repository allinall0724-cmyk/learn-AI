# AI Learning Sandbox — Design Doc

## Concept

A local app where you spin up new AI agents by writing a short prompt describing what you want them to do — a "concept." A two-stage pipeline turns that into a living, learning creature you can watch improve in real time:

1. **Prompt Enhancer** (LLM) — takes your rough concept ("a fast swimmer that avoids predators") and expands it into a strict, structured spec: environment layout, what the agent can sense, what actions it can take, the reward rules, and when an episode ends.
2. **Learner** (RL agent) — starts with zero knowledge (random behavior) and is dropped into the environment defined by that spec. It improves purely through trial-and-error against the reward rules — no language understanding, no pretraining, genuinely from scratch.

You watch it train live: the creature's behavior getting less random, plus a fitness/reward curve climbing over time.

**Important distinction:** the Enhancer understands language because it's a pretrained LLM. The Learner does not — it never sees your prompt, only the numeric rules the Enhancer produced. That's what makes the "learns from scratch" part real rather than hand-waved.

## Architecture

```
[Your prompt] 
     ↓
[Prompt Enhancer: LLM call → structured JSON spec]
     ↓ 
[Environment Generator: builds a simulated arena from the spec]
     ↓
[Learner: agent w/ random init, trained via RL/genetic algorithm against reward rules]
     ↓
[Live Visualization: render behavior + reward curve, generation by generation]
     ↓
[Save agent's learned weights/config for later — reload, compare, or pit two AIs against each other]
```

## Recommended Stack (solo, local, phase 1)

- **Python** — richest ecosystem for this, and none of it requires a GPU for phase 1.
- **PyMunk or Box2D** — lightweight 2D physics for the arena and creature bodies.
- **Hand-rolled genetic algorithm / NEAT-style evolution** for the learner. Recommended over deep RL (PPO) to start: no ML framework needed, easier to write yourself end-to-end, and visually it's the most satisfying — you see a population of creatures get better across generations, not just one agent slowly improving.
- **Pygame** for the live view (simplest path to "watch it learn" with minimal setup).
- **Prompt Enhancer**: a single API call to an LLM (Claude or similar) with a system prompt that forces strict JSON output matching a schema you define — no free text, no ambiguity for the environment generator to parse.
- No auth, no multi-user, no server needed — it's just for you right now, so keep it a single local script/app.

## Enhancer Output Schema (example)

```json
{
  "goal_type": "flee_and_survive",
  "observations": ["nearest_predator_distance", "nearest_predator_angle", "energy_level"],
  "actions": ["turn_left", "turn_right", "accelerate", "decelerate"],
  "reward_rules": {
    "survive_per_tick": 0.1,
    "caught_by_predator": -100,
    "distance_from_predator_bonus": 0.05
  },
  "termination": ["caught_by_predator", "max_ticks_reached"],
  "arena": { "size": [800, 600], "obstacles": 3, "predators": 1 }
}
```

## MVP Scope (Phase 1)

- One environment template to start (e.g. flee/survive, or chase/hunt — pick whichever concept you want to try first).
- Fixed enhancer output schema (above) — expand schema later once the loop works end to end.
- Genetic algorithm: population of ~50-100 creatures per generation, simple feedforward neural net "brain" (a few dozen weights), mutate + select the fittest each generation.
- Visualization: 2D arena view + a live reward-over-generations line chart.

## Phase 2 (once MVP works)

- Swap in PPO (e.g. via Stable-Baselines3) for smoother continuous movement — walking, swimming — once you want more lifelike motion than genetic algorithms give you.
- More environment templates (hoard/collect, obstacle course, multi-agent chase).
- Save/load/breed trained agents; pit two of your own AIs against each other.
- Let the Enhancer's schema grow (custom reward shaping, richer observation spaces) instead of the fixed v1 schema.

## Repo Structure

```
ai-learning-sandbox/
  README.md
  docs/
    design.md              <- this doc
  prompt_enhancer/
    enhancer.py             # LLM call, prompt template, JSON schema validation
    schema.json
  env/
    arena.py                # builds physics arena from spec
    creatures.py             # creature body + sensor definitions
  agent/
    genetic.py               # phase 1: genetic algorithm / NEAT-style evolution
    ppo.py                    # phase 2: deep RL (stub for now)
  viz/
    renderer.py               # pygame live view
    charts.py                  # reward/fitness-over-time plot
  saved_agents/
  requirements.txt
```

## Open Decisions Before You Start Coding

- **Engine**: Python (recommended — ML ecosystem, faster to prototype the learning loop) vs. Godot (you already know it well, but you'd be hand-rolling neural nets/genetic algorithms in GDScript with no library support).
- **First concept to try**: pick one goal type for the MVP (flee/survive, chase/hunt, or collect/hoard) rather than building the generic schema before you've proven the loop works on one case.
- **Enhancer LLM**: any API works here since output is just structured JSON — doesn't need to be fancy, just needs to reliably hit the schema.
