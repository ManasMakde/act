# 🎭 Act Pattern
![Status: Alpha](https://img.shields.io/badge/status-alpha-orange)	 
![License](https://img.shields.io/github/license/ManasMakde/act?style=flat&color=yellow)  


A game programming pattern for creating and managing complex behaviors with parallelism at its core.

> ## ⚙️ Game Engine Specific   
> ### 1. [act-unity](https://github.com/ManasMakde/act-unity)  
> ### 2. [act-godot](https://github.com/ManasMakde/act-godot)  
> ### 3. [act-unreal](https://github.com/ManasMakde/act-unreal)  
> 
> Need a port for a different game engine? Open a feature request to let the developer know.



## 💡 Principle
The entire pattern revolves around 2 things:
1. **Act:** This is the smallest unit of behaviour that can be carried out in the game e.g. Walk Act, Run Act, Jump Act, Shoot Act, etc.
2. **Theater:** This is responsible for keeping track of all acts and making them tick.

You define your desired behaviour in the `Act` and then "perform" it whenever you want that behaviour to run. The perform follows this lifecycle:  

<img src="images/act-lifecycle.png" width="450" alt="act lifecycle">

Since every `Act` is meant to be self-contained they can all perform in parallel (e.g. Look Act, Walk Act & Shoot Act). However, in cases where some acts conflict (e.g. Walk Act & Run Act) or depend on other acts (e.g. Reload Act & Shoot Act) there are 2 mechanisms to handle such situations: **Blocking** & **Prologuing**

### 1. Blocking
Each act has a list of other acts it should block when it performs. A blocked act cannot perform nor block any other act.  
There are 2 types of blocks:  
- `Persistent Block`: This type makes the block last for the entire duration of the blocker act's perform.
- `Oneshot Block`: This type only interrupts the blockee act's perform when the blocker act starts to perform, The blockee act can then again be performed even if the blocker act is still performing.

### 2. Prologuing
In situations where an act needs another act to perform before itself, the prologuing mechanism is used.  
Each act has prologue acts i.e. acts that need to be performed before the main act can perform.

<img src="images/prologue-example-1.png" width="400" alt="prologue chain example"><br/>
In example 1, The arrow pointing from an act denotes "who is my prologue" therefore B is a prologue of A and C is a prologue of B and D is a prologue of C. So in this example first D performs then C then B then A

> **Terminology Note:**  
> 1. Acts that comes before an act are called **prologue** acts (e.g. B is a prologue of A)  
> 2. Acts that come after an act are called **epilogue** acts (e.g. C is an epilogue of D)
> 2. Acts that are at the top of the chain are called **top epilogue** acts (e.g. A is a top epilogue of B, C, D)

<br/>

An act (B) can also have more than 1 prologues in such a case both prologues work in parallel (C & D) and only when both have finished does the act (B) perform:  
<img src="images/prologue-example-2.png" width="300" alt="prologue branched example"><br/>

Similarly, An act (C) can have more than 1 epilogues (A & B) and both of them will wait for the prologue (C) to complete first before performing themselves:  
<img src="images/prologue-example-3.png" width="300" alt="prologue merging example"><br/>

However cyclic prologues are not allowed:  
<img src="images/prologue-example-4.png" width="380" alt="prologue cyclic incorrect example"><br/>

> **Note:**  
> No acts wthin the same prologue chain can block each other.



## 🧭 Usage
> **Note:**  
> All examples are in Godot for simplicity but the concept applies to other engines as well.   
> **For examples in other engines take a look at [Game Engine Specific Implementations](#-engine-specific)**

### How to create an Act?
Start by extending/inheriting from the `Act` class:
```gdscript
class MyAct extends Act:
```

Now you can implement the desired behaviour by overriding the protected methods or assign values to protected properties. Here are the rules of thumb for each lifecycle method:

1. `_setup()`: Invoked once during initialization
2. `_can_perform()`: Define conditions that allow/disallow performing
3. `_enter()`: Where the actual core logic lives on each perform
4. `_tick()`: Incase the core logic needs continious updates while performing
5. `_exit()`: Used for cleanup after core logic on each perform
6. `_cleanup()`: Invoked once during deinitialization

And here are the rules of thumb for properties:

1. `_can_reperform`: Set true if act is allowed to perform again while already ongoing.
2. `_tick_flags`: The type of ticking (if any) while performing

<br/>

> **Note:**  
> For an act to tick there are 2 conditions:  
> a. The `_tick_flags` must set to something other than `TickFlags.NONE`  
> b. `_enter()` must return `Outcome.NONE`, Returning anything else will immediately lead to `_exit()`

### How to create a theater?
Unlike act, a theater is not meant to be extended instead it needs to be added as a component to the owner (The owner can be anything e.g. Player, Enemy, Machinery, etc)

```gdscript
class_name Player 
extends CharacterBody2D

@onready var theater:Theater = $Theater
```

<img src="images/theater-scene-tree.png" width="250" alt="Theater Scene Tree Example"><br/>

### How to use Acts & Theater?
Here's an example of a simple platformer character:

```gdscript
class_name Player extends CharacterBody2D

@onready var theater:Theater = $Theater
var walk_act := ExampleActs.MoveAct.new()
var run_act := ExampleActs.MoveAct.new()
var jump_act := ExampleActs.JumpAct.new()
var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")

func _ready():

	# Setup Walk
	walk_act.speed = 300.0
	walk_act.init(theater)

	# Setup Run
	run_act.speed = 500.0
	run_act.init(theater)

	# Setup Jump
	jump_act.speed = -400.0
	jump_act.init(theater)

func _physics_process(delta: float) -> void:

	# Gravity & No Slide
	velocity.y += gravity * delta
	velocity.x = 0.0

	# Jump
	if Input.is_action_just_pressed("ui_accept"):
		jump_act.perform()

	# Walk/Run
	var direction := Input.get_axis("ui_left", "ui_right")
	var movement_act := run_act if Input.is_action_pressed("ui_shift") else walk_act
	movement_act.direction = direction
	movement_act.perform()

	# Disable/Enable controls
	if Input.is_action_just_pressed("ui_text_backspace"):
		theater.set_enabled(!theater.is_enabled())

	move_and_slide()
```

> **Note:**  
> You must initialize an act using `init()` before you can perform it.


Here are examples of the act classes used:

```gdscript
class MoveAct extends Act:
	
	var speed := 100.0
	var direction := 0.0

	func _can_perform():
		return !is_zero_approx(direction)
	func _enter():
		get_owner().velocity.x = direction * speed
		return Outcome.SUCCESS
	func _exit():
		direction = 0.0
class JumpAct extends Act:
	
	var speed := -400.0
	var direction := Vector2.ZERO

	func _can_perform():
		return get_owner().is_on_floor()
	func _enter():
		get_owner().velocity.y = speed
		return Outcome.SUCCESS
```


## ❤️ Sponsors
If this has been useful in your projects consider supporting its development.  
Any support motivates to keep the project well maintained, documented and growing.


## 🔑 License
MIT © [Manas Ravindra Makde](https://manasmakde.github.io/)
