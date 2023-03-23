# Accessing C++ from Blueprints
Today we will add two features to our project from last class. Both will take advantage of Blueprints, and part of the goal of this class is to spend a bit more time in the various environments Blueprints gives us while also connecting 
Blueprints to C++.

1. We’ll attach a particle system (electricity) to connect our player and our target when we hit the target with a trace.
2. *(delayed until next class)* We’ll create a simple GUI to display, for example, player health.

**IF YOU HAVEN’T DONE IT YET, PLEASE DOWNLOAD THE CONTENT EXAMPLES PROJECT FROM THE EPIC GAMES LAUNCHER.** You can find it under the “Samples > UE Feature Samples” tab.

## Adding a particle system
### Importing our particles
In our [last class](ue_tutorial_day3.md) we added a line trace that checked to see if the “shot” our sphere pawn fired intersected with an enemy cube 
(spheres are natural enemies of cubes natch).  Now we’re going to add a particle emitter that will be placed when a hit occurs… 
this effect will create a stream between the player and the target object.

First thing we need to do is migrate our particle effect from our Content Examples project. You can do this in different ways, but the easiest is to
simply find the `.uasset` file you're interested in the Context Examples project and copy it into the `Content` directory of your Unreal project. You can
find the file in `ContextExamples > Content > ExampleContent > Effects > ParticleSystems`. I'll use the `P_electricity_arc.uasset` file. Copy this into
your project directory (again, into the `Content` folder).

Great! You’ve now added your particle system to the project. With your project open, you should see the effect in your content browser.

### Creating a “hit” event for Blueprints
We’re going to edit / dynamically place our particle emitter using Blueprints, but before we can do that we need to create 
an event that will tell us when our hit test actually strikes the enemy cube. 

We’ll add this “HitSomething” function to the header for our `SpherePawn` class:

```c++
UFUNCTION(BlueprintNativeEvent)
void HitSomething(class UStaticMeshComponent *m);
void HitSomething_Implementation(class UStaticMeshComponent *m);
```

Important (and bizzaro) things to point out, that all relate to the bindings between these C++ classes, the editor, and the reflection engine:

1. The `BlueprintNativeEvent` specifier will enable us to create a node in Blueprints that outputs an event trigger.
2. We need to create two separate functions. One generates our blueprints event, and one contains an implementation for the event. 
3. You must name the implementation the same as the Blueprints event + `_Implementation` or the compiler will throw an error. This is a very specific way to create bindings between C++ / Blueprints / the editor. Ugh.

OK, with that out of the way, we can all our `HitSomething` method from our `Fire` method (assuming we do actually hit something). In the `SpherePawn` C++ file, switch the following line in the `Fire` method:

```c++
hr.GetActor()->Destroy();
```

 to the following:
```c++
HitSomething_Implementation(Cast<UStaticMeshComponent>(hr.GetActor()->GetRootComponent()));
```

The root component of our cube will typically (but not necessarily) be a StaticMeshComponent, so we’ll cast it as such, and then pass it to our `HitSomething_Implementation`, which we’ll define as below:

```c++
void ASpherePawn::HitSomething_Implementation(class UStaticMeshComponent *meshThatWasHit ) {
	HitSomething(meshThatWasHit);
	UE_LOG(LogTemp, Warning, TEXT(“HIT SOMETHING CALLED”));
}
```

Note that calling `HitSomething` (which we defined as a UFUNCTION with the BlueprintNativeEvent specifier) is what actually triggers the event in Blueprints. We have to include that call.

Compile your C++ code and then move on to creating / editing a blueprint subclass of it.

### Into the Blueprint
OK, so now we want to make a Blueprint “subclass” out of our `SpherePawn` C++ class, so that we can access the Blueprint event we created. 
Go ahead and delete any previous instance of the C++ `SpherePawn` class that you had placed in the scene. 
Next, right-click on the SpherePawn C++ class in your content browser
and choose “Create Blueprint class based on SpherePawn”.  Name the new blueprint `SpherePawn_BP`.

Open up the resulting Blueprint. Click on the “Event Graph” tab to open the visual programming environment, or just select the corresponding tab if it's already open.
We’re not going to use any of the events they give us, instead we’ll use the one we just added in C++... go ahead and delete the other events.

