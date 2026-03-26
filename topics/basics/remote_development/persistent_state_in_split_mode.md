<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Persistent State Component in Split Mode

<link-summary>Synchronize persistent plugin settings correctly between frontend and backend in Split Mode.</link-summary>

This article shows how to make a **`PersistentStateComponent`** synchronize correctly between the frontend and the backend.

At a high level, you will:

1. Create your persistent settings component.
2. Register sync metadata with a `RemoteSettingInfoProvider`.
3. Declare the settings in XML so the initial synchronization can happen.
4. Choose the right sync direction for your use case.

This setup is especially important in [split mode](remote_development.md), where settings may exist on both sides and need to stay in sync.

## Create Your Settings Component

Start by implementing your settings component as you normally would.

If you use `SimplePersistentStateComponent`, it is a good idea to override `noStateLoaded()`. This helps handle the case in which the remote side sends an empty state: instead of leaving the component in an unexpected state, you can explicitly reset it to its defaults.

```kotlin
@State(name = "MySettings", storages = [Storage("my-settings.xml")])
class MySettings : SimplePersistentStateComponent<MySettings.State>(State()) {
    class State {
        var mySetting: Boolean = true
    }

    override fun noStateLoaded() {
        loadState(State()) // Reset to defaults when remote sends empty state
    }
}
```

### Why this matters

During synchronization, one side may receive no previously stored state. If that happens, `noStateLoaded()` gives you a safe and predictable fallback.

## Register a RemoteSettingInfoProvider

Next, tell the platform that this settings component should participate in frontend/backend synchronization.

To do that, create an implementation of `RemoteSettingInfoProvider` and register it on both the frontend and the backend. In practice, this usually means either:

* placing it in a shared module, such as `intellij.platform.split`, or
* registering the same provider in both plugin XML files.

Example:

```kotlin
class MySettingsRemoteInfoProvider : RemoteSettingInfoProvider {
    override fun getRemoteSettingsInfo() = mapOf(
        "MySettings" to RemoteSettingInfo(direction = Direction.InitialFromFrontend)
        // Use InitialFromBackend for project-level settings
    )
}
```

Then register it in `plugin.xml`:

```xml
<extensionPoint name="remoteSettingInfoProvider"
                interface="...RemoteSettingInfoProvider"/>

<extensions defaultExtensionNs="...">
    <remoteSettingInfoProvider
        implementation="com.example.MySettingsRemoteInfoProvider"/>
</extensions>
```

### What This Provider Does

`RemoteSettingInfoProvider` supplies synchronization metadata for your settings component. In particular, it tells the platform which component should be synced and in which direction the initial state should flow.

## Declare the Settings in XML

This step is easy to miss, but it is required for initial synchronization.

If you do not declare the settings in XML, synchronization will not start until the user changes the setting manually for the first time.

Use one of the following declarations, depending on the scope of your settings.

For application-level settings:

```xml
<applicationSettings service="com.example.MySettings"/>
```

For project-level settings:

```xml
<projectSettings service="com.example.MySettings"/>
```

### Why This Is Required

These declarations make the settings visible to the synchronization infrastructure from the start. Without them, the platform does not know that the settings should be included in the initial synchronization.

## Choose the Right Sync Direction

When registering your settings, you need to decide where the initial value should come from.

In most cases, the correct choice depends on whether the settings are application-level or project-level.

| Direction             | Recommended use                                                 |
|-----------------------|-----------------------------------------------------------------|
| `InitialFromFrontend` | Application-level settings. This is usually the default choice. |
| `InitialFromBackend`  | Project-level settings. This is usually the default choice.     |
| `OnlyFromBackend`     | Use when the frontend does not understand or use this setting.  |
| `OnlyFromFrontend`    | Use when the setting is owned entirely by a frontend plugin.    |

### General Guidance

* Use `InitialFromFrontend` for application-level settings unless you have a specific reason not to.
* Use `InitialFromBackend` for project-level settings unless your architecture requires something different.
* Use one-way synchronization (`OnlyFromBackend` or `OnlyFromFrontend`) when only one side is able to interpret or manage the setting.

## Complete Checklist

Before testing your setup, make sure you have done all of the following:

* Implemented your `PersistentStateComponent`
* Added a `noStateLoaded()` fallback if needed
* Created a `RemoteSettingInfoProvider`
* Registered that provider on both frontend and backend
* Declared the settings in XML
* Chosen the correct synchronization direction

## Example Summary

For a typical application-level setting:

* implement the settings as a `PersistentStateComponent`
* declare it in XML with `<applicationSettings ... />`
* register it in `RemoteSettingInfoProvider`
* use `Direction.InitialFromFrontend`

For a typical project-level setting:

* declare it with `<projectSettings ... />`
* use `Direction.InitialFromBackend`
