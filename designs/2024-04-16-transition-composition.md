---
created: 2024-04-16
last updated: 2024-04-29
status: approved
reviewers:
  - gregestren
title: Starlark Transition Composition
authors:
  - katre
discussion thread: https://github.com/bazelbuild/bazel/discussions/22019
---

# Overview

Bazel's facility for [configuration
transitions](https://bazel.build/extending/config#user-defined-transitions) are
widely used in rules today when rules or their dependencies need to change the
configuration. Bazel has functionality to compose multiple native transitions
into one (see
[`ComposingTransition`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/transitions/ComposingTransition.java),
but that hasn't yet been exposed to Starlark for custom transitions.

# Composition Semantics

The existing native composition takes two transitions (either of which may
already be a composed transition), and has fairly basic semantics.

1. The first transition is applied to the configuration.
   1. This will produce either a single result configuation (for a 1:1
      transition), or several (for a 1:2+ transition).
2. The second transition is then applied to each result configuration.
   1. If the second transition has any `input` flags, those are read from the
      first result. The second transition may then decide based on its logic
      what the `output` flags are.
   2. If the first transition was a 1:2+ transition, the second transition must
      be a 1:1 transition.
   3. If the first transition was a 1:1 transition, the second transition can be
      a 1:1 transition or a 1:2+ transition.
3. All result configurations from both transition applications are then
   collected and used as normal. Rules see a single (composed) transition, and
   cannot tell how many transitions were actually applied.
4. Composed transitions can themselves be composed, with the same limitation
   that there is only a single 1:2+ transition allowed anywhere in the chain.

These semantics will be kept in the Starlark version of transition composition.

# Proposal

A new method will be added to [the existing Starlark
ConfigurationTransitionApi](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/starlarkbuildapi/config/ConfigurationTransitionApi.java?q=symbol%3A%5Cbcom.google.devtools.build.lib.starlarkbuildapi.config.ConfigurationTransitionApi%5Cb%20case%3Ayes)
to enable composing a transition object with the current transition, called
`and_then`. The method will not mutate the existing transition but will return a
new value. Only one argument is allowed, but since the result is also a
`ConfigurationTransitionApi` instance further calls can be chained.

The legacy string values `"exec"` and `"target"` will not be allowed, only
actual transition objects. To enable no-op transitions, the `"target"`
transition will also be exposed via `config.target`.

Arguments may be:
1. An existing native transition exposed to Starlark, such as
   [`config.exec`](https://bazel.build/rules/lib/toplevel/config#exec) or
   `android_common.multi_cpu_configuration`.
2. An existing Starlark transition, from [the existing `transition`
   API](https://bazel.build/rules/lib/builtins/transition).

(This may require some extra overloads or superinterfaces in the Java
implementation top handle the distinctions between bare transitions and
transition factories, but this shouldn't be visible to Starlark code.)

The composed transitions can be used like other Starlark transitions, including
on rules and attributes, although the usual restriction that a 1:2+ transition
is not allowed as an incoming rule transition will be checked.
