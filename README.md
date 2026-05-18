# Building Adapter-Driven Home Assistant Integrations

> A general guide for HA custom_component authors who want to support many vendors, models, or device variants with one codebase — without ending up with a forest of `if device.brand == "x"` branches.

This guide describes a **runtime-configurable adapter pattern** for Home Assistant integrations. The core idea: every fact that varies by vendor, model, or device variant lives in a *config dict registered at runtime*, and the framework code reads from that registry instead of containing brand-specific names, vocabularies, or wire formats. Adding support for a new brand becomes a config-only change — you write a new adapter dict, register it, ship it.

The pattern was developed during a refactor of a multi-thousand-line custom vacuum integration that originally hardcoded one brand's entity names, dropdown options, payload shapes, and completion signals. The refactor pulled all of those into adapter configs and turned the rest of the integration into vendor-agnostic framework code. The result handles brand additions as ~200-line config files instead of cross-cutting code edits.

Examples in this guide are HA-flavored (vacuum, HVAC, sensor) but the pattern itself is domain-agnostic. Anywhere this guide says "vacuum" you could substitute "thermostat", "lock", "amplifier", "irrigation controller", "EV charger" — the architecture is the same.

---

## Table of Contents

### Part I — Core Architecture

1. [When to use this pattern](#1-when-to-use-this-pattern)
2. [The mental model](#2-the-mental-model)
   - [Core Rules of the Pattern](#core-rules-of-the-pattern)
3. [Designing the adapter config](#3-designing-the-adapter-config)
4. [The adapter registry](#4-the-adapter-registry)

### Part II — Runtime Systems

5. [Entity resolution by role](#5-entity-resolution-by-role)
6. [Vocabulary and dropdown options](#6-vocabulary-and-dropdown-options)
7. [Dispatch and wire-format payloads](#7-dispatch-and-wire-format-payloads)
8. [Capability gating](#8-capability-gating)
9. [Background listeners (state-watching by role)](#9-background-listeners-state-watching-by-role)
10. [Storage decoupling](#10-storage-decoupling)
11. [The service-layer auto-resolve pattern](#11-the-service-layer-auto-resolve-pattern)

### Part III — Frontend Integration

12. [Frontend / card integration](#12-frontend--card-integration)

### Part IV — Discovery & Extensibility

13. [Discovery helpers for new brands](#13-discovery-helpers-for-new-brands)
14. [The porting workflow](#14-the-porting-workflow)

### Part V — Operations & Maintenance

15. [Anti-patterns and pitfalls](#15-anti-patterns-and-pitfalls)
16. [Testing strategy](#16-testing-strategy)
17. [Versioning the adapter contract](#17-versioning-the-adapter-contract)
18. [Observability, validation, and diagnostics](#18-observability-and-diagnostics)
19. [Adapter declarations vs. user preferences](#19-adapter-declarations-vs-user-preferences)

### Part VI — Appendices

20. [Appendix: minimal scaffold](#20-appendix-minimal-scaffold)
21. [Appendix: Adapter contract reference](#21-appendix-adapter-contract-reference)

> **Short on time?** Read Part I (§1–§4) for the philosophy and Part V §15 for the failure modes. That's the 80% — enough to decide whether the pattern fits and to sketch out a first design. The rest is reference material for when you actually start building.
>
> **Navigation by purpose:** Part I is decision-making; Part II is "how the framework reads from adapters at runtime"; Part III is the frontend boundary; Part IV is how new brands enter the system; Part V is how the system stays healthy in production; Part VI is the contract spec and scaffold.

---

## 1. When to use this pattern

This pattern is right when **both** of these are true:

1. **Your integration manages a class of devices with similar shape but different vendors.** A vacuum is a vacuum: it cleans rooms, returns to a dock, reports a state. The shape doesn't change between Eufy, Roborock, Dreame, Narwal. What changes is which entity reports `task_status`, what value that entity uses to mean "completed", and what the wire payload looks like when you tell it to start cleaning.
2. **You either already have multiple vendors to support, or you reasonably expect to.** "Reasonably expect" is doing real work in that sentence — see the single-vendor question below.

The pattern also pays off most when **the vendor surface is non-trivial** — dozens of brand-specific facts (entity roles, vocabularies, completion sentinels, dispatch shapes, maintenance components) rather than three constants. For tiny surfaces you can implement the pattern shape but the discipline overhead won't pay back.

If you're building on top of someone else's existing HA integration (e.g. a meta-integration that consumes device entities provided by other custom components), this pattern is doubly useful: the underlying integrations have already done the brand-specific protocol work, and your adapter config just maps roles ("which entity reports task status?") onto the entities they've already created.

### The single-vendor question

This is where most readers end up stuck, so address it head-on.

**Don't use the pattern if** you're building for one vendor and you can honestly say you'll never expand. A truly one-off integration is faster to ship without the indirection layer; the abstraction adds friction the project never pays back.

**Do use the pattern if** you're single-vendor *for now* but can see plausible second and third vendors on the horizon — or you're not sure either way. The refactor cost of introducing the pattern *after* the codebase has accreted brand-specific assumptions is far higher than the cost of building with it from day one. The adapter dict feels silly when there's one entry; it feels like clarity when there are three.

The honest test: imagine writing the second adapter. If your gut says "yeah, that'd be straightforward, I can see exactly where each brand fact would land," you're already there. If your gut says "I'd never do that, this is one brand forever," skip the pattern and don't apologize.

### When *not* to use it (other reasons)

- **Devices where the brands are *too* different to share a code path** — if the conceptual model itself diverges (vacuum vs. lawn-mower vs. pool-cleaner), an adapter dict won't paper over the differences. You probably need separate integrations.
- **When the vendor differences are entirely in *protocol* (not in *shape*)** — if the only difference is how bytes go over the wire and everything above that is identical, that's a job for a protocol abstraction layer (something like a base class with overridable transport methods), not a config-dict registry.

---

## 2. The mental model

Picture three layers:

```
┌─────────────────────────────────────────────────────────────┐
│  Framework code (90% of the integration)                    │
│  Knows: there are vacuums, they have rooms, jobs have       │
│         states, payloads get dispatched, sensors update     │
│  Does NOT know: any brand-specific name, vocabulary, value  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │  reads from
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Adapter registry                                           │
│  In-memory: { entity_id: adapter_config_dict }              │
│  Lookup: get_adapter_config(vacuum_entity_id) -> dict       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │  populated by
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Adapters (one per brand/model variant)                     │
│  - Code adapters: shipped with the integration              │
│  - Stored adapters: user-built via discovery UI             │
│  Each is just: a config dict + a register() function        │
└─────────────────────────────────────────────────────────────┘
```

The framework code is the bulk of the integration — the state machine, queue engine, service handlers, entity platforms, storage, event bus, background watchers, panel. None of it references a brand-specific name, value, or string. When it needs a brand-specific fact ("which entity reports dock status?", "what value means 'washing the mop'?", "what shape is the start-cleaning payload?"), it looks it up in the adapter registry.

The registry is a flat in-memory dict keyed by the device's primary entity ID. One lookup function, `get_adapter_config(entity_id)`, fronts it. Populated at integration setup; updated when the user changes which adapter applies.

The adapters themselves are plain config dicts — one per device variant. The integration ships some; the user can build more via a discovery wizard (§13). Each is a leaf, not a class hierarchy. It declares what it knows. The framework reads what it declared.

The contract is **runtime lookup, not import-time dispatch**. The framework never says "if this is the Eufy adapter, do X" — it asks "what does the adapter tell me to do here?". A user-built stored adapter and a code-shipped adapter get identical treatment; they're both just dicts in the registry.

---

## Core Rules of the Pattern

Six rules. The rest of the guide is implementation detail for living these out.

1. **Framework code never references brand-specific strings.** No literal vendor entity names, no literal vendor values. If you're typing one in framework code, you've found a missing adapter field.
2. **Adapters are data, not behavior.** Plain dicts. No callables. No subclasses. No imports. Behavior lives in the framework's *named-builder* registries; the adapter selects by name (§7, §15).
3. **Framework logic resolves by role, never by entity naming convention.** "What entity reports task_status for this device?" is a registry lookup, not a `f"sensor.{object_id}_task_status"` (§5).
4. **Capabilities are explicit declarations, not inferred state.** Boolean flags in the adapter config, default-off (§8). Don't probe entity presence to guess what a device supports.
5. **Stored adapters must remain JSON-serializable.** This is what makes user-built adapters work without a code release (§13). Every rule above is partly in service of this one.
6. **User preferences are separate from adapter declarations.** "What does this device do?" vs. "how does this user want their install to behave?" — two different config layers (§19).

These get referenced throughout the rest of the guide as "Rule #1" through "Rule #6." When a design decision feels arbitrary, check it against the rules.

---

## 3. Designing the adapter config

The adapter config is the contract between the framework and the brand. Get this shape right and everything else falls into place; get it wrong and you'll bleed brand assumptions into framework code anyway.

A useful starting point is to group facts by what they describe. Here's a worked example for a vacuum integration:

```python
EUFY_X10_PRO_OMNI_ADAPTER = {
    # 1. IDENTITY — who am I?
    "adapter_name": "eufy_x10_pro_omni",
    "display_name": "Eufy X10 Pro Omni",
    "model_codes": ["T2351"],                 # vendor model codes this adapter handles
    "vacuum_entity_id": "vacuum.alfred",      # the primary HA entity

    # 2. ENTITIES — what HA entities does this device expose?
    # Keys are *roles*, values are *entity IDs*.
    # Framework code looks up roles, never entity IDs.
    "entities": {
        "task_status":             "sensor.alfred_task_status",
        "dock_status":             "sensor.alfred_dock_status",
        "active_map":              "sensor.alfred_active_map",
        "active_cleaning_target":  "sensor.alfred_active_cleaning_target",
        "battery_level":           "sensor.alfred_battery_level",
        "robot_position_x":        "sensor.alfred_position_x",
        "robot_position_y":        "sensor.alfred_position_y",
        "dry_duration":            "select.alfred_dry_duration",
        # ...
    },

    # 3. VOCABULARY — what string values does the device use?
    # Framework code matches against these, never against literal strings.
    "completion": {
        "task_status_value":         "completed",   # what task_status equals when done
        "cleared_target_sentinels":  {"", "unknown", "unavailable", "none", "null"},
    },

    # 4. DROPDOWN OPTIONS — what choices does the user get?
    # Card and entity platforms read these to populate UI.
    "dispatch": {
        "clean_mode_options":      ["Vacuum", "Vacuum and mop", "Mop"],
        "fan_speed_options":       ["Quiet", "Standard", "Turbo", "Max"],
        "water_level_options":     ["Off", "Low", "Medium", "High"],
        "clean_intensity_options": ["Quick", "Standard", "Deep", "Narrow"],
        "path_type_options":       ["None", "Y", "S"],
        "clean_passes_options":    [1, 2, 3],
    },

    # 5. WIRE FORMAT — what does the payload look like going to the device?
    # Field names and types vary per brand even when meaning is identical.
    "dispatch_payload": {
        "map_id_field":     "map_id",
        "map_id_type":      "int",
        "rooms_field":      "rooms",
        "room_id_field":    "id",
        "passes_field":     "clean_times",
        # ...
    },

    # 6. MAINTENANCE COMPONENTS — what consumables does the device have?
    # Drives which maintenance reset buttons and sensors appear.
    "maintenance_components": {
        "rolling_brush": {
            "label":          "Rolling Brush",
            "interval_hours": 300,
            "source_entity":  "sensor.alfred_rolling_brush_usage",
        },
        "side_brush": { ... },
        "filter":     { ... },
        # ...
    },

    # 7. EVENT TRIGGERS — what state values fire which integration events?
    "dock_events": {
        "triggers": {
            "last_mop_wash":   {"washing", "washing mop"},
            "last_dust_empty": {"emptying dust", "emptying dust bin"},
            "last_dry_start":  {"drying", "drying mop"},
        },
    },

    # 8. CAPABILITIES — what features does this model support?
    # Hard flags. Cheap to evaluate. Drive feature gating.
    "capabilities": {
        "supports_mop":            True,
        "supports_dock_washing":   True,
        "supports_path_control":   True,
        "supports_edge_mopping":   True,
        "supports_passes":         True,
        "supports_room_priority":  False,
    },
}
```

Several design rules emerge from this shape:

### Rule 1: Roles are framework-defined, values are adapter-defined

The framework decides that "task_status" is a meaningful role — every adapter must be able to declare which entity fills it (or explicitly declare that the role is unsupported). The framework does *not* decide that the entity must be called `sensor.<vacuum_name>_task_status` — that's the adapter's business.

When you're tempted to write framework code like `f"sensor.{object_id}_task_status"`, stop. Add the role to your adapter contract and look it up at runtime.

### Rule 2: Match by role + value, never by literal

When the framework needs to know "did the device just finish cleaning?", the code looks like this:

```python
state = hass.states.get(adapter["entities"]["task_status"])
if state and state.state == adapter["completion"]["task_status_value"]:
    # Cleaning finished
```

Not like this:

```python
state = hass.states.get(f"sensor.{vacuum_object_id}_task_status")
if state and state.state == "completed":   # ← literal
    # Cleaning finished
```

Even when *every* brand happens to use the string `"completed"`, the literal in framework code is a bug waiting to happen. The first brand that ships with `"finished"` or `"done"` or `"task_completed"` will break the world. Adapter-declared values cost nothing and prevent the trap.

### Rule 3: Dropdown options as ordered lists, not sets

Order matters in the UI. The first option in `fan_speed_options` is the default. Use lists, not sets. Document the convention.

### Rule 4: Sentinel sets as actual sets

For "this value means the thing is cleared/unavailable" lookups, use Python sets so membership tests are O(1) and obviously unordered:

```python
"cleared_target_sentinels": {"", "unknown", "unavailable", "none", "null"},
```

### Rule 5: One adapter dict per *device variant*, not per *brand*

If two models within the same brand have different capabilities or vocabularies, they get separate adapter dicts. The Eufy X10 Pro Omni and the Eufy RoboVac G30 are not the same device just because they're both Eufy. Adapters are cheap; copy-paste-and-edit is fine.

The `display_name` and `model_codes` fields exist so the discovery UI can pick the right adapter automatically when the user adds a device.

#### Composition for near-identical variants

Copy-paste works fine for two or three variants. Eighteen near-identical Roborock models generates noise — a fix to one shared default has to be made eighteen times.

The pattern that scales: **deepcopy the base, then assign changes directly.** Not inheritance, not class hierarchies, not dict-spread merging.

```python
import copy

BASE_ROBOROCK = {
    "schema_version":   3,
    "entities":         { ... },
    "completion":       { ... },
    "dispatch":         { ... },
    "dispatch_payload": {
        "fields": {"map_id": {"name": "map_id", "type": "int"}, ...},
    },
    "capabilities": {
        "supports_mop":           True,
        "supports_hot_water_wash": False,
    },
}

QREVO_MAXV = copy.deepcopy(BASE_ROBOROCK)
QREVO_MAXV["adapter_name"] = "roborock_qrevo_maxv"
QREVO_MAXV["display_name"] = "Roborock Qrevo MaxV"
QREVO_MAXV["model_codes"]  = ["roborock.vacuum.a87"]
QREVO_MAXV["capabilities"]["supports_hot_water_wash"] = True   # safe: deep copy

S8_PRO_ULTRA = copy.deepcopy(BASE_ROBOROCK)
S8_PRO_ULTRA["adapter_name"] = "roborock_s8_pro_ultra"
S8_PRO_ULTRA["display_name"] = "Roborock S8 Pro Ultra"
S8_PRO_ULTRA["model_codes"]  = ["roborock.vacuum.a70"]
```

The key insight: **dict spread (`{**base, ...}`) is shallow and silently shares nested state.** If you write `QREVO_MAXV = {**BASE_ROBOROCK, ...}`, then `QREVO_MAXV["dispatch_payload"]["fields"]` is *the same dict object* as `BASE_ROBOROCK["dispatch_payload"]["fields"]`. A later mutation on either propagates to the other plus every other variant derived from the base. Silent cross-variant pollution.

`copy.deepcopy(base)` produces structurally independent dicts. Mutations stay local. Two rules to keep it that way:

1. **One `copy.deepcopy(base)` per variant, then direct key assignment.** Never spread.
2. **Don't inherit at runtime.** Variants resolve to flat dicts at import time. Don't store `"_inherits": "BASE"` and resolve later — that breaks Rule #5 (stored adapters must remain JSON-serializable).

When a user-built stored adapter is exported to JSON, what gets serialized is the resolved flat dict — composition is a maintenance convenience for code adapters, invisible to storage and the registry.

### Rule 6: Capabilities are boolean flags, not strings

`supports_mop: True` not `mop_support: "yes"` or `mop_support: "full"`. If you genuinely need three states (supported / unsupported / partial), use an enum-like string constant set, but evaluate the design — usually three states means you have two capabilities, not one.

### Rule 7: Nothing in the adapter dict references the framework

The adapter is data. It doesn't import framework modules, it doesn't subclass anything, it doesn't carry behavior. This makes it trivially serializable, which matters for the stored-adapter use case (§13).

---

## 4. The adapter registry

The registry is the mapping from device entity ID to adapter config. **Scope it per config entry, under `hass.data[DOMAIN][entry.entry_id]`.** This is the only safe choice for HA integrations; the rest of this section explains why, then shows the recommended access pattern that keeps call sites clean despite the per-entry scoping.

### The coordinator pattern (recommended)

The cleanest way to expose the registry to the rest of the integration is through a per-entry coordinator object that owns the registry as part of its state. The coordinator is constructed in `async_setup_entry`, stashed under the entry's `hass.data` slot, and pulled out by anything that needs to look up adapter config:

```python
# adapters/coordinator.py

from __future__ import annotations
from typing import Any
from homeassistant.core import HomeAssistant
from homeassistant.config_entries import ConfigEntry


class AdapterCoordinator:
    """Per-entry coordinator owning the adapter registry and its lookups."""

    def __init__(self, hass: HomeAssistant, entry: ConfigEntry) -> None:
        self.hass = hass
        self.entry = entry
        self._registry: dict[str, dict[str, Any]] = {}

    def register_adapter_config(self, entity_id: str, config: dict[str, Any]) -> None:
        self._registry[entity_id] = dict(config)

    def unregister_adapter_config(self, entity_id: str) -> None:
        self._registry.pop(entity_id, None)

    def get_adapter_config(self, entity_id: str) -> dict[str, Any] | None:
        return self._registry.get(entity_id)

    def get_adapter_value(
        self, entity_id: str, *path: str, fallback: Any = None,
    ) -> Any:
        node: Any = self._registry.get(entity_id) or {}
        for key in path:
            if not isinstance(node, dict):
                return fallback
            node = node.get(key)
            if node is None:
                return fallback
        return node

    def all_registered_entity_ids(self) -> list[str]:
        return list(self._registry)
```

```python
# __init__.py — wiring

from .adapters.coordinator import AdapterCoordinator


async def async_setup_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    hass.data.setdefault(DOMAIN, {})
    coordinator = AdapterCoordinator(hass, entry)
    hass.data[DOMAIN][entry.entry_id] = {"coordinator": coordinator}
    # ... register code adapters and load stored adapters via the coordinator ...
    return True


async def async_unload_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    # Tearing down the slot drops the coordinator and its registry.
    hass.data.get(DOMAIN, {}).pop(entry.entry_id, None)
    return True
```

Every call site that needs adapter lookups grabs the coordinator from `hass.data`:

```python
coordinator = hass.data[DOMAIN][entry.entry_id]["coordinator"]
adapter = coordinator.get_adapter_config(entity_id)
fan_speeds = coordinator.get_adapter_value(
    entity_id, "dispatch", "fan_speed_options", fallback=[]
)
```

For long-lived objects (your manager, your tracker, your service handlers), pass the coordinator in at construction so they don't need to do the `hass.data` walk every call. The coordinator is the unit of per-entry isolation; everything downstream just talks to it.

The rest of this guide shows calls in the shorter form `get_adapter_config(entity_id)` for readability — substitute `coordinator.get_adapter_config(entity_id)` wherever it appears.

### Why per-entry scoping is mandatory

- **Multi-instance isolation.** A user sets up two config entries — primary home and guest house, two cloud accounts, two physical bridges. Each entry's registry lives in its own coordinator under its own `hass.data` slot. No cross-contamination. The lookup `coordinator.get_adapter_config(entity_id)` is unambiguous because the coordinator *is* the namespace.
- **Reload safety.** When `async_unload_entry` runs, popping the entry's `hass.data` slot drops the coordinator (and its registry) along with everything else. Even if `async_unload_entry` raises midway through teardown, there's no module-level state for the next setup to inherit from. The next `async_setup_entry` constructs a fresh coordinator and starts from zero.
- **HA convention.** Anyone reading HA core code expects per-entry state. Following the convention reduces the cognitive cost for reviewers and contributors.
- **Test isolation.** Pytest fixtures already reset `hass.data` between tests via the standard HA test scaffolding. Tests construct a fresh coordinator per test without any module-state-clearing dance.

### A note on module-level state

Module-level registries are tempting for real reasons. The lookup code is trivial — `get_adapter_value(entity_id, "entities", "task_status")` reads like a free function with no plumbing. No coordinator reference to pass through the manager, the tracker, the service handlers, the entity platforms, the listeners. No "where do I get the coordinator from?" question at every layer. For a fresh integration with one config entry, module state is *visibly* less code, and that simplicity is genuine.

The coordinator pattern accepts a small amount of wiring complexity — every long-lived object holds a coordinator reference, every short-lived call site fetches the coordinator from `hass.data` once — in exchange for HA lifecycle correctness. That's a real tradeoff, not a strawman.

**The reason to make the trade anyway:** the failure modes of module-level state are silent and asymmetric. The cost shows up only under conditions you don't usually test for:

- If anyone later removes the single-instance guard, two config entries silently share a registry. The bug shows up as "this device is reading another device's settings" — months after the change.
- If `async_unload_entry` raises partway through, stale entries survive into the next setup. Diagnosis requires guessing that a hot-reload between two specific actions caused it.
- Tests that don't religiously `_REGISTRY.clear()` in `autouse=True` fixtures leak state across tests. The leak shows up as "this test passes alone, fails in CI."

The coordinator pattern eliminates all three by construction. The wiring complexity it adds is concrete and visible up front; the bugs it prevents are silent and visible only in production. That's the real shape of the tradeoff, and that's why the recommendation flips.

(The eufy-vacuum-manager project that this guide grew out of *originally* used module-level state, with `async_set_unique_id(DOMAIN)` enforcing single-instance and disciplined manual unregistration. It worked, but the success was despite the choice, not because of it — and the project subsequently migrated to a coordinator using the deprecation-shim technique below. If you're starting fresh, skip the detour.)

### Migrating from module-level state to a coordinator

If you've already shipped with module-level state — like the eufy-vacuum-manager project did — the migration is far less risky than "rewrite every call site" suggests. The honest hand-wave "the swap is mechanical" hides the fact that a real codebase has ~50–100 call sites across ~20 files reaching for `get_adapter_config(entity_id)` directly. Coordinating all of them in one PR is the easy way to introduce subtle bugs in rarely-exercised code paths.

The technique that actually works: **keep the bare module-level functions as deprecation shims that route to the active coordinator.** The migration becomes structural (storage moves to per-entry instance state, unload becomes safe) without requiring any call site to change until you want to.

#### The shim shape

```python
# adapters/registry.py — migration-window state

# Fallback storage, used only when no coordinator is active
# (tests, pre-setup paths). Empty in normal operation.
_REGISTRY: dict[str, dict[str, Any]] = {}

# Module-level pointer to the active coordinator. Set by the
# coordinator's constructor, cleared by its shutdown(). The
# single-instance guard means there's never more than one alive.
_active_coordinator: AdapterCoordinator | None = None


class AdapterCoordinator:
    def __init__(self, hass, entry):
        self.hass = hass
        self.entry = entry
        self._registry: dict[str, dict[str, Any]] = {}
        global _active_coordinator
        _active_coordinator = self

    def shutdown(self):
        global _active_coordinator
        if _active_coordinator is self:
            _active_coordinator = None
        self._registry.clear()

    def get_adapter_config(self, entity_id):
        return self._registry.get(entity_id)
    # ... other methods unchanged from a from-scratch design ...


# Bare-function shims route to the active coordinator. Existing
# call sites continue to work without modification.
def get_adapter_config(entity_id):
    if _active_coordinator is not None:
        return _active_coordinator.get_adapter_config(entity_id)
    return _REGISTRY.get(entity_id)

def register_adapter_config(entity_id, config):
    if _active_coordinator is not None:
        _active_coordinator.register_adapter_config(entity_id, config)
        return
    _REGISTRY[entity_id] = config

# ... same shape for the other functions ...
```

Construct the coordinator in `async_setup_entry`, drop it in `async_unload_entry`. The shims handle every existing call site transparently.

#### What the shims buy you

- **Storage is per-entry from the day you ship Stage 1.** All three silent failure modes above are eliminated. The coordinator instance owns the registry; unload drops the whole instance; no module-level dict accumulates state across reloads.
- **Zero call-site churn risk.** The ~50–100 sites still calling `get_adapter_config(entity_id)` work exactly as before. They route through the shim, hit the active coordinator, get the same answer.
- **Migration becomes optional.** You can convert call sites to `coordinator.get_adapter_config(...)` explicitly later, file-by-file, on your schedule. Or never — the shims aren't technical debt, they're a stable abstraction layer.

#### What "Stage 1" lands

A single commit delivers:

1. The `AdapterCoordinator` class definition.
2. The module-level `_active_coordinator` pointer and a `_set_active_coordinator` helper.
3. The bare-function shims (replacing whatever the old bare functions did with routing logic).
4. Construction in `async_setup_entry`, stashed under `hass.data[DOMAIN][DATA_ADAPTER_COORDINATOR]` (or the equivalent per-entry slot if you're really running multi-instance).
5. Teardown in `async_unload_entry`, wrapped in `try/finally` so the coordinator always detaches even if other unload steps raise.

Behavior change for users: none. The integration loads, services respond, entity platforms render — all identical. The only observable difference is in test runs and reload behavior, where stale state can no longer leak.

#### Subsequent stages are cosmetic

Stages 2–4 of a "full" migration would be:

- **Stage 2** — Pass the coordinator explicitly to long-lived objects (manager, tracker, etc.) at construction. They hold a reference and call `self.coordinator.get_adapter_config(...)` instead of the module-level function.
- **Stage 3** — Migrate short-lived call sites (service handlers, listeners) to fetch the coordinator from `hass.data` and use it directly.
- **Stage 4** — Delete the bare-function shims and the `_active_coordinator` pointer.

These are pure idiomatic cleanups. They make the dependency explicit at every call site. They don't fix any bug, prevent any failure mode, or change any behavior. **For a project that's already shipped, stages 2–4 are aesthetic preference, not architectural necessity.** Stages 2–3 can be done in small file-scoped PRs that are individually reviewable; Stage 4 can ship whenever the last bare-function caller has been converted. Or never.

#### The trap to avoid

Don't try to do Stages 1–4 in a single PR. The temptation is real because "while we're rewriting the storage layer, we may as well clean up the call sites." But the call-site churn is where bugs hide — typos in renamed signatures, missed sites in obscure code paths, helpers that aliased the original function name and don't get caught by find-replace. Stage 1 alone delivers the structural win at near-zero risk; the rest is a slow burn the project can do at leisure.

The eufy-vacuum-manager project executed exactly this staging: Stage 1 landed in a single commit, integration loaded cleanly on first restart, smoke tests passed, no call sites needed updating. The remaining stages were explicitly deferred as low-value cleanup.

### Lifecycle

The registry is populated during `async_setup_entry`, immediately after the coordinator is constructed. Two sources:

1. **Code adapters** — Python modules shipped with the integration. Each has a `register_<name>_adapter_for_device(coordinator, entity_id)` function that builds the config dict (using e.g. the device's object_id to fill in entity IDs) and calls `coordinator.register_adapter_config(entity_id, config)`. Imported from `adapters/<brand>/` packages and called for any device the user has assigned to that adapter.
2. **Stored adapters** — user-built configs persisted to integration storage. A helper like `load_stored_adapter_configs(coordinator, manager)` reads them on startup and registers each via `coordinator.register_adapter_config`. This is what makes the integration extensible without a code release.

On `async_unload_entry`, popping the entry's `hass.data` slot drops the coordinator and the registry along with it. No explicit unregistration loop required.

---

## 5. Entity resolution by role

This is the most common adapter operation. The framework needs to know "what HA entity plays role X for this device?" and the answer comes from the adapter config.

The naive pattern is direct lookup at every call site:

```python
adapter = get_adapter_config(entity_id) or {}
task_status_entity = adapter.get("entities", {}).get("task_status")
if task_status_entity:
    state = hass.states.get(task_status_entity)
    # ...
```

That works but it's verbose, and forgetting the `or {}` or the existence check is an easy source of `AttributeError: 'NoneType' object has no attribute 'get'` bugs.

Hang the helpers off the coordinator so they share the per-entry scope:

```python
class AdapterCoordinator:
    # ... (registry methods from §4) ...

    def get_role_entity_id(self, entity_id: str, role: str) -> str | None:
        """Return the HA entity ID that plays `role` for the given device."""
        return self.get_adapter_value(entity_id, "entities", role, fallback=None)

    def get_role_state(self, entity_id: str, role: str):
        """Return the State of the entity playing `role`, or None if absent."""
        role_entity = self.get_role_entity_id(entity_id, role)
        if not role_entity:
            return None
        return self.hass.states.get(role_entity)
```

Now call sites look like:

```python
state = coordinator.get_role_state(vacuum_entity_id, "task_status")
if state and state.state == coordinator.get_adapter_value(
    vacuum_entity_id, "completion", "task_status_value"
):
    # Cleaning finished
```

That's the actual idiom used throughout the framework. Direct registry access is reserved for code that needs the full adapter config (e.g. iterating all maintenance components).

### Handling missing entities

`hass.states.get(...)` returns `None` in more cases than you'd think:

- The adapter declares a role but the underlying entity has been renamed or removed.
- HA is still booting; the upstream integration hasn't populated the entity yet.
- The entity exists but is in `"unknown"` or `"unavailable"` during a vendor-integration reconnection.
- A rapid state transition catches the entity between writes.

All four are normal. The framework should not crash on any of them; it should treat the absent or sentinel-valued state as "unknown" and gracefully degrade.

A clean pattern: every "what's the current state of role X?" helper returns `None` when the entity is missing, unavailable, or sentinel-valued, and callers branch on `None` to mean "unknown".

```python
def get_role_state_value(self, entity_id: str, role: str) -> str | None:
    """Return the current state value of the entity playing `role`,
    or None if the entity is missing or sentinel-valued."""
    state = self.get_role_state(entity_id, role)
    if state is None:
        return None
    value = state.state
    if value in ("unknown", "unavailable", "", None):
        return None
    return str(value)
```

This keeps every consumer of role state on the same "None means unknown" contract.

**Don't build a separate cache layer.** A tempting wrong fix when cold-boot None values show up is to wrap `hass.states.get` with a framework-owned cache fed by `async_track_state_change_event`. Don't. HA's state machine *is* the cache — it's already updated by reactive listeners under the hood. A second cache in front of it gives you two sources of truth that can drift, plus subscription bookkeeping per role entity. The graceful-degradation pattern above handles the same scenarios with less infrastructure: callers see `None` for a moment during boot, treat it as "unknown," and re-read on the next state-change event (which §9 already wires up for the cases that actually need reactive behavior).

Reach for `async_track_state_change_event` when you need to *react* to a state change (fire an event, update a sensor, advance a state machine). Use `coordinator.get_role_state_value(...)` when you need to *read* current state. Don't mix the two.

---

## 6. Vocabulary and dropdown options

Anywhere your code shows the user a choice — a dropdown, a list of selectable modes, a set of named profiles — read the choices from the adapter config.

A typical mistake: hardcoding a fan-speed list in a `SelectEntity` because the platform setup is "obviously" device-specific. Then someone wants to add a brand with `["silent", "balanced", "powerful"]` and you discover the list is referenced from three platforms.

The right pattern:

```python
# select.py — platform setup

async def async_setup_entry(hass, entry, async_add_entities):
    manager = hass.data[DOMAIN][DATA_RUNTIME]
    for vacuum_entity_id, vacuum_data in manager.get_all_vacuums():
        adapter = get_adapter_config(vacuum_entity_id) or {}
        options = adapter.get("dispatch", {}).get("fan_speed_options", [])
        if not options:
            continue
        async_add_entities([
            FanSpeedSelect(hass, vacuum_entity_id, options=options),
        ])
```

The select entity stores its options list at construction time and exposes it through `_attr_options`. The framework doesn't know what "Max" means; it just knows that "Max" is a legal fan-speed value for this device and the user can pick it.

### Label vs. value

Often the user-facing label differs from the wire-format value. The Eufy adapter ships `clean_mode_options: ["Vacuum", "Vacuum and mop", "Mop"]` because those are the strings the user sees in the dropdown *and* the strings sent to the device. But what if a brand sends `"VAC"`/`"VAC_MOP"`/`"MOP"` on the wire and you want to show pretty labels?

Two options:

1. **Adapter declares both, framework picks based on context.** Add a parallel `fan_speed_labels: {"low": "Low", ...}` and have the select entity show labels while sending values.
2. **Adapter declares one canonical list, framework label-cases for display.** Cheap but assumes uniform conversion.

Option 1 is more flexible; option 2 is enough for many real cases. Pick whichever fits, but make it part of the adapter contract — never put the conversion logic in framework code with brand-specific branches.

### Dropdown order is part of the adapter contract

The first option in `fan_speed_options` is treated as the default whenever the framework needs a fallback. Document this in the adapter config reference and don't reorder lists without realizing what you're changing.

---

## 7. Dispatch and wire-format payloads

This is where the adapter pattern earns its keep. Brand A's "start cleaning" payload might be:

```python
{
    "map_id": 6,
    "rooms": [
        {"id": 5, "clean_times": 2, "fan_speed": "Max", "clean_mode": "Vacuum"},
    ],
}
```

Brand B's might be:

```python
{
    "map_id": "6",
    "room_ids": [5],
    "settings": {
        "5": {"passes": 2, "suction": "high", "mode": "vacuum_only"},
    },
}
```

These are conceptually identical. The framework's queue engine doesn't care which shape goes over the wire. The adapter does.

Two complementary patterns:

### Pattern A: Field-name and field-type declarations

For simple shape differences (different field names, different type coercions), declare the mapping:

```python
"dispatch_payload": {
    "map_id_field":   "map_id",     # vs. "mapId" or "map" or omitted
    "map_id_type":    "int",        # vs. "str"
    "rooms_field":    "rooms",
    "room_id_field":  "id",
    "passes_field":   "clean_times",
    "fan_speed_field": "fan_speed",
    "clean_mode_field": "clean_mode",
}
```

Then the framework's payload builder reads these:

```python
def build_dispatch_payload(adapter, map_id, rooms):
    p = adapter["dispatch_payload"]
    map_id_value = int(map_id) if p["map_id_type"] == "int" else str(map_id)
    return {
        p["map_id_field"]: map_id_value,
        p["rooms_field"]: [
            {
                p["room_id_field"]: room["id"],
                p["passes_field"]: room["passes"],
                p["fan_speed_field"]: room["fan_speed"],
                p["clean_mode_field"]: room["clean_mode"],
            }
            for room in rooms
        ],
    }
```

This is enough for most "they renamed the fields but the shape is the same" cases.

### Pattern B: Named-builder indirection

When shapes diverge more than declarative mapping can handle — different nesting depth, conditional fields, vendor-specific transforms — the adapter still doesn't ship a callable. The framework owns a registry of *named* builder functions, and the adapter selects one by name:

```python
# In framework code (dispatch.py)

def _build_brand_b_payload(adapter, map_id, rooms):
    return {
        "map_id": str(map_id),
        "room_ids": [r["id"] for r in rooms],
        "settings": {
            str(r["id"]): {
                "passes": r["passes"],
                "suction": "high" if r["fan_speed"] == "Max" else "low",
                "mode":   adapter["dispatch_payload"]["mode_map"][r["clean_mode"]],
            }
            for r in rooms
        },
    }

_PAYLOAD_BUILDERS = {
    "declarative": _build_declarative_payload,
    "brand_b":     _build_brand_b_payload,
}
```

```python
# In the adapter dict — a string, never a callable

BRAND_B_ADAPTER = {
    # ...
    "dispatch_payload": {
        "builder": "brand_b",
        "mode_map": {"Vacuum": "vacuum_only", "Mop": "mop_only", "Vacuum and mop": "both"},
    },
}
```

At dispatch time:

```python
def build_dispatch_payload(adapter, map_id, rooms):
    name = adapter["dispatch_payload"].get("builder", "declarative")
    builder = _PAYLOAD_BUILDERS[name]
    return builder(adapter, map_id, rooms)
```

Why named builders, not callables in the dict? Two reasons:

1. **Stored adapters stay JSON-serializable.** The `save_adapter_config` service in §13 persists adapter dicts to integration storage as JSON. A callable can't be serialized. A string `"brand_b"` can. This is the same reason §15 calls out callable-in-dict as an anti-pattern.
2. **The framework still controls every code path.** When something goes wrong in dispatch, the bug is in framework code (the builder) — not in a function the adapter author wrote and forgot about. Easier to debug, easier to test, easier to audit.

The cost is one extra mapping step. The benefit is that "the adapter is data" stays true.

### Validating dispatched payloads

You'll want a unit test per adapter that asserts the wire shape. The framework provides a known queue + known room set; the test calls `build_dispatch_payload(adapter, ...)` and compares to a golden output. When someone changes a builder or an adapter's `dispatch_payload` block, the test catches it.

### Validating dispatched payloads

You'll want a unit test per adapter that asserts the wire shape. The framework provides a known queue + known room set; the test calls the adapter's payload builder and compares to a golden output. When someone changes the builder, the test catches it.

---

## 8. Capability gating

Not every device supports every feature. Brand A might not support edge mopping; brand B might not support per-room cleaning passes. The framework needs to know which features to expose for which device.

Capabilities are boolean flags in the adapter config:

```python
"capabilities": {
    "supports_mop":           True,
    "supports_path_control":  False,
    "supports_edge_mopping":  True,
    "supports_passes":        True,
    "supports_room_priority": False,
},
```

The framework reads them at three places:

1. **Entity platform setup** — don't register a `select` entity for `path_type` if `supports_path_control` is False. The user won't see a useless control.
2. **UI rendering** — the card hides the edge-mopping toggle for any vacuum whose adapter has `supports_edge_mopping: False`. (See §12 — capabilities are shipped to the frontend as part of the snapshot payload.)
3. **Service validation** — a `set_path_type` service call against a device that doesn't support path control should fail fast with a clear error, not silently no-op.

Three rules:

### Rule 1: Default-off

If an adapter doesn't declare a capability, assume it's unsupported. Adapters opt *in* to features, never out of them. This means a new adapter can be minimal and only declare the features it actually has.

```python
def supports(adapter, capability: str) -> bool:
    return bool(adapter.get("capabilities", {}).get(capability, False))
```

### Rule 2: Capabilities are flat booleans

Not nested, not enums (with rare exceptions). If you find yourself wanting `supports_mop: "partial"`, you probably have two capabilities (`supports_dry_mop`, `supports_wet_mop`) hiding in one flag. Split them.

### Rule 3: Capabilities are *not* the same as "the device has an entity for this"

A device might have a `select.path_type` entity but the adapter author decided not to expose path control in this integration. Capability declarations are an integration-level statement of intent, not a state-machine probe. Keep them in the adapter config, evaluated by the framework, not derived from entity presence.

---

## 9. Background listeners (state-watching by role)

HA integrations almost always need to react to state changes — when the device transitions to "cleaning", when the dock starts washing, when the battery drops below a threshold. The standard pattern is `async_track_state_change_event(hass, entity_ids, callback)`.

The adapter-driven variant: read the entity IDs from the adapter config at registration time, not from hardcoded names.

```python
# __init__.py — background listener setup

def _get_lifecycle_watch_entities(vacuum_entity_id: str) -> list[str]:
    """Return entity IDs to watch for lifecycle state changes."""
    adapter = get_adapter_config(vacuum_entity_id) or {}
    entities = adapter.get("entities", {})
    watch: list[str] = [vacuum_entity_id]  # always include the primary entity
    for role in ("task_status", "dock_status", "active_cleaning_target", "active_map"):
        eid = entities.get(role)
        if eid:
            watch.append(eid)
    return watch


def _register_lifecycle_listeners(hass: HomeAssistant) -> None:
    manager = hass.data[DOMAIN][DATA_RUNTIME]
    watched: dict[str, str] = {}   # entity_id -> vacuum_entity_id
    for vacuum_entity_id in manager.get_all_vacuum_entity_ids():
        for eid in _get_lifecycle_watch_entities(vacuum_entity_id):
            watched[eid] = vacuum_entity_id
    if not watched:
        return

    @callback
    def _on_state_change(event: Event) -> None:
        entity_id = event.data.get("entity_id")
        vacuum_entity_id = watched.get(entity_id)
        if vacuum_entity_id is None:
            return
        # delegate to manager
        manager.on_lifecycle_state_change(vacuum_entity_id, event)

    unsub = async_track_state_change_event(hass, list(watched), _on_state_change)
    hass.data[DOMAIN].setdefault("_lifecycle_unsubs", []).append(unsub)
```

Key idea: the framework builds the watch list at startup by *asking each adapter* what to watch, then maintains a single `entity_id → vacuum_entity_id` map so the callback can dispatch back to the right device. No hardcoded entity name appears anywhere in the framework code path.

When the adapter doesn't declare an entity for a particular role, that role is simply not watched. The framework's downstream logic should handle the absence gracefully (typically by treating the unwatched signal as "unknown" or "always-true depending on default semantics).

### State-value triggers

Some listeners care not just *that* a state changed but *what value* it changed to. Dock events are a good example: you want to record a `last_mop_wash` event when `dock_status` transitions to any of several adapter-declared strings.

```python
@callback
def _on_dock_status(event: Event) -> None:
    new_value = (event.data.get("new_state") or _SENTINEL).state.strip().lower()
    if new_value in {"unknown", "unavailable", ""}:
        return

    vacuum_entity_id = watched.get(event.data.get("entity_id"))
    if vacuum_entity_id is None:
        return

    triggers = get_adapter_value(
        vacuum_entity_id, "dock_events", "triggers", fallback={}
    )

    for event_type, trigger_values in triggers.items():
        normalized = {str(v).strip().lower() for v in trigger_values}
        if new_value in normalized:
            manager.record_dock_event(vacuum_entity_id, event_type)
            break   # don't double-fire for overlapping trigger sets
```

The framework iterates whatever triggers the adapter declared. New adapter? Just declare a `dock_events.triggers` dict; no code change.

---

## 10. Storage decoupling

The integration's persisted state (`.storage/<domain>.<key>`) needs to be readable across adapters. A user might switch from one adapter to another — say, after re-importing the device under a renamed entity, or after editing the adapter config. The persisted state should survive.

Two guidelines:

### Guideline 1: Store generic shapes, not brand-specific ones

Bad:

```python
{
    "rooms": {
        "5": {
            "eufy_clean_mode": "Vacuum and mop",   # ← brand-specific key
            "eufy_fan_speed": "Max",
        },
    },
}
```

Good:

```python
{
    "rooms": {
        "5": {
            "clean_mode": "Vacuum and mop",       # generic key
            "fan_speed":  "Max",                  # generic key, brand-specific value
        },
    },
}
```

The key names should be generic, even though the values are brand-vocabulary. When the framework reads `clean_mode: "Vacuum and mop"` and needs to validate it, it looks the value up in the adapter's `dispatch.clean_mode_options` list.

### Known sharp edge: switching adapters on a populated install

Generic keys solve the *shape* problem but not the *vocabulary* problem.

**The scenario:** a user has been running for six months on Adapter A with `clean_mode: "Vacuum and mop"` stored on every room. They switch to Adapter B whose vocabulary is `["vacuum_only", "mop_only", "combined"]`. The stored values are now strings the new adapter doesn't recognize.

This is the largest unsolved sharp edge in the pattern. Be honest about it.

Three strategies, cheapest to most robust:

1. **Fail loud, default soft.** On reading a stored value that isn't in the new adapter's vocabulary, log a warning and substitute the new adapter's default (typically the first option). The user loses their setting but the framework keeps running. Cheapest; least friendly.
2. **Per-adapter vocabulary aliases.** Each adapter declares a `vocabulary_aliases` block mapping known-foreign values to its own canonical form. Framework translates on read. Works for the common case but scales poorly past two or three adapters — each new one would need aliases for all the others.
3. **Store value with originating vocab tag.** Every persisted setting is a `{"value": "Vacuum and mop", "vocab": "brand_a"}` tuple. On read, if the stored `vocab` doesn't match the current adapter, run a migration step (using strategy 1 or 2 internally) and rewrite the stored shape. Most robust; most invasive.

#### Picking the right strategy

The key question is **how often the adapter changes on a populated install**:

- **Adapter set once, never changes** — the common case. Strategy 1 (fail loud, default soft) is the right call. Anything more is over-engineering for a scenario you'll never hit.
- **Adapter occasionally edited via the discovery wizard** — Strategy 1 still works; the edits are usually role-entity remappings (not vocabulary changes), and the rare vocabulary churn can be handled with default substitution.
- **Adapter switching is a normal user flow** — meta-integrations that rebind devices to different underlying brands, multi-tenant deployments, etc. Strategy 3 (originating-vocab tagging) is the durable answer.

> ⚠️ **Don't elevate Strategy 3 to a universal default.** Tagging every persisted field with its originating vocab doubles storage size for every per-room setting, adds a translation step to every read path, and ripples through your entire storage layer. Paying that cost in every install to defend against a scenario most installs never encounter is exactly the kind of premature generalization §1 warns about. The strategies are listed in increasing cost order; pick the cheapest one that solves your project's actual use case.

#### Strategy 3 pseudocode (for the projects that actually need it)

If adapter switching *is* a real user flow in your project, here's the concrete shape:

**Storage shape** — every per-device field that comes from adapter vocabulary becomes a tuple:

```python
# Before
room["clean_mode"] = "Vacuum and mop"

# After
room["clean_mode"] = {
    "value": "Vacuum and mop",
    "vocab": "brand_a",
    "field": "clean_mode",                 # for diagnostics; optional
}
```

**Read path** — every stored-value read goes through a resolver that checks the vocab tag:

```python
def resolve_vocab_value(
    stored: dict | str,
    current_adapter: dict,
    field: str,
) -> str | None:
    """Return the stored value translated to the current adapter's vocabulary,
    or None if no translation is possible (caller falls back to default).
    """
    # Legacy untagged values — assume they were written by the current adapter
    if not isinstance(stored, dict):
        return stored

    stored_value = stored["value"]
    stored_vocab = stored["vocab"]
    current_vocab = current_adapter.get("adapter_name")

    if stored_vocab == current_vocab:
        return stored_value                # no translation needed

    # Cross-adapter — try alias map first
    aliases = current_adapter.get("vocabulary_aliases", {}).get(stored_vocab, {})
    if stored_value in aliases.get(field, {}):
        return aliases[field][stored_value]

    # No translation available — let caller decide (warn + default, or fail)
    return None
```

**Write path** — every write tags the value with the current adapter's vocab:

```python
def tag_vocab_value(value: str, current_adapter: dict, field: str) -> dict:
    return {
        "value": value,
        "vocab": current_adapter["adapter_name"],
        "field": field,
    }
```

**Migration on adapter switch** — when the active adapter changes for a device, optionally walk all stored values for that device and rewrite the vocab tags to match the new adapter (so subsequent reads don't repeatedly translate). This is the most invasive piece; it runs once on switch, then the install pays no per-read cost until the next switch:

```python
async def migrate_stored_values_after_adapter_switch(
    manager, entity_id: str, old_adapter: dict, new_adapter: dict,
) -> None:
    for device_state in manager.iter_device_state(entity_id):
        for field in _VOCAB_FIELDS:          # framework-known list
            stored = device_state.get(field)
            if not isinstance(stored, dict):
                continue
            resolved = resolve_vocab_value(stored, new_adapter, field)
            if resolved is None:
                _LOGGER.warning(
                    "%s/%s: no translation from %s to %s; defaulting",
                    entity_id, field, stored["vocab"], new_adapter["adapter_name"],
                )
                resolved = _default_for(new_adapter, field)
            device_state[field] = tag_vocab_value(resolved, new_adapter, field)
    await manager.async_save()
```

That's the whole pattern. The work is in the resolver, the writer, the alias maps, and the migration walker — each is small, but together they touch most of the storage layer. Hence the warning: this is real architectural cost. Only pay it if you'll actually use it.

#### Why this is the right framing

The pattern doesn't make the vocabulary-drift problem disappear; it makes the problem *legible*. Brand-keyed storage hides the issue until it bites. Generic-keyed storage surfaces it loudly. The strategies above are the well-understood responses — Strategy 1 if the bite is rare, Strategy 3 if it's frequent. The mistake to avoid is paying Strategy 3's price as insurance against a bite most projects never get.

### Guideline 2: Keep adapter config separate from operational state

Don't co-mingle adapter declarations with stored vacuum state. If you let users edit adapter configs at runtime (the stored-adapter path), keep those in a separate top-level storage key:

```python
{
    "vacuums": { ... },           # operational state per device
    "maps":    { ... },
    # ...
    "adapter_configs": {          # user-customized adapter overrides
        "vacuum.alfred": { ... },
    },
}
```

Stored adapters load into the registry on startup with explicit precedence over code adapters. If the user edits the stored adapter, the change applies on next reload (or, if you want hot-reload, fire a registry-replace event).

---

## 11. The service-layer auto-resolve pattern

This one's not strictly part of the adapter pattern but it falls out of it naturally and dramatically improves the public surface.

**The problem:** Many service handlers need to operate on a specific *context* — for a vacuum, that's typically a `map_id`. For an HVAC system, maybe a `zone_id`. For a multi-channel amplifier, maybe an `input_id`. Originally those parameters were required on every service call: callers had to know the right context value and pass it. That's bad UX for automation authors.

**The insight:** Adapters can declare which entity reports "the current context". `entities.active_map` is the entity whose state holds the currently active map ID for a vacuum. If the caller omits `map_id`, the service layer can resolve it via the adapter.

**The pattern:**

```python
# services.py

def _resolved_call_data(hass: HomeAssistant, call: ServiceCall) -> dict:
    """Return call.data with map_id auto-resolved when absent."""
    data = dict(call.data)
    if data.get("map_id"):
        return data
    vacuum_entity_id = data.get("vacuum_entity_id")
    if not vacuum_entity_id:
        return data
    state = get_role_state(hass, vacuum_entity_id, "active_map")
    if state and state.state not in ("unknown", "unavailable", "", None):
        data["map_id"] = state.state
    return data


async def _handle_get_active_job(hass: HomeAssistant, call: ServiceCall) -> dict:
    """Get current active job state."""
    manager = hass.data[DOMAIN][DATA_RUNTIME]
    payload = manager.get_active_job(**_resolved_call_data(hass, call))
    return payload
```

The voluptuous schema lists `map_id` as `vol.Optional("map_id")` rather than required. The `services.yaml` file lists `required: false` with `description: Leave blank to use the current active map.`. Every handler that touches `map_id` dispatches via `_resolved_call_data` instead of `**call.data`.

**Behavior:**

- Caller passes `map_id` explicitly → handler uses it.
- Caller omits `map_id`, adapter declares `entities.active_map`, entity has a usable state → handler resolves and uses it.
- Caller omits `map_id`, adapter doesn't declare the role, or the entity is unavailable → handler dispatches with `map_id` absent, and the manager method's required-kwarg validation raises a clear error. No silent fallback to a wrong context.

**Why this is the right shape:**

- Single helper, one place to maintain the resolution rule.
- No framework code knows the entity name for "active map" — just the role.
- Adapters that don't support auto-resolve don't have to do anything special; explicit `map_id` still works.
- The optional-everywhere surface is consistent: there's no service where some-services-want-it-some-don't.

The same shape works for any "implicit current context" you want to support. HVAC `zone_id` defaulting to the active zone, audio `input_id` defaulting to the selected input, sprinkler `program_id` defaulting to the scheduled program — same pattern, different role name.

---

## 12. Frontend / card integration

If your integration ships a custom Lovelace card, the same adapter-driven discipline applies to the frontend: the card must not know which brand is on the other end. It reads adapter vocabulary from the backend and renders accordingly.

Two integration points:

### Integration point 1: The dashboard snapshot

Wire a service that returns *everything* the card needs to render, including adapter vocabulary:

```python
async def _handle_get_dashboard_snapshot(hass, call) -> dict:
    manager = hass.data[DOMAIN][DATA_RUNTIME]
    vacuum_entity_id = call.data["vacuum_entity_id"]
    adapter = get_adapter_config(vacuum_entity_id) or {}
    return {
        "lifecycle":    manager.get_lifecycle_state(vacuum_entity_id),
        "active_job":   manager.get_active_job(...),
        "rooms":        manager.get_managed_rooms(...),
        # Adapter vocabulary passed through to the card:
        "vocabulary": {
            "clean_mode_options":      adapter.get("dispatch", {}).get("clean_mode_options", []),
            "fan_speed_options":       adapter.get("dispatch", {}).get("fan_speed_options", []),
            "water_level_options":     adapter.get("dispatch", {}).get("water_level_options", []),
            "clean_intensity_options": adapter.get("dispatch", {}).get("clean_intensity_options", []),
        },
        "capabilities": adapter.get("capabilities", {}),
    }
```

The card calls this service on render, reads `vocabulary.*` to populate dropdowns, reads `capabilities.*` to decide which controls to show.

This means the *card* doesn't know which brand it's facing. It just reads the dropdown options for this device and shows them. A new brand with different vocabulary works without a card change.

### Integration point 2: No hardcoded labels in the card

Bad:

```js
// card source
const labels = {
  "Vacuum and mop": "Vacuum + Mop",
  "Vacuum":         "Vacuum",
};
return labels[room.clean_mode] || room.clean_mode;
```

Good:

```js
// backend (per-room snapshot) includes pre-formatted labels:
{
  clean_mode:        "Vacuum and mop",   // wire value
  clean_mode_label:  "Vacuum + Mop",     // pretty label
}
// card just renders room.clean_mode_label
```

The label transform lives in the backend, where it has adapter context. If a brand wants different labels, that's an adapter-config or backend-label-mapping change — not a card edit.

### Integration point 3: Theme tokens, not brand-specific styling

If you ship a theme system, make sure theme tokens are about *what UI element* not *what brand*. `--evcc-color-active-job-glow` is fine. `--evcc-color-eufy-status-line` is not — that locks the theme to the Eufy adapter.

### The boundary rule: backend is the source of truth

This is the single rule the frontend has to live by:

> **The backend owns adapter vocabulary and capabilities. The card displays them. The card never decides what they are.**

Everything else follows:

- **Vocabulary drift.** When an adapter ships a new clean-mode option, the backend snapshot includes it on the next service call. The card renders whatever options it receives. If the card kept its own list (cached or hardcoded), it would drift the moment the backend changed — visible as "the option's there in the backend but the card doesn't show it." Don't cache vocabulary client-side; re-read it on every snapshot.
- **Live capability changes.** A capability can change at runtime (user edits the stored adapter overlay, framework migrates a schema, hardware is rebound). The card must not assume capability flags are stable — pull them from the snapshot and use them at render time, not at component-construction time.
- **Optimistic updates.** Acceptable for *user-initiated* state changes (toggle a room enabled, change clean mode) — the card can render the new value immediately and roll back if the service call fails. Not acceptable for *capability* or *vocabulary* changes — those always wait for backend confirmation. The asymmetry is: state is what the user controls; vocabulary and capabilities are what the device declares, and the card doesn't get to forecast them.

### Snapshot delivery: polling vs. websocket

Two delivery models, neither inherently right:

- **Service call polling** — the card calls `get_dashboard_snapshot` on a timer (every 2–5 seconds during an active job, every 30 seconds otherwise). Simple, cache-friendly, plays well with HA's existing service infrastructure. Latency floor is your polling interval.
- **Websocket subscription** — the card subscribes to an HA event (`eufy_vacuum_dashboard_updated` or similar), and the backend pushes a snapshot whenever relevant state changes. Lower latency, more code on both ends, and you need to be disciplined about *what* triggers a push (or you'll fire a snapshot on every position-sample update and saturate the websocket).

Either works. The pattern doesn't care. Pick polling unless you have a specific reason — most adapters don't generate state at a rate that justifies the push complexity. When you do switch to websocket, the snapshot shape stays exactly the same; only the delivery channel changes.

### What the card is allowed to know

A short list:

- Generic UI state: which view is active, which form fields are dirty, which dropdown is open.
- The current snapshot payload from the backend (vocabulary, capabilities, rooms, active job, lifecycle).
- User-initiated edits *in progress* (the user has typed a new value but hasn't submitted yet).

A short list of what the card is **not** allowed to know:

- Which adapter is loaded (Rule #1 at the frontend boundary).
- Brand-specific entity IDs.
- The factory defaults for any field (those come from the backend if needed).
- The shape of the dispatch wire format.

If you find frontend code doing `if (adapter.name === "...")`, the boundary leaked. The fix is in the backend snapshot — add whatever fact the card was switching on so it can render generically.

---

## 13. Discovery helpers for new brands

The point of the adapter pattern is that a new brand should be a config-only addition. But writing a 200-line config dict from scratch is intimidating. **This is the section that turns the pattern from "good architecture" into "maintainable ecosystem."** Get the discovery flow right and adapter authoring scales beyond your in-house contributors.

The discovery surface is five services, each small, that compose into a complete onboarding flow: scan → score → preview → validate → save → import/export.

### `discover_adapter_entities` — candidate scanning

A service that scans the HA state machine for entities that look like they belong to one device, returning candidates per role. The naive version (substring matching) is the floor; a real implementation does better.

```python
async def _handle_discover_adapter_entities(hass, call):
    """Scan HA for entities that look like they belong to one device."""
    device_entity_id = call.data["device_entity_id"]
    object_id = device_entity_id.split(".", 1)[1]
    candidates = {}
    for role, heuristics in _ROLE_HEURISTICS.items():
        scored = []
        for state in hass.states.async_all():
            score = _score_role_match(state.entity_id, state.attributes, object_id, heuristics)
            if score > 0:
                scored.append({
                    "entity_id":  state.entity_id,
                    "score":      score,
                    "score_reasons": _explain_score(state.entity_id, heuristics),
                })
        scored.sort(key=lambda c: c["score"], reverse=True)
        candidates[role] = scored[:5]
    return {"candidates": candidates}
```

#### Confidence scoring

The result for each role isn't just a list — it's a ranked list with confidence scores. The score combines multiple signals:

- **Object-id prefix match** — `sensor.alfred_task_status` for a vacuum entity `vacuum.alfred` scores high.
- **Role keyword match** — `_task_status` in the entity ID, weighted by how specific the keyword is.
- **Device class / unit hints** — battery-percentage roles match entities with `device_class: battery`. Position roles match entities with `°` or no unit but numeric state.
- **Attribute hints** — many integrations expose role context in attributes (`friendly_name`, `state_class`, `entity_category`). Match against them when they're present.
- **State value hints** — for `task_status`-style roles, prefer entities whose current state looks like a status string (short, lowercase, in a small enum) over entities whose state is a number or timestamp.

A score of 100 means "almost certainly this one"; below 30 means "weak guess, ask the user." The UI uses the score to decide between auto-selecting (high confidence, single candidate dominating) and presenting a chooser.

#### Fuzzy role matching

The substring approach in the naive example breaks the moment a brand uses different terminology. Roborock calls it `state` instead of `task_status`; Dreame uses `cleaning_state`; Narwal uses `work_mode`. A real heuristic table mixes substring with alias sets:

```python
_ROLE_HEURISTICS = {
    "task_status": {
        "keywords":   ["task_status", "task_state", "cleaning_state", "work_mode", "robot_state"],
        "device_classes": {"enum"},
        "value_pattern":  "short_string",      # heuristic flag, not a regex
    },
    "battery_level": {
        "keywords":   ["battery", "battery_level"],
        "device_classes": {"battery"},
        "units":     {"%"},
    },
    # ...
}
```

The heuristics table is data, not code — and it's allowed to grow. Each new brand someone onboards may surface new aliases; check those in.

### `observe_entity_states` — preview before commit

A service that returns current state snapshots for a list of entity IDs (`state`, `last_changed`, useful `attributes`). The discovery UI uses this to show the user "here's what each candidate entity is reporting right now" so they pick the right one with confidence:

```
Candidate for task_status:
  sensor.alfred_task_status     state: "cleaning"      ← currently active
  sensor.alfred_dock_status     state: "docked"
  sensor.alfred_active_target   state: "5"             ← probably wrong
```

This step prevents the most common discovery error: the heuristic suggested `sensor.alfred_active_target` for `task_status` because both contain `task` somewhere, and the user hit "next" without checking. Showing live values makes the mistake obvious.

### Persisted discovery drafts

Discovery is iterative. The user starts a draft, picks a few entities, smoke-tests, fixes one, refines vocabulary, comes back tomorrow. Don't make them start over.

Drafts live alongside committed adapter configs but in a separate storage bucket:

```python
{
    "adapter_configs": { ... },        # committed, in the registry
    "adapter_drafts":  {               # work-in-progress, not yet in the registry
        "vacuum.alfred": {
            "draft_id":     "draft_2026-05-18T14-22",
            "started_at":   "...",
            "last_edited":  "...",
            "config":       { ... partial adapter ... },
            "issues":       [ ... last validation result ... ],
        },
    },
}
```

The wizard works on the draft, runs `_validate_adapter` after every change, and only promotes to `adapter_configs` (and the live registry) on explicit commit. Drafts that haven't been touched in N days are auto-pruned with a warning so storage doesn't grow indefinitely.

### `save_adapter_config` / `delete_adapter_config`

Persist user-committed adapter configs to integration storage. On next reload (or immediately, if you support hot-reload), `load_stored_adapter_configs` reads them and registers them in the registry. The user-built adapter is indistinguishable from a code-shipped one as far as the framework is concerned.

Hot-reload is worth the extra wiring. When the user commits a draft, fire a registry-replace event; the next service call sees the new adapter. Without hot-reload the user has to restart HA to test changes, which kills the iteration loop.

### `export_adapter_config` / `import_adapter_config`

The final piece. A user who's built a working adapter for Brand X shouldn't be the only person who benefits.

- **Export** returns a JSON-safe payload that captures the full adapter config plus metadata (origin user, schema version, validation status):

```python
{
    "schema":         "eufy_vacuum_adapter_export_v1",
    "exported_at":    "2026-05-18T14:22:00Z",
    "framework_min":  3,                     # min schema_version that handles this
    "adapter":        { ... full config ... },
    "discovery_notes": "Roborock Qrevo MaxV — picked task_status from sensor.x, ...",
}
```

- **Import** validates the payload, runs migrations (§17), and either commits directly or stages as a draft for the user to review first.

The exported payload is what gets shared in GitHub issues, Discord screenshots, gist links. Combined with the porting workflow (§14), this is what lets the community add brand support without going through your release cycle.

### Together, these turn discovery into an ecosystem

The pattern up to §12 makes new-brand support *technically possible* — write a dict, register it, you're done. The discovery services in this section make it *operationally scalable*. A user with no Python skills can run a wizard, the wizard scans-scores-previews-validates-saves-exports, and the user posts the resulting JSON for someone else to import. That's the difference between a pattern that requires you to do all the brand work and a pattern that lets your community do most of it.

Together: scan → score → preview → validate → save → import/export. Each service is small. The flow is large.

---

## 14. The porting workflow

What does it look like for someone to add support for a new brand? With the adapter pattern, it's a config edit and a smoke test.

### Step 1: Identify the device variant

The user has a vacuum from Brand X, Model Y. They want to use the integration with it.

### Step 2: Run the discovery flow

The integration's UI offers an "Add new device" wizard:

1. User points the wizard at the brand's existing HA integration (which exposes the underlying device entity and its related entities).
2. Wizard calls `discover_adapter_entities` to scan the HA state machine for likely entity matches.
3. User confirms or corrects each role assignment.
4. Wizard calls `observe_entity_states` on each picked entity and shows current values, helping the user verify they picked the right ones.
5. User fills in vocabulary (clean modes, fan speeds, etc.) — either by hand or by selecting from a presets library.
6. Wizard saves the result via `save_adapter_config`.

### Step 3: Smoke test

The wizard offers a "send a test command" step: starts a small one-room job, watches the state machine, and confirms the lifecycle progressed as expected. If it didn't, the user is shown which role's reading didn't match expectations and prompted to fix the adapter config.

### Step 4: Polish (optional)

If the user wants to upstream the new adapter for everyone, they:

1. Export the adapter config to JSON (the persisted shape is already JSON-friendly because the config is plain data).
2. Open a PR adding it as a code adapter under `adapters/<brand>/` in the integration repo.
3. The PR is reviewable as a config file — the framework changes are zero.

This is the payoff. Brand support becomes a config delivery, not a code delivery. Reviewers can read the config and reason about whether it's correct without spelunking through cross-cutting code edits.

---

## 15. Anti-patterns and pitfalls

A non-exhaustive catalog of mistakes the pattern catches — and mistakes you can still make even when applying the pattern.

### Anti-pattern 1: Brand-named functions

If your framework code has a function called `_eufy_normalize_clean_mode(...)` or `_roborock_payload_builder(...)`, the brand has leaked. Names like that should appear *only* inside `adapters/<brand>/` modules — never in framework code paths.

The framework calls things like `_normalize_clean_mode(adapter, value)` or `_build_dispatch_payload(adapter, ...)`. The adapter passed in tells those functions which brand they're working with, implicitly.

### Anti-pattern 2: Hardcoded literals "just for now"

```python
if state.state == "completed":   # TODO: read from adapter
    ...
```

That TODO will outlive you. The cost of doing the adapter lookup right now is one extra line; the cost of finding all the stale TODOs in two years is a rewrite.

### Anti-pattern 3: Adapter dicts that contain framework imports or callables

The adapter is data. If your adapter dict says `"completion_handler": MyCompletionHandler` or `"build": build_payload_brand_b`, you've turned the config into a Python module disguised as a dict, and you've lost the serialization property that makes stored adapters (§13) work.

Whenever you'd reach for a callable in the dict, use the named-builder indirection from §7 instead: the framework owns a registry of named functions; the adapter holds the *name* (a string). Stored adapters stay JSON-serializable, and the framework still controls every executed code path.

### Anti-pattern 4: Capability flags that drift from real capability

It's tempting to set `supports_mop: True` and ship before testing the mop dispatch path. Then someone sends a mop command and it errors out because the wire format isn't quite right. Capability flags should describe what the integration *correctly handles*, not what the device *theoretically supports*. Test before you flag.

### Anti-pattern 5: Adding adapter fields without adding adapter doc

A new role, a new dispatch field, a new capability — every addition to the framework's expectations needs to be in the adapter config reference docs *at the same time*. The reference doc is the contract. Adapters can't conform to a contract they can't read.

### Pitfall 1: Forgetting to handle adapter-less devices

A device might be registered but the adapter not yet loaded (race during startup, the user hasn't run the discovery wizard yet, the stored config failed to load). Framework code should degrade gracefully — return empty data, hide UI elements, log a warning, but not crash. The standard pattern: every adapter access goes through `coordinator.get_adapter_value(...) or fallback`. Treat `None` as "unknown" everywhere; never let it become an `AttributeError`.

### Pitfall 2: Treating the coordinator as singleton-ish

The coordinator is **per config entry**. Don't cache it at module level, don't pass it around as a global, don't construct it outside `async_setup_entry`. Long-lived objects (manager, tracker) hold a reference passed in at construction; short-lived call sites (service handlers, state listeners) fetch it from `hass.data[DOMAIN][entry_id]["coordinator"]` on each call. If your code "just grabs the coordinator from somewhere," that somewhere is going to be wrong the moment a user adds a second config entry.

### Pitfall 3: Letting the card know which adapter is active

If the card has a code path like `if (adapter.name === "eufy")`, the abstraction broke at the boundary. The card should only know about vocabulary and capabilities — never about the adapter identity itself. (Restating §12's rule because this one re-emerges every time a developer reaches for a quick fix.)

### Pitfall 4: Mutating an adapter dict after registration

Rule #2 says adapters are data. *Read-only* data, in practice. If you mutate `coordinator._registry["vacuum.alfred"]["dispatch"]["fan_speed_options"].append("Turbo")` from somewhere in your manager code, you've just modified an object that's also referenced by every state listener, every entity platform, every service handler. The change is invisible to validation (validation already ran), invisible to migration (migration already ran), and survives until next reload.

If you genuinely need to update an adapter — say, the user edited a stored adapter overlay — call `coordinator.register_adapter_config(entity_id, new_config)` to *replace* the entry. Replace, never mutate. The four-function API is intentionally write-only-via-register; respect it.

---

## 16. Testing strategy

Adapter-driven integrations have a clear testing surface: you can test the framework with synthetic adapters, and you can test each real adapter independently.

### Framework tests

Build a minimal synthetic adapter that covers all the framework's expectations. Run the full integration lifecycle (setup, services, state changes, unload) against it. The synthetic adapter uses obviously-fake values (`"clean_mode_options": ["TEST_MODE_1", "TEST_MODE_2"]`) so that if a brand string leaks into framework code, the test breaks loudly.

```python
SYNTHETIC_ADAPTER = {
    "adapter_name":  "synthetic_test",
    "vacuum_entity_id": "vacuum.test",
    "entities": {
        "task_status": "sensor.test_task_status",
        "dock_status": "sensor.test_dock_status",
        # ...
    },
    "completion": {
        "task_status_value": "SYNTHETIC_COMPLETE",
        "cleared_target_sentinels": {"SYNTHETIC_CLEARED"},
    },
    "dispatch": {
        "clean_mode_options": ["SYNTHETIC_MODE_A", "SYNTHETIC_MODE_B"],
        # ...
    },
    # ...
}
```

If any test fails with `"completed"` or `"Vacuum"` appearing where it shouldn't, you've found a brand leak. Fix it.

### Per-adapter tests

Each real adapter gets its own test suite that verifies:

1. **Schema conformance** — every required role and vocabulary key is present, with the right types.
2. **Payload shape** — for known input (queue + rooms), the adapter produces the exact expected wire format. Golden-file tests work well here.
3. **Completion semantics** — the adapter's declared `task_status_value` matches what real devices send (verify against a recorded sample or, if you can, a live device).
4. **Discovery heuristics** — the discovery service's role-pattern table successfully identifies the right entity IDs for this brand's typical naming convention.

### Integration tests across adapters

Run the framework's main flows against multiple adapters in sequence:

```python
@pytest.mark.parametrize("adapter", [BRAND_A_ADAPTER, BRAND_B_ADAPTER, BRAND_C_ADAPTER])
async def test_full_job_lifecycle(adapter, hass):
    register_adapter_config(adapter["vacuum_entity_id"], adapter)
    # ... drive the full lifecycle ...
    # assert against framework-level invariants, not brand-specific values
```

These cross-adapter tests are your safety net against brand assumptions creeping back in. If a test that runs successfully against Brand A fails against Brand B with no code change between them, you've found a hidden assumption.

---

## 17. Versioning the adapter contract

The framework's expectations of an adapter evolve. You'll:

- Add a new required role.
- Rename a vocabulary key.
- Change the shape of `maintenance_components`.

Existing stored adapters built against the old contract will break the moment the framework demands the new fields. The fix has two pieces:

- A `schema_version` field on every adapter.
- A migration chain in the registry that runs old configs up to current before they enter the live registry.

### `schema_version` as a top-level adapter field

Every adapter declares which version of the contract it was built against:

```python
EUFY_X10_PRO_OMNI_ADAPTER = {
    "schema_version": 3,
    "adapter_name":   "eufy_x10_pro_omni",
    # ...
}
```

The framework declares its currently-supported schema version as a module-level constant (`CURRENT_ADAPTER_SCHEMA_VERSION = 3`).

### Migration hooks in the registry

When `register_adapter_config` is called with a config whose `schema_version` is below current, run it through a chain of migration functions:

```python
# adapters/registry.py

CURRENT_ADAPTER_SCHEMA_VERSION = 3

_MIGRATIONS = {
    1: _migrate_v1_to_v2,
    2: _migrate_v2_to_v3,
}


def register_adapter_config(entity_id: str, config: dict[str, Any]) -> None:
    version = int(config.get("schema_version", 1))
    while version < CURRENT_ADAPTER_SCHEMA_VERSION:
        migrate = _MIGRATIONS.get(version)
        if migrate is None:
            raise ValueError(
                f"No migration from adapter schema v{version} for {entity_id}"
            )
        config = migrate(config)
        version = int(config.get("schema_version", version + 1))
    _REGISTRY[entity_id] = dict(config)
```

Each migration is a pure function from one schema to the next:

```python
def _migrate_v1_to_v2(config: dict) -> dict:
    """v2 introduced `dispatch.clean_intensity_options`; default to single-option."""
    new = dict(config)
    new.setdefault("dispatch", {}).setdefault("clean_intensity_options", ["Standard"])
    new["schema_version"] = 2
    return new
```

### Persisting migrations

For *stored* adapters, run the migration once on load and rewrite the stored shape. The next load won't need to migrate again:

```python
async def load_stored_adapter_configs(hass, manager):
    raw = manager.data.get("adapter_configs", {})
    dirty = False
    for entity_id, config in raw.items():
        original_version = int(config.get("schema_version", 1))
        register_adapter_config(entity_id, config)         # runs migrations
        migrated = get_adapter_config(entity_id)
        if int(migrated.get("schema_version", 1)) != original_version:
            manager.data["adapter_configs"][entity_id] = migrated
            dirty = True
    if dirty:
        await manager.async_save()
```

For *code* adapters, you can either bump them inline (edit the dict, ship the new version) or let the migration run at register-time so old code adapters keep working through a deprecation period.

### What counts as a breaking change

| Change | Migration needed? |
|---|---|
| Adding a new optional field | No — readers tolerate absent fields |
| Adding a new required role | Yes — migration must supply a sensible default or refuse the adapter |
| Renaming a key | Yes — migration translates old key to new |
| Removing a key | Sometimes — only if framework code still tries to read it during a transition |
| Changing a value type | Yes — migration coerces or refuses |
| Reordering a list | Maybe — only if the framework treats position semantically (e.g. "first option is default") |

When in doubt, bump the schema version and write the migration. Schema bumps are cheap; mysterious breakage from a stored adapter built two years ago is expensive.

---

## 18. Observability, validation, and diagnostics

This section covers three related concerns that all answer the same question — "is the adapter doing what we think it's doing, and how do we know?":

- **Observability** — runtime visibility into adapter reads and resolved state.
- **Validation** — the contract check that runs at every boundary where an adapter enters the framework's care.
- **Diagnostics** — the service surface a maintainer uses when something is wrong.

When something goes wrong, the question is almost always "what did the adapter say at that point?". The three subsystems below make that easy to answer.

### `dump_adapter_state` diagnostic service

A service that returns the full current adapter config for a device — plus, optionally, the resolved state of every role entity it declares:

```python
async def _handle_dump_adapter_state(hass: HomeAssistant, call: ServiceCall) -> dict:
    entity_id = call.data["device_entity_id"]
    adapter = get_adapter_config(entity_id) or {}
    role_states = {}
    for role, role_entity in adapter.get("entities", {}).items():
        state = hass.states.get(role_entity)
        role_states[role] = {
            "entity_id":     role_entity,
            "state":         state.state if state else None,
            "last_changed":  state.last_changed.isoformat() if state else None,
            "attributes":    dict(state.attributes) if state else None,
        }
    return {
        "device_entity_id": entity_id,
        "schema_version":   adapter.get("schema_version"),
        "adapter_name":     adapter.get("adapter_name"),
        "adapter_config":   adapter,
        "role_states":      role_states,
    }
```

This service is what you ask the user to run when they report "the integration thinks my vacuum hasn't finished cleaning even though it's docked". The output shows what `task_status` entity the adapter is reading, what value it's currently reporting, and what value the adapter expects to mean "completed". Most adapter-config bugs become obvious in one read.

### Structured logging at adapter boundaries

Every framework function that reads from the adapter should log the read at debug level with structured fields. A simple wrapper:

```python
def get_adapter_value_logged(
    entity_id: str, *path: str, fallback: Any = None, ctx: str = "",
) -> Any:
    value = get_adapter_value(entity_id, *path, fallback=fallback)
    _LOGGER.debug(
        "adapter_read entity=%s path=%s value=%r ctx=%s",
        entity_id, "/".join(path), value, ctx,
    )
    return value
```

In production this is a no-op (debug level is off). When you turn debug on for the integration, you get a trace of every adapter lookup and what it resolved to. Combined with `dump_adapter_state`, this gets you from "something's wrong" to "the adapter declares X but the device reports Y" in minutes rather than hours.

### Validation architecture

Validation gets referenced in many places throughout this guide — discovery drafts (§13), schema migrations (§17), import/export (§13), storage load (§10), registry registration (this section). That's because validation isn't a single event; it's a contract enforced at every boundary where adapter configs enter or leave the framework's care. Before showing the validator itself, the architecture needs one canonical answer to "where and when does validation run?".

#### The four validation boundaries

Validation happens at exactly these four points. Outside of these, framework code may *assume* the adapter is valid — it has already been checked.

1. **Discovery draft updates** — every time the discovery wizard saves a partial draft. Runs the validator and stores the issues alongside the draft so the UI can show "you still need to pick a `task_status` entity." Doesn't reject the draft for missing fields; that's the whole point of a draft. Rejects only structural impossibilities (the config isn't even a dict).
2. **Import (from exported payload)** — when a user imports a JSON adapter someone else exported. Runs the validator; runs the migration chain to bring stale schema versions up to current; refuses the import if validation still fails after migration. Imported configs that pass go into the user's drafts (so they can review before committing) — not directly into the registry.
3. **Storage load on startup** — when `load_stored_adapter_configs` reads persisted adapters. Migrates each up to `CURRENT_ADAPTER_SCHEMA_VERSION`, validates, registers only the ones that pass. Failed entries log a warning and are *skipped*, not dropped from storage — the user can still see and fix them via the wizard.
4. **Registry registration** — the last line of defense. Every `coordinator.register_adapter_config(entity_id, config)` call runs the validator. Hard-fails (raises) on structurally unusable configs; warn-and-register on configs with non-blocking issues. Nothing enters the live registry without passing through this.

The boundaries are in increasing order of strictness: drafts tolerate the most, registry registration tolerates the least. The same `_validate_adapter` function runs at all four points; what differs is what the caller does with the resulting issues list.

#### Strictness rules

The validator returns a list of structured issues. Each issue carries a severity. The framework's response depends on severity, not on which boundary triggered the check:

| Severity | What triggers it | Framework response |
|---|---|---|
| `structural` | Config isn't a dict; required top-level key (`schema_version`, `adapter_name`, `entities`) is absent or wrong type | Always hard-reject (raise on register; refuse on import; mark draft as "unusable") |
| `required_role_missing` | `entities` block is present but a framework-required role (e.g. `task_status`) is missing | Hard-reject on register; surface in draft as a blocking issue |
| `inconsistency` | A capability is `True` but the matching dispatch vocabulary is empty; declared maintenance source entity doesn't exist; vocab list contains duplicates | Warn on register, surface as a non-blocking issue in drafts |
| `schema_drift` | `schema_version` is below current (migrations exist) | Migrate transparently; no warning |
| `schema_future` | `schema_version` is above current (no migration available) | Hard-reject; the framework can't know what newer schemas mean |

The validator is the single source of truth for these classifications; everywhere validation runs, it runs through the same function with the same return shape.

#### Unknown fields: ignored, not rejected

Adapters often carry brand-specific keys the framework doesn't know about (`roborock_protocol_version`, `eufy_internal_quirk`, etc.). The validator **ignores unknown top-level keys**. Reserved namespaces (§21) are off-limits; everything else is the adapter author's space. This keeps the contract forward-compatible: an adapter built for a future framework version (which knows about new fields) doesn't fail on an older framework — older framework just doesn't read the new fields.

The cost of this rule: typos in known field names get silently ignored too (writing `entites: {...}` instead of `entities: {...}` looks like an adapter-specific extension). The TypedDict hint layer (below) catches that at edit-time. Runtime validation catches it indirectly when the framework can't find the role and the adapter falls into the "missing role" branch.

#### Validation depth

The validator goes one level deep on known blocks:

- Top-level keys: type-checked.
- `entities` block: each declared role is type-checked (must be a string in `domain.object_id` shape); presence of framework-required roles is enforced.
- `completion`, `dispatch`, `capabilities`, `dispatch_payload`, `maintenance_components`, `dock_events`: structural checks on the block's own keys.
- Beyond that: the validator stops. The framework doesn't validate, e.g., that every entry inside `maintenance_components` has a `source_entity` that resolves to a real HA entity — that's the job of the discovery flow's "preview before commit" step (§13) and the diagnostic dump service (this section).

Shallow validation is intentional. Deeper validation would couple the validator to the HA state machine (we'd need to call `hass.states.get` from inside `_validate_adapter`), which makes the function async, which makes it harder to call from the registry registration path, which is exactly what you don't want.

#### Failure semantics

When validation fails at each boundary:

- **Draft update**: store the draft and the issues; UI shows what's wrong; user keeps editing.
- **Import**: refuse the import; show the issues to the user; nothing is persisted.
- **Storage load**: skip the bad entry; log a warning with the entity ID and the issues; the bad entry stays in storage so the user can fix it via the wizard.
- **Registry registration**: raise `AdapterConfigError` with the issues list; the calling code (typically `async_setup_entry`) decides whether to abort setup or continue with degraded function for that one device.

The shared rule: **never silently drop an adapter the user worked to build**. Reject, surface, but keep the source readable.

### Validation on register

Run a sanity check whenever a config enters the registry. Catch the things that will hurt later: missing required roles, unrecognized schema version, malformed entity IDs, vocabulary lists empty where the framework treats them as required.

#### The recommendation, stated plainly

**For HA adapter configs: plain dicts plus a lightweight runtime validation function. Don't use Pydantic, dataclasses, TypedDict, or JSON Schema as the validation mechanism.**

The reasoning, in one paragraph: strong static typing is useful for framework internals (the manager class, the queue engine, the service handlers). Adapters are a different category — they're *data* that needs to be easy to serialize, diff, export to JSON, hand-edit in a discovery wizard, and round-trip through a stored-adapter overlay. Plain dicts give you all of that with zero ceremony. The contract is informal convention enforced by a single function from dict to list-of-issues:

```python
def _validate_adapter(config: dict) -> list[str]:
    issues = []
    if "entities" not in config:
        issues.append("missing 'entities' block")
    for required_role in ("task_status", "dock_status"):   # framework-required
        if not config.get("entities", {}).get(required_role):
            issues.append(f"missing entities.{required_role}")
    if not config.get("dispatch", {}).get("clean_mode_options"):
        issues.append("dispatch.clean_mode_options must be a non-empty list")
    if config.get("schema_version", 0) > CURRENT_ADAPTER_SCHEMA_VERSION:
        issues.append(
            f"schema_version {config.get('schema_version')} is newer than "
            f"framework's {CURRENT_ADAPTER_SCHEMA_VERSION}"
        )
    return issues
```

Called from `register_adapter_config`:

```python
def register_adapter_config(entity_id, config):
    issues = _validate_adapter(config)
    if issues:
        for issue in issues:
            _LOGGER.warning("adapter %s: %s", entity_id, issue)
        # Hard-fail only on structurally unusable configs
        if any(i.startswith("missing entities.") for i in issues):
            raise ValueError(f"adapter for {entity_id} is unusable: {issues}")
    _REGISTRY[entity_id] = dict(config)
```

#### Why not Pydantic / dataclasses / TypedDict / voluptuous / JSON Schema?

You can use any of them, but each costs more than it gives:

| Tool | Cost | Pays back? |
|---|---|---|
| **Pydantic** | Adapter dicts become `BaseModel` instances; serialization needs `.model_dump()`; user-built stored adapters need round-trip through the model | Only if you want IDE autocomplete on adapter access, which you mostly don't (lookups are by string path) |
| **dataclasses** | Same shape issue as Pydantic — adapters stop being plain dicts; spread composition (§3 Rule 5) breaks | No |
| **TypedDict** | Static typing only; no runtime validation; adapter authors get type hints in their editor but the registry still has to validate | Maybe, as a *parallel* hint layer — not as the validation mechanism |
| **voluptuous** | Already an HA dependency; readable schemas; but the error messages aren't tuned for adapter config use and the schema lives in code where humans rarely read it | Useful for service-call schemas (where HA already uses it). Overkill for adapter validation. |
| **JSON Schema** | Most portable; works if you want a separate schema file documenting the contract | Worthwhile *only* if you're publishing the adapter format for outside contributors to target. For internal use it's ceremony. |

The function-from-dict-to-issues approach is the right default because:

- Adapter dicts stay plain dicts (Rule #2 stays clean).
- Validation logic lives next to the framework code that uses it.
- New required fields are added by editing one function.
- Test coverage is trivial (pass a dict, assert on the issues list).
- Stored adapter round-trips don't go through any framework-specific type.

The shape of the adapter contract is **informal convention enforced by runtime validation**, not a formally-typed schema. Document the contract in `adapter-config-reference.md` so adapter authors know what to write; enforce it with `_validate_adapter` so wrong configs don't fail silently.

If the user-built adapter is missing fields the framework needs, *tell them*. Silent degradation is the enemy.

#### TypedDict as an optional parallel hint layer

The one place strong typing helps without conflicting with anything above is **catching typos in adapter authoring**. A capability flag like `supports_hot_water_wash` is exactly the kind of thing a developer will mistype as `supports_hotwater_wash` in a new variant; both spellings pass Python parsing and the wrong one silently never gets matched at lookup time.

`TypedDict` is the lightest tool for this job. It's *only* a static type hint — adapters stay plain dicts, serialization stays identical, runtime behavior is unchanged. The only thing it adds is editor autocomplete plus mypy errors on key typos:

```python
from typing import TypedDict


class EntitiesBlock(TypedDict, total=False):
    task_status:            str
    dock_status:            str
    active_map:             str
    active_cleaning_target: str
    battery_level:          str
    # ...


class CapabilitiesBlock(TypedDict, total=False):
    supports_mop:             bool
    supports_passes:          bool
    supports_path_control:    bool
    supports_edge_mopping:    bool
    supports_water_control:   bool
    supports_dock_washing:    bool
    supports_dust_emptying:   bool
    supports_hot_water_wash:  bool
    # ...


class AdapterConfig(TypedDict, total=False):
    schema_version:    int
    adapter_name:      str
    display_name:      str
    model_codes:       list[str]
    entities:          EntitiesBlock
    completion:        dict
    dispatch:          dict
    dispatch_payload:  dict
    capabilities:      CapabilitiesBlock
    # ...
```

Adapter authors who type their code as `EUFY_ADAPTER: AdapterConfig = {...}` get squiggly underlines under typos before they save the file. The runtime validator still runs at registration (TypedDict has no runtime effect), but most typos get caught at edit-time instead.

Use this if your codebase already runs mypy or your editor surfaces type errors. Skip it if you don't — adapters work fine without the hint layer and adding it just for this guide isn't worth the ceremony.

This is the one exception to the "don't use static-typing tools" rule. The reason it's an exception: TypedDict doesn't change the data shape, doesn't break serialization, and doesn't substitute for runtime validation. It's purely additive editor help.

---

## 19. Adapter declarations vs. user preferences

Not everything that varies should go in the adapter config. Some facts vary by *device* (entity IDs, vocabulary, wire format) — those are adapter business. Other facts vary by *install* (the user prefers brush replacement reminders at 200 hours instead of 300, or wants edge mopping always on, or has a non-standard dock placement) — those are user preferences, not adapter declarations.

The distinction matters because conflating them produces two failure modes:

1. **User preferences in the adapter config** are wiped on adapter update or replacement. The user's customizations disappear when the integration ships an updated adapter dict.
2. **Adapter declarations as user preferences** scatter brand-specific facts across user storage. The same Eufy X10 Pro Omni configured on two different installs ends up with two different stored "intervals" for the same brush — and any framework code that reads them has to merge or pick.

### Three-layer config resolution

The clean architecture is three layers:

```
┌──────────────────────────────────────────────────────────┐
│  User preferences (per-install, persisted)               │
│    e.g. rolling_brush_interval_hours: 250                │
│         (user wants reminders sooner than default)       │
└──────────────────────────────────────────────────────────┘
                         ▲
                         │  overrides
                         │
┌──────────────────────────────────────────────────────────┐
│  Stored adapter overlay (per-install, persisted)         │
│    e.g. user-tweaked entities.task_status                │
│         (user remapped after re-import)                  │
└──────────────────────────────────────────────────────────┘
                         ▲
                         │  overrides
                         │
┌──────────────────────────────────────────────────────────┐
│  Code adapter (shipped, immutable in storage)            │
│    e.g. rolling_brush.interval_hours: 300                │
│         (factory default for this device)                │
└──────────────────────────────────────────────────────────┘
```

Reads walk top-down: user preference, then stored adapter overlay, then code adapter. The framework exposes one resolved view:

```python
def resolve_setting(entity_id: str, *path: str, fallback=None) -> Any:
    """Return user pref -> stored adapter -> code adapter -> fallback, in order."""
    user_pref = get_user_preference(entity_id, *path)
    if user_pref is not None:
        return user_pref
    return get_adapter_value(entity_id, *path, fallback=fallback)
```

(The stored-adapter overlay is already merged into the registry by `register_adapter_config`, so framework code doesn't see three layers — it sees one resolved adapter plus a user-preferences side channel.)

### What goes where

Some practical rules:

- **Adapter declarations:** anything that's a fact about the device or vendor (entity IDs, vocabulary, dispatch shape, completion sentinels, factory maintenance intervals, capability flags).
- **Stored adapter overlay:** user corrections to adapter declarations after a discovery flow (e.g. they picked a different `task_status` entity because the heuristic guessed wrong).
- **User preferences:** anything that's about *how this user wants their install to behave* (custom maintenance intervals, default profile choices, notification preferences, custom room nicknames).

If you're not sure, ask: "would two users of the same device, on the same model, sensibly want different values here?" If yes, it's a user preference. If no, it's an adapter declaration.

### Service surface

User preferences get their own service group — `get_user_preference`, `set_user_preference`, `clear_user_preference` — distinct from the adapter-config services in §13. The card edits user preferences through one path and adapter declarations through another. The boundary stays visible in the UI as well as the code.

---

## 20. Appendix: minimal scaffold

A skeleton you can copy as a starting point. This is the absolute minimum required to make the pattern work; a real integration would expand each module substantially.

```
custom_components/my_integration/
├── __init__.py
├── manifest.json
├── const.py
├── adapters/
│   ├── __init__.py
│   ├── coordinator.py
│   ├── config_loader.py
│   └── brand_a/
│       ├── __init__.py
│       └── adapter.py
├── services.py
└── services.yaml
```

### `adapters/coordinator.py`

```python
from __future__ import annotations
from typing import Any
from homeassistant.core import HomeAssistant
from homeassistant.config_entries import ConfigEntry


class AdapterCoordinator:
    """Per-entry coordinator owning the adapter registry and its lookups."""

    def __init__(self, hass: HomeAssistant, entry: ConfigEntry) -> None:
        self.hass = hass
        self.entry = entry
        self._registry: dict[str, dict[str, Any]] = {}

    def register_adapter_config(self, entity_id: str, config: dict[str, Any]) -> None:
        self._registry[entity_id] = dict(config)

    def unregister_adapter_config(self, entity_id: str) -> None:
        self._registry.pop(entity_id, None)

    def get_adapter_config(self, entity_id: str) -> dict[str, Any] | None:
        return self._registry.get(entity_id)

    def get_adapter_value(
        self, entity_id: str, *path: str, fallback: Any = None,
    ) -> Any:
        node: Any = self._registry.get(entity_id) or {}
        for key in path:
            if not isinstance(node, dict):
                return fallback
            node = node.get(key)
            if node is None:
                return fallback
        return node

    def all_registered_entity_ids(self) -> list[str]:
        return list(self._registry)
```

### `adapters/brand_a/adapter.py`

```python
from __future__ import annotations
import copy

from ..coordinator import AdapterCoordinator


def _build_brand_a_adapter(object_id: str) -> dict:
    return {
        "adapter_name": "brand_a",
        "display_name": "Brand A Model 1",
        "model_codes":  ["BA1"],
        "vacuum_entity_id": f"vacuum.{object_id}",
        "entities": {
            "task_status":            f"sensor.{object_id}_task_status",
            "dock_status":            f"sensor.{object_id}_dock_status",
            "active_map":             f"sensor.{object_id}_active_map",
            "active_cleaning_target": f"sensor.{object_id}_active_target",
        },
        "completion": {
            "task_status_value":         "completed",
            "cleared_target_sentinels":  {"", "unknown", "unavailable", "none"},
        },
        "dispatch": {
            "clean_mode_options":  ["Vacuum", "Mop", "Vacuum and mop"],
            "fan_speed_options":   ["Quiet", "Standard", "Max"],
            "clean_passes_options": [1, 2, 3],
        },
        "dispatch_payload": {
            "map_id_field":   "map_id",
            "map_id_type":    "int",
            "rooms_field":    "rooms",
            "room_id_field":  "id",
            "passes_field":   "clean_times",
        },
        "capabilities": {
            "supports_mop":           True,
            "supports_passes":        True,
            "supports_path_control":  False,
        },
        "dock_events": {
            "triggers": {
                "last_mop_wash":   {"washing", "washing mop"},
                "last_dust_empty": {"emptying dust"},
            },
        },
    }


def register_brand_a_adapter_for_vacuum(
    coordinator: AdapterCoordinator, vacuum_entity_id: str,
) -> None:
    object_id = vacuum_entity_id.split(".", 1)[1]
    config = _build_brand_a_adapter(object_id)
    coordinator.register_adapter_config(vacuum_entity_id, config)
```

### `services.py` (excerpt)

```python
from __future__ import annotations
import voluptuous as vol
from homeassistant.core import HomeAssistant, ServiceCall
from homeassistant.helpers import config_validation as cv

from .const import DOMAIN


def _get_coordinator(hass: HomeAssistant, entry_id: str):
    return hass.data[DOMAIN][entry_id]["coordinator"]


def _resolved_call_data(hass: HomeAssistant, call: ServiceCall) -> dict:
    """Auto-resolve map_id from the adapter's active_map role."""
    data = dict(call.data)
    if data.get("map_id"):
        return data
    entity_id = data.get("vacuum_entity_id")
    entry_id  = data.get("entry_id") or _infer_entry_id(hass, entity_id)
    if not (entity_id and entry_id):
        return data
    coordinator = _get_coordinator(hass, entry_id)
    role_entity = coordinator.get_adapter_value(entity_id, "entities", "active_map")
    if role_entity:
        state = hass.states.get(role_entity)
        if state and state.state not in ("unknown", "unavailable", "", None):
            data["map_id"] = state.state
    return data


GET_ACTIVE_JOB_SCHEMA = vol.Schema({
    vol.Required("vacuum_entity_id"): cv.entity_id,
    vol.Optional("map_id"): cv.string,
})


async def _handle_get_active_job(hass: HomeAssistant, call: ServiceCall) -> dict:
    entry_id = _infer_entry_id(hass, call.data["vacuum_entity_id"])
    manager  = hass.data[DOMAIN][entry_id]["manager"]
    return manager.get_active_job(**_resolved_call_data(hass, call))


async def async_register_services(hass: HomeAssistant) -> None:
    hass.services.async_register(
        DOMAIN,
        "get_active_job",
        lambda call: _handle_get_active_job(hass, call),
        schema=GET_ACTIVE_JOB_SCHEMA,
        supports_response=True,
    )
```

(`_infer_entry_id(hass, entity_id)` walks the HA entity registry to find which config entry owns a given entity — standard pattern; left as an exercise so the example stays focused on adapter wiring.)

### `services.yaml` (excerpt)

```yaml
get_active_job:
  name: Get active job
  description: Return the current active job state.
  fields:
    vacuum_entity_id:
      name: Vacuum
      required: true
      selector:
        entity:
          domain: vacuum
    map_id:
      name: Map ID
      description: Leave blank to use the current active map.
      required: false
      selector:
        text: {}
```

### `__init__.py` (excerpt)

```python
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant

from .const import DOMAIN
from .adapters.coordinator import AdapterCoordinator
from .adapters.brand_a.adapter import register_brand_a_adapter_for_vacuum
from .adapters.config_loader import load_stored_adapter_configs
from .services import async_register_services


async def async_setup_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    hass.data.setdefault(DOMAIN, {})
    entry_data = hass.data[DOMAIN].setdefault(entry.entry_id, {})

    coordinator = AdapterCoordinator(hass, entry)
    entry_data["coordinator"] = coordinator

    manager = MyIntegrationManager(hass, entry, coordinator)
    await manager.async_initialize()
    entry_data["manager"] = manager

    # Register code adapters for known devices
    for vacuum_entity_id in manager.get_all_vacuum_entity_ids():
        adapter_name = manager.get_adapter_name(vacuum_entity_id)
        if adapter_name == "brand_a":
            register_brand_a_adapter_for_vacuum(coordinator, vacuum_entity_id)
        # ... additional adapters here

    # Overlay stored (user-built) adapters on top
    await load_stored_adapter_configs(coordinator, manager)

    await async_register_services(hass)
    return True


async def async_unload_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    # Popping the entry's slot drops the coordinator and registry along with it.
    hass.data.get(DOMAIN, {}).pop(entry.entry_id, None)
    return True
```

This is the skeleton. From here you'd add:

- Entity platforms (sensor, switch, etc.) that read role entity IDs from the adapter.
- A manager class that owns persistent state and orchestrates the device.
- Background listeners that read watched entity IDs from the adapter.
- Discovery services for the porting wizard.
- A frontend panel that consumes adapter vocabulary via a dashboard snapshot.

But all of that is built *on top of* the adapter registry; none of it knows the brand.

---

## 21. Appendix: Adapter contract reference

The single canonical answer to "what is a valid adapter?". This is what `_validate_adapter` enforces and what adapter authors target. Everything else in the guide is rationale; this section is the spec.

### Top-level shape

```python
{
    # === Identity (required) ===
    "schema_version":    int,                       # required, matches CURRENT_ADAPTER_SCHEMA_VERSION
    "adapter_name":      str,                       # required, snake_case, unique within the integration
    "display_name":      str,                       # required, human-readable, shown in UI
    "model_codes":       list[str],                 # required (may be empty), vendor model IDs this adapter handles

    # === Entity role mapping (required) ===
    "entities":          dict[str, str],            # required, role -> HA entity_id

    # === Vocabulary and completion semantics (required if framework uses them) ===
    "completion":        dict,                      # required if framework does completion detection
    "dispatch":          dict,                      # required if framework dispatches payloads or shows dropdowns

    # === Wire format (required if framework dispatches) ===
    "dispatch_payload":  dict,                      # required if framework builds payloads

    # === Capability declarations (required) ===
    "capabilities":      dict[str, bool],           # required, boolean flags, default-off

    # === Optional, framework-feature-specific ===
    "maintenance_components": dict[str, dict],      # optional, omit if no maintenance UI
    "dock_events":            dict,                 # optional, omit if no dock-event tracking
    "vocabulary_aliases":     dict[str, dict],      # optional, only for Strategy-3 adapter-switch projects (§10)
}
```

### `entities` block — framework-required roles

These roles must be declared (mapped to a non-empty entity ID string) by every adapter the framework expects to drive end-to-end. Validation fails hard if any are missing:

| Role | Type | Purpose |
|---|---|---|
| `task_status` | str (entity_id) | Reports current task/job state (`"cleaning"`, `"docked"`, `"completed"`, …) |
| `dock_status` | str (entity_id) | Reports dock activity (`"idle"`, `"washing"`, `"emptying"`, …) |

### `entities` block — framework-optional roles

These roles are read when present and degraded gracefully when absent. Adapters declare them only if the underlying device exposes the signal:

| Role | Type | Used by |
|---|---|---|
| `active_map` | str (entity_id) | §11 service-layer map_id auto-resolve |
| `active_cleaning_target` | str (entity_id) | §8 auto-finalization completion check |
| `battery_level` | str (entity_id) | Battery-aware estimation, mid-job recharge detection |
| `robot_position_x` / `robot_position_y` | str (entity_id) | Position-trace mapping subsystem |
| `dry_duration` | str (entity_id) | Captured on `last_dry_start` dock events |
| *additional roles* | str (entity_id) | Reserved namespace — framework code may add new roles in future schema versions; older adapters tolerate their absence |

### `completion` block

```python
"completion": {
    "task_status_value":         str,           # required if completion block present
    "cleared_target_sentinels":  set[str],      # required if completion block present
}
```

- `task_status_value` — the string value of `entities.task_status` that means "job is done." Framework checks `state == this_value`, never a hardcoded `"completed"`.
- `cleared_target_sentinels` — the set of string values for `entities.active_cleaning_target` that mean "no active target." Sentinel matching, not equality.

### `dispatch` block

```python
"dispatch": {
    "clean_mode_options":      list[str],       # required if device supports clean modes
    "fan_speed_options":       list[str],       # required if device supports fan speeds
    "water_level_options":     list[str],       # required if supports_water_control capability
    "clean_intensity_options": list[str],       # required if device supports intensity
    "path_type_options":       list[str],       # required if supports_path_control capability
    "clean_passes_options":    list[int],       # required if supports_passes capability
    "edge_mopping_options":    list[bool],      # optional, defaults to [False, True]
}
```

All option lists are **ordered**. First element is treated as the default whenever the framework needs a fallback. Empty lists are invalid; if a device doesn't support a feature, omit the key (and clear the matching capability flag).

### `dispatch_payload` block

Either declarative (field mapping) or named-builder. Pick one shape per adapter:

```python
# Declarative variant
"dispatch_payload": {
    "map_id_field":   str,           # e.g. "map_id", "mapId", "map"
    "map_id_type":    "int" | "str", # cast for the map_id field
    "rooms_field":    str,           # e.g. "rooms", "room_list", "areas"
    "room_id_field":  str,           # e.g. "id", "room_id", "area_id"
    "passes_field":   str,           # e.g. "clean_times", "passes", "repeat"
    # ... additional field-name declarations as needed
}

# Named-builder variant
"dispatch_payload": {
    "builder":  str,                 # key into framework's _PAYLOAD_BUILDERS registry
    # ... arbitrary builder-specific keys (read by the named builder)
}
```

### `capabilities` block

Flat dict of `str -> bool`. Adapters opt *in* to features; default is off (Rule #4). Standard framework capability flags:

| Flag | Meaning |
|---|---|
| `supports_mop` | Device has a mop |
| `supports_passes` | Per-room pass count is honored |
| `supports_path_control` | Device accepts path-type instructions |
| `supports_edge_mopping` | Edge-mopping toggle is honored |
| `supports_water_control` | Device accepts water-level control |
| `supports_dock_washing` | Dock can wash the mop |
| `supports_dust_emptying` | Dock can empty the dust bin |
| `supports_mop_drying` | Dock can dry the mop |
| `supports_room_priority` | Order of rooms in payload is honored |

Adapters may declare additional brand-specific capabilities; the framework ignores unknown keys. Other framework code that grows new feature flags should declare them in the integration's `adapter-config-reference.md` and add `_validate_adapter` rules to enforce presence where required.

### `maintenance_components` block (optional)

```python
"maintenance_components": {
    "<component_key>": {
        "label":          str,             # human-readable, shown in UI
        "interval_hours": int | float,     # factory default interval
        "source_entity":  str (entity_id), # HA entity that reports cumulative usage hours
    },
    # ... one entry per supported component
}
```

Component keys are arbitrary strings; convention is snake_case (`rolling_brush`, `side_brush`, `filter`, `mopping_cloth`, …). Omit the block entirely if the integration doesn't surface maintenance counters for this adapter.

### `dock_events` block (optional)

```python
"dock_events": {
    "triggers": {
        "<event_type>": set[str],     # event_type -> set of dock_status values that fire it
        # ...
    },
}
```

Event types are framework-defined (`last_mop_wash`, `last_dust_empty`, `last_dry_start`, …); the set of recognized event types is part of the framework contract, not the adapter. Adapters declare which dock-status string values map to each event type — `dock_status` transitions to any value in the set fires the corresponding event.

### Reserved namespaces

Some top-level keys are reserved by the framework and should not be repurposed by adapters:

- `schema_version` — migration machinery (§17)
- `adapter_name` — registry identity
- `vocabulary_aliases` — Strategy-3 vocab tagging (§10)
- Any key beginning with `_` — internal use; framework may add diagnostic keys here

Adapters may add their own top-level keys for adapter-specific data (e.g. `roborock_protocol_version`), but should namespace them with the brand name to avoid collisions: `roborock_protocol_version`, not `protocol_version`.

### Migration expectations

When the framework introduces a new required field or renames an existing one:

1. Bump `CURRENT_ADAPTER_SCHEMA_VERSION` in the framework.
2. Add an entry to `_MIGRATIONS` that translates the previous schema to the new one (see §17 for the migration function shape).
3. Update this reference section with the new field's spec.

Adapter authors update their `schema_version` and adjust their dicts to match. Stored adapters built against the old version migrate automatically on load.

### Minimum viable adapter

The smallest config that passes framework validation:

```python
{
    "schema_version":  3,
    "adapter_name":    "myvac_minimal",
    "display_name":    "MyVac (minimal)",
    "model_codes":     [],
    "entities": {
        "task_status": "sensor.myvac_status",
        "dock_status": "sensor.myvac_dock",
    },
    "completion": {
        "task_status_value":         "completed",
        "cleared_target_sentinels":  {"", "unknown", "unavailable", "none", "null"},
    },
    "capabilities": {},
}
```

This adapter can do completion detection but nothing else — no dispatch, no maintenance, no dock events. The framework will gracefully degrade every feature whose adapter declarations are absent. Use this shape as the starting point when bootstrapping a new brand.

---

## Closing thoughts

The adapter-driven pattern doesn't make brand support free. You still have to do the protocol work, the entity mapping, the payload validation, the completion-signal reverse engineering. What it does is *isolate* that work: the cost of supporting a new brand is paid entirely inside the adapter, not spread across the whole codebase.

When you have one brand, this looks like over-engineering. When you have three, it looks like clarity. By five, it's the only way the codebase stays reviewable. If a second brand is on your horizon — even speculatively — start with the pattern. The refactor cost of introducing it after the codebase has accreted brand assumptions is the highest cost you'll pay. If you're certain you'll stay single-vendor, skip the pattern and don't feel bad about it; see §1 for the honest test.

Two final rules of thumb (assuming you're using the pattern):

1. **If you're typing a brand name in framework code, stop and write an adapter field instead.** Even when there's only one brand right now. The discipline is what makes adding the second brand cheap.
2. **When adding a new adapter, the diff should be one new file under `adapters/<brand>/`, plus possibly one line in `__init__.py` to wire the registration call.** If you find yourself editing services, manager logic, or entity platforms to add brand support, your adapter contract is missing a field — fix the contract.

That's the pattern.
