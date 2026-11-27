---
title: "From SDK Dump to Trainer: How Unreal Engine Games Give Themselves Away"
date: 2026-11-27 12:00:00 -0800
categories: [Guides, Game Reversing]
tags: [unreal-engine, reverse-engineering, dumper-7, cpp, dll, single-player]
description: Why almost every Unreal Engine game ships a map of its own internals, how Dumper-7 turns that into a C++ SDK, and how that SDK becomes an internal DLL trainer for a single-player game.
---

Most reverse engineering starts with a binary and no names. You stare at `sub_14183F2A0` and work out what it does one instruction at a time.

Unreal Engine games are different. Almost every one of them ships with the name, type, and memory offset of nearly every class, field, and function the game uses. Nobody leaks these. The engine needs them to run. Once you understand why, turning a UE game into a working trainer stops being guesswork.

This post is the overview: what the engine exposes, what an SDK dumper does with it, and how the result becomes an internal DLL. It isn't a full template. That comes later.

> Everything here is about **single-player games** you own, for your own use. Don't use it in online games, against other players, or to get around anti-cheat. That's where this stops being reversing and starts being someone else's problem.
{: .prompt-warning }

## Why Unreal Engine exposes itself

Unreal has a **reflection system**. Every gameplay class derives from `UObject`, and the engine keeps a description of each one at runtime:

- **`UClass`** describes a class: its parent, its size, its properties, and its functions.
- **`FProperty`** describes one field: its name, its type, and its byte offset inside the object.
- **`UFunction`** describes a callable function, including the layout of its parameters.

Unreal keeps all of this around because it uses it constantly. Blueprints call C++ functions by name. Save games and network replication walk properties by offset. The editor shows them in the details panel.

Two global tables make it reachable:

- **`GObjects`** (`FUObjectArray`): every live `UObject`, including every `UClass` and `UFunction`.
- **`GNames`** (`FNamePool` in UE4.23+ and UE5): the string table behind every `FName`, so you can turn the IDs stored in objects back into text like `Health` or `BP_Zombie_C`.

Find those two, and you can walk the entire game's type system.

## What a dumper does

