---
author: "alessio-paoletti"
layout: post
did: "blog11"
title: "Multiple game viewports in Unreal Engine"
slug: "MultipleViewportWithCustomScenes"
date: 2020-07-24 01:00:00
categories: graphics
image: images/banners/bim-int-rendering.png
tags: graphics
description: "How to render multiple game viewports with custom scenes in Unreal Engine."
---

*What if you want to have a new viewport to display 3D content in an Unreal Engine application?*

There are some examples out there - like this one [UE4 Plugin for scene capture rendering to a separate window](http://www.geodesic.games/2019/02/28/ue4-plugin-for-scene-capture-rendering-to-separate-window/)) - but they usually render the same application's scene from another point of view, using a scene capture component.

But *what if you want to display a completely new and detached world in the new viewport?*

In this post, I will show you how I have achieved this in **Unreal Engine 4**.

<br>

## Widget and Viewport creation

First, let's create a widget that will contain the new viewport.

Our widget inherits from `SCompoundWidget`, which is the base from which most widgets should be built. Her we need to create a viewport and a viewport client: the `construct` parent function is the place where this happens:

```c++
class SMyWidget : public SCompoundWidget {
public:
    SLATE_BEGIN_ARGS(SMyWidget) {}
    SLATE_END_ARGS()

    ~SMyWidget() {}

    // Begin SWidget override
    void Construct(const FArguments& InArgs);
    virtual void Tick(const FGeometry& AllottedGeometry, const double InCurrentTime, const float InDeltaTime) override;
    // End SWidget override
};

SMyWidget::construct(...)
{
    Viewport = SNew(SViewport);

    ViewportClient = NewObject<UMyGameViewportClient>(...);

    ...

    SceneViewport = new FSceneViewport(ViewportClient, Viewport));
    ViewportClient->Viewport = SceneViewport;
    ViewportClient->SetViewportFrame(SceneViewport->GetViewportFrame());

    Viewport->SetViewportInterface(SceneViewport);

    FSlateApplication::Get().RegisterGameViewport(Viewport);

    this->ChildSlot[Viewport];
}

SMyWidget::Tick(...)
{
    // Call the scene draw that calls the FViewportClient draw
    SceneViewport->Draw();
}
```

A scene viewport is created with the `SViewport` and the `ViewportClient` objects.
This object is responsible for drawing the viewport calling `SceneViewport->Draw()` at event tick of the widget. This will call the `Draw(...)` function of the viewport client, which is the one that actually render the scene. We will discuss this in the next section.

Let's create a window to display the widget:

```c++
// Window creation
window = SNew(SWindow)
    .Title(FText::FromString("EditorWindow"))
    .Type(EWindowType::GameWindow)

FSlateApplication::Get().AddWindow(window);

// Set the widget to be displayed in the window
window->SetContent(myWidget)
```

Now, let's see what `UMyGameViewportClient` is and why we need it.

<br>

## Viewport Client setup

The **GameViewportClient** is the engine's interface to a game viewport, an high-level class for the platform specific rendering, audio, and input subsystems.

We would like to have the same rendering process of our game, but we need a different scene and an independent view from the one in the game instance.

So what we need to do is override the `World` member to set our own world and override the `FinalizeViews(...)` function to set our own (otherwise, the client would get the view from the controller that is set in the current game instance, and the newly created viewport would move together with the main one).

This is a little bit hacky, and definitely not a good performance wise choice, but the problem is that as the documentation says:

> exactly one GameViewportClient is created for each instance of the game. The only case (so far) where you might have a single instance of Engine, but multiple instances of the game (and thus multiple GameViewportClients) is when you have more than one PIE window running

So overriding `FinalizeViews(...)` is the best solution I've found at the moment to achieve what we want without modifying the engine. Note that having another player controller is not a solution since it will create a split screen viewport, that is not what we want. We will use the same player controller.

Here is a sample of the class:

```c++
class UMyGameViewportClient : public UGameViewportClient
{
public:
    // Begin UGameViewportClient override
    virtual void FinalizeViews(class FSceneViewFamily* ViewFamily, const TMap<ULocalPlayer*, FSceneView*>& PlayerViewMap) override;

    virtual bool InputKey(const FInputKeyEventArgs& EventArgs) override;
    virtual bool InputAxis(
        FViewport* InViewport, int32 ControllerId, FKey Key, float Delta, float DeltaTime, int32 NumSamples = 1, bool bGamepad = false) override;
    // End UGameViewportClient override

    void SetWorld(UWorld* InWorld) { World = InWorld; }

    virtual void FinalizeViews(class FSceneViewFamily* ViewFamily, ...) override;
    ...
};

void UMyGameViewportClient::FinalizeViews(FSceneViewFamily* ViewFamily, ...) {
    for (int32 ViewIndex = 0; ViewIndex < ViewFamily->Views.Num(); ++ViewIndex) {
        // Remove the current view
        ViewFamily->Remove(View);

        // Setup the view options manually
        FSceneViewInitOptions ViewInitOptions;
        ViewInitOptions.ViewOrigin = ...;
        ViewInitOptions.ViewRotationMatrix = ...;
        ViewInitOptions.ProjectionMatrix = ...;

        // Create a new view with our options and overwrite the existing one
        FSceneView* NewView = new FSceneView(ViewInitOptions);
        ViewFamily->Add(NewView);
    }
}

bool UMyGameViewportClient::InputKey(const FInputKeyEventArgs& EventArgs) {
    if (EventArgs.Key == EKeys::MouseScrollUp) {
        // handle the zoom
    }
    if (EventArgs.Key == EKeys::W) {
        // Move forward
    }

    ...

    return true;
}

bool UMyGameViewportClient::InputAxis(FViewport* InViewport, int32 ControllerId, FKey Key, float Delta, float DeltaTime, int32 NumSamples, bool bGamepad) {
    const float mouseDragX = (Key == EKeys::MouseX) ? Delta : 0.f;
    const float mouseDragY = (Key == EKeys::MouseY) ? Delta : 0.f;

    // handle your view options here

    return true;
}
```

The `SetWorld(UWorld* InWorld)` is essential to let us override the world of the viewport with our custom scene.

<br>

## Creating the custom scene

With the term *custom scene* I mean a collection of meshes, actors, lights that are not in the actual game's world.

The Unreal Engine class that represent a scene on the render thread is the `FScene` class, that has its own replication on the game thread called `UWorld`.

To create a scene by code, you need to create your own world that needs to be registered in a new world context. When initialized, the world object will allocate a scene for the render thread, and that is the one that you will use for your purposes.

Basically, you can have a scene that is a simple class like:

```c++
class  FCustomScene {
private:
    UWorld* _customWorld;
    TArray<UActorComponent*> _allComponents;

public:
    FCustomScene() {
        _customWorld = NewObject<UWorld>(...);
        _customWorld->WorldType = EWorldType::Game;

        FWorldContext& WorldContext = GEngine->CreateNewWorldContext(_customWorld->WorldType);
        WorldContext.SetCurrentWorld(_customWorld);

        _customWorld->InitializeNewWorld(...);
        _customWorld->InitializeActorsForPlay(...);
    }

    virtual ~FCustomScene() {
        // Remove all the attached components
        for (auto component : _allComponents) {
            component->UnregisterComponent();
        }
        _allComponents.Clear();

        _customWorld->CleanupWorld();
        GEngine->DestroyWorldContext(_customWorld);
    }

    void AddComponent(class UActorComponent* component) {
        _allComponents.AddUnique(component);
        component->RegisterComponentWithWorld(_customWorld);
    }

    UWorld* GetWorld() const { return _customWorld; }
};
```

And so now, we can set the `ViewportClient`'s world to the one we've just created. In the `SMyWidget` construct, we can set our new world:

```c++
FCustomScene* newScene = new FCustomScene();
ViewportClient->SetWorld(newScene->GetWorld());
```

The easiest example of how to build a scene from scratch is the `FPreviewScene` source file within the engine. This is the base class for the more complicated `FAdvancedPreviewScene`, that is the one used in the material editor to display the material's preview on a mesh.
