<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Implementing a Feature for Split Mode

<link-summary>Refactor an existing plugin feature, or design a new one, so it behaves well in both split-mode and monolithic IDEs.</link-summary>

<tldr>

**Reference**: [](remote_development.md), [](modular_plugins.md), [](rpc.md), [](frontend_backend_shared_apis.md), [](persistent_state_in_split_mode.md)

**Code**: [IntelliJ Platform Modular Plugin Template](https://github.com/JetBrains/intellij-platform-modular-plugin-template)

</tldr>

This page describes a practical flow for implementing a feature that works natively in Split Mode and still behaves the same in a monolithic IDE.
The steps apply both when migrating an existing plugin and when designing a new one.

![Typical Split Mode refactoring path](remdev_refactoring_path.png){width=900}

This article shows a step-by-step instruction on how to refactor the existing IntelliJ plugin or write a new one in a way it works natively in Split Mode and provides best possible UX to its users.

1. Identify or create necessary plugin modules.

    * Please refer to the plugin template [https://github.com/JetBrains/intellij-platform-modular-plugin-template](https://github.com/JetBrains/intellij-platform-modular-plugin-template) for module structure and necessary dependencies and **\<TODO proper link to the Article 2\>** for modular plugin concept description
    * Create at least those three modules:
        1. \<YourPlugin\>.Shared \- with little as possible dependencies
        2. \<YourPlugin\>.Backend \- with intellij.platform.backend dependency
        3. \<YourPlugin\>.Frontend \- with intellij.platform.frontend dependency
    * Make sure the dependencies are properly described in the build scripts \- again, refer to the plugin template
    * Put a plugin.xml file into the root plugin module’s resources/META-INF directory
    * Put the \<YourPlugin\>.\<ModuleType\>.xml file into the resources directory in all three freshly created modules
    * Reference content modules in the root plugin.xml file

   *Expected outcome: a plugin consists of at least three modules, namely frontend, backend and shared \+ possibly the existing code which resides in a root plugin module or in other submodules.*

2. Put the existing or newly written code into appropriate module types:
    * Identify what kind of features dominates in your plugin
        1. See which APIs it uses more and which type of module they belong to by referring to the **\<TODO proper link to the Article 4\>**
        2. If there is mostly backend extensions in the plugin, consider it backend functionality, otherwise \- frontend
    * Extract your existing functionality you have just analyzed into the module type determined during the previous step
    * See some APIs are still used in a wrong module type, that’s expected and will be addressed in the due course
    * Continue moving the code between modules with now increased granularity level
        1. Example: have a plugin that provides a toolwindow which receives data to display from an external CLI tool that analyzes a project.
        2. First ,move the toolwindow implementation to the frontend and register it in its XML descriptor
        3. Second, extract the code responsible for external process spawning and reading its output to the backend module
        4. Ensure no more APIs are reported as being used in an inappropriate module type

   *Expected outcome: your plugin is not functioning properly since the UI is now completely detached from the business logic. The code though is properly distributed between modules which do not communicate with each other yet. The root plugin module carries only the plugin.xml descriptor in its resources folder. All the code from it is extracted to one of the freshly created modules.*

3. Create DTO classes required for data exchange between frontend UI and backend logic:
    * Investigate what kind of information is necessary for the now extracted UI components but is known on the backend side only
    * Create classes representing the data and annotate them with @Serializable annotation
    * Consider using one of the primitive types as DTO property or have your own structure with a proper custom Serializer implementation
    * Consider using serializable form of some major platform primitives if necessary:
        1. To pass a Project \- use project.projectId() to create a serializable ID and projectId.findProjectOrNull() to resolve a project by id
        2. To pass an Icon \- icon.rpcId() and iconId.icon()
        3. To pass a VirtualFile \- virtualFile.rpcId() and virtualFileId.virtualFile()

   *Expected outcome: there are serializable (in terms of kotlinx.serialization framework) DTO classes in the shared module representing the data to be sent over RPC calls.*

4. Add the transport layer to connect your UI with the backend model
    * Introduce RPC interface in the shared plugin module
        1. RPC interface must have an @Rpc annotation
        2. Must implement RemoteApi\<Unit\> interface
        3. Must have only suspend functions
        4. Must use only @Serializable DTO classes as function return type and parameters
    * Introduce RPC implementation in the backend plugin module
        1. RPC implementation must implement the corresponding RPC interface
        2. It must be registered in the backend module xml descriptor via the *platform.rpc.backend.remoteApiProvider* extension point
    * For more details about RPC refer to the **\<TODO proper link to the Article 3\>**
    * Use DTOs created on step 3 as input parameters and return values. Get back to step 3 if some data is missing.
    * Call the RPC where the backend data is required
        1. It is a crucial detail that RPC calls are always suspending. It could be that one can not allow using the suspending code in a particular place in the frontend functionality. Either because it’s an old implementation written in java, and it’s not ready for suspend functions at all, or because the data must be available immediately, otherwise causing poor UX or even freezes. Remember that a proper UX is one of the main reasons we initiated the entire splitting process for\! **\<TODO
           proper link to the Article 1\>**
        2. You can’t call RPC on the event dispatch thread (EDT). Avoid wrapping it into runBlockingCancellable unless absolutely necessary and you understand all the consequences of such decision (namely, blocking the caller thread, breaking the structured concurrency and suspending api concepts)
        3. Consider using the existing platform abstraction for shared state as a reference [https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/lang-impl/src/com/intellij/build/FlowWithHistory.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/lang-impl/src/com/intellij/build/FlowWithHistory.kt)
    * RPC is designed to be initiated by the frontend, which implies users always interact with one of the IDE UI components that naturally belong to the frontend. In some cases, you may want to initiate some UI displaying from within the backend code, however. For instance, a long backend process wants to show a notification after it finishes. Consider using the RemoteTopic API in such cases
        1. Declare a project- or application-level topic in the shared plugin module by using ProjectRemoteTopic() or ApplicationRemoteTopic() method respectively
        2. Subscribe to the topic in the frontend plugin module via the *platform.rpc.applicationRemoteTopicListener* extension point \- here you may call a code for displaying the desired notification, for instance
        3. Push the Serializable DTO class into the topic in the backend plugin module where necessary \- as soon as the DTO gets delivered, your frontend topic listener will do its job
        4. Example: [https://github.com/JetBrains/intellij-community/blob/1b63f9058d6285980c1eac14b8b59fca251751b7/platform/structure-view-impl/frontend/src/ShowStructurePopupRemoteTopicListener.kt](https://github.com/JetBrains/intellij-community/blob/1b63f9058d6285980c1eac14b8b59fca251751b7/platform/structure-view-impl/frontend/src/ShowStructurePopupRemoteTopicListener.kt)

   *Expected outcome: your frontend UI exchanges serializable data with backend via RPC or RemoteTopic API*

5. After having all infrastructure implemented, it’s now time to verify the feature behaviour and polish it. Refer to the guide on how to manually test the split mode, check monolith IDE as well \- it is expected the behaviour is exactly the same. **\<TODO proper link to the Article 1\>**
   *Expected outcome: the code is valid from the current guide POV, and the behaviour is as expected in both split mode and monolith*
6. Now the general functionality works as expected, consider reviewing the list of frequently occurring problems and suggested solutions for them. Depending on the feature specifics, you might not necessarily need to tune the code.
    * Handle reconnection: wrap RPC calls into *durable{...};* this wrapper will restart the call should a network error occur. Be careful with any side effects your code inside the durable block produces \- ideally, avoid them.
      Example [https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/frontend/src/com/intellij/platform/recentFiles/frontend/model/RecentFileModelSynchronizer.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/frontend/src/com/intellij/platform/recentFiles/frontend/model/RecentFileModelSynchronizer.kt)
    * Register actions in proper plugin modules
        1. We strongly encourage registering actions on the frontend side, if possible. The possibility is determined by the action's update method: should it require backend entities, the action belongs to the backend then. Otherwise \- frontend.
        2. Frontend actions are rendered immediately and do not bring delays into context menu/popups/toolbars displaying. Should the frontend action decide to touch backend entities in the AnAction.actionPerformed method, it is completely fine to call RPC there.
        3. *(\*\*\*advanced\*\*\*) In some complicated cases action.update() method might require both frontend and backend entities available simultaneously. Such cases must be addressed individually depending on a specific feature description. There are examples of shared eventually synchronized state implementations in the IntelliJ Platform codebase, for instance in
           the [https://github.com/JetBrains/intellij-community/tree/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles](https://github.com/JetBrains/intellij-community/tree/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles) . There an eventually-consistent mutable shared state is implemented to make it possible to access the data model on both frontend and backend with no RPC calls required for the access*
        4. Consider approaching us on the platform forum [https://platform.jetbrains.com](https://platform.jetbrains.com) to discuss what could be done in your specific case.
    * Display empty state: UI must render without waiting for backend; show placeholders and progressively fill in data.
    * Load large state: avoid “send everything at once” RPC implementations; use paging/lazy loading, and request only what the UI needs now.
      Example: [https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/backend/src/com/intellij/platform/recentFiles/backend/BackendRecentFileEventsModel.kt\#L164](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/backend/src/com/intellij/platform/recentFiles/backend/BackendRecentFileEventsModel.kt#L164)
    * Do not load data too frequently: avoid chatty RPC (per keystroke, per scroll tick). Batch requests, cache results, debounce UI events.
      Example: [https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/frontend/src/com/intellij/platform/recentFiles/frontend/RecentFilesEditorTypingListener.kt](https://github.com/JetBrains/intellij-community/blob/1c3952828ff3af2d18f99a6721c48bb22f97bd57/platform/recentFiles/frontend/src/com/intellij/platform/recentFiles/frontend/RecentFilesEditorTypingListener.kt)

   *Expected outcome: known issues are mitigated and now the plugin quality is good enough, finally.*

7. Fix the split feature behaviour and quality with unit and integration tests, should you not have used the TDD approach earlier.  **\<TODO proper link to the Article 1, specifically testing description\>**
   We suggest to pay attention to:

* Data transformations: correct de/serialization
* Data consistency: correct data merging in case of complicated features with eventually consistent backend/frontend state
* Proper behaviour under latency: artificial delay in a test implementation backend service does not bring freezes or broken UX
  *Expected outcome: the feature implementation has tests covering its correct behaviour in remote and local scenarios.*

Should you have any questions or uncertainties regarding the splitting process, you are very welcome on our forum [https://platform.jetbrains.com](https://platform.jetbrains.com) \- we’ll try to provide as much help as possible there, reconsider and adjust what is inconvenient for you.
