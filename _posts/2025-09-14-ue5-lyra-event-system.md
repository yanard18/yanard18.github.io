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

We will start simple, then we will ask questions. How the system could be better? Then gradually improve the code. I will also start with Unity C# then transition to Unreal C++ when we start to review the **Game Message Subsystem**. Here is the classic observer pattern:

![observer-pattern-diagram-wikipedia](images/observer-pattern-uml-diagram-wikipedia.png){: w="720"}

```c#
// IObserver.cs
public interface IObserver
{
    void OnNotify();
}

// ISubject.cs
using System.Collections.Generic;

public interface ISubject
{
    void Register(IObserver observer);
    void Unregister(IObserver observer);
    void Notify();
}

// PlayerSubject.cs
using System.Collections.Generic;
using UnityEngine;

public class Player : MonoBehaviour, ISubject
{
    private readonly List<IObserver> observers = new();

    public void Register(IObserver observer) => observers.Add(observer);
    public void Unregister(IObserver observer) => observers.Remove(observer);


    public void TakeDamage(int dmg)
    {
        health -= dmg;
        if (health <= 0) 
			Notify();
    }

    public void Notify()
    {
        foreach (var obs in observers) obs.OnNotify();
    }
}
```

```C#
// RespawnManagerObserver.cs
using UnityEngine;

public class RespawnManagerObserver : MonoBehaviour, IObserver<PlayerDiedEvent>
{
    [SerializeField] private ISubject playerSubject; // could be found at runtime or set by a bootstrapper
    [SerializeField] private GameObject playerPrefab;

    void OnEnable()
    {
        if (playerSubject != null) playerSubject.Register(this);
    }

    void OnDisable()
    {
        if (playerSubject != null) playerSubject.Unregister(this);
    }

    public void OnNotify()
    {
        Debug.Log("Observer received PlayerDiedEvent. Respawning...");
        Instantiate(playerPrefab, Vector3.zero, Quaternion.identity);
    }
}
```

`Player` is the "Subject" and the `Respawn Manager` is the "Observer".  At this point it's already decoupled the `Player` and the `Respawn Manager` right?

Then what are the problems with this implementation?

- Event system do not pass any paramateres through `Notify()`. Example, who killed the player? Wouldn't it be greatt o pass the **Instigator** game object who is resonsible by death event? 
- Player must be inherited by the `ISubject` interface and Respawn Manager by the `IObserver`. Defining `OnNotify()` and `Notify()` methods for each event again is repitition.
- Player needs to know how to notify and this is not the responsibility of the player. 
- Player and respawn manager events required to be wired. There are no global event bus so actually, respawn manager needs to find the player and register itself into player's `ISubject`. 

Considering what is common and what is differ from the others also could guide us to design a better system.

Common:

- Behaviour of an event. Something happens, we broadcast what happens then sombe observers catches. This is same for all event system. 
- implementations of the `Notify()` and `OnNotify()` methods are always the same. There are no need to define them again for each new event.

Differ:

- Passed paramaters may differ for each kind of event. I.e., `Game.Player.Die` event passes the instigator.
- Event itself is different. Current implementation requires wiring subject and the observer, this way events are differing from each others.


### Passing Paramterers

Most obvious problem is that previous example do not pass any parameters at all. Obviously it's possible to write `ISubject` interfaces for each specific cases.

```c#
public interface IPlayerDeathSubject
{
    void Register(IPlayerDeathObserver observer);
    void Unregister(IPlayerDeathObserver observer);
    void Notify(GameObject instigator);
}
```

Great work! Now we need to implement a specific interface for each event. That's a big NO!

```c#
// IObserver.cs
public interface IObserver<T>
{
    void OnNotify(T data);
}

// ISubject.cs
using System.Collections.Generic;

public interface ISubject<T>
{
    void Register(IObserver<T> observer);
    void Unregister(IObserver<T> observer);
    void Notify(T data);
}

// PlayerSubject.cs
using System.Collections.Generic;
using UnityEngine;

public class PlayerSubject : MonoBehaviour, ISubject<PlayerDiedEvent>
{
    private readonly List<IObserver<PlayerDiedEvent>> observers = new();

    public void Register(IObserver<PlayerDiedEvent> observer) => observers.Add(observer);
    public void Unregister(IObserver<PlayerDiedEvent> observer) => observers.Remove(observer);

    private int health = 100;


    public void TakeDamage(int dmg)
    {
        health -= dmg;
        if (health <= 0) Notify(new PlayerDiedEvent { position = transform.position, player = gameObject });
    }

    public void Notify(PlayerDiedEvent data)
    {
        foreach (var obs in observers) obs.OnNotify(data);
    }
}
```

```c#
// RespawnManagerObserver.cs
using UnityEngine;

public class RespawnManagerObserver : MonoBehaviour, IObserver<PlayerDiedEvent>
{
    [SerializeField] private Player playerSubject; // Player looks like dependency but we just use it to find object itself. It could be done via bootstrap code as well.
    [SerializeField] private GameObject playerPrefab;

    void OnEnable()
    {
        if (playerSubject != null) playerSubject.Register(this);
    }

    void OnDisable()
    {
        if (playerSubject != null) playerSubject.Unregister(this);
    }

    public void OnNotify(PlayerDiedEvent data)
    {
        Debug.Log("Observer received PlayerDiedEvent. Respawning...");
        Instantiate(playerPrefab, Vector3.zero, Quaternion.identity);
    }
}
```

