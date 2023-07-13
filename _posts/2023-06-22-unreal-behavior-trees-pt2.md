---
title: "Unreal Behavior Trees: A Blogtorial Pt. 2"
date: 2023-06-22
---

In part 1 of this series I described setting up a super-basic three-state NPC in Behavior Tree. 
In this part I want to get into adding a couple more little tasks to juice one of the states up and make them seem more lively, and fixing an undocumented issue with AI Perception.

### Adding Some Juice

So to begin, let’s take a look at the Patrol behavior as currently defined:

<img src= "https://github.com/DKesserich/Blog/blob/main/images/Basic_Patrol.png?raw=true" alt="A Screenshot of the basic patrol behavior">

Three simple tasks run in sequence: Set the state of the animation, get the next patrol point, and move to the patrol point.

Let’s look at this in action using Simulate:

<iframe width="560" height="315" src="https://www.youtube.com/embed/_oPMkinNLuY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

As you can see, they’re just sort of ping-ponging between patrol points. That’s kind of silly looking. 
So what we want is for them to stop and look around randomly for a few seconds when they reach a patrol point, and then go to the next one.

To do that the first two things we’ll need are a new Vector type Blackboard entry for our Look At Location, 
and a custom Task that will get a random location within a radius of the NPC (This can also be done with the Environment Query System, but for my purposes right now 
random location works fine). 

So we make a new Blueprint of the type BTTask Blueprint Base and name it Get Random Location. This Task needs four variables: two of the type Blackboard Key Selector, 
one for the Origin (the point that we’re searching in a radius around), one for Found Location output, one Float variable for the radius, and one Vector for an internal origin reference. 
The Origin, Found Location, and Radius variables should all be Instance Editable.

In the Blueprint graph, get the Origin variable and drag a ‘Get Blackboard Value as Actor’ and a ‘Get Blackboard Value as Vector’ node out from it. 
From the ‘As Actor’ node drag out an Is Valid macro. For the Valid branch, we want to Get Actor Location and set it to our internal origin variable. 
For the Invalid branch, just set the ‘As Vector’ value to the internal origin. This way we can use either an Actor’s location, or just any location we want as our origin.

Add a ‘GetRandomLocationInNavigableRadius’ node and plug the internal origin variable to the Origin input, and the Radius variable into the Radius input. 
Also plug the execution output pins from the Set Variables into the input on the GetRandomLocation node. Then put a branch with the Return Value bool as the input, 
and for the True branch grab the Found Location variable and drag out Set Blackboard Value as Vector and plug the Random Location into the value, 
and then from there add a Finish Execute and mark Success to True. For the False branch, add a Finish Execute and leave Success as false.

You should end up with something that looks like this:

<img src = "https://github.com/DKesserich/Blog/blob/main/images/Get_Random_Loc_BP.png?raw=true" alt = "A screenshot of the GetRandomLocation BP logic">

Now that that’s set up, time to go back to our Behavior Tree and add a Look Around task (I say task, but this is really more of a sub-behavior) to the Patrol behavior. 

I find that the easiest way to set up a timed task is with the Simple Parallel Composite node. There is a Time Limit decorator, but that returns a Fail when the time limit expires, 
which can cause a fail all the way back up the tree, and we don’t want that. 

