---
title: "Unreal Behavior Trees: A Blogtorial Pt. 3"
date: 2023-06-30
---

In part 2 of this series I described setting up some new tasks to make an existing behavior a little more interesting.

In this part I want to get into adding a whole new behavior in the existing tree, and a little bit on some of the settings in AI Perception.

### On Perception and the AI's Memory

Before I dive into adding a new behavior to the tree, I’m going to do a brief little dive into how I’ve got the AI Perception component set up.

Up until this point I’ve been using a very basic sight-only set up for the AI Perception component where when I get the OnTargetPerceptionUpdated event 
I check the ‘Stimulus Successfully Sensed’ bool and if it’s True, set the SeenPlayer Blackboard variable to the actor that was sensed, and if False, clear SeenPlayer. 
This works, but with the default Sight Sense settings leads to some real goldfish brain behavior where the AI immediately forgets the player exists 
and thus returns to the Patrol state as soon as it can’t see the player. You could even evade the AI while it’s in combat with you just by running around behind it.

There are a couple of settings that help with this a bit in the AI Sight Config. Most importantly is the Auto Success Range From Last Seen Location setting, 
which lets you specify a range that, if the player remains within, the AI Perception will continue to return True for Stimulus Successfully Sensed even if the AI can’t actually see it. 
Setting this to around 5 meters or so (depending on how large your NPC is) solves the ‘run behind’ problem. Another useful setting is the Max Age setting, 
which can be used for two things: it lets you set a time limit after which the stimulus will be updated, so you can set it to less than a second if you want to get refreshed stimulus 
info with a high frequency (this does come with some additional cost on the game thread, so probably not the best idea to use it this way); 
and it can get you a ‘forget’ event when the stimulus expires, which can be useful for setting up a timeout for something like the Pursuit behavior.

Now I say it _can_ get you a forget event instead of it _does_ get you a forget event because the Forget event only fires if the `bForgetStaleActors` variable is set to true, 
and inexplicably that variable is not set by default (if you look at the header for AI Perception in the Unreal source code it looks like it’s set to true, 
but then in the constructor it makes a ‘get value from config’ call and since `bForgetStaleActors` isn’t in BaseEngine.ini it sets it to false. Annoying.)

In order to enable it you have to go to Project Settings-> Engine->AI System and tick ‘Forget Stale Actors.’

That done, let’s get into adding a new Investigate behavior to our Behavior Tree.

### Investigation and Services

For an Investigate behavior we’re going to need a new Vector blackboard key, and we’re going to need the OnTargetPerceptionUpdated event in the AI Controller to write to that 
key when Stimulus Successfully Sensed is true. I’ve named the BB key LastSeenPosition, and in the AI Controller I write the Stimulus Stimulus Location to that key.

Since the OnTargetPerceptionUpdated event only fires when Successfully Sensed changes, or MaxAge for the stimulus is reached while the player can still be seen, 
we don’t get a continuous update on the Player’s location while it is being seen.

So we need our first Service.

A Service is sort of like a Decorator in that it is something that attaches to a node in the Behavior Tree instead of being a node in and of itself, 
but unlike a Decorator it doesn’t control the execution flow of the tree (at least not directly). What it does do is allow you to run some continuous logic for 
as long the node it’s attached to is active. Sort of like a Simple Parallel, but it doesn’t return a success or a fail.

What we need this Service to do is update the LastSeenLocation key for as long as we can see the player. Which is pretty simple. Make a new Blueprint of the 
type BTService_BlueprintBase and add Event Receive Tick. I use my interface to call GetTarget on the OwnerActor (which is the AI Controller), 
which then passes that to its Controlled Pawn. I then GetActorLocation from the returned actor and set the LastSeenLocation blackboard key.

This is what it looks like:
<img src="https://github.com/DKesserich/Blog/blob/main/images/Update_Target_Pos_Service.PNG?raw=true" alt="A screenshot of the BP graph for the Update Target Position Service">