Feels better, to be honest for respawn manager it looks okey and we solved the parameter passing without repitition but for the `Player` class I feel a little irritated. Here are the problems that bugs me:

1. Adding interfaces to player for each event can be messy really quick especially considering player might have lots of different events.
2. It brokes the single responsibility of the `Player` class. Well, that may depend on the specifics of your project. And I think what makes Unreal Engine's designs are great they break those rules often to make something better. Still, at least we could separate the logic how it will be triggered and rather player just trigger the event.
3. **Wiring subject and observer cause coupling. Currently a boostrap code can wire those two.**

```c#
// Bootstrap class
public class GameBootstrap : MonoBehaviour
{
    [SerializeField] private GameObject playerPrefab;

    void Start()
    {
        // Spawn player
        GameObject playerObject = Instantiate(playerPrefab);
        Player player = playerObject.GetComponent<Player>();

        // Cast to ISubject
        ISubject<PlayerDead> playerSubject = player;

        // Find and cast RespawnManager
        RespawnManager respawnManager = FindObjectOfType<RespawnManager>();
        IObserver<PlayerDead> respawnObserver = respawnManager;

        // Register subject to observer
        playerSubject.RegisterObserver(respawnObserver);
    }
}
```

Again we are asking ourselves what is COMMON here!? **All subjects needs to be wired its related observers**. But we are writing new code for each wiring operation.

Let's imagine real life radio towers. They broadcast signals without caring who receive it. I bet they do not wire cables from radio towers to your pocket phone. 

Then how the signals find the correct device? They do **NOT!** But each signal has its own frequency, and signature messages. As a cyber sec entusiast I can tell that this is what we hackers do right? We listen those free signals on air.

Nature can inspire our design (well how much you can consider nature the 400 meter tall metal tower). Instead adding `int hz` field into our code we will define an event channel. However we have limitations, we do not have physical radio waves. Well maybe that's the limiatation of real life.

Here we need to design a transmission system. 

> Out of the topic, but wouldn't it be amazing if we would transmit messages in game engine with real physical waves!? Just imagine there are no manager class, just phsyical waves for class communications.

Well some ideas to design transmission system and channels:
1) Having a manager class in the game. All trasnmitted events will be catched by it and observers will subscribe to this one object.
2) Events can be declared as data assets (or ScriptableObjects in Unity).

As a Unity dev I seen that second option with ScriptableObjects choosen for many indie projects. So if I would go with Unity Scriptable Objects, I would end up with something like that:

```c#
[System.Serializable]
public class PlayerDeathPayload
{
    public GameObject Instigator;
    public GameObject Victim;
    public Vector3 Position;
}

public abstract class EventChannel<T> : ScriptableObject
{
    public event Action<T> OnRaised;

    public void Raise(T payload) => OnRaised?.Invoke(payload);

    public void Register(Action<T> listener) => OnRaised += listener;
    public void Unregister(Action<T> listener) => OnRaised -= listener;
}

[CreateAssetMenu(menuName = "Events/PlayerDeath")]
public class PlayerDeathEvent : EventChannel<PlayerDeathPayload> {}
```

That's looking really solid, and I designed my events like that for many projects. However now I'm using Lyra's event system and using its interface feels much smoother comparing to this event system. I ask myself why is that, what's really more appealing on the Lyra's sytem? 

At this point, we manage to  write a generic scriptable object however for each kind of event payload (i.e., `PlayerDeathPayload`), it's required to code a new struct.

On the other hand, Lyra says that, nah I will just write a massive payload that works under many circumstances. And it works perfectly.

With that said, let's modify our `PlayerDeathPayload` as:

```c#
[System.Serializable]
public class EventVerbPayload
{
    public GameObject Instigator;
    public GameObject Target;
	public String[] Tags;
	public float Magnitude = 1.0f; 
}
```

What we do here, generalizing the event payload. Looks simple but this setup we can handle various game events. Yes we lost the data of where the player death, but still we can use **Target** GameObject's position data to find player's position.

Philosophy learned here, **sometimes monster methods and data with many configurations better than a minimal one.**

## Design of the Lyra's Game Message Subsystem

At this point we build fairly good event system. Now we will transition to analyzing actual code of the **Lyra's Game Message Subystem** and see what we can improve in our design.

### Event Manager - Game Subsystem

We prefered to use scriptable objects for each event. But Unreal Engine prefers to use managers well in Unreal Engine ecosystem they called subsystems. All events are registered into one persistent manager.

Managers generally breaks SOLID principles, due to their nature of having multiple responsiblities. Even our system didn't use a manager, so our code was better? Well when I work with Unreal Engine I felt that their design works smoother. So I want to understand what are the reasons they prefer to go with managers and break the rules. 

1. 






