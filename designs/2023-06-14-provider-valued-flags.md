---
created: 2023-06-14
last updated: 2023-06-15
status: Draft
reviewers: 
title: Provider-Valued Flags
authors: katre
discussion thread: 
---

# Background

Many rules need to implement [custom configuration
transitions](https://bazel.build/extending/config) in order to perform
rule-specific logic like enabling low-lying libraries or building dependencies
for multiple architectures. In many cases, this is fairly simple, but some
rules (like [rules\_android](https://github.com/bazelbuild/rules_android) and
[rules\_apple](https://github.com/bazelbuild/rules_apple)) have more complex
needs.

Specifically, these rules want to take a list of labels of potential values,
and choose based on the underlying target which to use. One example might be
that a `--apple_platforms` flag with a list of platforms, the transition wants
to identify which of those transitions are specifically iOS versus macOS or
tvOS platforms.

Currently, the [Android rules perform this by requiring specific names for the
Android
platforms](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/rules/android/AndroidSplitTransition.java;drc=368fcf758ea2fc7a57ebe5fef06853bcd0da68a6;l=147),
which works but is inelegant.

This proposal suggests a way to allow flags where the value is a
[provider](https://bazel.build/extending/rules#providers), not a bare label,
allowing transitions (and other users of the flag) to directly check the value
of the provider without an additional dependency lookup.

# Syntax

This proposal adds a new flag type which is set like a label-valued flag but
instead holds a provider returned by the target at that label. If the flag is
list type, then labels can be separated by commas, and a list of providers is
available.

## Native API

The native Options API will add a new `providerKey` parameter to `@Options`.

```py
  @Option(
      name = "fruit",
      defaultValue = "",
      providerKey = PlatformInfo.PROVIDER,
      documentationCategory = ...,
      effectTags = {
        ...
      },
      help = "Sets the type of fruit to process.")
  public FruitInfo fruit;
```

The flag can be set via `--fruit=//path/to/my:apple`, and the new `fruit` field
will be populated during flag parsing.

## Starlark API

Similarly, the Starlark build setting API will add a new `provider_key` parameter.

```py
FruitInfo = provider(....)
fruit_flag = rule(
    implementation = _impl,
    build_setting = config.provider(
        flag = True,
        provider_key = FruitInfo,
    ),
)
```

The flag can be set via `--//path/to:fruit_flag=//path/to/my:apple`.

# Implementation

When the
[`BuildConfigurationValue`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/analysis/config/BuildConfigurationValue.java)
is created, provider-valued flags are resolved by the following steps:

1. The `NoConfigTransition` is applied, giving an explicitly blank configuration.
    1. Any provider-valued flags in the remaining fragments are explicitly set to null.
2. The `ConfiguredTarget` for the label value is fetched.
3. The requested provider is founded and stored in the configuration.

If the configured target fails to analyze (possibly because it tries to access
configuration data), or does not contain the correct provider, an error is
thrown and analysis stops.

# Benefits

This enables both the [Better
Transitions](https://docs.google.com/document/d/1LD4uj1LJZa98C9ix3LhHntSENZe5KE854FNL5pxGNB4/edit#heading=h.viy871qwv1r7)
and [Rich
Platforms](https://docs.google.com/document/d/1LD4uj1LJZa98C9ix3LhHntSENZe5KE854FNL5pxGNB4/edit#heading=h.5ih3qvldvdkn)
platforms proposals, by making the providers a configuration-level element
rather than something loaded into the rule context only.

# Impact

This change will add a new dependency-gathering stage to
ConfiguredTargetFunction (and AspectFunction). However, it should not create
more skyframe edges, merely reorder existing ones: for the example of
`--platforms`, the current edge is created in ToolchainResolutionFunction, and
now will be created earlier, before the configuration is realized.

By adding the `NoConfigTransition` step, this should reduce the overall number
of skyframe nodes, because (same example) previously there would be separate
`ConfiguredTargetKey`s for a target platform in every configuration, whereas with
this proposal they will all use the same `ConfiguredTargetKey`.

