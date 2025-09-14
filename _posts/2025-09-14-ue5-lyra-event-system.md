---
layout: post
title:  "How Unreal Engine's Lyra Handles Game Events"
date: 2025-09-14
categories: [game-development]
tags: [unrealengine, c++]
---

## Unreal Engine's Game Message System

Unreal Engine has one of the most elegant architecture designs I seen. Probably I will look more inside Unreal Engine and write about it in the future. In this post, we will turn our microscopes towards to Unreal Engine Lyra's Game Message System. It's more advanced way of game events or observer pattern. However, rather than trying to understand how it works, this writing will try to understand how the developers came up with this design. I believe such understanding is more important than the system itself.

### What is Unreal Engine Lyra Starter Game Project

Lyra is a simple gameplay project by Unreal Engine itself that aims to show good practicies for game development. Lyra is a multiplayer third person shooter, it works very well and robust and contains very smartly designed systems such as `GAS` and `Gameplay Message System` and more.

### Understanding Gameplay Message System

Either be an game developer or software engineer, observer pattern is a common design pattern we all know. Well if you don't know what's it check it out on YouTube really quick. In game development it's also often seen as game events.

- Unreal Engine decided to name it as `Message System` or `Gameplay Message Subsystem`.
- Comparing to classic observer pattern, It uses Tags as event channels. We will also talk about what are Tags.
- It decouples the systems. Player health UI, score UI, respawn system shouldn't know about the player. With the message subsytem they do not care about the player but just the player events.
- For example death of the player may trigger an event over `Game.Player.Death` channel, and all UIs and systems that listens this event will react to player's death.
- System is very generic and Lyra use it for all kinds of communication between its systems.

## Why we Need Event System in the First Place

Starting with two simple systems of `Player` and the `Respawn Manager`. At the moment there are nothing special. Whenever a damage taken `Player` code check if it's below zero and if yes it directly communicate with `Respawn Manager` and thell that we want to respawn a new player.

```cpp
void Player::OnHealthChanged(AActor* InstigatorActor, UHarvestAttributeComponent* OwningComp,
                                        float NewHealth,
                                        float Delta)
{
	// When a player die
	if (NewHealth <= 0 && HasAuthority())
	{
		AHarvestGameMode* SurvivalGameMode = GetWorld()->GetAuthGameMode<AHarvestGameMode>();

		if (SurvivalGameMode)
		{
            // Here is SurvivalGameMode is the Respawn Manager
			SurvivalGameMode->KillPlayer(InstigatorActor, this)
		}
	}
}
```

By the way, this is an real life scenario that I faced when I was working on my survival game project **Tarnish**, so if you love competitive games and the survival genree give it some love!

As seen it's very basic system `OnHealthChanged` method inside of the `Player` class. The problem about it, when I change the `SurvivalGameMode` to another game mode (i.e., "ArenaGameMode") I have to re-write the same logic that handles death of the player. So I can call the same exact method: `ArenaGameMode->KillPlayer()` Also it requires change on the `Player` code itself.

With the help of game events, `Player` will only send a game event as `Game.Player.Die` and any game mode that listens this event will trigger the respawn mechanism. So new code looks like:

```cpp
void Player::OnHealthChanged(AActor* InstigatorActor, UHarvestAttributeComponent* OwningComp,
                                        float NewHealth,
                                        float Delta)
{
	// Whenever a player die
	if (NewHealth <= 0 && HasAuthority())
	{
		AHarvestGameMode* SurvivalGameMode = GetWorld()->GetAuthGameMode<AHarvestGameMode>();
		AController* CharacterController = GetController();
		if (SurvivalGameMode && CharacterController)
		{

			FHarvestMessageVerb Message(FGameplayTag::RequestGameplayTag(FName("Game.Player.Die")) ,InstigatorActor, this);

			UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());

			MessageSystem.BroadcastMessage(Message.Verb, Message);
		}
	}
}
```

No longer `Player` communicates with the `SurvivalGameMode` just send the event and take care of its own business. 

## Building an Event System from Scratch

We will start simple, then we will ask questions. How the system could be better? Then gradually improve the code. Here is the classic observer pattern:

![observer-pattern-diagram-wikipedia]("images/observer-pattern-uml-diagram-wikipedia.png"){: w="720"}

`Player` is the "Subject" and the `Respawn Manager` is the "Observer". I won't go into details of the code and just use the UML. At this point we already decoupled the `Player` and the `Respawn Manager` right? `Player` class will be inherited by `Subject` so it can use `NotifyObservers()` method to trigger its observers.

However problem shines when we want to pass parameters via this event. I.e., who killed the player? Maybe we want to pass the **Instigator** game object who is resonsible by death event. Traditionally observer pattern creates different **Subject** and **Observer** base classes per system.

Also at the moment our player must be inherited by **Subject** class which something I do not like.

