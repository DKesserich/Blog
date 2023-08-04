---
title: "Unreal Behavior Trees: A Blogtorial Pt. 5"
date: 2023-08-04
---

Hi there! It’s been a little bit since my last post. Last time I redesigned the Behavior Tree so it has the capability of being somewhat more dynamic, 
with a modular ‘Combat’ branch that I can change to use different Behavior Trees depending on what weapon or items the NPC is carrying. 
Since then I’ve been figuring out how I want the item pickup system to work and I think I’m mostly there, so it’s time to blog about it. 
(Oh, also I’ve added the Unreal project for this to Github [here](https://github.com/DKesserich/UnrealAI_Project), 
so you can clone/fork that and follow along if you want).

## Item Class And Querying, In the Abstract

One of the first things I needed for the pickup system is a new base Item class that I can put some shared, 
editable properties and interface implementations into. For instance: Each item has an associated Behavior Tree, 
so I want a Behavior Tree Object Reference variable, and I want an interface call that will return that Behavior Tree reference 
so it can be assigned to the Combat Dynamic Behavior node when the NPC picks up the item.

But the other thing I want is to use tags so when I use the Environment Query System to find an item I’m not creating a bunch of hard references 
to the various Item sub-classes. To understand why I might end up in a situation where I’d need to be referencing the sub-classes instead 
of the base Item class, consider a situation where the NPC has a shield, but doesn’t have a weapon. I can’t just search for 'all actors of class Item' 
any more, because that might find another shield, or another secondary item, or a two handed weapon like a bow that they can’t use 
because they have a shield, so the query has to either be ‘all actors of class Item that are not Shield or Throwable or Consumable or Bow’ 
or it has to be ‘all actors of class Sword' or 'all actors of class Spear,’ in either case, the query would have to have hard references to those classes.

So that’s where tagging comes in.

## Tags, You're It

Unreal has two systems for tagging: Actor Tags, and Gameplay Tags. Actor tags are pretty easy to use:
you just go to the Actor Tags section of your Blueprint or the properties of the Actor in the level, hit the plus button and type in whatever you want. 
But that’s also one of their big downsides. You have to type it in. And when you’re checking tags on an actor, 
or looking for an actor with tags you have to type the tags in again. 
This can be a little error prone because if you typo the tag somewhere you won’t get the actor you’re looking for.

The other thing about Actor tags is that when querying them Unreal is just doing an FName comparison, 
which is essentially the same as a String comparison. And String comparisons can get expensive, computationally. 
At smaller scales it’s probably fine (and there are a lot of Epic sample projects and examples that use Actor tags, 
so it’s not like they’re a feature that Epic recommends not be used, like the Level Blueprint), 
but as things scale up and you have more things doing Actor tag queries at the same time, it can start affecting performance.

Gameplay Tags, on the other hand, are a little more complicated to use but they’re also more powerful, and computationally much cheaper. 
I actually used a Gameplay Tag once already for the Combat Dynamic Behavior node in the Behavior Tree. 
They can be defined either in the Project Settings, or the option to define a new Gameplay Tag is given anywhere you can use them. 
You can also create Gameplay Tag data assets and use them as a Gameplay Tag Source if you want to do something like keep the Item Gameplay Tags 
separate from the Behavior Tree Gameplay Tags. They’re also a lot less error prone to use because once a Gameplay Tag is defined, 
you can just select the tag you want instead of manually typing it in every time.

The powerful thing about them is that they’re hierarchical, so when defining a new Gameplay Tag I can put in ‘Item.Primary.OneHanded’ 
and in the Gameplay Tag selector there’ll be a little inheritance tree that I can expand, where OneHanded is a child of Primary, 
which is a child of Item. This allows queries to be as specific or loose as I want.

They’re computationally cheaper because a GameplayTagContainer is basically an n-dimensional table of booleans, 
so a Gameplay Tag query is just doing a table lookup, and that’s practically free, computationally.

Now for the con of using Gameplay Tags: There is a built-in Gameplay Tag Query interface, but that interface is not implemented by default, 
and it can not be implemented in Blueprint. So, once again, we have to make a native C++ class for the Item base class 
(or you can manually roll your own Blueprint Interface that does the same thing as the built-in one, but that’d be silly). 
I’ve already done this in the shared Github project, so if you want to use that and skip ahead, go ahead. 
For C++ noobs who want to learn a bit, this is how to implement an interface in C++.

## C++ Interfaces for Noobs (and just some C++ for noobs)

First things first, we need to create our native class. All the way back in Part 2 I said to go to the Engine Content C++ classes and 
search for the class we want to sub-class and right-click it and select ‘Create C++ Class Derived From [Classname].’ We can still do that, 
but there’s actually another, slightly easier, way to add a C++ class. Go to ‘Tools->Add C++ Class,’ select ‘Actor’ and then 
do pretty much the same thing as we did back in Part 2. In the shared project I named the new class ‘Pickup_Base,’ 
if you’re making your own project from scratch, name it whatever you want.

Once Unreal finishes doing its thing and inevitably tells you you need to close Unreal and re-build manually, 
close Unreal and open up Visual Studio (or your IDE of choice). Before we start editing the new class, we need to add a couple things to the 
project module’s Build.CS file, because Gameplay Tags is an Unreal module that doesn’t get pulled in by C++ projects by default. 
So open up YourProject.Build.CS (in the shared project this is ZeldaAI.Build.CS. Because I’m trying to make a Zelda style AI). 
In ‘PublicDependencyModuleNames.AddRange’ add “GameplayTags” and, just to save us some time later, “AIModule”. 
The full line should look something like this: 
```
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "GameplayTags", "AIModule" });
```
Now open up the .h file for the class you created (Pickup_Base.h in the shared project). In the includes add `#include "GameplayTagContainer.h" ` 
and `#include "GameplayTagAssetInterface.h"`. To implement interfaces in C++ we just have to add them, separated by commas, 
to the class declaration after the main class that we’re inheriting from. So where it says `public AActor` near the top of your .h file, 
just add `, public IGameplayTagAssetInterface` at the end.

We also need a GameplayTagContainer variable that can be edited in all of our Pickup blueprints. 
You can copy and paste this code snippet into the public section of your header:
```
UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Gameplay Tags")
FGameplayTagContainer ItemTags;
```
That UPROPERTY line is a macro for Unreal, and it tells Unreal that this variable can only be edited in the blueprint, 
not in a placed actor (EditDefaultsOnly), that the blueprint script can read the variable and also write to it (BlueprintReadWrite), 
and that we want the variable to be listed in a Gameplay Tags section of the class defaults (Category=”Gameplay Tags”).

Now if you try to compile the code, it’ll fail. Because I lied, and there’s a little bit more we need to do to implement the interface. 
The thing about interfaces in C++ is that they’re pretty much just headers with declarations of functions, but no definitions for those functions. 
And the C++ compiler doesn’t like it when a function is undefined. So we need to override all the functions that we got from the 
GameplayTagAssetInterface and actually define what they do. Luckily there are only four. 
To make it easy for you, copy and paste the following code snippet into the header:
```
//GameplayTagAssetInterface
public:
    virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const override;

    virtual bool HasMatchingGameplayTag(FGameplayTag TagToCheck) const override;

    virtual bool HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const override;

    virtual bool HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const override;
    //End GameplayTagAssetInterface
```
And then copy and paste the following into your .cpp file (if you named your class something other than Pickup_Base, 
replace `APickup_Base` with whatever your class name is):
```
void APickup_Base::GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const
{
    TagContainer = ItemTags;
}

bool APickup_Base::HasMatchingGameplayTag(FGameplayTag TagToCheck) const
{
    return ItemTags.HasTag(TagToCheck);
}

bool APickup_Base::HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const
{

    return ItemTags.HasAll(TagContainer);
}

bool APickup_Base::HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const
{
    return ItemTags.HasAny(TagContainer);
}
```
All pretty easy. GetOwnedGameplayTags just gets a reference to our GameplayTagContainer property, and for HasMatchingGameplayTag, 
HasAllMatchingGameplayTags, and HasAnyMatchingGameplayTags we just want to return those queries from our GameplayTagContainer.

Now, before we build the code and go back to the Unreal Editor, there’s one more thing we should add to the Pickup class. 
Since all of the Item pickups are going to have an associated Behavior Tree, we can just add the variable for that here, 
instead of in blueprint (this is why we added the AIModule to the Build.CS). Go back to the .h file, and add the following to 
the `public` section where the ItemTags variable was added:
```
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
UBehaviorTree* Item_Behavior;
```
After adding that, UBehaviorTree will probably be underlined in red. There are two ways to fix that: We can add the header file for BehaviorTree 
to the includes in the .h file (`#include "BehaviorTree/BehaviorTree.h"`), or we can add a forward declaration to the .h file, 
and add the header file to the includes in the .cpp file.

In the shared project I went with the forward declaration method. A forward declaration is done by just adding `class ClassName;` 
(in this case the ClassName is UBehaviorTree) after the includes. A forward declaration is basically pinky swearing that a class actually exists 
and has a full definition somewhere. They’re mainly used to prevent creating circular references where you have a header file 
loading another header file that loads the first header file. Strictly speaking, using forward declarations for classes that are built-in 
to Unreal isn’t totally necessary because there’s no way Unreal is referencing a brand new class you just made, 
but it’s still a good habit to get into using them.

“Why didn’t we use a forward declaration for the FGameplayTagContainer?” I hear you ask. And y’know what? Good point. 
Let’s do that. FGameplayTagContainer is actually a struct, not a class, so the forward declaration needs to be `struct FGameplayTagContainer;` 
And then we just cut the GameplayTagContainer.h include from our header file, and paste it into the .cpp and we’re good. 
Build the code, and if you set everything up right the build should succeed, and we can open up the Unreal Editor again.

## Wrapping Up

Once we’re back in Unreal, it’s time to make the Blueprint base class for Items. We don’t want to go straight into making the actual items because 
there are Blueprint Interfaces to implement, and we don’t want to have to re-implement them for every type of Item. 
So we make a new Blueprint that inherits from Pickup_Base (or whatever you named the native class). I’ve named mine BP_Pickup_Base. 
I’ve also made a new Blueprint Interface called BPI Items to put some functions that I don’t think are necessarily combat related into. 
Right now the only two functions in that interface are Set Animation and Get Behavior (which is configured to return a Behavior Tree object). 
In BP_Pickup_Base we want to add BPI Items to the Implemented Interfaces in Class Settings. We’ll leave the combat interface out of the base pickup 
bp because not every Pickup needs to be combat oriented.

Then we just have to define the Get Behavior function to get the Item Behavior property from the C++ class and return it through the interface, and 
now every blueprint we make that inherits from BP_Pickup_Base will automatically have the BPI Items interface and return their Behavior Tree.

And I think that’ll do it for now. Next time I’ll get into setting up the first actual pick-up item (There are some quirks to how I’ve set this up 
in the shared project that come from the simplistic visual design of the NPC, but some of the fundamental ideas should transfer to more 
realistic/complex character designs), and creating a Behavior Tree to find and pick up items when the NPC doesn’t have any 
(I’ll finally be using the Environment Query System. And maybe Smart Objects, too).
