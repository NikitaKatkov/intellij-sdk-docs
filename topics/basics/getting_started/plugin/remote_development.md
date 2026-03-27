<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Remote Development and Split Mode

<link-summary>Design plugins for both monolithic IDEs and split-process IDE sessions, and validate frontend and backend behavior from the start.</link-summary>

<tldr>

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
This article explains the terminology, motivation, and architecture behind remote development in JetBrains IDEs, also known as **Split Mode**. It also describes how to run, debug, and test an IDE in Split Mode.

## Why It’s Important for Users

* Use more powerful hardware than a local laptop.
* Work on a different OS than the one you’re running locally (useful for cross-platform development).
* Use the laptop as a thin client (often no need to keep source code locally).
* Keep sensitive code on company servers while still working “from anywhere”.

## Why It’s Important for Plugin Developers

* Responsive UX not affected by latency or unstable connection issues
* Some APIs naturally belong to either the backend or the frontend, and it is crucial that they work on the proper side. Take a file system events listener, for instance: the plugin would likely want to observe events in the file system where the project source code is located, not on the client side.
* Split Mode becomes more and more popular, and is demanded by JetBrains IDE customers, which in turn are potential plugin customers as well.

## What Would Happen If You Choose to Do Nothing

Some features do not require anything specific to work properly in Split Mode. These include:

* LSP
* MCP
* Inspections and annotators
* Quick fixes and intentions
* Completion contributors
* Reference providers
* Run configurations
* Find usages providers
* Inlays

If a backend plugin provides other functionality, such as UI or editing assistance, it will be available on the frontend as well, but will be affected by latency and result in suboptimal UX. For example:

* Language support, lexers, and parsers get processed on the backend and expose their output, such as the PSI tree, only there. The frontend does not have information about the file structure and only renders the highlighting received from the backend by applying it to the corresponding ranges in an edited file.
* Typing assistance gets processed on the backend and then sent to the frontend, making it subject to latency issues.
* UI components, actions, dialogs, and popups get rendered on the backend side, and rendering commands are then transmitted to the frontend. They do not require additional effort to work, but they are subject to latency issues, and the resulting UX is usually of insufficient quality.

## Architecture in a Nutshell

![High-level architecture overview](remdev_arch_high_level.png){width=706}

A remote development-native plugin, or **Split Plugin** for short, is a plugin consisting of
**modules** that implement different parts of its functionality and are loaded either on the
backend of a JetBrains IDE, on the frontend, or everywhere. Throughout the documentation, these
modules are referred to as **backend, frontend, and shared plugin modules**. An IDE running in
Split Mode is essentially two separate processes. A monolithic IDE is a single process. A split
plugin works just fine in a monolithic IDE; the only difference is that both frontend and backend
functionality are combined in the same process. IntelliJ Platform determines whether a particular
module belongs to the frontend or the backend by examining its dependencies for the corresponding
`intellij.platform.frontend/backend` module name entries. If a dependency is not satisfied, the
module does not get loaded. Both frontend and backend dependencies are satisfied when an IDE runs
in monolithic mode.

Let’s zoom in on the Split Plugin scheme and see what is typically inside its parts:

![Low-level architecture overview](remdev_arch_low_level.png){width=706}

Frontend and backend code communicate with each other via RPC calls. The RPC framework provided by
the IntelliJ Platform implies the existence of a shared plugin module, which gets loaded
regardless of which IDE mode is active. Inside it, RPC interfaces are defined. The frontend module
depends on the shared module and calls RPC via these interfaces. The backend module depends on the
shared module and defines RPC interface implementations. Thus, the primary flow is to get or send
data from the frontend to the backend via RPC.

Data sent through RPC must be serializable; the *kotlinx.serialization* framework is used for that
purpose. The data is sent over a secure protocol under the hood, so there is no need to apply
additional measures.

In a monolithic IDE, the whole machinery works inside a single process, and there is no network
access during RPC execution. In fact, one can consider the RPC function call a regular suspend
function call in Kotlin, and thus features properly implemented for Split Mode naturally work well
in a monolithic IDE.

## How to Run the IDE in Split Mode with Gradle

IntelliJ Platform Gradle Plugin (2.x) is required to build plugins with split mode support. In the `intellijPlatform` section, there are two special options:

* `splitMode` to indicate that the target IDE must run in split mode
* `splitModeTarget` to indicate where (backend full-fledged or frontend) your plugin must be installed

```kotlin
intellijPlatform {
    splitMode = true // false for monolith IDE
    splitModeTarget = SplitModeTarget.BOTH // also FRONTEND, BACKEND
}
```

## How to Run Tests in Split Mode with Gradle

There are two common needs here:

A) Plugin business logic can be tested by regular unit tests using the IntelliJ test framework, see [IntelliJ Platform Testing Extension | IntelliJ Platform Plugin SDK](https://plugins.jetbrains.com/docs/intellij/tools-intellij-platform-gradle-plugin-testing-extension.html#intellijPlatformTesting)

You may put small tests that verify the functionality of a specific class in any content module. However, if you decide to test the plugin as a whole, put the test classes into the root plugin module \- this is necessary for correct classpath assembly, which will include the plugin.xml file and properly register all plugin extensions from all content modules.

B) Plugin UI in Split Mode can be tested by the integrated UI test framework, see [Integration Tests: UI Testing](https://plugins.jetbrains.com/docs/intellij/integration-tests-ui.html#interaction-with-components). There are publicly available tests that can be used for reference,
see [PluginTest.kt example](https://github.com/JetBrains/intellij-ide-starter/blob/master/intellij.tools.ide.starter.examples.plugins/src/integrationTest/kotlin/PluginTest.kt).

## How to Debug the IDE in Split Mode with Gradle

To be able to run and debug your plugin, you will need to use a run configuration <control>Run IDE (Split Mode)</control> which will start both frontend and backend processes locally.

The [modular plugin template](https://github.com/JetBrains/intellij-platform-modular-plugin-template) already provides such a configuration out of the box.

To add one to your existing project, do the following:

1. Make sure you are using the IntelliJ Gradle plugin version 2.14 or higher
2. Call the `:generateSplitModeRunConfigurations` task via Execute Gradle Task action or via terminal
3. Once finished, the task will produce a run configuration that can be selected in the run widget and used for debugging or running as usual

## How to Test Split Mode Manually and Emulate Latency

Deploying the backend to a real remote machine is not the easiest or fastest way to check that the feature has been properly split. The major problem with features working in a Split Mode IDE is that they are exposed to latency issues. This can be easily emulated within a locally running Split Mode.

1. Enable internal mode in the IDE your plugin is being tested against by specifying the system property `-Didea.is.internal=true`
2. Open the **Split Mode** widget in the upper left corner of the target IDE, switch to the **Connection Config (internal)** tab, and select the **Enable Connection Widget** checkbox
3. Specify a reasonably large custom value in the **Direct Ping** field \- this will delay all communication that goes through RPC in the entire IDE. Be careful not to set it too high \- you will experience very poor UX if a particular feature has not yet been properly split. This will let you experience something close to the real feel of the features provided by your plugin. If you notice no annoying or incorrect behavior, you have likely done a good job refactoring your plugin.
   ![Split Mode widget in the IDE](remdev_split_mode_widget.png){width=706}