[Dumper-7](https://github.com/Encryqed/Dumper-7) is an SDK generator you inject into the running game. It:

1. Finds `GObjects`, the name table, the `AppendString` function that turns an `FName` into a string, and the vtable slot of `ProcessEvent` (more on that below).
2. Walks every `UClass`, `UScriptStruct`, `UEnum`, and `UFunction`.
3. Writes them out as compilable C++ headers: real classes with real member offsets, and wrappers for every reflected function.

You get folders like `CppSDK/SDK/` with `Basic.hpp`, `CoreUObject_classes.hpp`, `Engine_classes.hpp`, and the game's own `<Game>_classes.hpp`. A generated class looks roughly like this (simplified):

```cpp
// Class Engine.Pawn
// 0x0068 (0x0328 - 0x02C0)
class APawn : public AActor
{
public:
    uint8                Pad_2C0[0x8];
    class APlayerState*  PlayerState;    // 0x02C8(0x0008)
    // ...
    class AController*   Controller;     // 0x02F8(0x0008)
};
```

Every field comes with its offset, and the headers carry `static_assert`s that check those offsets at compile time. Read `Engine_classes.hpp` and `<Game>_classes.hpp` like documentation. Search for words like `Health`, `Ammo`, `Stamina`, or `Inventory`, and you'll usually find what you want in minutes.

## Turning the SDK into a DLL

The project itself is ordinary C++:

1. **New Visual Studio project:** a Dynamic-Link Library, x64, C++20, Release.
2. **Add the SDK:** include `SDK.hpp` and compile the `.cpp` files you actually use. You almost always need `Basic.cpp`, `CoreUObject_functions.cpp`, and `Engine_functions.cpp`. Add the game's `_functions.cpp` when you call its functions.
3. **Add your libraries:** [MinHook](https://github.com/TsudaKageyu/minhook) for hooking, and [Dear ImGui](https://github.com/ocornut/imgui) for the menu.
4. **`DllMain`** starts a thread that runs your setup, so you don't do heavy work while the loader lock is held.

From there, getting to your own player is a short pointer walk through the SDK's own types:

```cpp
SDK::UWorld* World = SDK::UWorld::GetWorld();
if (!World || !World->OwningGameInstance) return;

auto& LocalPlayers = World->OwningGameInstance->LocalPlayers;
if (LocalPlayers.Num() == 0) return;

SDK::APlayerController* PC = LocalPlayers[0]->PlayerController;
if (!PC) return;

SDK::APawn* Me = PC->AcknowledgedPawn;   // the pawn you're controlling
```

Every link in that chain can be null during loading screens, menus, and level transitions. Check each one, every frame.

## Calling the game's own functions

Reading and writing fields is only half of it. Every reflected function in the SDK is a wrapper around **`ProcessEvent`**, the engine's one entry point for calling a `UFunction` by reflection. So you can use the game's own code instead of rebuilding it:

```cpp
SDK::FVector Pos = Me->K2_GetActorLocation();

SDK::FVector2D Screen;
if (PC->ProjectWorldLocationToScreen(Pos, &Screen, false))
{
    // Pos is on screen at Screen.X, Screen.Y: draw a box or label there
}
```

That's world-to-screen for an ESP box without writing any matrix math. The same pattern gets you teleporting (`K2_TeleportTo`), line-of-sight checks, and most of what the game's Blueprints can do.

For enemies and items, walk the current level's actor list and filter by class:

```cpp
SDK::ULevel* Level = World->PersistentLevel;
for (SDK::AActor* Actor : Level->Actors)
{
    if (!Actor || !Actor->IsA(SDK::ACharacter::StaticClass())) continue;
    // ...
}
```

## Drawing a menu

To draw anything, hook the swap chain's `Present` (DirectX 11 or 12) with MinHook, set up ImGui on the first call, and render your menu and ESP in that hook every frame. That's a post of its own. The short version: DX11 is straightforward, while DX12 needs a command queue and per-backbuffer resources, so expect to spend more time there.

## Lessons from real projects

None of this is in the README. All of it came from things breaking.

### Do game work on the game thread

The `Present` hook runs on the render thread. Reading there is mostly fine. Writing gameplay values from it can crash the game. In one project, writing a replicated field fired its `OnRep` callback on the wrong thread and caused access violations.

The fix is to hook `ProcessEvent` itself and do your writes when you see a per-frame function like `ReceiveTick` go by. Then your changes happen on the game thread, the same way the game's own code runs.

### Don't trust the ProcessEvent index blindly

Dumpers find `ProcessEvent` by scanning the `UObject` vtable. On one game, the scan accepted the wrong slot (index `0x6`). The dump still "passed" its health checks, but the game name came back blank, the version came back as `UnknownVersion`, and every SDK function call went to the wrong place. Re-dumped correctly, the real index was `0x49`.

If the game name or version is empty, or the index is suspiciously low, check the vtable before you trust any function call.

### Updates move globals, not much else

On one game, I diffed the dumps across a patch: **all six global addresses moved**, but only **171 of about 54,000 class members** did. In practice, that means:

- Find globals like `GObjects` at injection time with **signatures** (byte patterns with the addresses wildcarded), not hardcoded offsets.
- For the handful of fields you use, look up the offset through reflection at runtime (`FProperty::Offset`) instead of compiling it in. Then a small update doesn't break the trainer.

### When the SDK won't compile, read the asserts

The `static_assert`s are your friend. When thousands of them fail at once, the dumper got one basic thing wrong for that build, not thousands of things. Twice that happened to me:

- **Enum sizes guessed wrong:** about 1,500 enums came out the wrong size, which shifted every struct that contained one.
- **A name lookup mismatched on case:** the game had registered `assetpath` instead of `AssetPath`, so a core type was missing from `Basic.hpp`.

Both were dumper bugs, fixed once in the dumper instead of patched by hand in every SDK.

### Blueprint classes are where the game lives

Classes ending in `_C` (like `BP_Zombie_C`) are Blueprint-generated, and that's often where game-specific data actually sits. Their dumped headers can look nearly empty because the useful fields come from a C++ parent class. Read up the inheritance chain before you decide a field doesn't exist.

### Some stats aren't plain numbers

Many newer UE games use the **Gameplay Ability System**, where health and similar stats are *attributes* owned by an `AbilitySystemComponent`, with a base value and a current value. Writing the float directly often gets overwritten on the next update. Go through the attribute system the way the game does.

## Where to go from here

A dump turns a binary into something closer to source code. From there, a single-player trainer is mostly ordinary C++, plus care about threads, null pointers, and game updates.

Next time, I'll go through a full project template: the DLL entry point, the DX11 and DX12 `Present` hooks, the ImGui menu, and the `ProcessEvent` hook that keeps game writes on the game thread.
