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

**Origin story (must include).** This pattern wasn't reached from first
principles. It was forced. The author was building a card for a vacuum
integration with 10 rooms and 8 per-room attributes (water level, clean
mode, suction, path type, edge mopping, pass count, queue order, and a
"selected for next job" boolean) → 80 entities just for room
configuration, before any sensor or status entity. The entity model
collapsed before any architectural decision was made.
Service-as-storage wasn't chosen *over* entities — it was what remained
after the entity approach proved unworkable.

That's the honest framing readers need. The pattern isn't a better
mousetrap discovered through careful design; it's what happens when
the default model breaks under load and you happen to also be shipping
the UI yourself. If you're not in that exact corner — building a card
with high-arity per-object config — you probably shouldn't be looking
at this pattern. The default exists for good reasons.

**Meta: the development order was backwards from conventional advice.**
Standard engineering wisdom says model the data first, then build the
API, then put the UI on top. This project went the opposite direction:
the system existed → the card was built on top → the card's needs drove
the backend complexity. Each "I need to render per-room settings in a
grid" pulled a structured storage shape behind it. Each "I want
drag-and-drop ordering" pulled an `order` attribute and a queue
rebuilder. Each "show learning insights" pulled a trace capture and
analysis pipeline. The 75 services are not a designed API surface —
they're the accumulated artifact of the card needing things and the
backend growing to provide them.

This is **product-led / frontend-driven development**, and it's
disparaged in API-first culture. In this case it was correct, because:
  - The card is the only realistic consumer (no third-party integrators
    competing for API attention)
  - Each backend feature has a concrete frontend justification, so
    nothing is built speculatively
  - Iteration is cheap when you control both ends

The tradeoff: the backend has weird shapes that exist only because the
card happened to want them. The services aren't a clean general-purpose
API; they're the services *this card* needed, in the order it needed
them. Anyone trying to use the integration without the card hits that
shape and finds it confusing.

This is fine for a one-author project shipping its own UI. It is **not**
a recommendation for multi-author projects, libraries, or anything
other people extend. For those, API-first / model-first remains correct.
The pattern in this guide is a description of what *worked here*, not
a prescription for what to do generally.

When to write this up:
- The right framing isn't "services beat entities," it's "**how often
  does this attribute change, and who changes it?**"
  - Frequently mutated, especially via automations / dashboards / voice
    → entity (you want history, state tracking, automation triggers,
    template access, the UI affordances HA already gives you for free).
    Example: per-room *order* in a clean queue — users drag-rearrange
    constantly, automations might bump priority, voice might say
    "clean Kitchen first." Entity is unambiguously right.
  - Rarely mutated, "set once per install" config → service /
    structured storage. No reason to pay the entity registry tax for
    something that almost never changes. Example: per-room cleaning
    parameters (Bathroom = Quiet because hardwood, set once, stays
    forever).
- Arity is a secondary factor — high arity *amplifies* the entity cost
  for rarely-mutated config (55+ entities of dead weight), but doesn't
  flip the rule on its own. A single rarely-mutated attribute is still
  fine as one entity if you want one for it.
- **Theoretical mutability ≠ actual mutation.** In a stable household,
  per-room cleaning settings are configured once during onboarding and
  then sit untouched for years. The attribute *could* change daily in
  theory; in practice, "Bathroom is hardwood, give it Quiet+Low" is
  set on day one and that's it. Use *steady-state* mutation frequency
  in a real install, not theoretical mutability, to decide.
- **Mutation profiles vary by user, not by attribute.** Within the same
  per-room cleaning setting list, one user sets everything once and
  forgets. Another (the author of this integration) tweaks `path_type`
  frequently because it has a huge effect on run time. Same attribute,
  same integration, opposite mutation profiles. This means you can't
  globally categorize attributes as "service" or "entity" — the right
  design lets the *same* attribute be reachable both ways, with the
  card surfacing whichever access pattern the user wants. The
  service-as-storage layer is the source of truth; specific attributes
  can also be exposed as entities for users who want HA-native tweaking.
- **Different users tweak different attributes, and the author can't
  predict which.** The author cares about `path_type` because runtime
  matters to them. Another user cares about `clean_mode` because they
  switch between vacuum-only and vacuum-mop seasonally. A third
  rearranges `order` constantly while leaving everything else alone.
  A fourth never touches any of it. The integration author has zero
  visibility into which attribute will be "hot" for any given install,
  which means they *cannot* statically pre-promote N of M attributes
  to entities and leave the rest as services. Every static choice is
  wrong for some non-trivial slice of users. Service-as-storage as the
  ground truth, with the card (or HA itself, via a user-configurable
  "expose as entity" mechanism) deciding what gets surfaced, is the
  only design that survives user-specific access patterns.
- **Automation users want trigger access to *everything*, even niche
  state.** HA's user base is weird in the best way. Someone, somewhere,
  wants an automation that fires "when room D is configured for 2
  passes + high water." The integration author would never build
  per-room-per-attribute entities for that — it's a fringe use case
  that doesn't justify the entity explosion. But the *automation* has
  to be possible, or you've quietly locked out a legitimate HA-user
  workflow.

  The escape hatch is **structured attributes on the parent entity**.
  In this integration, the vacuum entity carries a `rooms` attribute
  with the full per-room state dict. Service-backed storage is the
  source of truth; the parent entity's attribute mirrors it. Automations
  can trigger on that attribute via template triggers (awkwardly, but
  it works). Templates can read it. Voice can't (yet), but most uses
  can. So the dichotomy was never really "service OR entity" — it was
  "service-backed storage with a structured-attribute window into it
  from the parent entity." That window is what preserves HA's
  template-and-trigger power without per-attribute entity explosion.

  This is the missing piece. The article should foreground it rather
  than treating entities and services as exclusive options.
- **Storage shapes behavior, not just reflects it.** An entity with a
  slider in the UI invites users to tweak. A service requires intent
  to invoke. Making something an entity creates an attractor for
  mutation that wouldn't otherwise exist. So the choice isn't neutral
  — it's partly self-fulfilling. If you don't *want* users tweaking
  this daily, don't put it behind a slider.
- **The pattern is rare for a reason.** Entities buy you a lot for
  free: state history, template access, automation triggers, voice
  addressability, Lovelace rendering, areas/labels, the registry's
  built-in tooling. Service-as-storage walks away from all of it.
  That cost is only acceptable when you've replaced it with something
  better — typically a custom card you ship alongside. Most integration
  authors don't ship a card, so the entity model is correct for them
  by default. Service-as-storage isn't a better mousetrap; it's a
  different shape with prerequisites.
- **Prerequisites for choosing service-as-storage:**
  1. You ship (or accept hard dependency on) a card / custom UI
  2. The configured objects have high arity × wide attribute set
  3. Steady-state mutation frequency is low (set-once-per-install)
  4. You don't need state history, templates, or voice on the config
     (or you can synthesize them where needed)
  When fewer of these hold, the entity model is the right default —
  even if the abstract attribute *could* be modeled as services.
- Use eufy-vacuum-manager as the case study: per-room *order* is an
  entity (frequently mutated, automation-target). Per-room *fan_speed*
  is service-backed config (rarely mutated, "set once"). Same domain,
  different mutation profiles, different storage choices.
- Think about whether this is its own section or fits inside the
  "Storage decoupling" section (§10) of the main guide.

(Captured 2026-05-17 as a follow-up after the eufy-clean PR scoping
exercise — that PR would have been a polish wrapper that didn't address
the actual UI gap. The gap exists because stock HA has no clean answer
to high-arity per-object configuration without entity explosion.)