An alternative way to do this would be to get the SeenPlayer blackboard key and GetActorLocation from that, but since in my setup SeenPlayer gets unset as soon as the AI can’t see them,
but the Pawn’s Target doesn’t get unset until Forget fires I think the way I have it offers a little more flexibility.

Then we just attach this Service to the Pursue and Attack sequence nodes and we’re good.

Now let’s set up an Investigate behavior. As with all behaviors, we’re going to start by plugging a Sequence node into our main Selector. 
I feel like the priority of Investigation falls between Patrol and Attack, so that’s where we’ll drop the sequence in the graph. 

Before we define the behavior, let’s set up the conditions under which this behavior is valid with our Decorators. We want our guy to investigate if they can’t currently see the player,
but they have seen them recently enough that they haven’t forgotten. So we add two Blackboard Decorators to the Investigate sequence: 
One that checks that SeenPlayer is not currently set (which should also be set to Abort Both), and one that checks if LastSeenLocation is set. 
We also want to add another Decorator to the Patrol behavior to check if LastSeenLocation is not set. That way we don’t fall into Patrol when we still remember having seen the player.

Now let’s define the Investigate behavior. We want our guy to MoveTo the LastSeenLocation, then wait a second, and then we want them to start checking around 
looking for the player within a radius of the LastSeenLocation for a little while, and then give up and go back on Patrol.

The 'checking around looking for the player' part is very similar to the LookAround sub-behavior we set up last time, so we want to start with a Simple Parallel with a Wait task 
in the primary and another Sequence in the secondary. Set the Wait to 10 seconds or so. Also like in LookAround we want to start that sequence with GetRandomLocation 
(this is another place where using EQS is a valid approach, and probably will get better results, but random location will do for now). 
But unlike LookAround we want the Origin to be the LastSeenLocation (we can re-use LookAtLoc to store the found random location). 
We then want to MoveTo the LookAtLoc, Wait a second, and then go back to the LastSeenLocation. 
So for the duration of the primary Wait task, our guy will move back and forth between the LastSeenLocation and a random location. 

Here's that sub-behavior in BT:

<img src="https://github.com/DKesserich/Blog/blob/main/images/Investigate_Sub_Behavior.PNG?raw=true" alt="Screenshot of the sub-behavior for investigating after reaching the last seen location">

Once that ten seconds is up we want to go back to Patrol, so we need one more task after the Simple Parallel to clear the LastSeenLocation key. 
Even though this seems like something that would be very commonly used and perfectly logical to have, there is no built-in Behavior Tree Task to clear a Blackboard value. 
It’s a simple enough task to create, we just need a Blackboard Selector variable, and on Receive Execute just get that variable, drag out Clear Blackboard Value, 
and then Finish Execute Success.

The Blueprint should look something like this:

<img src="https://github.com/DKesserich/Blog/blob/main/images/Clear_BB_Value_Task.PNG?raw=true" alt="Screenshot of the BP graph for a Clear Blackboard Value task">

We add that task after the Simple Parallel finishes, and that's our Investigate behavior.

<img src="https://github.com/DKesserich/Blog/blob/main/images/Full_Investigate_Behavior.PNG?raw=true" alt="Screenshot of the full investigate behavior">

Let's take a look at what we hath wrought:
<iframe width="560" height="315" src="https://www.youtube.com/embed/PR0Jok3OSbY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

And this is what our full Behavior Tree looks like now:
<img src="https://github.com/DKesserich/Blog/blob/main/images/Full_BehaviorTree_With_Investigation.PNG?raw=true" alt="Screenshot of the full Behavior Tree with the Investigation Behavior">

Looking at that we can see that it's _already_ getting a little unwieldy, and it'll only get moreso when we start adding more complex combat behavior, 
so in the next part I’ll get into using the Run Behavior and Run Behavior Dynamic tasks and how those can be used to both clean up the main tree and make 
everything a little more modular and reusable, and hopefully _finally_ get into making Combat better.
