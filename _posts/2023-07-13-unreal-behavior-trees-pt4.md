---
title: "Unreal Behavior Trees: A Blogtorial Pt. 4"
date: 2023-07-13
---

This is going to be a short one where I mostly get into refactoring the existing behavior tree and the benefits of the Run Behavior and Run Behavior Dynamic Tasks. 
I know I keep saying I’m going to get into making combat better, but the fact is: I am not a combat designer. I’ve got some high level ideas for what I want to do with 
the actual combat AI, but I have to work through them and figure out what works and what doesn’t before I actually tutorialize it. 
Another thing is that what I’m trying to work towards is sort of an approximation of the way that combat works in The Legend of Zelda: Windwaker, 
and a lot of the finer points of that system are animation driven and I am also not much of an animator.

So let’s get into refactoring.

## Cleaning Things Up A Bit

When we left off last time the Behavior Tree was already starting to get a bit messy.

<img src="https://github.com/DKesserich/Blog/blob/main/images/Full_BehaviorTree_With_Investigation.PNG?raw=true" alt="Screenshot of the full Behavior Tree with the Investigation Behavior">

So what we want to do is clean up this main Behavior Tree. First off, when we look at it we can see that there are two sort of super states to the tree: Can’t See The Player, 
and Can See The Player. So we can add a new Selector for Can’t See the Player and plug the Patrol and Investigate branches into that, and move the Can’t See Player Decorator 
into that selector and remove it from the Patrol and Investigate Sequence nodes. But, in a key difference from how it was configured before, 
I want this Decorator to only abort Self (I’ll get into the why of this later).

We could also add a new Selector for the Can See The Player branch and plug Combat and Pursue into that, but I actually don’t want to do that either.

What we’re going to do is start using the Run Behavior and Run Behavior Dynamic tasks. These are really useful, potentially extremely powerful, ways of managing your Behavior Tree, 
and go back to what I’d said in part one of this series about how what I like about Behavior Tree is its modularity and reusability. 
Because what these two tasks do is pull in another Behavior Tree asset and run it within the main tree.

So we want to add a Run Behavior task, and plug that into the Can’t See The Player Selector, move the ‘Last Seen Location is not set’ decorator from the Patrol Sequence 
on to the Run Behavior task, and then unhook the Patrol Sequence.

We then want to create a whole new Behavior Tree asset (I’ve named mine BT_Patrol) and cut and paste all the nodes for the Patrol Behavior into that, 
hook up the Root, and save it. Back in the main BT, select the Run Behavior that we added and for the Behavior Asset in settings select BT_Patrol.

<sub>A quick Note about Blackboards: Any Behavior Trees that are going to be used by the Run Behavior tasks must use the same Blackboard asset as the main tree.
I find this a little annoying, to be honest. Unreal does have a concept of Parent and Child Blackboards, but Child blackboards won’t work in this context. 
To me it seems like sub-behaviors will likely have data that is only relevant to them and it should be fine to use a Child blackboard just for that data, 
but Epic apparently feels differently. This is probably a relatively easy change in the engine source code, but I haven’t looked into it.</sub>

Now do the same thing with the Investigate tree: Add a Run Behavior task, move the decorator, make a new BT, cut and paste the nodes from the main BT to the new Investigate BT, 
assign the new BT as the asset for the task.

Now when we run our guy should work exactly the same as they did before. But now we have these pre-defined behaviors that we can plug into other Behavior Trees, 
or even just run them as the main Behavior Tree all by themselves (if, for example, we wanted a guy who patrolled and did nothing else).

So now we’ve got to deal with the Pursuit and Combat branches. This is where we’re going to use a Run Behavior Dynamic task, and this is where things get really interesting and powerful.

## Dynamic Behaviors

Some of the most common complaints about Behavior Trees is that they become increasingly hard to manage as complexity increases, 
and that they also become more brittle as complexity increases. Meaning that adding additional behaviors increasingly risks breaking the existing ones. 
The Run Behavior Dynamic task is something that I think is unique to Unreal’s implementation of Behavior Tree, and, in my opinion, does a lot to mitigate those complaints.

To better illustrate this, let’s go back to what I said in the intro to this post about how the ultimate goal of the AI I’m building is to have something like what the AI in Windwaker does.

If we look at the Windwaker enemies, one of the more notable things they do is that some of the primary enemy types (most notably the moblins and bokoblins), 
don’t actually seem to have hardcoded static weaponry. I. e. there are times you’ll find a bokoblin that doesn’t have a weapon at all, 
and when you initiate combat with them they will run away to find a weapon. You can also force them to drop their weapons, at which point they’ll either attack you unarmed, 
or try to find a new weapon. Enemies will also behave differently depending on what kind of weapon they have, 
or if they have a secondary combat item like a shield that they can block with, or a lantern that they can throw at the player.

So we can sort of break this down into a bunch of primary weapon types and secondary combinations, like: unarmed, ranged, short range melee, short range melee with defense, 
short range melee with throwable, two-handed/mid-range melee, and two-handed/mid-range melee with throwable.

