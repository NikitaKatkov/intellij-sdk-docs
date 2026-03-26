<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Configuring a Plugin for Split Mode

<link-summary>Configure Gradle so a plugin can run, debug, and test with separate frontend and backend processes.</link-summary>

[Split Mode](remote_development.md) configuration is the Gradle layer of remote development support.
It does not replace modularization, but it is the entry point for running a plugin with separate frontend and backend processes during development.

See also [](remote_development.md) and [](modular_plugins.md).

## Required Tooling

Split Mode requires [](tools_intellij_platform_gradle_plugin.md).
The older [](tools_gradle_intellij_plugin.md) does not support modular plugins or Split Mode development.

## Basic Configuration

Enable Split Mode in the `intellijPlatform {}` extension:

```kotlin
intellijPlatform {
  splitMode = true
  splitModeTarget = SplitModeAware.SplitModeTarget.BOTH
}
```

The two relevant properties are:

- [`splitMode`](tools_intellij_platform_gradle_plugin_extension.md#intellijPlatform-splitMode), which starts separate frontend and backend processes
- [`splitModeTarget`](tools_intellij_platform_gradle_plugin_extension.md#intellijPlatform-splitModeTarget), which selects where the plugin is installed

## Choosing the Install Target

Choose `splitModeTarget` according to the code being exercised:

- `BACKEND` for backend-only functionality
- `FRONTEND` for frontend-only functionality
- `BOTH` for typical modular plugins that contain frontend, backend, and shared modules

For split  plugin development, `BOTH` is the most common choice.

## Custom Run and Test Tasks

Custom split-mode tasks can be declared with `intellijPlatformTesting`:

```kotlin
val runIdeSplitMode by intellijPlatformTesting.runIde.registering {
  splitMode = true
  splitModeTarget = SplitModeAware.SplitModeTarget.BOTH
}

val testIdeUiSplitMode by intellijPlatformTesting.testIdeUi.registering {
  splitMode = true
  splitModeTarget = SplitModeAware.SplitModeTarget.BOTH
}
```

These tasks get dedicated sandboxes and can be used like other development or test tasks.
See [](tools_intellij_platform_gradle_plugin_testing_extension.md).

## Typical Next Step

After the Gradle configuration is in place, the next step is deciding how the plugin code should be distributed between frontend, backend, and shared modules.
See [](split_mode_feature_development.md).
