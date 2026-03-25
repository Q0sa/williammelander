+++
date = '2026-03-23'
draft = true
title = 'Specialisation: Warframe Movement Controller'
tags = ["C++", "Custom Engine", "Solo","Jolt Physics", "Third Person Camera"]

summary = """Specialisation project, recreating Warframe's Motion Controller, along with a simple Third Person Camera."""

showTableOfContents = true

+++

## Intro
This is a project I developed as part of the specialisation course at The Game Assembly. This course allowed us to create a project of something that we were interested in. Since I felt quite daring I decided to recreate Warframes Motion Controller.

My reasons choosing this as a Specialisation:
- Warframe, at least in my eyes, is the King of Fast-Paced Third Person Movement.
- Wanted to see how far I could push my limits when comes to Gameplay Programming.
- Seemed like an incredibly fun challenge (it was hehe).

### Minimum Features
Here were the minimum features I planned to include:
- [A Third Person Camera](#simple-third-person-camera)
- [Basic Movement (Walking with WASD)](#basics)
- [Sprinting](#normalonground)
- [Jumping](#jump)
- [Sliding](#slide)
- [Dodge Rolling](#dodge-roll)
- [Bullet Jumping](#bullet-jump)
- [Air Gliding](#air-gliding)


### Wish Features
Here are the features that I wanted to add if I was ahead of schedule:
- [Camera handling for crouch movement](#crouch-and-shoulder-swap)
- [Aim Zoom-In/Out](#aim-zooming)
- Camera Lag when reaching high speeds
- Force Input Release
- Sliding takes slopes into account, for slowing down / speeding up
- Heavy Landing state that stuns player for a few seconds

### Additional Features that I added down the line
Here are features that I didn't plan to add but became a "spur of the moment" additions:
- [Shoulder Swapping](#crouch-and-shoulder-swap)
- Mixamo Animations and Model

I was able to implement most of the features that I had laid out which I am happy with, with few Wish features as well. I was also able to see how incredibly subtle and complex the motion system is when it comes to its camera, input handling, and physics. The amount of polish that has gone into the game is incredible which makes me happy that I chose this as my specialisation!

I developed the specialisation with my groups engine, the "RatTrap Engine". The physics engine we use is Jolt Physics.


## Namespace Constants
Before diving into how things are implemented in the states themselves there is an important implementation decision that I made. All constant variables that don't change under runtime are stored as constexpr's within a namespace called `RatFrameConstants`. 

It does mean that these variables are globally accessible, however it drastically reduces the amount of getters needed to access how fast the player should jump, how long it takes to crouch, aim speed, etc. It also has the benefit of keeping player related constant variables all in the same place for quick adjustments. In passed game projects it has also allowed my programmer group members to easily access player constant values.

For this specialization, most of the values are approximations from estimated distances and time to reach the useable values, as well as checking different Warframe forum posts where players discuss the differing speeds of each playable character.
<details>
<summary><b>Click here to view : RatFrameConstants Namespace</b></summary>

```cpp{title = "RatFrameConstants.h"}

namespace RatFrameConstants
{
    namespace ColliderDimensions
    {
        constexpr float STANDING_HEIGHT(175.f);
        constexpr float COLLISION_RADIUS(50.f);
        constexpr float STANDING_CYLINDER_COLLIDER_HALF_HEIGHT((STANDING_HEIGHT + COLLISION_RADIUS * 0.5f) * 0.5f); 
    }
    
    namespace Camera
    {
        constexpr float MODEL_ROTATION_SMOOTHING(25.f); //Arbitrary number that feels good
     
        constexpr float TIME_TO_FROM_CROUCHING(0.35f);
        constexpr float CROUCH_ALPHA_CHANGE_RATE(1.f / TIME_TO_FROM_CROUCHING);
        
        constexpr float TIME_TO_FROM_AIM(0.25f);
        constexpr float AIM_ALPHA_CHANGE_RATE(1.f / TIME_TO_FROM_AIM);
    }
    
    namespace Movement
    {
        constexpr float RATFRAME_GRAVITY(23.f * 100.f); //Warframes approximate gravity, 23m/s compared to earths 9.8m/s
        constexpr float TIME_TO_MAX_SPEED_ON_GROUND(0.05f); //seconds, one could argue that this is so small is might be worth to directly set
        constexpr float TIME_TO_NO_SPEED_ON_GROUND(0.1f);
        constexpr float TIME_TO_WALK_SPEED_IN_AIR(0.5f);
        
        constexpr float MAX_WALKING_SPEED(600.f); //cm/s
        constexpr float WALKING_SPEED_ACCELERATION(MAX_WALKING_SPEED / TIME_TO_MAX_SPEED_ON_GROUND); //Topspeed / Time to Topspeed = Acceleration
        constexpr float WALKING_SPEED_DEACCELERATION(MAX_WALKING_SPEED / TIME_TO_NO_SPEED_ON_GROUND); 
        
        constexpr float MAX_SPRINT_SPEED(750.f); //cm/s
        constexpr float SPRINTING_SPEED_ACCELERATION(MAX_SPRINT_SPEED / TIME_TO_MAX_SPEED_ON_GROUND);
        
        constexpr float IN_AIR_ACCELERATION(MAX_WALKING_SPEED / TIME_TO_WALK_SPEED_IN_AIR);
    
        constexpr float JUMP_HEIGHT( 200.f ); // 2 meters
        const float JUMP_VELOCITY{ sqrtf(2.f * JUMP_HEIGHT * RATFRAME_GRAVITY)}; //sqrtf(2 * Jump Height (in CM) * Adjusted Gravity (from M to CM))
    
        constexpr float DODGE_ROLL_DISTANCE (1000.f); // 10m
        constexpr float DODGE_ROLL_DURATION (0.6f); //seconds
        constexpr float DODGE_ROLL_VELOCITY (DODGE_ROLL_DISTANCE / DODGE_ROLL_DURATION);
        
        constexpr float BULLET_JUMP_DISTANCE (1200.f); //12m
        constexpr float BULLET_JUMP_DURATION (0.6f); //seconds
        constexpr float BULLET_JUMP_VELOCITY (BULLET_JUMP_DISTANCE / BULLET_JUMP_DURATION);
        
        constexpr float SLIDE_DISTANCE (700.f); //7m
        constexpr float SLIDE_TIME_TO_COVERED_DISTANCE(0.6f); //Time it takes to cover the slide distance without friction
        constexpr float SLIDE_TIME_TO_NO_VELOCITY(0.5f); //Time it takes to slow down from sliding on flat ground
        constexpr float SLIDE_VELOCITY(SLIDE_DISTANCE / SLIDE_TIME_TO_COVERED_DISTANCE);
        constexpr float SLIDE_DEACCELERATION(SLIDE_DISTANCE / SLIDE_TIME_TO_NO_VELOCITY);
        
        constexpr float AIR_GLIDE_DURATION ( 3.f );
        constexpr float AIR_GLIDE_GRAVITY_MODIFIER( RATFRAME_GRAVITY * 0.2f ); //20% Gravity, also used as Y velocity clamp
        constexpr float AIR_GLIDE_VERTICAL_VELOCITY_DRAG( 0.5f ); //Rapidly reduce speed if the Y velocity is positive
        constexpr float AIR_GLIDE_DRAG_DURATION( 0.15f ); //Duration / Fixed Update = Number of frames where drag is applied
        
    }

    namespace Input
    {
        constexpr float DODGE_ROLL_INPUT_BOUNDS(0.2f); //If player releases sprint within this time frame, perform dodge roll
        constexpr float ANTI_SPAM_TIMING (1.f); //This is the minimum amount of time that needs to run between momentum changing states
    }
}

```
</details>

## Finite State Machine
Early on I had planned on potentially hardcoding the movement controller directly into the player. I believed that this would allow movement actions to be performed immediately upon request from the player.

But as I started implementing Dodge Rolling and looking deeper into what movement states are/aren't allowed to transition into each other in Warframe I realised that hardcoding would be :
- Rigid
- Hard to read
- A debugging nightmare

These things would be time wasters, as well looked smelly.  
I opted for the most straight forward solution : Finite State Machine.
- Would allow for properly separated states with clear purpose
- State transitions are predictable and easily read
- Much easier to debug

Since the RatTrap Engine didn't have any dedicated State parent class, I created one that could be used for not only my movement controller but also for other future applications such as AI, Animation Trees, other FSM's, etc.

<details>
<summary><b>Click here to view : BehaviorMachineState parent class</b></summary>

```cpp{title = "BehaviorMachineState.h"}
template<typename StateContext> 
class BehaviorMachineState
{
public:
    BehaviorMachineState(int aEnumID) : myCurrentStateID{aEnumID} {}
    
    virtual void Update([[maybe_unused]] float aDeltaTime, [[maybe_unused]] StateContext& aContext){ }      
    
    virtual void FixedUpdate([[maybe_unused]] float aFixedDeltaTime, [[maybe_unused]] StateContext& aContext){ } 
    
    virtual void OnEnter([[maybe_unused]]int aPreviousState = -1,  [[maybe_unused]] StateContext& aContext = {}){}
    virtual void OnExit(){}
    
    void AddStateTransition(BehaviorMachineState* aTransition){ myValidTransitions.push_back(aTransition); }
    void AddStateTransition(const std::vector<BehaviorMachineState*>& aTransitions)
    {
        myValidTransitions.insert(myValidTransitions.end(), aTransitions.begin(), aTransitions.end());
    }
    
    void ClearStateChangeRequest(){ myRequestedStateChange = nullptr; }
    BehaviorMachineState* GetRequestedNextState() const { return myRequestedStateChange; }
    
    int GetStateID() const { return myCurrentStateID; }
    
protected:
    virtual void CheckUpdateTransitions([[maybe_unused]] StateContext& aContext){}
    virtual void CheckFixedUpdateTransitions([[maybe_unused]] StateContext& aContext){}

    void RequestStateChange(int aNextStateID)
    {
        for (const auto& transition : myValidTransitions)
        {
            if (transition->GetStateID() == aNextStateID)
            {
                myRequestedStateChange = transition;
                return;
            }
        }
        
        assert(false && "Attempted to transition to an invalid state!");
    }
    
private:
    const int myCurrentStateID{};
    BehaviorMachineState* myRequestedStateChange{nullptr};
    std::vector<BehaviorMachineState*> myValidTransitions{}; //This eliminates the need to have multiple enums just to manage states
};
```
</details>

The State parent class is quite straight forward. It has an `Update`, `FixedUpdate`, and separated transition functions for transitions that occur after an `Update` (often input based) and transitions after `FixedUpdate` (often physics based). 

### State Context Templating

```cpp
template<typename StateContext>
class BehaviorMachineState
{...
```

Due to there being a decent amount of information that needs to be passed to and from each state at different parts I decided to pass contexts into states. Each of these contexts would then be recreated every frame and then passed into the active state. The state machine operator (in this case our player class) would recreate the state context every Update and then pass them into the states update.

If the template was set up differently then there would be a possibility to have multiple types of context if needed, for example one for `Update` and one `FixedUpdate`. However the main reason I am not directly wanting to grab data from the static physics system is that it feels somewhat like an overreach. However I am already grabbing information outside of the context, such as getting the `Transform` component of the player entity. So technically I am breaking my own guidelines when it comes to the amount of access the states should have. But nonetheless `StateContext` would still be used in different capacities, such as sending input data.

In a previous iteration before templating `StateContext`, it was a empty parent struct, where the `PlayerStateContext` was a child class which was passes into the active states update where it would have to be `static_cast` from the parent struct to the child struct. This was already a code smell which only worsened when I had to do it multiple times the same frame when it was passed to the transition check functions. 

```cpp{title = "RatFrameBehaviour.cpp"}
void RatFrameBehaviour::ReconstructPlayerStateContext()
{
    //Reconstruct player state context
    bool prevOnGround = myPlayerStateContext.isOnGround;
    myPlayerStateContext =
    {
        .entity = myEntity,
        
        .isOnGround = prevOnGround, //should only be updated in fixed update
        .isAiming =  myIsAiming,
        
        .wantsToCrouch = myWantsToCrouch,
        .wantsToSprint = myWantsToSprint,
        .wantsToJump = myWantsToJump,
        .wantsToDodgeRoll = !myWantsToSprint && //Sprint has been released
                            mySprintHoldTime != 0.f && //Was held previously
                            mySprintHoldTime <= Input::DODGE_ROLL_INPUT_BOUNDS //Was held and released quickly
       
        .cameraRelativeInputDirection = myCameraRelativeInputDirection,
       
        .hasDoubleJumped = &myHasDoubleJumped, //These are pointers that would then be modified
        .hasBulletJumped = &myHasBulletJumped, //   within states, these change what transitions
        .antiSpamTime = &mySpamTime,           //   are and aren't allowed
        .currentVelocity = &myCurrentVelocity,
    };    
}
```

### Valid Transition Storage
```cpp {title = "BehaviorMachineState.h -- continued"}
...
protected:
...
    void RequestStateChange(int aNextStateID)
    {
        for (const auto& transition : myValidTransitions)
        {
            if (transition->GetStateID() == aNextStateID)
            {
                myRequestedStateChange = transition;
                return;
            }
        }
        
        assert(false && "Attempted to transition to an invalid state!");
    }
    
private:
    const int myCurrentStateID{};
    BehaviorMachineState* myRequestedStateChange{nullptr};
    std::vector<BehaviorMachineState*> myValidTransitions{}; //This eliminates the need to have multiple enums just to manage states
};
```

Before each state `Update` is called the controller checks whether or not a state has been requested, which is done by checking if `myRequestedStateChange` is no longer `nullptr`. If this is the case the controller would run the `OnExit` function before swapping the active state and then clearing the request from the previous state. Upon which is performs an `OnEnter` followed immediately by the new states `Update`. 

Each state has a container for storing what states they are allowed to transition to. These would be added when the state machine (our RatFrame player class) is initialised. When a state is constructed they are each given a respective state ID (which is originally an enum). Since all states have their own ID, the state would then run `RequestStateChange` with the requested StateID and then perform the state change next `Update` (which is our RatFrame player class). If a state wants to transition to a state that does not exist within `myValidTransitions` then the program pushes an assert.

The main reasoning behind the transitions is so that if someone who hasn't worked with the player needs to bug fix something or implement a new feature, they are able to look at all the state transitions to gain a good understanding of the overall setup. I took a certain amount of inspiration from Unity's Animation Tree component.

<details>
<summary><b>Click here to view : State Initialisation and Transition Linking</b></summary>

```cpp{title = "RatFrameBehaviour.cpp"}
void RatFrameBehaviour::CreateAndInitStateMachine()
{
    
    myStates.try_emplace(RatFrameMovementState::Idle,           std::make_unique<Idle>(static_cast<int>(RatFrameMovementState::Idle)));
    myStates.try_emplace(RatFrameMovementState::NormalOnGround, std::make_unique<NormalOnGround>(static_cast<int>(RatFrameMovementState::NormalOnGround)));
    myStates.try_emplace(RatFrameMovementState::CrouchOnGround, std::make_unique<CrouchOnGround>(static_cast<int>(RatFrameMovementState::CrouchOnGround)));
    myStates.try_emplace(RatFrameMovementState::NormalInAir,    std::make_unique<NormalInAir>(static_cast<int>(RatFrameMovementState::NormalInAir)));
    myStates.try_emplace(RatFrameMovementState::Jump,           std::make_unique<Jump>(static_cast<int>(RatFrameMovementState::Jump)));
    myStates.try_emplace(RatFrameMovementState::Dodge,          std::make_unique<Dodge>(static_cast<int>(RatFrameMovementState::Dodge)));
    myStates.try_emplace(RatFrameMovementState::Slide,          std::make_unique<Slide>(static_cast<int>(RatFrameMovementState::Slide)));
    myStates.try_emplace(RatFrameMovementState::BulletJump,     std::make_unique<BulletJump>(static_cast<int>(RatFrameMovementState::BulletJump)));
    
    myStates[RatFrameMovementState::Idle]->AddStateTransition(
    {
        myStates[RatFrameMovementState::NormalOnGround].get(),
        myStates[RatFrameMovementState::CrouchOnGround].get(),
        myStates[RatFrameMovementState::NormalInAir].get(),
        myStates[RatFrameMovementState::Dodge].get(),
        myStates[RatFrameMovementState::Jump].get(),
        myStates[RatFrameMovementState::BulletJump].get(),
    });
    
    myStates[RatFrameMovementState::NormalOnGround]->AddStateTransition(
    {
        myStates[RatFrameMovementState::Idle].get(),
        myStates[RatFrameMovementState::NormalInAir].get(),
        myStates[RatFrameMovementState::Dodge].get(),
        myStates[RatFrameMovementState::Jump].get(),
        myStates[RatFrameMovementState::Slide].get(),
    });
    
    myStates[RatFrameMovementState::CrouchOnGround]->AddStateTransition(
    {
        myStates[RatFrameMovementState::Idle].get(),
        myStates[RatFrameMovementState::NormalOnGround].get(),
        myStates[RatFrameMovementState::NormalInAir].get(),
        myStates[RatFrameMovementState::Dodge].get(),
        myStates[RatFrameMovementState::BulletJump].get(),
    });
    
    myStates[RatFrameMovementState::NormalInAir]->AddStateTransition(
    {
        myStates[RatFrameMovementState::Idle].get(),
        myStates[RatFrameMovementState::NormalOnGround].get(),
        myStates[RatFrameMovementState::CrouchOnGround].get(),
        myStates[RatFrameMovementState::Jump].get(),
        myStates[RatFrameMovementState::Dodge].get(),
        myStates[RatFrameMovementState::Slide].get(),
        myStates[RatFrameMovementState::BulletJump].get(),
    });
    
    myStates[RatFrameMovementState::Jump]->AddStateTransition(myStates[RatFrameMovementState::NormalInAir].get());
    
    myStates[RatFrameMovementState::Dodge]->AddStateTransition(
    {
        myStates[RatFrameMovementState::NormalOnGround].get(),
        myStates[RatFrameMovementState::NormalInAir].get(),
        myStates[RatFrameMovementState::CrouchOnGround].get(),
        myStates[RatFrameMovementState::Jump].get(),
        myStates[RatFrameMovementState::BulletJump].get(),

    });
    
    myStates[RatFrameMovementState::BulletJump]->AddStateTransition(
    {
        myStates[RatFrameMovementState::NormalOnGround].get(),
        myStates[RatFrameMovementState::NormalInAir].get(),
        myStates[RatFrameMovementState::CrouchOnGround].get(),
        myStates[RatFrameMovementState::Jump].get(),
    });
    
    myStates[RatFrameMovementState::Slide]->AddStateTransition(
    {
        myStates[RatFrameMovementState::Idle].get(),
        myStates[RatFrameMovementState::NormalOnGround].get(),
        myStates[RatFrameMovementState::NormalInAir].get(),
        myStates[RatFrameMovementState::CrouchOnGround].get(),
        myStates[RatFrameMovementState::Jump].get(),
        myStates[RatFrameMovementState::BulletJump].get(),
    });
    
    myCurrentState = myStates[RatFrameMovementState::Idle].get();
}
```
</details>


### Transition Requests

For the typical implementation of the Movement states all transitions are handled within two functions :
- `CheckUpdateTransitions(StateContext& aContext)`
    - Handles mostly if not exclusively Input based transitions, like going from Idle to Jumping or In Air Movement to a Dodge Roll. 
<details>
<summary><b>Click here to view : Average Update State Transition</b></summary>

```cpp{title = "NormalInAir.cpp"}
void NormalInAir::CheckUpdateTransitions(PlayerStateContext& aContext)
{
    if (aContext.wantsToJump && !*aContext.hasDoubleJumped)
    {
        aContext.wantsToCrouch && !*aContext.hasBulletJumped ? RequestStateChange(static_cast<int>(RatFrameMovementState::BulletJump)) : 
                                                               RequestStateChange(static_cast<int>(RatFrameMovementState::Jump));
    }
    
    if (aContext.cameraRelativeInputDirection.LengthSqr() > 0.f)
    {
        if (aContext.wantsToCrouch)
        {
            RequestStateChange(static_cast<int>(RatFrameMovementState::Slide));
        }
    }

    if (aContext.wantsToDodgeRoll)
    {
        RequestStateChange(static_cast<int>(RatFrameMovementState::Dodge));
    }
}

```
</details>


- `CheckFixedUpdateTransitions(StateContext& aContext)`
    - Handles physics based transitions, such as when if the player has dodged rolled for the full duration, or if player is no longer touching the ground / is now in air
<details>
<summary><b>Click here to view : Average Fixed Update State Transition</b></summary>

```cpp{title = "Dodge.cpp"}
void Dodge::CheckFixedUpdateTransitions(PlayerStateContext& aContext)
{
    if (myRollDuration >= DODGE_ROLL_DURATION)
    {
        if (myWantsToJumpAtEnd)
        {
            RequestStateChange(static_cast<int>(RatFrameMovementState::Jump));
        }
        else if (aContext.isOnGround)
        {
            aContext.wantsToCrouch ? RequestStateChange(static_cast<int>(RatFrameMovementState::CrouchOnGround)) : 
                                     RequestStateChange(static_cast<int>(RatFrameMovementState::NormalOnGround));
        }
        else
        {
            RequestStateChange(static_cast<int>(RatFrameMovementState::NormalInAir));
        }
    }
}


```
</details>




As stated previously the state would then request a transition to a different state ID, if allowed the transition would occur at the beginning of the next Update. One thing I want to experiment with in the future is allowing the transitions to occur at the start of FixedUpdate however this could have side effects. But an experiment for the future nonetheless! 



## Movement States
<b>Now onto the fun stuff!</b><br>
Here is a quick walkthrough of how the movement is implemented and how it compares to Warframe. One thing to note is that the different movement states also act as states in a pseudo `Animation Controller`, so each state sets their own animations in their `OnEnter`.
<br>

This is mainly due to the animations being a spur of the moment addition, with the player model and animations taken from [Mixamo](https://www.mixamo.com/#/?page=1&type=Motion%2CMotionPack). I felt at the time that this would make the different movement states visually clear for me and others, rather than a tall cube that is flying around.
<br>

In a more proper implementation, the animations would be set in an external controller, checking the active state of the movement controller and setting/calling animations accordingly. 

### Basics
The regular movement found in most games. 

#### Idle
Simple and straight forward if the player is standing still and not providing any input, wait for either an input or a change on the physics side.



#### NormalOnGround
#### CrouchOnGround
#### NormalInAir

### Mobility!
#### Jump
#### Dodge Roll
#### Slide
#### Bullet Jump
#### Air Gliding and Gravity Handling


## Simple Third Person Camera
### Crouch and Shoulder Swap
### Aim Zooming

## Summary


