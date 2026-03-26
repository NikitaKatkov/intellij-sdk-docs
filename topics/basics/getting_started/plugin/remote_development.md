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
This article explains the terminology, motivation, and architecture behind remote development in JetBrains IDEs, also known as **Split Mode**. It also describes how to run, debug, and test an IDE in Split Mode mode.

## WHY IT’S IMPORTANT FOR USERS

* Use more powerful hardware than a local laptop.
* Work on a different OS than the one you’re running locally (useful for cross-platform development).
* Use the laptop as a thin client (often no need to keep source code locally).
* Keep sensitive code on company servers while still working “from anywhere”.

## WHY IT’S IMPORTANT FOR PLUGIN DEVELOPERS

* Responsive UX not affected by latency or insatiable connection issues
* Some APIs naturally belong to either backend or frontend, and it is crucial they are working on a proper side. Take a file system events listener, for instance: it is likely that the plugin would like to observe events in the FS where the project source code is located, not on the client side.
* Split Mode becomes more and more popular, and is demanded by JetBrains IDE customers, which in turn are potential plugin customers as well **\<TODO Give me some figures to share, not from the web, but from our DevEco or other surveys \- reliable data required to convince devs\>**

## WHAT WOULD HAPPEN IF YOU CHOOSE TO DO NOTHING

Some features do not require anything specific to work properly in Split Mode. These are:

* LSP
* MCP
* Inspections and annotators
* Quick fixes and intentions
* Completion contributors
* Reference providers
* Run configurations
* Find usages providers
* Inlays


Should a backend plugin provide other functionality like UI or editing assistance, it will be available on the frontend as well, but will be affected by latency and result in suboptimal UX. Namely:

* Language support, lexer, parser \- get processed on backend, expose their output like PSI tree only on backend. Frontend does not have information regarding the file structure and only renders highlighting received from backend by applying it to the corresponding ranges in an edited file
* Typing assistance \- gets processed on backend and then sent to frontend, subject for latency issues
* UI components, actions, dialogs, popups \- gets rendered on the backend side and then rendering commands transmitted to the frontend, does not require additional effort to work, is subject to latency issues, resulting UI components UX usually has insufficient quality

## ARCHITECTURE IN A NUTSHELL

![High-level architecture overview](remdev_arch_high_level.png){width=900}

