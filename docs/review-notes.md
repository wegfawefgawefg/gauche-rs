# Review Notes

Codebase review taken on March 11, 2026.

## Overall Read

This does **not** read like AI-vibe-code. It reads like a hand-built prototype that moved quickly and accumulated systems before a later cleanup pass.

What looks good:

- coherent game loop
- intentional entity/version handle model
- distinct state, render, input, and asset layers
- generally modern-feeling presentation for a small raylib project

The main risks are not "bad globals" so much as:

- a few brittle runtime assumptions
- oversized policy-heavy files
- some product-surface modes that are still stubs

## Findings

### 1. Screen-to-world conversion will drift if the real window size changes

Severity: High

Rendering already uses actual screen size in fullscreen in [src/render.rs](/home/vega/Coding/GameDev/gauche/src/render.rs), but input conversion still scales against fixed `window_dims` in [src/graphics.rs](/home/vega/Coding/GameDev/gauche/src/graphics.rs).

Why it matters:

- fullscreen or resize support will make mouse targeting inaccurate
- video settings work will be harder to finish correctly

Relevant code:

- [src/graphics.rs](/home/vega/Coding/GameDev/gauche/src/graphics.rs)
- [src/render.rs](/home/vega/Coding/GameDev/gauche/src/render.rs)

### 2. Entity free-list can be corrupted by double-deactivation

Severity: High

Both `set_inactive` and `set_entity_inactive` push IDs back into `available_ids` without checking whether the entity is already inactive.

Why it matters:

- the same slot can be returned twice later
- that can become hard-to-reproduce entity corruption

Relevant code:

- [src/entity_manager.rs](/home/vega/Coding/GameDev/gauche/src/entity_manager.rs)

### 3. Runtime safety depends on stale VIDs never happening

Severity: Medium

There are many `unwrap()` calls on entity fetches in hot gameplay paths. That is okay while the spawn/despawn timing remains simple, but one stale handle bug will turn into a hard crash.

Why it matters:

- future combat/spawn/despawn changes become riskier
- runtime failures will be abrupt instead of gracefully contained

Relevant code examples:

- [src/step.rs](/home/vega/Coding/GameDev/gauche/src/step.rs)
- [src/inputs.rs](/home/vega/Coding/GameDev/gauche/src/inputs.rs)
- [src/entity_behavior.rs](/home/vega/Coding/GameDev/gauche/src/entity_behavior.rs)

### 4. Several advertised modes are still stub surfaces

Severity: Medium

`Settings`, `VideoSettings`, and `Win` exist in the mode enum and render/input dispatch, but do not have real behavior yet.

Why it matters:

- architecture implies more completeness than the repo actually has
- dead/stub surface area makes future cleanup noisier

Relevant code:

- [src/state.rs](/home/vega/Coding/GameDev/gauche/src/state.rs)
- [src/render.rs](/home/vega/Coding/GameDev/gauche/src/render.rs)
- [src/inputs.rs](/home/vega/Coding/GameDev/gauche/src/inputs.rs)

### 5. A few files still own too much policy

Severity: Medium

The biggest architectural pressure is not the globals themselves, but the fact that a few large files still own too many unrelated responsibilities.

Main files:

- [src/entity_behavior.rs](/home/vega/Coding/GameDev/gauche/src/entity_behavior.rs)
- [src/render_ui.rs](/home/vega/Coding/GameDev/gauche/src/render_ui.rs)
- [src/inputs.rs](/home/vega/Coding/GameDev/gauche/src/inputs.rs)
- [src/step.rs](/home/vega/Coding/GameDev/gauche/src/step.rs)

## Note On Globals

Global or singleton-ish ownership is not inherently a problem here. Old C/C++ engines and plenty of game prototypes do this successfully.

The real issues are:

- boundary discipline
- header/module hygiene
- oversized policy files
- correctness around reload/init/despawn assumptions

If this repo kept growing, the next cleanup pass should focus on:

1. fixing the free-list safety issue
2. making mouse/world conversion use the actual live window size
3. shrinking the largest policy files
4. either implementing or removing the stub modes
