# TODO — future sections

## Services-as-config-surface vs. entities

Stock Home Assistant's default unit of configuration is the entity:
a select per dropdown, a number per slider, a switch per toggle. That
works cleanly when each device exposes a handful of independent
settings. It scales badly when the configuration is "N things with M
settings each" — e.g. N rooms with M per-room cleaning parameters.

eufy-vacuum-manager hit that wall and walked the other way: 75 services
that read and write per-room state as structured blobs, with the card
acting as the UI on top. Stock HA would express the same configuration
as a forest of entities (5+ per room × 11 rooms = 55+ entities just for
cleaning settings) — which is technically possible but counterintuitive,
hard to template against, and fights the registry.

The eufy-vacuum-manager pattern is itself counterintuitive — services
historically mean "actions you trigger," not "the storage layer your
config lives in." But for high-arity configuration it's the leaner shape.

When to write this up:
- The right framing isn't "services beat entities," it's "what's the
  arity of your config surface, and does the entity registry make sense
  as the storage?"
- Use eufy-vacuum-manager as one concrete case study, ideally pair it
  with a counter-example where entities are unambiguously right.
- Think about whether this is its own section or fits inside the
  "Storage decoupling" section (§10) of the main guide.

(Captured 2026-05-17 as a follow-up after the eufy-clean PR scoping
exercise — that PR would have been a polish wrapper that didn't address
the actual UI gap. The gap exists because stock HA has no clean answer
to high-arity per-object configuration without entity explosion.)
