---
title: "Unreal Behavior Trees: A Blogtorial Pt. 1"
date: 2023-06-20
---

A recent conversation on Mastodon about Unreal’s ‘Behavior Tree’ system got my brain all buzzing with THOUGHTS and I wanted to get them all written down. 
In short this is going to be a ‘Why I like Behavior Tree (and you should like it too)’ blog, 
but I’m also going to talk about game AI programming in general (at least my understanding of it. I am by no stretch an expert in game AI), 
and how I think about Behavior Trees and some of what I feel like the theory of them is, using an AI character that I’m currently working on as an example.
(If you've watched [Paolo Souza's excellent Behavior Tree Tutorial](https://www.youtube.com/watch?v=iY1jnFvHgbE) on Youtube, he covers a lot of the same ground as this post,
but glosses over some details like [interface usage](#a-note-on-chain-of-command-and-blueprint-interfaces), so maybe skip to the end here because it's going to come up
in future posts)

So, to start with…

### All AI is a Lie.

‘Artificial Intelligence’ is a lie. The only accurate word there is ‘artificial.’ 
(there’s a whole other rant about the current ‘revolution’ in AI I could do, but it’s not entirely pertinent to games, 
and Adam Conover actually did a pretty good one a little while ago) But in games especially, AI is really just a bunch of simple tricks and nonsense. 
It boils down to long chains of if then statements, like ‘If I can see an enemy then I should try to shoot the enemy.’ 

That out of the way lets talk AI programming sans Behavior Tree.

### A Pseudocode AI Example

If you’re writing an AI in code you’re most likely putting together a finite state machine setup, and you’ve got something like an enum for the possible states for your NPC to be in
(i.e. patrolling, pursuing, attacking), a function that does a switch on that enum being called from tick, and that switch function in turn calling other functions for those states, 
and those functions having their own switches and conditionals that control what the NPC actually does. 
So for a simple three state AI we probably end up with something that looks like this:

```
Enum AIStates (PATROL, PURSUE, ATTACK)

CurrentAIState = PATROL

Tick(deltaTime)
{
	UpdateAI(deltaTime)
}

UpdateAI(deltaTime)
{
	switch CurrentAIState:
		case PATROL: DoPatrol(deltaTime)
		case PURSUE: DoPursue(deltaTime)
		case ATTACK: DoAttack(deltaTime)
}

DoPatrol(deltaTime)
{
	if(!CanSeePlayer)
	{
		MoveTo(PatrolPoint)
		If (AtPatrolPoint)
			{
				PatrolPoint = NextPatrolPoint
			}
	}
	else
	{
		CurrentAIState = PURSUE
	}
}

DoPursue(deltaTime)
{
	if(CanSeePlayer and !PlayerInAttackRange)
	{
		MoveTo(Player)
	}
	else if (CanSeePlayer and PlayerInAttackRange)
	{
		CurrentAIState = ATTACK
	}
	else
	{
		CurrentAIState = PATROL
	}
}

DoAttack(deltaTime)
{
	if(PlayerInAttackRange and CanSeePlayer)
	{
		DoAnAttackAnimation()
	}
	else if (!PlayerInAttackRange)
	{
		CurrentAIState = PURSUE
	}
}
```

Which is fine, I guess. But what if you need another type of AI? Sure, you could put all this in a generalized base class and have all of your NPCs be subclasses of that, 
but what if one of those NPCs needs a new state? You have to add it into the base class, even though every other NPC doesn’t use that state. 
Or what if you need only part of one of those state functions. You’re either breaking the state functions up into smaller functions, 
or you’re overriding without calling Super::Function and copy-pasting the chunks of the original function into your new class. 
Scaling an AI like this up or down can quickly get unwieldy, and iterating on it will get slower as complexity increases. 
And if you’re doing all this in Blueprint, then gods help you because the pile of spaghetti you’ll end up looking at will be absolutely wild.

So that brings us to Behavior Trees.

### The Big Benefit of Behavior Trees in Unreal

By far the biggest benefit of Behavior Trees is code reusability. 

When we break down the pseudo code three state guy, we see that there are a handful of functions that are called from each state (MoveTo, and DoAnAttackAnimation). 
In Unreal Behavior Tree parlance these are ‘Tasks.’ And in Unreal, these tasks are actually unique blueprints (or native code classes, if that’s your jam), 
so they can be reused in any Behavior Tree. There are even a bunch of built in Tasks that cover a lot of expected NPC functionality, like MoveTo, Wait, LookAt, 
and playing animations and sounds. 

So you get to build up your AI from different building blocks, and you can use the same blocks over and over again.

### How To Think About A Behavior Tree

The first thing about Behavior Trees is that we need to sort of change how we think about the AI. In the pseudo code example of our basic three state guy, 
the states of the guy are a pretty standard finite state machine, with three rigid, discrete states. A Behavior Tree is still pretty much a finite state machine, 
but the states can be a little bit softer (you can make them pretty rigid, but doing so brings some drawbacks), and it also has to be organized by priority. 
In code, you can put the states in whatever order you want, in Behavior Tree you need to know what your AI wants. I find it useful to think of it in terms of a "hierarchy of needs."

So instead of just thinking about “what are our states and what changes our current state?” we have to think about “what is the state that our guy **wants** to be in, 
and what do they fall into when they can’t be in that state?” And then we organize the tree from left to right in the order of where we want to fall down or up the tree.

### The Blackboard

Another key component of the Behavior Tree is the Blackboard. One way to think about the Blackboard is that it’s the header file for the Behavior Tree. 
This is where all the public variables that are accessible to every Task, Decorator (more on those in a second) and Service (more on these in a future post) are kept. 
Strictly speaking you don’t NEED a Blackboard. You could have your tasks pull variables straight from your Controller or Pawn classes, 
but that means either casting to those classes (which is bad. More on this later), or a larger interface class (more on THIS later), 
but all of the built in tasks (and decorators and services) use the Blackboard, so it’s a good idea to use it. 

<sub>(You can also set blackboard variables to share their values across blackboard instances, 
which is handy for doing something like having a Camera type NPC that relays information to Guards, or Guards informing other Guards. 
But that’s outside the scope of this particular blog/tutorial)</sub>

### A Behavior Tree

So breaking down Three State Guy, we could say that the thing that they want most in the world is to be on patrol. So long as they’re on patrol, not seeing anything, 
they’re as happy as a clam. Their next highest priority is **killing anything they see**. Because seeing something has ruined their patrol. 
But, since they have a limited attack range they have to get close enough to whatever they saw to kill it, 
so their third highest priority is getting close enough to whatever they saw to kill it.

So here’s a view of the three state guy as a Behavior Tree:

<img src = https://github.com/DKesserich/Blog/blob/main/images/ThreeStateBT.PNG?raw=true> </div>

So we can clearly see our three states: Patrol, Attack, Pursue. These nodes are ‘Composite’ nodes. ‘Composites’ do pretty much what the name suggests: 
they let you composite other nodes, be they other composites or tasks, into an overall task, or Behavior. The ‘Main’ node is a ‘Selector’ type, 
which means it will execute its leftmost (highest priority) child repeatedly until it fails, then move to the next leftmost, and repeat. 
The Patrol, Attack, and Pursue nodes are ‘Sequence’ types, which will execute their children one at a time from left to right until one of them fails, 
at which point the entire composite will report itself as having failed. 

The blue squares on the nodes are Decorators. Decorators are effectively the ‘If’ statement of the node they’re attached to, and they can be attached to Composite nodes and Task nodes, 
and you can stack them, though multiple decorators are treated as ‘And’ booleans, not ‘Or.’ So if one Decorator fails, the entire node will be prevented from executing. 
Decorators can also be set to interrupt their attached node, any lower priority members of the tree, or both. So, by default, if a Decorator is watching a boolean value and that value 
flips from `true` to `false`, it will wait for the task it's attached to to complete before blocking the task from executing again. If set to interrupt Self it will immediately stop 
the attached task, but if `false` flips back to `true`, it will wait for the lower priority task to finish before taking over. If set to interrupt Lower Priority it will cut the lower 
priority task off, and if set to Both, it will instantly transition in and out as the watched value changes.

So Patrol will execute if they Can Not See Player (which evaluates based on checking whether a particular Blackboard key is set, and that key is set from the AI Controller 
using the AI Perception component), and the tasks for Patrol are to set the animation state of the pawn, get the next location they’re supposed to move to, and move to that location. 
Since 'Can Not See Player' is set to 'aborts both', as soon as they Can See Player, Patrol will be cancelled and the tree will move to Attack, 
but in all likelihood the player will not be in attack range, so Attack will fail and Pursue will execute instead (at least so long as the player can still be seen). 
Pursue sets animation state and moves toward the player, and once in range the tree will fall back up to Attack, which will set anim state, wait half a second, and then Do Attack.

And that's our basic Three State Guy.

But before I close out this post,

### A note on 'Chain of Command' and Blueprint Interfaces

I like to follow a structure for my NPCs that I call ‘Chain of Command.’ (I don’t know how common this is or if there’s another name for it, but that’s what I’m calling it.)

It breaks down pretty much like this: A standard NPC in Unreal consists of an AI Controller class, a Pawn class (this is usually a subclass of the Character class), 
an Animation Blueprint for the skeletal mesh of the Pawn/Character, and the Behavior Tree and Blackboard. Technically speaking, 
it is possible for the Behavior Tree to directly set values in the AI Controller, Pawn, or Animation BP, and all of them could potentially set values in each other easily enough.

But it can be a real mess if you do that, because if you’re not careful you could have values being changed by multiple things on the same frame and you’ll get unexpected behaviors 
and have a hard time tracking down where it’s coming from.

So following ‘Chain of Command’: The AI Controller and the Behavior Tree can directly set values in the Blackboard. The Behavior Tree can make requests of the AI Controller, 
the AI Controller can call functions on the Pawn, and the Pawn can make requests of the AI Controller and call functions on and retrieve values from the Animation BP, and the 
Animation BP can retrieve values from the Pawn. 
This way I know that if something happens on the AnimBP, I know it came from the Pawn, and if it happens on the Pawn, it was because the Pawn asked for it from the AnimBP 
or it came from the AI Controller, and if something changes in the Behavior Tree it was either a result of what the Behavior Tree was doing, or it came from the AI Controller.

And this is where Blueprint Interfaces come in. A lot of tutorials written or made by people who don’t work at Epic skip over the importance of Blueprint Interfaces. 
They are SUPER important. If you’re moderately experienced with Unreal and looking at that ‘Chain of Command’ breakdown you’re probably thinking: 
‘Okay the custom Behavior Tree Tasks all have casts to the AI Controller class, and the AI Controller does a bunch of casts to the Pawn class, 
and the Pawn class has a bunch of casts to the AnimBP class and also vice versa.’ Not so! Firstly: If I did that, I wouldn’t be able to reuse any of my code. 
If I made a new AI Controller class, none of my custom tasks would work with it, and if I made a new Pawn, my AI Controller wouldn’t work with it, and if I made a new AnimBP… 
You get the picture. Secondly: **Blueprint Casts to Blueprint Classes Are Bad**. Every time a Blueprint cast is used, it creates a hard reference to the class that is being cast to. 
If you cast to a blueprint class that ALSO has blueprint casts, you’re creating hard references to those classes. If you’re not careful, 
a single cast can cause every class in your game to load into memory at once. And then that all has to get garbage collected, and it’ll kill your performance.

So I use an interface, and I have the AI Controller, the Pawn, and the AnimBP all implement the same interface.

Lets look at a practical example with the GetNextPatrolPoint task:

The Patrol Points are actually an instance editable array of actor references on the Pawn class, so GetNextPatrolPoint sets a Patrol Point Index on the Blackboard by 
incrementing that value and checking against the length of the array, which is retrieved with a GetNumPatrolPoints interface call on the AI Controller, 
which passes it to the Pawn which returns the length of the array, and then a call is made to the GetPatrolPoint function of the AI Interface on the AI Controller, 
which also passes it to the Pawn which returns the actor reference.

Easy.

The REAL flexibility of this approach is shown in the DoAttack task. When DoAttack fires in the Behavior Tree, it sends the DoAttack interface message to the AI Controller, 
which passes it along to the Pawn via the same interface, and because the attack for this particular Pawn is a melee attack, the Pawn passes the message to the AnimBP, 
which plays a montage of the attack (and returns a float value for the length of the montage, which we set to the blackboard so WaitBlackboardTime will hold until the montage finishes). 
If I wanted to I could make a new Pawn that has a gun instead of a sword which implements the same interface, and I could use the same AI Controller with the same Behavior Tree, 
and the only thing that would have to change is that in Gun Pawn’s implementation of DoAttack I would spawn projectiles instead of continuing to pass the message to the AnimBP. 
Or I could use the same Sword Pawn and assign a new AnimBP on an instance of that Pawn that does a different montage when DoAttack fires, and nothing has to change anywhere up the chain. 

Much easier to re-use code, requires no casting.

So that's basic Behavior Tree stuff. In my next post I’m going to get into adding some juice to the existing behaviors, 
and adding new behaviors into the existing structure in a robust way.







