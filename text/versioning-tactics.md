- Title: Versioning Tactics
- Driver: Hugo
- Status: Draft

# Context

The dialectic between evolving and compatibility is a standard issue for Coq. 

Like other proof assistants, Coq is exploring new directions and ideas of proof techniques and proof languages, and, often it needs to evolve going beyond the limitations, fragile design or underspecifications revealed by experimenting, also integrating new ideas, providing more expressiveness, finer-grained design.

At the same time, there is a need for stability to support the corpus of existing developments, to let users be free of upgrading or not, also allowing the constitution of a persistent library of formal knowledge which de facto include libraries for which no maintainers exist.

# Summary

We propose a model to deal with configurability of tactics (for options of the style `Set Structural Injection` etc.).

# Proposal

- There are two levels of configuration:
  - a tactic-based configuration called tactic configuration applicable to only the given tactic
  - a global configuration called global version configuration applicable to all tactics

- Tactics supporting different behaviors take a specific tactic configuration record as an option. If `None`, the tactics takes as argument the dynamic configuration state relevant for this tactic at runtime (see e.g. PR #677 for an example with `injection`). The dynamic configuration state can itself mix local parametrization (e.g. the flag `Set Structural Injection`) with global parametrization (e.g. `Set Version 8.6.`).

  Adding tactic support for configuration can be done on a by-need basis, tactic by tactic. For instance, in PR #677, `injection` takes the following optional record as argument:
```coq
type inj_flags = {
    keep_proof_equalities : bool;
    injection_in_context : bool;
    injection_pattern_l2r_order : bool;
  }
```

- At the level of the Coq vernacular, parametrization of tactics is done using specific tactic-level parametrization commands (e.g. `Set Structural Injection`) or global versioning parametrization (e.g. `Set Version 8.6.`).

- At the level of the tactic expressions, parametrization is done by using tacticals of the form `version 8.6 in tac`. There is also a special `version runtime in tac` which says that the tactic is supposed to take its configuration dynamically. (In this proposal, only global versioning can be set, but one can imagine a more general framework where each individual configuration flag of a tactic can be modified as a tactical.)

- The role of the runtime version number is to satisfy wishes like `Ltac f x := ... rewrite x ....` and having "rewrite" parametrized with the global flags at runtime (wish from @JasonGross).
 
- Internalization of tactics is done in an environment telling if a version number is set or if the version number is dynamic. By default, in the absence of a `version ... in ...` tactical, the version number is the version number defined globally (e.g. `8.7`).

- When internalizing a tactic, whether it is as toplevel or within an Ltac definition, the current tactic configuration (or `None` for runtime configuration) is attached to the tactic. This is done by providing, for each tactic, an ML function of the following form:
```coq
let get_keep_proof_equalities version =
  !keep_proof_equalities_for_injection

let get_injection_in_context version =
  !injection_in_context && Flags.version_strictly_greater Flags.V8_5

let get_injection_pattern_l2r_order () =
  !injection_pattern_l2r_order

let get_inj_flags dynamic =
  if dynamic then None else
   Some {
    keep_proof_equalities = get_keep_proof_equalities ();
    injection_in_context = get_injection_in_context ();
    injection_pattern_l2r_order = get_injection_pattern_l2r_order ();
   }
```
and, by writing rules like:
```
let myinj dyn a b c = injClause (get_inj_flags dyn) None false None

TACTIC EXTEND injection
| [ "injection" ] -> [ mying !dynamic None false None ]
| [ "injection" destruction_arg(c) ] -> [ mytclWithHoles (myinj !dynamic None) false c ]
END
```
Note: the `dynamic` parameter is here taken global but it could also be a value provided by `TACTIC EXTEND` (that would prevent problems with eta-conversion).

- Internally, any ML function calling an ML tactic decide on its own what to configuration to pass (e.g. `decide equality` would call `injection` with the exact flags it needs, not with `None`).

This is a proposal to deal with configurability and versioning of tactics. I find it well-suited for tactics such as `subst`, `injection`, `destruct`, etc. This is not exclusive of other or complementary approaches such as proof modes, deprecation, canonical notations for primitive tactics, versioned plugins, etc. For instance, if any primitive tactic comes with a prefix syntax, one could more easily pass specific flags to them, using a syntax say of the form `Coq.Init.Notations.injection ~version:8.6 ~keep_proofs term`, etc. Also, I guess that for the Meta-based `apply`, the plugin approach shall be the reasonable one.