A Remote development-native or for short **Split Plugin** is a plugin consisting of **modules** that implement different parts of its functionality and get loaded either on backend of JetBrains IDE, or on frontend, or everywhere. Throughout the documentation these modules will be referred to as **backend, frontend and shared plugin module**. An IDE running in a split mode is essentially two separate processes. A monolithic IDE is a single process. A split plugin works just fine in a monolithic IDE, the only difference is that both frontend and backend functionality are combined in the same process. IntelliJ Platform determines if a particular module belongs to frontend or backend by examining its dependencies for the corresponding \`intellij.platform.frontend/backend\` module name entries. If a dependency is not satisfied, the module does not get loaded. Both frontend and backend dependencies are satisfied when an IDE runs in monolithic mode.

Let’s zoom in the scheme of the Split Plugin and see what's typically inside its parts:

![Low-level architecture overview](remdev_arch_low_level.png){width=900}

Frontend and backend code communicate with each other via RPC calls. The RPC framework provided by IntelliJ platform implies the existence of a shared plugin module which gets loaded no matter what IDE mode is active. Inside it, RPC interfaces are defined. The frontend module depends on the shared and calls RPC via these interfaces. Backend module depends on the shared and defines RPC interfaces implementations. Thus the primary flow is to get/send data from frontend to backend via RPC.

Data being sent through the RPC must be serializable, *kotlinx.serialization* framework is used for that purpose. The data is sent over a secure protocol under the hood, there is no need for additional measures to be applied.

In the monolithic IDE, the whole machinery works inside a single process, and there is no network access during RPC execution. In fact, one can consider the RPC function call as a regular suspend function call in kotlin, and thus the features properly implemented for split mode naturally work well in a monolithic IDE.

## HOW TO RUN THE IDE IN SPLIT MODE WITH GRADLE

IntelliJ Platform Gradle Plugin (2.x) is required to build plugins with split mode support. In the \``intellijPlatform`\` section, there are two special options:

* \``splitMode`\` to indicate that the target IDE must run in split mode
* \``splitModeTarget`\` to indicate where (backend full-fledged or frontend) your plugin must be installed


```kotlin
intellijPlatform {
    splitMode = true // false for monolith IDE
    splitModeTarget = SplitModeTarget.BOTH // also FRONTEND, BACKEND
}
```

## HOW TO RUN TESTS IN SPLIT MODE WITH GRADLE

There are two common needs here:

A) Plugin’s business logic can be tested by regular unit tests using the intellij test framework, see [IntelliJ Platform Testing Extension | IntelliJ Platform Plugin SDK](https://plugins.jetbrains.com/docs/intellij/tools-intellij-platform-gradle-plugin-testing-extension.html#intellijPlatformTesting)

You may put small tests that verify a specific class functionality in any content module. However, should you decide to test the plugin altogether, kindly put the test classes into the root plugin module \- this is necessary for correct classpath assembly which will include the plugin.xml file and properly register all plugin extensions from all content modules

B) Plugin’s UI in split mode can be tested by integrated UI tests framework, see [Integration Tests: UI Testing | IntelliJ Platform Plugin SDK](https://plugins.jetbrains.com/docs/intellij/integration-tests-ui.html#interaction-with-components) . There are publicly available tests that can be used for reference, see [https://github.com/JetBrains/intellij-ide-starter/blob/master/intellij.tools.ide.starter.examples.plugins/src/integrationTest/kotlin/PluginTest.kt](https://github.com/JetBrains/intellij-ide-starter/blob/master/intellij.tools.ide.starter.examples.plugins/src/integrationTest/kotlin/PluginTest.kt) .

## HOW TO DEBUG THE IDE IN SPLIT MODE WITH GRADLE

To be able to run and debug your plugin, you will need to use a run configuration Run IDE (Split Mode) which will start both frontend and backend processes locally.

The plugin template [https://github.com/JetBrains/intellij-platform-modular-plugin-template](https://github.com/JetBrains/intellij-platform-modular-plugin-template) already provides such a configuration out of the box.

To have one in your existing project, please do the following:

1. Make sure you are using the IntelliJ Gradle plugin version 2.13.**\<TODO: paste proper version after it is released\>** or higher
2. Call the *:generateSplitModeRunConfigurations* task via Execute Gradle Task action or via terminal
3. Once finished, the task will produce a run configuration that can be selected in the run widget and debugged/run as usual

## HOW TO TEST SPLIT MODE MANUALLY AND EMULATE LATENCY

Deploying the backend to a real remote machine is not the easiest and fastest way to check that the feature has been properly split. The major problem with features working in Split Mode IDE is that they are exposed to latency issues. This could be easily emulated within a locally running Split Mode.

1. Enable internal mode in the IDE your plugin is being tested against (via specifying the system property *\-[Didea.is](http://Didea.is).internal=true*)
2. Open the **Split Mode** widget in the upper left corner of the target IDE, switch to the **Connection Config (internal)** tab and tick the **Enable Connection Widget** checkbox
3. Specify a custom reasonably big value in the **Direct Ping** field \- this will delay ALL communication that goes through the RPC in the entire IDE. Only be careful with setting too big values \- you’ll experience a really bad UX, if a particular feature is not yet properly split. Thus you’ll be able to experience a close to real feeling of the features provided by your plugin. Should you notice no annoying or incorrect behaviour, you have likely done a good job refactoring your plugin\! Congrats :)
![Split Mode widget in the IDE](remdev_split_mode_widget.png){width=900}
