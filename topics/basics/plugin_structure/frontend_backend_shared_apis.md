<!-- Copyright 2000-2026 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Frontend, Backend, and Shared APIs

<link-summary>Common IntelliJ Platform APIs that usually belong on the frontend, backend, or shared side of a split-aware plugin.</link-summary>

This page lists APIs that commonly indicate where plugin code should live in a split-aware architecture.
The lists are intentionally conservative.
Treat them as placement defaults rather than an exhaustive type system.
If a feature needs a different placement, review the specific API behavior and validate the result in Split Mode.

The IntelliJ DevKit plugin is expected to provide inspections for inappropriate API usage.

See also [](modular_plugins.md) and [](split_mode_feature_development.md).

## Frontend APIs

The following APIs usually belong in the frontend module:

- `com.intellij.openapi.wm.ToolWindowFactory`
- `com.intellij.openapi.wm.ToolWindow`
- `com.intellij.openapi.editor.toolbar.floating.FloatingToolbarProvider`
- `com.intellij.openapi.ui.DialogWrapper`
- `com.intellij.openapi.fileEditor.FileEditorProvider`
- `com.intellij.openapi.ui.popup.JBPopupFactory`
- `com.intellij.codeInsight.editorActions.TypedHandlerDelegate`
- `com.intellij.codeInsight.editorActions.enter.EnterHandlerDelegate`
- `com.intellij.codeInsight.editorActions.enter.EnterHandlerDelegateAdapter`
- `com.intellij.codeInsight.highlighting.BraceMatcher`
- `com.intellij.openapi.fileEditor.FileEditorManager.getFocusedEditor()`
- `com.intellij.execution.services.ServiceViewContributor`
- `com.intellij.ui.tree.AsyncTreeModel`
- `com.intellij.ui.jcef.JBCefClient`
- `com.intellij.notification.NotificationGroup`
- `com.intellij.openapi.wm.StatusBarWidgetFactory`
- `com.intellij.openapi.options.Configurable`
- `com.intellij.codeInsight.editorActions.CopyPastePostProcessor`
- `com.intellij.openapi.editor.actionSystem.EditorActionHandler`
- `com.intellij.openapi.editor.event.CaretListener`
- `com.intellij.openapi.editor.event.EditorMouseMotionListener`
- `com.intellij.openapi.editor.event.EditorMouseListener`
- `com.intellij.openapi.editor.event.SelectionListener`
- `com.intellij.openapi.editor.event.VisibleAreaListener`
- `com.intellij.openapi.editor.ex.FoldingListener`
- `com.intellij.openapi.ui.popup.JBPopupListener`
- `com.intellij.openapi.actionSystem.CommonDataKeys.EDITOR`
- `com.intellij.openapi.actionSystem.PlatformDataKeys.TOOL_WINDOW`
- `com.intellij.openapi.actionSystem.PlatformCoreDataKeys.SELECTED_ITEM`
- `com.intellij.openapi.actionSystem.PlatformCoreDataKeys.SELECTED_ITEMS`
- `com.intellij.openapi.actionSystem.PlatformCoreDataKeys.CONTEXT_COMPONENT`

## Backend APIs

The following APIs usually belong in the backend module:

- `com.intellij.openapi.vfs.VirtualFileManager`
- `com.intellij.openapi.vfs.VfsUtil`
- `com.intellij.psi.stubs.StubIndex`
- `com.intellij.util.indexing.IndexExtension`
- `com.intellij.execution.configurations.ConfigurationType`
- `com.intellij.psi.search.searches.ReferencesSearch`
- `com.intellij.psi.search.SearchService`
- `com.intellij.util.Query`
- `com.intellij.ide.FileIconProvider`
- `com.intellij.codeInsight.hints.InlayHintsProvider`
- `com.intellij.codeInsight.hints.declarative.InlayHintsProvider`
- `com.intellij.codeInsight.daemon.LineMarkerProvider`
- `com.intellij.codeInspection.LocalInspectionTool`
- `com.intellij.psi.PsiReferenceContributor`
- `com.intellij.codeInsight.completion.CompletionContributor`
- `com.jetbrains.jsonSchema.extension.JsonSchemaProviderFactory`
- `com.intellij.execution.actions.RunConfigurationProducer`
- `com.intellij.ide.structureView.StructureViewBuilder`
- `com.intellij.openapi.vfs.AsyncFileListener`
- `com.intellij.openapi.vfs.VirtualFileListener`
- `com.intellij.platform.backend.workspace.WorkspaceModelChangeListener`
- `com.intellij.openapi.project.ProjectManagerListener`
- `com.intellij.execution.process.ProcessHandler`
- `com.intellij.openapi.actionSystem.CommonDataKeys.SYMBOLS`
- `com.intellij.openapi.actionSystem.PlatformCoreDataKeys.MODULE`
- `com.intellij.openapi.actionSystem.LangDataKeys.MODULE_CONTEXT`
- `com.intellij.openapi.actionSystem.LangDataKeys.MODIFIABLE_MODULE_MODEL`

## Shared APIs

The following APIs are commonly used in shared code:

- `com.intellij.lang.ParserDefinition`
- `com.intellij.openapi.util.registry.RegistryValueListener`
- `com.intellij.openapi.application.ApplicationListener`
- `com.intellij.openapi.project.ProjectListener`
- `com.intellij.ide.plugins.DynamicPluginListener`
- `com.intellij.openapi.fileEditor.FileEditorManagerListener`
- `com.intellij.openapi.editor.event.DocumentListener`

Shared code should stay lightweight.
If shared logic starts depending on frontend-only or backend-only APIs, it usually belongs in a dedicated content module instead.