When you add a Simple Parallel node, you’ll see that the bottom of the node is separated into a purple section and a grey section, 
where all other composite nodes just have a grey section. Whatever task or composite is plugged into the purple section (Edit: Turns out composites can't be plugged into the purple section, only tasks. But you CAN use Run Behavior tasks. See Part 4 of this series for more on them) is what determines when the Simple Parallel completes, 
and whether its completion is a success or a failure. So we just want to plug a Wait task into the purple section. Into the grey section we want to plug a Sequence composite, 
and from there we want our Get Random Location task, a Rotate To Face BB Entry task, and another wait task. 
For the Get Random Location set the Found Location key to the Look At Location Blackboard Entry and the Origin to SelfActor, and the radius to a meter or so. 
For Rotate to Face BB Entry set the Blackboard Key to the Look At Location. 
And for the final Wait task, set the time to something less than the Wait that’s plugged into the purple section.

So you should end up with a Patrol behavior that looks like this:

<img src = "https://github.com/DKesserich/Blog/blob/main/images/Patrol_With_LookAround.png?raw=true" alt = "A screenshot of the Patrol behavior with the added LookAround task">

That sequence will now run repeatedly until the primary Wait in the Simple Parallel finishes, and then Patrol will start again.

Now let’s look at it in action:

<iframe width="560" height="315" src="https://www.youtube.com/embed/D_9p_Zncxrk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

(I added a +- 0.5 seconds to the final 'Wait' in the lookaround sequence to make it even more random)

### Some Notes on Pawn and Perception Rotation

Taking a step back from Behavior Tree and AI design for a second: if this is your first time setting up an AI driven Pawn, 
you’ll probably notice that the way the pawn rotates is pretty jerky by default. This can be fixed up with a couple of settings in your AI Pawn blueprint. 
In the main settings of the Pawn look for the ‘Use Controller Rotation Yaw’ setting and make sure that it’s unticked. 
Then, in the Character Movement Component settings look for the 'Character Movement (Rotation)' settings and tick ‘Use Controller Desired Rotation’ and untick ‘Orient Rotation To Movement.’
They should now rotate smoothly. You can also change the Z value of the Rotation Rate vector to change how quickly they rotate.

But you may notice another issue when you actually play: The AI can sometimes see the player even when the AI doesn't appear to be facing towards them. 
If you turn on the AI perception debugging visualizer (press apostrophe on the keyboard to turn on AI debugging during play, 
then press 4 on the numpad to turn on the perception visualizer), you’ll see that the vision cone instantly snaps to point towards either the Patrol Point location, 
or the LookAtLocation if it’s doing the LookAround. Unfortunately, so far as I have been able to find, there is no combination of Blueprint settings that will fix this 
(the core issue is that when you use MoveTo or RotateToLookAt in Behavior Tree, those nodes set a Focus Point for the AI Controller, and if the AI Controller has a Focus Point, 
it rotates to face it immediately and ignores its controlled pawn’s rotation rate).

If you’re a pure Blueprint user, I’m sorry, but you’re gonna have to get your hands dirty with native code to fix this, 
as there is a function that is not exposed to Blueprint that we have make a native sub-class to override.

The easiest way to make a native sub-class is to open up the content drawer and in settings make sure ‘Show C++ Classes’ and ‘Show Engine Content’ are ticked. 
Go to the Engine folder in the Content tree and search for Character. You should see something that says ‘Character’ with the type ‘C++ Class.’ 
Right-click that and select ‘Create C++ Class Derived From Character.’ You’ll then have to pick a name for your new Character sub-class and click through the dialogs, 
at which point Unreal will try to compile the new code and usually something weird will happen and it will say you need to exit Unreal and build it yourself. 
Which is fine, we needed to exit Unreal anyway. 

Close Unreal and open the [YourProjectName].sln in Visual Studio. Open the [YourNewCharacterClass].h and add the following under Public:
```
virtual void GetActorEyesViewPoint(FVector& out_Location, FRotator& out_Rotation) const override;
```

You should then be able to right-click `GetActorEyesViewPoint`, click ‘Quick Actions and Refactoring’ and select ‘Create Declaration/Definition’ 
and it will automatically set up the function definition in [YourNewCharacterClass].cpp.

In the function definition you’ll just want to add
```
out_Location = GetPawnViewLocation();
out_Rotation = GetActorRotation();
```
(if you want to get really fancy you can get the location and rotation from a socket on the Character’s skeletal mesh, 
but for my purposes sticking with the default GetPawnViewLocation and getting the Actor rotation is good enough.)

Build your code and re-open Unreal. Open up your AI Pawn’s blueprint and select File->Reparent Blueprint, search for your new character class and select it. 
Then compile and save your Pawn BP. Now when you turn on the perception visualizer and run you should see the vision cone’s rotation properly connected to your pawn’s.

And that's Part 2. In Part 3 I'll get into adding a whole new state. And then, in Part 4 (hopefully), I'll start getting into making the Attack state more reactive.


