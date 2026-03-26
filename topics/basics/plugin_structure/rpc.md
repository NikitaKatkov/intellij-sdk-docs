<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# RPC for Split Mode Plugins

<link-summary>Connect frontend and backend plugin modules with RPC, serializable DTOs, and remote topics.</link-summary>

<tldr>

**Reference**: [](modular_plugins.md), [](split_mode_feature_development.md), [](frontend_backend_shared_apis.md)

**Code**: [IntelliJ Platform Modular Plugin Template](https://github.com/JetBrains/intellij-platform-modular-plugin-template)

</tldr>

RPC is the usual transport layer between frontend and backend modules in a split-aware plugin.
The shared module defines the API contract.
The backend module implements it.
The frontend module calls it.

## Module Setup

The shared module defines RPC interfaces and DTOs.
It typically applies the `rpc` and `kotlinx.serialization` plugins:

```kotlin
plugins {
  id("rpc")
  id("org.jetbrains.kotlin.jvm")
  id("org.jetbrains.kotlin.plugin.serialization")
}

dependencies {
  intellijPlatform {
    intellijIdea(libs.versions.intellij.platform)
  }
}
```

The frontend module depends on the shared module and frontend platform modules:

```kotlin
plugins {
  id("rpc")
  id("org.jetbrains.kotlin.jvm")
  id("org.jetbrains.kotlin.plugin.serialization")
}

dependencies {
  intellijPlatform {
    intellijIdea(libs.versions.intellij.platform)
    bundledModule("intellij.platform.frontend")
  }

  compileOnly(project(":shared"))
}
```

The backend module depends on the shared module and backend platform modules:

```kotlin
plugins {
  id("rpc")
  id("org.jetbrains.kotlin.jvm")
  id("org.jetbrains.kotlin.plugin.serialization")
}

dependencies {
  intellijPlatform {
    intellijIdea(libs.versions.intellij.platform)
    bundledModule("intellij.platform.kernel.backend")
    bundledModule("intellij.platform.rpc.backend")
    bundledModule("intellij.platform.backend")
  }

  compileOnly(project(":shared"))
}
```

The backend module descriptor should declare the platform and shared-module dependencies:

```xml
<idea-plugin>
  <dependencies>
    <module name="intellij.platform.backend"/>
    <module name="intellij.platform.kernel.backend"/>
    <module name="modular.plugin.shared"/>
  </dependencies>
</idea-plugin>
```

## Create an RPC Interface

Define the RPC interface in the shared module:

```kotlin
@Rpc
interface ChatRepositoryRpcApi : RemoteApi<Unit> {
  companion object {
    suspend fun getInstance(): ChatRepositoryRpcApi {
      return RemoteApiProviderService.resolve(remoteApiDescriptor<ChatRepositoryRpcApi>())
    }
  }

  suspend fun getMessagesFlow(projectId: ProjectId): Flow<List<ChatMessageDto>>

  suspend fun sendMessage(projectId: ProjectId, messageContent: String)
}
```

Use the following rules for RPC interfaces:

- annotate the interface with `@Rpc`
- inherit from `RemoteApi<Unit>`
- make every RPC method `suspend`
- use only serializable parameters and return values
- keep transport contracts in the shared module

Custom classes and enums should be annotated with `@Serializable`.
If a type is not natively serializable, provide a custom serializer.

## Implement the RPC on the Backend

Add an implementation in the backend module:

```kotlin
class BackendChatRepositoryRpcApi : ChatRepositoryRpcApi {
  override suspend fun getMessagesFlow(projectId: ProjectId): Flow<List<ChatMessageDto>> {
    val backendProject = projectId.findProjectOrNull() ?: return emptyFlow()
    return BackendChatRepositoryModel.getInstance(backendProject).getMessagesFlow()
  }

  override suspend fun sendMessage(projectId: ProjectId, messageContent: String) {
    val backendProject = projectId.findProjectOrNull() ?: return
    BackendChatRepositoryModel.getInstance(backendProject).sendMessage(messageContent)
  }
}
```

Register the implementation with `RemoteApiProvider`:

```kotlin
internal class BackendRpcApiProvider : RemoteApiProvider {
  override fun RemoteApiProvider.Sink.remoteApis() {
    remoteApi(remoteApiDescriptor<ChatRepositoryRpcApi>()) {
      BackendChatRepositoryRpcApi()
    }
  }
}
```

Register the provider in the backend descriptor:

```xml
<idea-plugin>
  <extensions defaultExtensionNs="com.intellij">
    <platform.rpc.backend.remoteApiProvider
        implementation="org.jetbrains.plugins.template.BackendRpcApiProvider"/>
  </extensions>
</idea-plugin>
```

## Use RPC on the Frontend

Call the shared API from the frontend:

```kotlin
ChatRepositoryRpcApi.getInstance().getMessagesFlow(project.projectId())
ChatRepositoryRpcApi.getInstance().sendMessage(project.projectId(), messageContent)
```

Because RPC methods are suspending, they must be called from an appropriate coroutine context.
Avoid running them on EDT.
See also [](threading_model.md).

If an RPC call fails because the backend is still initializing, reconnecting, or temporarily unavailable, the call may throw an exception.
Long-lived frontend subscriptions often benefit from the `durable {}` wrapper:

```kotlin
durable {
  ChatRepositoryRpcApi.getInstance().getMessagesFlow(project.projectId()).collect { valueFromBackend ->
    // Process remote state updates.
  }
}
```

## Common RPC Patterns

### Stream Backend State to the Frontend

Backend services often expose a `Flow` and let the RPC layer forward it:

```kotlin
@Service(Service.Level.PROJECT)
class BackendChatRepositoryModel {
  private val _messages = MutableStateFlow(listOf<ChatMessage>())

  fun getMessagesFlow(): Flow<List<ChatMessageDto>> {
    return _messages.map { messages -> messages.map(ChatMessage::toChatMessageDto) }
  }
}
```

On the frontend, convert the remote flow into local state:

```kotlin
@Service(Service.Level.PROJECT)
class FrontendChatRepositoryModel(
  private val project: Project,
  coroutineScope: CoroutineScope,
) {
  val messagesFlow: StateFlow<List<ChatMessage>> = flow {
    durable {
      ChatRepositoryRpcApi.getInstance().getMessagesFlow(project.projectId()).collect { valueFromBackend ->
        emit(valueFromBackend.map(ChatMessageDto::toChatMessage))
      }
    }
  }.stateIn(coroutineScope, SharingStarted.Lazily, emptyList())
}
```

### Use Serializable DTOs

Transport objects should stay simple and serializable:

```kotlin
@Serializable
data class ChatMessageDto(
  val id: String,
  val content: String,
  val author: String,
  val isMyMessage: Boolean,
  @Serializable(with = LocalDateTimeSerializer::class)
  val timestamp: LocalDateTime,
  val type: ChatMessageType,
)
```

For unsupported types, provide a custom serializer:

```kotlin
object LocalDateTimeSerializer : KSerializer<LocalDateTime> {
  private val formatter = DateTimeFormatter.ISO_LOCAL_DATE_TIME

  override val descriptor: SerialDescriptor =
    PrimitiveSerialDescriptor("LocalDateTime", PrimitiveKind.STRING)

  override fun serialize(encoder: Encoder, value: LocalDateTime) {
    encoder.encodeString(value.format(formatter))
  }

  override fun deserialize(decoder: Decoder): LocalDateTime {
    return LocalDateTime.parse(decoder.decodeString(), formatter)
  }
}
```

### Send Events from Backend to Frontend

For backend-initiated events, use `ApplicationRemoteTopic` or `ProjectRemoteTopic` instead of a request-response RPC call.

Define the topic and event in the shared module:

```kotlin
@Serializable
data class NewMessageEvent(
  val projectId: ProjectId,
  val messageId: String,
)

val NEW_MESSAGE_TOPIC: ProjectRemoteTopic<NewMessageEvent> =
  ProjectRemoteTopic("chat.newMessage", NewMessageEvent.serializer())
```

Send the event on the backend:

```kotlin
NEW_MESSAGE_TOPIC.send(NewMessageEvent(project.projectId(), userMessage.id))
```

Handle it on the frontend:

```kotlin
class NewMessageEventListener : ProjectRemoteTopicListener<NewMessageEvent> {
  override val topic = NEW_MESSAGE_TOPIC

  override fun handleEvent(event: NewMessageEvent) {
    // React to the event.
  }
}
```

Register the listener in the frontend descriptor:

```xml
<idea-plugin>
  <extensions defaultExtensionNs="com.intellij">
    <platform.rpc.projectRemoteTopicListener
        implementation="org.jetbrains.plugins.template.NewMessageEventListener"/>
  </extensions>
</idea-plugin>
```

## FAQ

### What Can Be Passed Through RPC

All RPC parameters and return values must be serializable.
Useful platform identifiers include:

- `Project.projectId()` and `ProjectId.findProjectOrNull()`
- `Editor.editorId()` and `EditorId.findEditor()`
- `VirtualFile.rpcId()` and `VirtualFileId.virtualFile()`
- `Icon.rpcId()` and `IconId.icon()`

When possible, prefer identifiers over heavyweight platform objects.

### What Causes `AbstractMethodError`

If an RPC stub fails with `AbstractMethodError`, first check that every RPC method in the shared interface is declared as `suspend`.

### How to Transfer Large Binary Data

Wrap large byte arrays in `fleet.rpc.core.Blob` to reduce serialization overhead.
