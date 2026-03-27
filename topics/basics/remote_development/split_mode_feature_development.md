<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Implementing a Feature for Split Mode

<link-summary>Refactor an existing plugin feature or design a new one, so it behaves well in both split-mode and monolithic IDEs.</link-summary>

<tldr>

**Code**: [IntelliJ Platform Modular Plugin Template](https://github.com/JetBrains/intellij-platform-modular-plugin-template)

</tldr>

This page describes a practical flow for implementing a feature that works natively in [Split Mode](split_mode_for_remote_development.md) and still behaves the same in a monolithic IDE.
The steps apply both when migrating an existing plugin and when designing a new one.

![Typical Split Mode refactoring path](remdev_refactoring_path.png){width=706}

This article shows a step-by-step instruction on how to refactor an existing IntelliJ plugin or write a new one in a way that works natively in Split Mode and provides the best possible UX to its users.

## 1. Identify or create necessary plugin modules

- Please refer to the [modular plugin template](https://github.com/JetBrains/intellij-platform-modular-plugin-template) for module structure and necessary dependencies and [Modular plugins](modular_plugins.md) for modular plugin concept description
- Create at least those three modules:
  1. `<YourPlugin>.Shared` - with as few dependencies as possible
  2. `<YourPlugin>.Backend` - with `intellij.platform.backend` dependency
  3. `<YourPlugin>.Frontend` - with `intellij.platform.frontend` dependency
- Make sure the dependencies are properly described in the build scripts - again, refer to the plugin template
- Put a `plugin.xml` file into the root plugin module’s `resources/META-INF` directory
- Put the `<YourPlugin>.<ModuleType>.xml` file into the `resources` directory in all three freshly created modules
- Reference content modules in the root `plugin.xml` file

> a plugin consists of at least three modules, namely frontend, backend, and shared, plus possibly the existing code that resides in a root plugin module or in other submodules.
>
{style="note" title="Expected Outcome"}

## 2. Put the existing or newly written code into appropriate module types

- Identify which kind of features dominate in your plugin
  1. See which APIs it uses more and which type of module they belong to by referring to [List of Frontend/Backend/Shared APIs in IntelliJ Platform](frontend_backend_shared_apis.md)
  2. If there are mostly backend extensions in the plugin, consider it backend functionality; otherwise, frontend functionality
- Extract the existing functionality you have just analyzed into the module type determined during the previous step
- If some APIs are still used in the wrong module type, that’s expected and will be addressed in due course
- Continue moving the code between modules with a higher level of granularity
  1. Example: have a plugin that provides a toolwindow which receives data to display from an external CLI tool that analyzes a project.
  2. First, move the toolwindow implementation to the frontend and register it in its XML descriptor
  3. Second, extract the code responsible for external process spawning and reading its output to the backend module
  4. Ensure no more APIs are reported as being used in an inappropriate module type

> your plugin is not functioning properly since the UI is now completely detached from the business logic. The code, though, is properly distributed between modules that do not communicate with each other yet. The root plugin module carries only the `plugin.xml` descriptor in its `resources` folder. All the code from it is extracted to one of the freshly created modules.
>
{style="note" title="Expected Outcome"}

## 3. Create DTO classes required for data exchange between the frontend UI and backend logic

- Investigate what information is necessary for the extracted UI components but is known only on the backend side
- Create classes representing the data and annotate them with `@Serializable`
- Consider using one of the primitive types as a DTO property, or use your own structure with a proper custom serializer implementation
- Consider using serializable form of some major platform primitives if necessary:
  1. To pass a `Project` - use `project.projectId()` to create a serializable ID and `projectId.findProjectOrNull()` to resolve a project by ID
  2. To pass an `Icon` - `icon.rpcId()` and `iconId.icon()`
  3. To pass a `VirtualFile` - `virtualFile.rpcId()` and `virtualFileId.virtualFile()`

> there are serializable (in terms of `kotlinx.serialization` framework) DTO classes in the shared module representing the data to be sent over RPC calls.
>
{style="note" title="Expected Outcome"}

## 4. Add the transport layer to connect your UI to the backend model

- Introduce RPC interface in the shared plugin module
  1. RPC interface must have an `@Rpc` annotation
  2. Must implement `RemoteApi<Unit>` interface
  3. Must have only `suspend` functions
  4. Must use only `@Serializable` DTO classes as function return type and parameters
- Introduce RPC implementation in the backend plugin module
  1. RPC implementation must implement the corresponding RPC interface
  2. It must be registered in the backend module XML descriptor via the `platform.rpc.backend.remoteApiProvider` extension point
- For more details about RPC refer to [RPC guideline](remote_procedure_calls.md)
- Use DTOs created on step 3 as input parameters and return values. Get back to step 3 if some data is missing.
- Call the RPC where the backend data is required
  1. It is a crucial detail that RPC calls are always suspending. It may be impossible to use suspending code in a particular place in the frontend functionality, either because it is an old implementation written in Java and is not ready for suspend functions at all, or because the data must be available immediately, otherwise causing poor UX or even freezes.
     Remember that a proper UX is one of the main reasons we initiated the entire splitting process for, see the [split-mode introduction](split_mode_for_remote_development.md).
  2. You can’t call RPC on the event dispatch thread (EDT). Avoid wrapping it in `runBlockingCancellable` unless it is absolutely necessary and you understand all the consequences of such a decision, namely blocking the caller thread and breaking the structured concurrency and suspending API concepts
  3. Consider using the existing platform abstraction for shared state as a reference: [FlowWithHistory.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/lang-impl/src/com/intellij/build/FlowWithHistory.kt)
- RPC is designed to be initiated by the frontend, which implies that users always interact with one of the IDE UI components that naturally belong to the frontend. In some cases, however, you may want to initiate some UI display from within the backend code. For instance, a long backend process may want to show a notification after it finishes. Consider using the RemoteTopic API in such cases
  1. Declare a project- or application-level topic in the shared plugin module by using the `ProjectRemoteTopic()` or `ApplicationRemoteTopic()` method, respectively
  2. Subscribe to the topic in the frontend plugin module via the `platform.rpc.applicationRemoteTopicListener` extension point - here you may call code to display the desired notification, for instance
  3. Push the serializable DTO class into the topic in the backend plugin module where necessary - as soon as the DTO is delivered, your frontend topic listener will do its job
  4. Example: [ShowStructurePopupRemoteTopicListener.kt](https://github.com/JetBrains/intellij-community/blob/1b63f9058d6285980c1eac14b8b59fca251751b7/platform/structure-view-impl/frontend/src/ShowStructurePopupRemoteTopicListener.kt)

> your frontend UI exchanges serializable data with backend via RPC or RemoteTopic API
>
{style="note" title="Expected Outcome"}

## 5. Verify and polish

After all infrastructure has been implemented, it is time to verify the feature behavior and polish it. Refer to [Introduction into Split Mode / Remote Development](split_mode_for_remote_development.md) on how to manually test Split Mode, and check the monolithic IDE as well - the behavior is expected to be exactly the same.

> the code is valid from the point of view of this guide, and the behavior is as expected in both Split Mode and a monolithic IDE
>
{style="note" title="Expected Outcome"}

## 6. Review common issues

Now that the general functionality works as expected, consider reviewing the list of frequently occurring problems and suggested solutions for them. Depending on the specifics of the feature, you might not necessarily need to tune the code.

- Handle reconnection: wrap RPC calls in `durable { ... }`; this wrapper will restart the call if a network error occurs. Be careful with any side effects your code inside the durable block produces - ideally, avoid them.
  Example: [RecentFileModelSynchronizer.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/frontend/src/com/intellij/platform/recentFiles/frontend/model/RecentFileModelSynchronizer.kt)
- Register actions in proper plugin modules
  1. We strongly encourage registering actions on the frontend side, if possible. The possibility is determined by the action's update method: if it requires backend entities, the action belongs on the backend. Otherwise - on the frontend.
  2. Frontend actions are rendered immediately and do not introduce delays when displaying context menus, popups, or toolbars. If the frontend action decides to touch backend entities in the `AnAction.actionPerformed` method, it is completely fine to call RPC there.
  3. *(\*\*\*advanced\*\*\*) In some complicated cases, the `action.update()` method might require both frontend and backend entities to be available simultaneously. Such cases must be addressed individually depending on a specific feature description. There are examples of shared, eventually synchronized state implementations in the IntelliJ Platform codebase, for instance in
     the [Recent Files implementation](https://github.com/JetBrains/intellij-community/tree/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles). There, an eventually consistent mutable shared state is implemented to make it possible to access the data model on both the frontend and the backend with no RPC calls required for access*
  4. Consider approaching us on the [JetBrains Platform Forum](https://platform.jetbrains.com) to discuss what could be done in your specific case.
- Display empty state: UI must render without waiting for backend; show placeholders and progressively fill in data.
- Load large state: avoid “send everything at once” RPC implementations; use paging/lazy loading, and request only what the UI needs now.
  Example: [BackendRecentFileEventsModel.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/backend/src/com/intellij/platform/recentFiles/backend/BackendRecentFileEventsModel.kt#L164)
- Do not load data too frequently: avoid chatty RPC (per keystroke, per scroll tick). Batch requests, cache results, debounce UI events.
  Example: [RecentFilesEditorTypingListener.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/frontend/src/com/intellij/platform/recentFiles/frontend/RecentFilesEditorTypingListener.kt)

> known issues are mitigated, and the plugin quality is now good enough.
>
{style="note" title="Expected Outcome"}

## 7. Add tests

Fix the split feature behavior and quality with unit and integration tests if you have not used the TDD approach earlier. See the [split-mode testing section](split_mode_for_remote_development.md#how-to-run-tests-in-split-mode-with-gradle).

We suggest paying attention to:

- Data transformations: correct de/serialization
- Data consistency: correct data merging in case of complicated features with eventually consistent backend/frontend state
- Proper behaviour under latency: artificial delay in a test implementation backend service does not bring freezes or broken UX

> the feature implementation has tests covering its correct behaviour in remote and local scenarios.
>
{style="note" title="Expected Outcome"}

Should you have any questions or uncertainties regarding the splitting process, you are very welcome on our [JetBrains Platform Forum](https://platform.jetbrains.com) - we will try to provide as much help as possible there and reconsider and adjust what is inconvenient for you.