And then we can further break these down into what the behaviors are when in combat: if I have a ranged weapon and I see the player I want to move to somewhere where I 
have both line of sight and am at the effective range of my weapon, and then shoot at the player. If the player is inside or outside my effective range I want to find a new spot. 
If I have a short ranged weapon I want to close the distance to the player and attack them. If I have a shield I can try to block when the player attacks me. 
If I have a mid-ranged weapon I want to close the distance to the player and attack them, but if the player is too close I want to do a shoving attack to push them away from 
me so I can use my weapon more effectively. If I have a throwable, I want to get within throwing distance of the player and throw it at them, 
and then revert to whatever my other weapon does. If I’m unarmed I want to find a weapon, but if I can’t find a weapon or the player is closer to me than a weapon 
is I want to do an unarmed attack.

Looking at that, we can see that trying to encapsulate all these possibilities in a single omnibus ‘Combat’ Behavior Tree would just be an absolute mess. 
And adding new types of weapons or other new behaviors into that big pile of stuff would indeed get really difficult. 
Especially since we wouldn’t only have to have all these behaviors in there, we’d have to sort their priorities properly and/or have a ton of decorators strictly controlling their flow.

Which brings us back to Run Behavior Dynamic. What Run Behavior Dynamic lets us do is swap out what Behavior Tree we run on that node during runtime. 
So what we can do is have a unique BT asset for every weapon type that we can swap to based on what weapon or weapons the guy is holding, 
and much more easily tune those individual behaviors. That also means that for adding a new type of weapon, instead of having to squeeze it into the existing tree 
we can just make a new one. We can also get really crazy and have nested dynamic behaviors and swap other things out. 
Like, for example, we could have a service monitoring the NPC’s health, and if it drops below a certain level switch the Attack behavior to a ‘go get health’ behavior, 
and then back to the Attack behavior after healing (or back to attack if ‘go get health’ fails), or just swap around the priorities of behaviors based on external conditions.

So, looking at our current main Behavior Tree we have our Attack and Pursue behaviors. Like with the Patrol and Investigate behaviors before, 
we want to create new Behavior Trees for Attack and Pursue (I’ve named mine BT_GenericAttack and BT_Pursue). But unlike those, 
we don’t want a Run Behavior node for each of them in the main tree. Instead we want another Behavior Tree that uses both the Attack and Pursue behaviors. 

This is the short range melee combat Behavior Tree (I’ve named mine BT_Combat_ShortRangeMelee). This BT should have a Selector and two Run Behavior nodes. 
The higher priority Run Behavior should use BT_GenericAttack, and the lower priority one should run BT_Pursue. We also want decorators on these tasks. 
Put a check if In_Attack_Range is true on the Attack node, and set it to abort both, and put a Seen_Player is set and an In_Attack_Range is false check on the Pursue node, 
both of them set to abort self. This is how it should look:

<img src="https://github.com/DKesserich/Blog/blob/main/images/Combat_ShortRangeMelee.PNG?raw=true" alt="Screenshot of the Short Range Melee Combat tree">

Back in our main Behavior Tree we now want to add the Run Behavior Dynamic task. You’ll notice that the settings are a little bit different than the Run Behavior task.
Where in Run Behavior it just had the Behavior Asset, here it has an Injection Tag and a Default Behavior Asset. Default Behavior Asset is exactly what it sounds like. 
It’s the behavior tree we want to run by default. In this case we want to set that to BT_Combat_ShortRangeMelee. 

Injection Tag is the important bit, because it is this tag that is used in Blueprint or code to set a different BT asset during gameplay. If you click the Edit button here you can pick any existing gameplay tags, or create a new gameplay tag for this task. The thing to keep in mind is that every Run Behavior Dynamic task in the Behavior Tree, 
and any in any Behavior Trees being called from your main Behavior Tree, should have a unique tag. I.E. if this one we just added were tagged Combat_Behavior (which it is), 
and if we wanted to be able to dynamically swap BT_GenericAttack in the BT_Combat_ShortRangeMelee tree with something else, we wouldn’t want that task to also be tagged Combat_Behavior.

The last thing we want to do is add the Update Target Position service to this Run Behavior Dynamic task, because regardless of what combat behavior is being run, 
we always want to be updating the Last Seen Location blackboard key as long as it’s running. 

You’ll also note that I’m not putting any decorators on the Run Behavior Dynamic task. This is related to why when we did the Can’t See Player branch we set that decorator 
to only abort self. Because when we _can_ see the player, we know we don’t want to do either the Patrol or Investigate tasks, but whether or not we should immediately 
give up on the Combat tree when we _can’t_ see the player is going to vary depending on what Combat tree we’re running, so the decorators in those trees will control that. 
In the current Melee tree, when we’re not in range and can’t see the player, we fail completely out of that tree and over to the Patrol/Investigate group. 
But if we were running a ranged weapon tree where we want to move to where we have line of sight on the last seen player location and are in effective range of 
the ranged weapon before shooting (if we can still see the player), we want to allow that move to to complete before we evaluate whether we can still see the 
player and then fail back out to Investigate if we can’t.

So to finish: This is what our main Behavior Tree should look like now:

<img src="https://github.com/DKesserich/Blog/blob/main/images/Refactored_Main_Tree.PNG?raw=true" alt="Screenshot of the refactored main Behavior Tree">

That’s MUCH better.

Looking back, this post ended up being longer than I originally anticipated. It might be a while before I can get to the next instalment of this blog series. 
I want to do the weapon pick-up system, and for that I have to change some stuff with the character model and animations I’m using for the NPC, and make some new classes.

Until next time.


