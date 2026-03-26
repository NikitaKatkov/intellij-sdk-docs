<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Persistent State in Split Mode

<link-summary>Synchronize plugin settings correctly between frontend and backend processes in split plugins.</link-summary>

This page explains how to make a `PersistentStateComponent` work correctly in Split Mode.
In split plugins, settings may exist on both sides and may need explicit synchronization metadata in addition to the regular persistence implementation.

See also [](persisting_state_of_components.md).

## Create the Settings Component

Implement the settings component as a regular `PersistentStateComponent`.
For `SimplePersistentStateComponent`, overriding `noStateLoaded()` is often a good idea so an empty remote state resets the component to defaults.

```kotlin
@State(name = "MySettings", storages = [Storage("my-settings.xml")])
class MySettings : SimplePersistentStateComponent<MySettings.State>(State()) {
  class State : BaseState() {
    var mySetting by property(true)
  }

  override fun noStateLoaded() {
    loadState(State())
  }
}
```

If one side has no stored state yet, `noStateLoaded()` provides a predictable fallback instead of leaving the component in a partially initialized state.

## Register Sync Metadata

Register a `RemoteSettingInfoProvider` so the split-mode settings infrastructure knows that the component should be synchronized.
The provider maps the persistent state name to synchronization metadata and direction.

```kotlin
class MySettingsRemoteInfoProvider : RemoteSettingInfoProvider {
  override fun getRemoteSettingsInfo(): Map<String, RemoteSettingInfo> = mapOf(
    "MySettings" to RemoteSettingInfo(direction = Direction.InitialFromFrontend),
  )
}
```

Register the provider in the plugin descriptors that participate in Split Mode.
In practice, that usually means either:

- keeping the provider in code shared by both sides and registering it accordingly
- or registering equivalent providers in both frontend and backend descriptors

## Declare the Settings in XML

Initial synchronization depends on explicit settings declarations.
Without them, synchronization may not start until the setting changes for the first time.

For application-level settings:

```xml
<extensions defaultExtensionNs="com.intellij">
  <applicationSettings service="com.example.MySettings"/>
</extensions>
```

For project-level settings:

```xml
<extensions defaultExtensionNs="com.intellij">
  <projectSettings service="com.example.MySettings"/>
</extensions>
```

See also:

- <include from="snippets.topic" element-id="epLink"><var name="ep" value="com.intellij.applicationSettings"/></include>
- <include from="snippets.topic" element-id="epLink"><var name="ep" value="com.intellij.projectSettings"/></include>

## Choose the Sync Direction

The initial direction usually follows the scope of the settings:

| Direction | Typical use |
|-----------|-------------|
| `InitialFromFrontend` | Application-level settings |
| `InitialFromBackend` | Project-level settings |
| `OnlyFromBackend` | Settings that are interpreted only on the backend |
| `OnlyFromFrontend` | Settings that are owned entirely by the frontend |

General guidance:

- use `InitialFromFrontend` for application-level settings unless the setting is backend-owned
- use `InitialFromBackend` for project-level settings unless a feature requires a different ownership model
- use one-way synchronization only when one side does not read or manage the setting

## Checklist

Before validating the feature in Split Mode, make sure that:

- the `PersistentStateComponent` implementation is complete
- `noStateLoaded()` provides safe defaults when needed
- a `RemoteSettingInfoProvider` is registered
- the settings are declared with `<applicationSettings>` or `<projectSettings>`
- the synchronization direction matches the feature ownership model

## Typical Setups

For a typical application-level setting:

- implement the setting as a `PersistentStateComponent`
- declare it with `<applicationSettings ... />`
- register it in `RemoteSettingInfoProvider`
- use `Direction.InitialFromFrontend`

For a typical project-level setting:

- declare it with `<projectSettings ... />`
- use `Direction.InitialFromBackend`
