<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Split Mode Feature Development

<link-summary>Refactor an existing plugin feature, or design a new one, so it behaves well in both split-mode and monolithic IDEs.</link-summary>

<tldr>

**Reference**: [](remote_development.md), [](modular_plugins.md), [](rpc.md), [](frontend_backend_shared_apis.md), [](persistent_state_in_split_mode.md)

**Code**: [IntelliJ Platform Modular Plugin Template](https://github.com/JetBrains/intellij-platform-modular-plugin-template)

</tldr>

This page describes a practical flow for implementing a feature that works natively in Split Mode and still behaves the same in a monolithic IDE.
The steps apply both when migrating an existing plugin and when designing a new one.

## 1. Define Plugin Modules

Start with a module layout that can express frontend, backend, and shared responsibilities.
[](modular_plugins.md) describes the packaging model in detail.

For a typical split-aware plugin:

- keep the root plugin module for the main <path>plugin.xml</path> descriptor and overall packaging
- add a shared module with as few dependencies as possible
- add a backend module with `intellij.platform.backend` and other backend-only dependencies
- add a frontend module with `intellij.platform.frontend` and other frontend-only dependencies
- place a module descriptor file in each content module and reference the modules from the root <path>plugin.xml</path>

The initial outcome is mostly structural.
At this stage, the plugin may stop behaving correctly because code has been separated but not yet reconnected.

## 2. Distribute Code by Side

Move code according to where the feature actually runs:

- frontend: tool windows, dialogs, popups, editor-side interactions, latency-sensitive actions
- backend: PSI, VFS, indexing, project model, execution, external process handling
- shared: DTOs, RPC contracts, remote topics, common identifiers, pure utilities

[](frontend_backend_shared_apis.md) lists common APIs that usually belong to one side.
The first pass is often coarse-grained.
That is acceptable.
The goal is to separate obviously frontend-only and backend-only code before reconnecting the feature.

## 3. Create Transport DTOs

Identify the data that the frontend needs but that is only available on the backend.
Represent that data with serializable DTOs in the shared module.

For RPC payloads:

- annotate custom classes with `@Serializable`
- keep DTOs focused on transport data rather than rich domain behavior
- prefer identifiers over heavyweight platform objects when possible

Common platform identifiers used in split-aware plugins include:

- `Project.projectId()` and `ProjectId.findProjectOrNull()`
- `Editor.editorId()` and `EditorId.findEditor()`
- `VirtualFile.rpcId()` and `VirtualFileId.virtualFile()`
- `Icon.rpcId()` and `IconId.icon()`

## 4. Add the Transport Layer

Introduce RPC in the shared module and implement it on the backend.
[](rpc.md) describes the concrete setup.

At a high level:

- define an `@Rpc` interface in the shared module
- make all RPC methods `suspend`
- use only serializable parameters and return values
- implement the interface on the backend
- register the implementation with <include from="snippets.topic" element-id="epLink"><var name="ep" value="com.intellij.platform.rpc.backend.remoteApiProvider"/></include>
- call the RPC from the frontend

When backend code needs to trigger frontend behavior without a direct request, use `ApplicationRemoteTopic` or `ProjectRemoteTopic`.
Register frontend listeners with:

- <include from="snippets.topic" element-id="epLink"><var name="ep" value="com.intellij.platform.rpc.applicationRemoteTopicListener"/></include>
- <include from="snippets.topic" element-id="epLink"><var name="ep" value="com.intellij.platform.rpc.projectRemoteTopicListener"/></include>

### Avoid Blocking the UI

RPC calls are always suspending.
Do not execute them on EDT.
Avoid forcing them through blocking wrappers unless the consequences are fully understood.
When a feature needs immediately available state on the frontend, prefer a dedicated frontend model that is eventually synchronized with the backend instead of blocking UI while waiting for a remote response.
See also [](threading_model.md).

## 5. Validate Behavior in Split Mode and Monolith

After the transport layer is in place, verify the feature in both execution models:

- Split Mode, to observe latency and frontend or backend placement issues
- monolithic mode, to confirm behavior stays consistent when both modules run in one process

[](remote_development.md) describes practical split-mode runs, debugging, and latency emulation.

## 6. Address Common Quality Issues

After the feature works functionally, tune it for resilience and UX:

- wrap long-lived or reconnect-prone RPC flows in `durable {}` so transient disconnections do not permanently break the feature
- register actions on the frontend whenever `AnAction.update()` can be resolved without backend-only data
- render empty or loading state first, then progressively fill the UI as remote data arrives
- page or lazily load large datasets instead of transferring everything at once
- debounce or batch high-frequency frontend events to avoid chatty RPC traffic

Settings and other persisted state may require extra synchronization metadata in split-aware plugins.
See [](persistent_state_in_split_mode.md).

## 7. Add Tests

Split-aware features benefit from the same test pyramid as other plugins, with additional checks around transport and latency:

- serialization and deserialization of DTOs
- consistency between frontend and backend state
- behavior under artificial latency
- integration and UI tests for interactive flows

See [](testing_plugins.md) and [](integration_tests_intro.md).
