---
title: Inversion of Control and Dependency Injection in Unity Using Zenject
date: 2026-11-13 12:00:00 +0530
categories: [Game Development, Unity]
tags: [unity, csharp, dependency-injection, ioc, zenject, testing]
redirect_from:
  - /docs/game-development/inversion-of-control-and-dependency-injection-in-unity.html
---

## Introduction

When you're building a game in Unity, the MonoBehaviour singleton pattern is great for prototyping. But once you start adding lots of features, scaling the game, or writing tests, it becomes a problem. The project gets harder to maintain, and the code starts to smell.

In this writeup I'll explain **Inversion of Control (IoC)**, and how **Dependency Injection (DI)** lets you use cleaner patterns (like state patterns) while keeping your project testable, maintainable, and scalable.

![Singleton vs dependency injection overview](https://cdn-images-1.medium.com/max/1080/1*HIstg3DZPmRQrc6jLMXYiw.png)

## Polymorphism and IoC (Inversion of Control)

Let's start with polymorphism. Say your game has an **enemy** that needs to attack the player. You might have different weapon types: a sword, a bow, a magic staff.

The enemy doesn't need to know *how* each weapon works. It just calls `Attack()`, and the weapon takes care of the rest. The sword swings, the bow fires an arrow, the staff casts a spell.

So instead of depending on one specific weapon class, the enemy depends on something general (an interface like `IWeapon`). Any weapon that follows that interface will work, and you can swap them without changing the enemy's code.

IoC is the same idea, but flipped. Instead of an object reaching out to find what it needs, **each object tells the system what it needs**, and the system provides it.

This suits Unity well. Unity creates MonoBehaviours for you, so you don't get to write their constructors. With DI, a MonoBehaviour can declare what it needs and have it handed over.

It's a much better pattern, and it opens the door to other useful patterns. That's the core idea of dependency injection: your class doesn't know *which* implementation it gets, only that it's something that works for the system.

![Enemy depending on an IWeapon interface](https://cdn-images-1.medium.com/max/1080/1*ItXBHHpxNCLD-tPdm9002A.png)

## Composition Root

With DI, each system is created and then **installed in the composition root**. The composition root keeps track of all your systems by their **interface types**.

So when a class asks for an `IGuiCamera` or an `IMusicService`, the container simply hands over whatever you installed.

You can build your own implementation, or use a library like **Zenject** (free and open source). In upcoming writeups we will see how to implement your own using Zenject.

![Composition root providing dependencies](https://cdn-images-1.medium.com/max/1080/1*tQ_q2DDUxWzgpDRtOFvcwg.png)

For example, a class `GameZone` asks for camera, music, and time. Whatever is installed in the composition root for those interfaces gets passed to the class that asked for it.

## Why This Is Better

- **No singletons everywhere.** All your systems are managed by the composition root, which means fewer chances to make mistakes.
- **No script execution order headaches.** Without DI, you end up relying on Unity's Script Execution Order to make sure things initialize correctly. Anyone who has done that knows how error-prone it is. With DI, dependencies are resolved for you.

![Dependencies resolved by the container](https://cdn-images-1.medium.com/max/1080/1*c1w9K76HWqPGm8bwtJ_o2w.png)

## Testing

Because it's easy to mock things in a composition root and scaffold around your code, **unit testing becomes easy**.

Testing matters. Many people think it slows you down, but consider this: System A is well tested, and you're making changes to System B. With tests in place, you can be sure you aren't breaking System A. Without them, you won't find out until QA does, and fixing it later costs far more.

![Unit testing with injected dependencies](https://cdn-images-1.medium.com/max/1080/1*qfrOyy0cq73RthZm2CyVaQ.png)

You can also integrate **NSubstitute**. You don't have to write mock classes by hand. Just create substitutes for the dependencies, inject them, and run your test.

![Mocking dependencies with NSubstitute](https://cdn-images-1.medium.com/max/720/1*RGcklGiYECHfEEbxVNH0mw.png)

## The Issue

This is all great, but you can end up with **one giant prefab** (the composition root setup), which is a nightmare whenever anyone edits it: you get merge conflicts and all sorts of problems.

In the next post I'll cover how to fix this. We want to split things up as small as possible, and we'll do that using **game states**.