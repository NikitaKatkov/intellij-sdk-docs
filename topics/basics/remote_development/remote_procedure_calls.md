<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Remote Procedure Calls (RPC)

<link-summary>Set up RPC between shared, frontend, and backend plugin modules in Split Mode.</link-summary>

This article walks through how remote calls (RPC) are set up in [Split Mode](split_mode_for_remote_development.md) and refers to code in the publicly available [modular plugin template](https://github.com/JetBrains/intellij-platform-modular-plugin-template). The plugin is split into three modules: **shared, frontend, and backend**. Let’s start with an explanation of how the module dependencies are configured.

### Shared module

The shared module defines the RPC interface. It needs the `rpc` and `kotlinx.serialization` plugins:

```kotlin
// shared/build.gradle.kts
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

### Frontend module

The frontend module depends on `:shared` and needs the `rpc` plugin as well:

```kotlin
// frontend/build.gradle.kts
plugins {
  id("rpc")
  id("org.jetbrains.kotlin.jvm")
  id("org.jetbrains.kotlin.plugin.serialization")
}
dependencies {
  intellijPlatform {
    intellijIdea(libs.versions.intellij.platform)
    bundledModule("intellij.platform.frontend")
    // ...
  }
  compileOnly(project(":shared"))
}
```

### Backend module

The backend module depends on `:shared` and requires `intellij.platform.kernel.backend` and `intellij.platform.rpc.backend`:

```kotlin
// backend/build.gradle.kts
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

Also declare the backend platform module dependency in `modular.plugin.backend.xml`:

```xml
<!-- backend/src/main/resources/modular.plugin.backend.xml -->
<idea-plugin>
    <dependencies>
        <module name="intellij.platform.backend"/>
        <module name="intellij.platform.kernel.backend"/>
        <module name="modular.plugin.shared"/>
    </dependencies>
    <!-- ... -->
</idea-plugin>
```

## Create an RPC Interface

Introduce the RPC interface in the shared module:

```kotlin
// shared/src/main/kotlin/org/jetbrains/plugins/template/ChatRepositoryRpcApi.kt
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

The rules for creating an RPC interface are:

1. Add `@Rpc` annotation to the interface.
2. All functions must be `suspend` (https://kotlinlang.org/docs/coroutines-basics.html\#suspending-functions).
3. All parameters and return types must be `@Serializable`. They are essentially data transfer objects (DTOs) one might be familiar with from client-server app development.
    * Primitives, `String`, `Flow`, `Deferred` are serializable by default.
    * Enums are not serializable by default. Mark them as `@Serializable` explicitly.
    * Classes must be annotated with `@Serializable` and must contain only other serializable fields
4. Introduce `suspend getInstanceAsync()` so the frontend can easily acquire the instance.
5. Implement the RPC interface on the backend.

Add a class implementing the RPC interface in the backend module:

```kotlin
// backend/src/main/kotlin/org/jetbrains/plugins/template/BackendChatRepositoryRpcApi.kt
class BackendChatRepositoryRpcApi : ChatRepositoryRpcApi {
    override suspend fun getMessagesFlow(projectId: ProjectId): Flow<List<ChatMessageDto>> {
        val backendProject = projectId.findProjectOrNull() ?: return emptyFlow()
        return BackendChatRepositoryModel.getInstance(backendProject).getMessagesFlow()
    }

    override suspend fun sendMessage(
        projectId: ProjectId,
        messageContent: String
    ) {
        val backendProject = projectId.findProjectOrNull() ?: return
        return BackendChatRepositoryModel.getInstance(backendProject).sendMessage(messageContent)
    }
}
```

Implement `RemoteApiProvider`, which registers the RPC implementation with the platform:

```kotlin
// backend/src/main/kotlin/org/jetbrains/plugins/template/BackendRpcApiProvider.kt
internal class BackendRpcApiProvider : RemoteApiProvider {
    override fun RemoteApiProvider.Sink.remoteApis() {
        remoteApi(remoteApiDescriptor<ChatRepositoryRpcApi>()) {
            BackendChatRepositoryRpcApi()
        }
    }
}
```

Register the provider in `modular.plugin.backend.xml`:

```xml
<!-- backend/src/main/resources/modular.plugin.backend.xml -->
<idea-plugin>
    <dependencies>
        <module name="intellij.platform.backend"/>
        <module name="intellij.platform.kernel.backend"/>
        <module name="modular.plugin.shared"/>
    </dependencies>

    <extensions defaultExtensionNs="com.intellij">
        <platform.rpc.backend.remoteApiProvider
            implementation="com.example.BackendRpcApiProvider"/>
    </extensions>
</idea-plugin>
```

## Use RPC on Frontend

Now you can call `ChatRepositoryRpcApi` on the frontend:

```kotlin
ChatRepositoryRpcApi.getInstance().getMessagesFlow(project.projectId())
ChatRepositoryRpcApi.getInstance().sendMessage(project.projectId(), messageContent)
```

Note that all `getInstanceAsync()`, `getMessagesFlow()`, and `sendMessage()` functions are `suspend`, so they must be called in some coroutine context.

Implementation detail:

If some problem occurs while trying to execute the RPC, the call will fail with a RpcClientException. For instance, this may happen if the client tries to execute the call while the backend is not fully initialized, if a network problem occurs, or if the backend is restarted while a call is being executed.

Such errors can be mitigated by using the `fleet.rpc.client.DurableKt.durable` wrapper function.

```kotlin
durable {
   ChatRepositoryRpcApi.getInstanceAsync().getMessagesFlow(project.projectId()).collect { valueFromBackend ->
	... // process the message
   }
}
```

It will retry the call in case discovery of the backend RPC implementation fails. Consider employing it especially when working with long-lived RPC flows, so that an exception there is handled properly and the corresponding feature keeps working.

## RPC Examples

### Subscription to the Backend State

Let’s have a look at the reference implementation from the [modular plugin template](https://github.com/JetBrains/intellij-platform-modular-plugin-template).

There, `BackendChatRepositoryModel` holds a `MutableStateFlow` of messages on the backend:

```kotlin
// backend/src/main/kotlin/org/jetbrains/plugins/template/BackendChatRepositoryModel.kt
@Service(Service.Level.PROJECT)
class BackendChatRepositoryModel {
    private val _messages = MutableStateFlow(listOf(/* initial messages */))

    fun getMessagesFlow(): Flow<List<ChatMessageDto>> {
        return _messages.map { messagesList -> messagesList.map(ChatMessage::toChatMessageDto) }
    }

    suspend fun sendMessage(messageContent: String) {
        // appends to _messages, triggering flow emissions
    }
}
```

The RPC implementation simply delegates to this service:

```kotlin
// backend/src/main/kotlin/org/jetbrains/plugins/template/BackendChatRepositoryRpcApi.kt
class BackendChatRepositoryRpcApi : ChatRepositoryRpcApi {
    override suspend fun getMessagesFlow(projectId: ProjectId): Flow<List<ChatMessageDto>> {
        val backendProject = projectId.findProjectOrNull() ?: return emptyFlow()
        return BackendChatRepositoryModel.getInstance(backendProject).getMessagesFlow()
    }
}
```

On the frontend, `FrontendChatRepositoryModel` subscribes to this flow and exposes it as a `StateFlow`:

```kotlin
// frontend/.../FrontendChatRepositoryModel.kt
@Service(Level.PROJECT)
class FrontendChatRepositoryModel(
    private val project: Project,
    coroutineScope: CoroutineScope
) : ChatRepositoryApi {
    override val messagesFlow: StateFlow<List<ChatMessage>> = flow {
        durable {
            ChatRepositoryRpcApi.getInstanceAsync().getMessagesFlow(project.projectId()).collect { valueFromBackend ->
                val mappedValue = valueFromBackend.map { messageDto -> messageDto.toChatMessage() }
                emit(mappedValue)
            }
        }
    }.stateIn(coroutineScope, initialValue = emptyList(), started = SharingStarted.Lazily)
}
```

As you can see, `messagesFlow` is initialized with an empty list.
Since services are initialized lazily, the first subscriber will trigger the RPC connection, but the state will be `emptyList()` until the first backend emission arrives.

If this matters for your feature, make sure the service is initialized before its first use — for example, via subscribing to updates from the backend inside a dedicated `ProjectActivity`.

### Use Serializable DTOs for RPC Transport

Domain objects are not always directly serializable for transport over RPC.
In this plugin, `ChatMessage` is the domain model used on both sides, but it contains `LocalDateTime`, which is not natively supported by `kotlinx.serialization`.

The solution is a dedicated DTO class in the shared module:

```kotlin
// shared/src/main/kotlin/org/jetbrains/plugins/template/dtos.kt
@Serializable
data class ChatMessageDto(
    val id: String,
    val content: String,
    val author: String,
    val isMyMessage: Boolean,
    @Serializable(with = LocalDateTimeSerializer::class)
    val timestamp: LocalDateTime,
    val type: ChatMessage.ChatMessageType
)

fun ChatMessageDto.toChatMessage(): ChatMessage { /* ... */ }
fun ChatMessage.toChatMessageDto(): ChatMessageDto { /* ... */ }
```

Custom `KSerializer` implementations can handle types that are not natively serializable:

```kotlin
// shared/src/main/kotlin/org/jetbrains/plugins/template/serializers.kt
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

The RPC interface uses the DTO, and each side converts to and from the domain model as needed:

* Backend: converts `ChatMessage -> ChatMessageDto` before emitting via `getMessagesFlow()`
* Frontend: converts `ChatMessageDto -> ChatMessage` after receiving via `toChatMessage()`

### Send Events from Backend to Frontend

When you need to push events from the backend to the frontend without an explicit request, use `ApplicationRemoteTopic` or `ProjectRemoteTopic` instead of a regular RPC call.

1. Define topic and event in the shared module

```kotlin
// shared module
@Serializable
data class NewMessageEvent(
    val projectId: ProjectId,
    val messageId: String
)

val NEW_MESSAGE_TOPIC: ProjectRemoteTopic<NewMessageEvent> =
    ProjectRemoteTopic("chat.newMessage", NewMessageEvent.serializer())
```

2. Send events from the backend

```kotlin
// backend module
class BackendChatRepositoryModel {
    suspend fun sendMessage(messageContent: String) {
        val userMessage = chatMessageFactory.createUserMessage(messageContent)
        _messages.value += userMessage

        NEW_MESSAGE_TOPIC.send(NewMessageEvent(project.projectId(), userMessage.id)
    }
}
```

3. Handle events on the frontend

```kotlin
// frontend module
class NewMessageEventListener : ProjectRemoteTopicListener<NewMessageEvent> {
    override val topic = NEW_MESSAGE_TOPIC

    override fun handleEvent(event: NewMessageEvent) {
        // React to the new message event
    }
}
```

4. Register the listener via extension point

In `modular.plugin.frontend.xml`:

```xml
<extensions defaultExtensionNs="com.intellij">
    <platform.rpc.projectRemoteTopicListener
        implementation="org.jetbrains.plugins.template.NewMessageEventListener"/>
</extensions>
```

You can read more about this approach in the `ApplicationRemoteTopic` and `ProjectRemoteTopic` KDocs.

## FAQ

**Q:** What classes can be passed through RPC?

**A:** All parameters and returned values must be `@Serializable`.
You can read more about `kotlinx.serialization` in the [Kotlin serialization documentation](https://kotlinlang.org/docs/serialization.html).

* Primitives, `String`, `Flow`, `Deferred` are serializable by default.
* Enums are not serializable by default. Mark them as `@Serializable` as well.
* For types like `LocalDateTime`, implement a custom `KSerializer` — see `LocalDateTimeSerializer` in this project.

IntelliJ Platform provides a way to pass some commonly used classes through RPC:

1. `Project` can be serialized and deserialized by `Project.projectId()` and `ProjectId.findProject()` functions.
   This plugin passes `ProjectId` to every RPC method so the backend can resolve the correct project instance.
2. `Editor` can be serialized and deserialized by `Editor.editorId()` and `EditorId.findEditor()` functions.
3. `VirtualFile` can be serialized and deserialized by `VirtualFile.rpcId()` and `VirtualFileId.virtualFile()` functions.
4. `Icon` can be serialized and deserialized by `Icon.rpcId()` and `IconId.icon()` functions.

Note that these objects are not fully serializable, so the frontend only receives parts of the backend object.
If possible, use only IDs on the frontend and work with the full objects on the backend side.

**Q:** What to do with AbstractMethodError?

```
java.lang.AbstractMethodError: Receiver class InterfaceApiClientStub
does not define or inherit an implementation of the resolved method
'abstract void foo()' of interface InterfaceApi.
```

**A:** Make sure that all the functions in the interface are `suspend`.

**Q:** How to efficiently transfer `byte[]`

**A:** Wrap the data into a `fleet.rpc.core.Blob` for the sake of reducing the serialization overhead.