Right-click in any of the blank space and then type “Hit” to search through the available nodes. 
You should see a “Event Hit Something” appear in the dropdown menu; select it. Great! We now have access to the C++ event we created.
The arrow at the top of this node is the control flow for event processing, while the blue output marked `M` is the mesh that has been hit.

Now let’s spawn our particle emitter. Right-click on the Blueprints canvas again and search for “Spawn Emitter Attached”. 
Connect the topmost control flow output from our “Event Hit Something” node to the topmost control flow input in our “Spawn Emitter Attached” node. 

Next, we need to select the particle system that will be spawned. Click on the “Emitter Template” input for the spawn node, 
and then select the “P_electricity_arc” that we migrated to this project earlier.

Almost there! Connect the “M” output to the “Attach to Component” input of our spawn node. Set “Location Type” on the spawn node to be “Snap to Target, Including Scale”.

Last step: drag an instance of your `SpherePawn_BP` component into your level. Assign it a mesh of your choosing for representation. 
Make sure you to tell the pawn to “possess” via the Auto Possess Player option. Make sure the SpherePawn is selected (not the mesh component) and 
then choose Pawn > Auto Possess Player > Player 0 from the Details tree menu.

OK! Compile your blueprint, hit Save, and then hit Play. You should see the particle system placed when you hit the cube with your line trace. 
It might help to make the cube a bigger target (by scaling it) so it’s easier to hit.

### Orient our particles
Right now, although the particles are placed correctly, they are not oriented to create a connection between our player and the cube. 
Let’s change this. First we need to edit the particle system itself. 
Find it in the Content Browser and then double click on it to edit. 

Click on the “Source” property of the “beam” emitter in the particle editor. 
Change its value to be “Emitter”. This means the particles will begin at the source of the emitter. 
Next click on the “Target” property of the beam emitter and set it to be “User set”. 
This will enable us to set its value from Blueprints. Save the particle blueprint and close the editor; 
alternatively, feel free to play around with some of the other properties (color, size, etc.)

Back in the blueprint for our “SpherePawn”, drag the “Mesh” component into our canvas from the Components panel in the upper left corner; 
this automatically creates a Mesh node that will represent whatever Mesh we assign to our `SpherePawn`. You might need to add the `BlueprintReadOnly` desgination
to the `UPROPERTY` declaration for this property, so that it reads:

`UPROPERTY(EditAnywhere, BlueprintReadOnly)
class UStaticMeshComponent * Mesh;
`

Drag off a connection and release, this will bring up the new node dialog box. Search for `GetWorldLocation`, 
this node will find the location of the mesh in the world. Once selected, the output of your mesh will be automatically connected to 
the relevant input (in this case there’s only one, `Target`) in the new node that is created.

OK, our next step is to set the target location of our beam. 
Drag a pin off of the “Return Value” output of our `GetWorldLocation` node, and then do a search for the “Set Beam Target Point” node; 
the location will be auto-connected to the “New Target Point” property. If `Set Beatm Target Point` doesn't appear in your Blueprints list, make sure you 
uncheck that `Context Sensitive` option in the upper right corner so that it shows all nodes available.

Connect the control flow output of our Spawn Emitter to our Set Beam Target Point control flow input, and then connect the Return Value (which represents the emitter) of the Spawn Emitter node two the “Target” attribute of the Set Beam Target Point node.  Compile your Blueprint, save it, and hit Play. You should now have correctly oriented particles!

### Set the particles to flow between pawn and enemy
Last but not least we’ll set the particles to flow between the pawn and the enemy. To do this, all we need to do is keep calling Set Beam Target Point with a brief delay.  Drag a pin out of the control flow output from Set Beam Target Point, and do a search for Delay. Set a duration of .05 (in seconds) by typing it into the property field. Last but not least, connect the output of the delay to the control flow input of the Set Beam Target Point. This, in effect, creates a recursive function that calls itself over time, constantly updating the location of the player and thus changing the positioning of the particle beam. 

Save and compile the blueprint, and then test your scene to see the results. Your final blueprint should look something [like this](https://github.com/imgd-4000-2020/syllabus-and-notes/blob/master/images/day4/beam_blueprint.png)
