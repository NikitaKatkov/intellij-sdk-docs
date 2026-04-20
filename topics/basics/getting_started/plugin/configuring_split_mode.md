<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Configuring a Plugin for Split Mode

<link-summary>Configure Gradle so a plugin can run, debug, and test with separate frontend and backend processes.</link-summary>

Split Mode configuration in the Gradle build script allows for running development sandbox IDEs in mode emulating the [remote development](split_mode_and_remote_development.md) scenario, with both frontend and backend processes running locally.
To make a plugin work natively in split mode, see [plugin modularization](modular_plugins.md).

## Basic Configuration

> Split Mode requires [IntelliJ Platform Gradle Plugin **2.x**](tools_intellij_platform_gradle_plugin.md).
>
{style="warning"}

Enable Split Mode in the `intellijPlatform {}` extension:

```kotlin
intellijPlatform {
  splitMode = true
  pluginInstallationTarget = SplitModeAware.PluginInstallationTarget.BOTH
}
```

The two relevant properties are:

- [`splitMode`](tools_intellij_platform_gradle_plugin_extension.md#intellijPlatform-splitMode) – starts separate frontend and backend processes
- [`pluginInstallationTarget`](tools_intellij_platform_gradle_plugin_extension.md#intellijPlatform-pluginInstallationTarget) – selects where the plugin is installed

## Choosing the Installation Target

Choose `pluginInstallationTarget` according to the code being exercised:

- `BACKEND` for backend-only functionality
- `FRONTEND` for frontend-only functionality
- `BOTH` for typical modular plugins that contain frontend, backend, and shared modules

For split plugin development, `BOTH` is the most common choice.

> `pluginInstallationTarget` only controls where the plugin is placed in locally run development sandboxes.
> It doesn't describe end-user installation or synchronization behavior in split mode.
> See [Plugin Management](plugin_management_in_split_mode.md).
>
{style="note"}

## Custom Run and Test Tasks

Custom split-mode tasks can be declared with `intellijPlatformTesting`:

```kotlin
val runIdeSplitMode by intellijPlatformTesting.runIde.registering {
  splitMode = true
  pluginInstallationTarget = SplitModeAware.PluginInstallationTarget.BOTH
}

val testIdeUiSplitMode by intellijPlatformTesting.testIdeUi.registering {
  splitMode = true
  pluginInstallationTarget = SplitModeAware.PluginInstallationTarget.BOTH
}
```

These tasks get dedicated sandboxes and can be used like other development or test tasks.
See [](tools_intellij_platform_gradle_plugin_testing_extension.md) for more details.

## Typical Next Step

After the Gradle configuration is in place, the next step is deciding how the plugin code should be distributed between frontend, backend, and shared modules.
See [](split_mode_feature_development.md).

#### Necessary plugin and library versions

The following table lists the recommended versions of plugins and libraries for split mode development, tailored for compatibility with the specified IntelliJ Platform versions.

| IntelliJ Platform                                    | 2025.3       | 2026.1         |
|------------------------------------------------------|--------------|----------------|
| `rpc` _plugin_                                       | 2.1.20-0.1   | 2.3.20-RC2-0.1 |
| `org.jetbrains.kotlin.plugin.serialization` _plugin_ | 2.1.20       | 2.3.20         |
| `kotlinx-serialization-core-jvm` _library_           | 1.7.3        | 1.9.0          |
| `kotlinx-serialization-json-jvm` _library_           | 1.7.3        | 1.9.0          |

