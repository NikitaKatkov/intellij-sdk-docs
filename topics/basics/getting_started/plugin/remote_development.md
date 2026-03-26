<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Remote Development and Split Mode

<link-summary>Design plugins for both monolithic IDEs and split-process IDE sessions, and validate frontend and backend behavior from the start.</link-summary>

<tldr>

**Reference**: [](modular_plugins.md), [](split_mode_feature_development.md), [](rpc.md), [](frontend_backend_shared_apis.md), [](persistent_state_in_split_mode.md)

**Code**: [IntelliJ Platform Modular Plugin Template](https://github.com/JetBrains/intellij-platform-modular-plugin-template), [Making an IntelliJ Plugin Remote Development-friendly](https://www.youtube.com/watch?v=T2VvY6kgALY)

</tldr>

## What Is Split Mode

The documentation uses the following basic terms:

- **Remote Development**: a workflow where the IDE frontend connects to a backend running locally or on another machine
- **Split Mode**: essentially a synonym for Remote Development, a JetBrains IDE setup where the frontend and backend run as separate processes
- **Split Plugin**: a plugin consisting of dedicated frontend, backend, and shared modules so each part runs on the appropriate side in Split Mode

Remote development changes where plugin code runs.
In Split Mode, the frontend process renders the user interface, while the backend process hosts the project model, indexing, analysis, execution, and other heavy work.
The backend may run on the same machine, on another host, in a container, or in the cloud.
Plugins that provide user interface, typing assistance, or other latency-sensitive features should account for this model from the beginning.

## Why It Matters

### For Users

Split Mode makes it possible to:

- use more powerful hardware than a local laptop
- work on another operating system while keeping the local machine as a thin client
- avoid storing project sources locally when that is preferred
- keep sensitive code on managed infrastructure while still using a local IDE client

### For Plugin Authors

Split Mode affects both architecture and user experience:

- latency becomes part of the feature design
- some APIs naturally belong to either the frontend or the backend
- plugin modules often need different dependencies on different sides
- a feature that is technically functional may still feel slow or inconsistent if it stays on the wrong side

## Behaviour Without Split-Aware Design

Many backend-driven features continue to work with little or no split-specific refactoring.
Typical examples include:

- LSP and MCP integrations
- inspections and annotators
- quick fixes and intentions
- completion contributors
- reference providers
- run configurations
- find usages providers
- inlays

Other features may remain functional but usually provide lower-quality UX when they stay backend-driven:

- language support infrastructure still runs on the backend, and the frontend renders results instead of owning a full local PSI model
- typing assistance and editor reactions that require frequent round-trips can feel laggy
- dialogs, popups, tool windows, and other interactive UI can work from the backend, but their responsiveness is then limited by latency

## Architecture in a Nutshell

A split plugin usually consists of several modules that are loaded on the frontend, on the backend, or everywhere.
Throughout this documentation, these modules are referred to as frontend, backend, and shared plugin modules.
See [](modular_plugins.md) for the packaging model and [](split_mode_feature_development.md) for the migration flow.

A typical split layout looks as follows:

- shared module: RPC interfaces, DTOs, remote topics, and other code used on both sides
- frontend module: user interface, editor-side interactions, and latency-sensitive behavior
- backend module: PSI, VFS, indexing, project model, external process execution, and other project-local logic

An IDE running in Split Mode is two separate processes.
A monolithic IDE remains a single process.
The same split plugin can work in both cases.
In monolithic mode, frontend and backend modules are loaded into the same process, and the same abstractions continue to work without network hops.

## Running Split Mode with Gradle

The [](tools_intellij_platform_gradle_plugin.md) is required for modular plugins and Split Mode development.
The `intellijPlatform {}` extension provides two key properties:

- [`splitMode`](tools_intellij_platform_gradle_plugin_extension.md#intellijPlatform-splitMode) enables separate frontend and backend processes
- [`splitModeTarget`](tools_intellij_platform_gradle_plugin_extension.md#intellijPlatform-splitModeTarget) selects whether the plugin is installed in the frontend, backend, or both

```kotlin
intellijPlatform {
  splitMode = true
  splitModeTarget = SplitModeTarget.BOTH
}
```

See also [](configuring_split_mode.md) and [](tools_intellij_platform_gradle_plugin_task_awares.md#SplitModeAware).

## Testing and Debugging

Split-aware plugins should be checked in both Split Mode and monolithic mode.

Model and business logic can still be covered with regular IntelliJ Platform tests.
For modular plugins, full-plugin tests are best placed in the root plugin module so the runtime classpath includes the root <path>plugin.xml</path> descriptor and all registered content modules.

User interface and end-to-end flows can be verified with [](integration_tests_intro.md).
The modular plugin template already includes a split-mode run configuration.
Existing projects can create equivalent run and debug tasks with `runIde` or `intellijPlatformTesting.runIde` entries configured for Split Mode.

## Manual Testing and Latency Emulation

A local Split Mode run is useful, but it does not reproduce network delays by default.
Artificial latency helps expose features that still depend on chatty or blocking frontend and backend interactions.

<procedure title="Emulate Latency in a Local Split Mode Run">

1. Enable internal mode in the target IDE instance with the `-Didea.is.internal=true` system property.
2. Open the Split Mode widget in the running IDE and enable the internal connection controls.
3. Increase the direct ping value to inject latency into frontend and backend communication.
4. Exercise the plugin UI and editor interactions.
   A well-split feature should remain responsive, show placeholder or stale state only when acceptable, and avoid freezes.

</procedure>
