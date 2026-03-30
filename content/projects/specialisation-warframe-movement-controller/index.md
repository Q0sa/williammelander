+++
date = '2026-03-23'
draft = false
title = 'Specialisation: Warframe Movement Controller'
tags = ["C++", "Custom Engine", "Solo","Jolt Physics", "Third Person Camera"]

summary = """Specialisation project, recreating Warframe's Motion Controller, along with a simple Third Person Camera."""

showTableOfContents = true

+++

## Intro
This is a project I developed as part of the specialisation course at The Game Assembly, totaling in ~80 hours of work. This course allowed us to create a project of something that we were interested in. Since I felt quite daring I decided to recreate Warframes Motion Controller.

My reasons choosing this as a Specialisation:
- Warframe, at least in my eyes, is the King of Fast-Paced Third Person Movement.
- Wanted to see how far I could push my limits when comes to Gameplay Programming.
- Seemed like an incredibly fun challenge (it was hehe).

### Minimum Features
Here were the minimum features I planned to include:
- ✅ [A Third Person Camera](#simple-third-person-camera)
- ✅ [Basic Movement (Walking with WASD)](#basics)
- ✅ [Sprinting](#normalonground)
- ✅ [Jumping](#jump)
- ✅ [Sliding](#slide)
- ✅ [Dodge Rolling](#dodge-roll)
- ✅ [Bullet Jumping](#bullet-jump)
- ✅ [Air Gliding](#air-gliding)


### Wish Features
Here are the features that I wanted to add if I was ahead of schedule:
- ✅ [Camera handling for crouch movement](#crouch-and-shoulder-swap)
- ✅ [Aim Zoom-In/Out](#aim-zooming)
- ❌ Camera Lag when reaching high speeds
- ❌ Force Input Release
- ❌ Sliding takes slopes into account, for slowing down / speeding up
- ❌ Heavy Landing state that stuns player for a few seconds

### Additional Features that I added down the line
Here are features that I didn't plan to add but became a "spur of the moment" additions:
- [Shoulder Swapping](#crouch-and-shoulder-swap)
- [Mixamo Animations and Model](https://www.mixamo.com/#/?page=1&type=Motion%2CMotionPack) 

I was able to implement most of the features that I had laid out which I am happy with, with few Wish features as well. I was also able to see how incredibly subtle and complex the motion system is when it comes to its camera, input handling, and physics. The amount of polish that has gone into the game is incredible which makes me happy that I chose this as my specialisation!

I developed the specialisation with my groups engine, the "RatTrap Engine". The physics engine we use is Jolt Physics.

## Simple Third Person Camera
First things first, the Camera. In our custom engine, the 'Rat Trap Engine', we do have a camera parent class which we use to create different camera implementations, however we didn't have a third person camera implementation, so this was the first thing I implemented. 

The reason why I call it 'simple' is due to it not having super sophisticated camera handling such as surface collision checking, velocity adjustment (speed affecting offsets), different actions affecting camera distance from the pivot point, etc.

I tried to match the cameras pivot offset and FoV as closely as I could to Warframe by comparing my implementation and Warframe side-by-side.


### Shoulder Swapping

![CamShoulderSwap](/img/CamShoulderSwap.webp)


Shoulder swapping in most games is where the cameras X offset is "swapped", or rather which side it is offset to. Warframe has the same functionality. 

```cpp{title="ThirdPersonCamera.cpp"}
void ThirdPersonCamera::ToggleShoulderSwap()
{
    if (myShoulderTransitionLerpProgress == 1.f)
    {
        myStartShoulderOffset = myDesiredOffsetFromPivot.x;
        myDesiredShoulderOffset = -myDesiredOffsetFromPivot.x;
        myCurrentTransitionTime = 0.f;
        myShoulderTransitionLerpProgress = 0.f;
    }
}

void ThirdPersonCamera::HandleShoulderSwap(float aDeltaTime)
{
    if (myShoulderTransitionLerpProgress != 1.f)
    {
        myCurrentTransitionTime += aDeltaTime;
        myShoulderTransitionLerpProgress = std::min(myCurrentTransitionTime * myTimeToCompleteShoulderSwapDiv, 1.f);
        myDesiredOffsetFromPivot.x = FMath::Lerp(
            myStartShoulderOffset, 
            myDesiredShoulderOffset, 
            EaseOutQuartConverter(myShoulderTransitionLerpProgress));
    }
}
```
Above is how I handled shoulder swapping. As I described, the concept behind it is very simple, upon toggling, take the current X offset and "swap" it to its opposite and lerp between them.

So each frame when `myShoulderTransitionLerpProgress` has not reached 1.f, it adds to the current transition time which is then multiplied by the `myTimeToCompleteShoulderSwapDiv` (1.f / shoulderSwapDuration). This does get the job done, when toggled it sets the start and end shoulder offset as well as resetting the progress values, then progressing each frame until reaching `myDesiredShoulderOffset`. 

A downside to this implementation is that it is a smidge overcomplicated, rather than directly manipulating the alphas direction of change it instead relies on `myCurrentTransitionTime` which then converts to the `myShoulderTransitionLerpProgress` which always transitions from 0 to 1. Due to this the player can't toggle the shoulder swap again until the transition has been completed, and this is a direct result from introducing to many moving parts to something that should simple.

This overcomplication is something I rectified when I implemented the crouch offset lerping.

### Crouch

![CamCrouch](/img/CamCrouch.webp)


In Warframe, regardless of what the current movement state is, if the player is holding down the `Crouch` button the cameras Y offset adjusts downward. I set the crouch camera offset to be ~60% of the regular standing height Y offset.

```cpp{title="ThirdPersonCamera.cpp"}
void ThirdPersonCamera::HandleCrouch(float aDeltaTime)
{
    float changeRate = RatFrameConstants::Camera::CROUCH_ALPHA_CHANGE_RATE * aDeltaTime;
    myCurrentCrouchAlpha += myShouldCrouch ? changeRate : -changeRate;
    myCurrentCrouchAlpha = std::clamp(myCurrentCrouchAlpha, 0.f, 1.f);
    
    myDesiredOffsetFromPivot.y = FMath::Lerp(myStandingYOffset, myCrouchingYOffset, EaseInOutQuartConverter(myCurrentCrouchAlpha) );
}
```
In the [Shoulder Swap](#shoulder-swapping) implementation, the alpha transitions from 0 to 1 regardless of going from standing to crouching or crouching to standing, always swapping a start point and end point.

When I implemented the crouch handling however, I instead use the `myStandingYOffset` and `myCrouchingYOffset` as bounds. So the `myCurrentCrouchAlpha` almost act like a bucket filling with water, where crouching increases the alpha to 1 (filling the bucket) and releasing crouch would then decrease the alpha back to 0 (emptying the bucket).

A potential improvement to reduce the coupling between the player and camera would be to remove the `myShouldCrouch` variable and setter from the camera, instead having the player request for the `myCurrentCrouchAlpha` to increase each frame. Then if/when the player has not requested an increase in the alpha that frame then it would start draining again, bring the player back to standing. 

A potential side affect of doing this would be that if you want to pause the player script for whatever reason AND still hold the camera in a crouched position you would need to either block the crouching to standing transition or continue requesting `myCurrentCrouchAlpha` increases each frame. But thats just an edge case.


### Aim Zooming
![CamZoom](/img/CamZoom.webp)

Since Warframe is a shooter, I felt that implementing some for of aiming functionality for the camera was important, even if the character model isnt holding a weapon. From what I can tell Warframe goes about this by decreasing the FoV rather than decreasing the Z offset.

There are three states that the zoom can be in:
- Unzoomed : Completely zoomed out
- In Air Aim Zoom : Partially zoomed in
- On Ground Aim Zoom : Completely zoomed in

```cpp{title = "ThirdPersonCamera.cpp"}
    void ThirdPersonCamera::HandleAim(float aDeltaTime)
    {
        float alphaUpperLimit = myIsInAir ? myInAirAlphaLimit : 1.f;
        float changeRate = RatFrameConstants::Camera::AIM_ALPHA_CHANGE_RATE * aDeltaTime;
        
        if (myShouldAim)
        {
            if (myCurrentFoVAlpha <= alphaUpperLimit)
            {
                myCurrentFoVAlpha += changeRate;
                myCurrentFoVAlpha = std::min(myCurrentFoVAlpha, alphaUpperLimit);
            }
            else if (alphaUpperLimit < myCurrentFoVAlpha)
            {
                myCurrentFoVAlpha += -changeRate;
                myCurrentFoVAlpha = std::max(myCurrentFoVAlpha, alphaUpperLimit);
            }
        }
        else
        {
            myCurrentFoVAlpha += -changeRate;
            myCurrentFoVAlpha = std::max(myCurrentFoVAlpha, 0.f);
        }
        
        
        SetFoV(std::lerp(myBaseFoV, myAimFoV, EaseInOutQuartConverter(myCurrentFoVAlpha)));
    }
```

How I handle the lerping between the FoV is very similar to how I handle crouching, using the `myBaseFoV` and `myAimFoV` as two points for the alpha to lerp between. However the more complicated aspect of the implementation is handling the alpha when transitioning between aiming on the ground and aiming while in air.

`alphaUpperLimit` controls how much the camera zoom should progress, with 1 being its max (thus being fully zoomed in). When in air, `alphaUpperLimit` is reduced to 0.6. So to avoid snapping when going from fully zoomed in to the partial zoom we check if the `myCurrentFoVAlpha` is larger than our upper limit, and if it is the alpha is reduced until it has reached below the upper limit, allowing for "smooth clamping". 

There is a noticeable difference in the aim zoom implementation when compared to Warframe. In Warframe when transitioning between the different zoom states, the amount of time it takes to transition is always the same, which makes it visually consistent. With my implementation the change rate is always the same, which causes the transition between in air zooming and on the ground zooming to almost feel like a snap, since the change rate is adjusted to go from 0 to 1 over the course of 0.25s, going from 0.6 to 1 therefore take ~0.1s.

If I had more time this would be one of the things that I would polish.

## Namespace Constants
Before diving into how things are implemented in the states themselves there is an important implementation decision that I made. All constant variables that don't change under runtime are stored as constexpr's within a namespace called `RatFrameConstants`. 

It does mean that these variables are globally accessible, however it drastically reduces the amount of getters needed to access how fast the player should jump, how long it takes to crouch, aim speed, etc. It also has the benefit of keeping player related constant variables all in the same place for quick adjustments. In passed game projects it has also allowed my programmer group members to easily access player constant values.

For this specialization, most of the values are approximations from estimated distances and time to reach the useable values, as well as checking different Warframe forum posts where players discuss the differing speeds of each playable character.

<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
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

<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
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

<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
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
<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
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
<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
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
Here is a quick walkthrough of how the movement is implemented and how it compares to Warframe. I attempted to match the movement as close as I could while preserving functionality, this includes comparing it side-by-side with Warframe and searching game forums for useable velocity data. Check the [RatFrameConstant Namespace](#namespace-constants) to see the values used!
<br>

One thing to note is that the different movement states also act as states in a pseudo `Animation Controller`, so each state sets their own animations in their `OnEnter`.
<br>

This is mainly due to the animations being a spur of the moment addition, with the player model and animations taken from [Mixamo](https://www.mixamo.com/#/?page=1&type=Motion%2CMotionPack). I felt at the time that this would make the different movement states visually clear for me and others, rather than a tall cube that is flying around.
<br>

In a more proper implementation, the animations would be set in an external controller, checking the active state of the movement controller and setting/calling animations accordingly. 

### Basics
The basic, regular movement found in most games. 

#### Idle
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/Idle.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/Idle.webp" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

Simple and straight forward if the player is standing still and not providing any input, wait for either an input or a change on the physics side.


#### NormalOnGround
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/NormalOnGround.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/NormalOnGround.webp" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

Player gives a directional input while in contact with the ground, which causes the player to start walking. The player can also sprint in this state.
<br>

The actual acceleration direction is relative to the [Third Person Camera's](#simple-third-person-camera) look direction and player input.

#### CrouchOnGround
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/CrouchOnGround.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/CrouchOnGround.webp" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

The same as the [NormalOnGround](#normalonground) state, however has having different transitions depending on if the player is in this state. This is a separate state to reduce the amount of clutter within the  [NormalOnGround](#normalonground) state, specifically when it came to transition handling. 
<br>

They can (and should) be combined however, but there would need to be some extra transition request handling for it to work properly. That being said it is a code stink seeing how they perform the same action slightly differently, with the major difference being what they transition to. 

#### NormalInAir
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/NormalInAir.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/NormalInAir.webp" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

Just like with [CrouchOnGround](#crouchonground) this also functions similarly when it comes to movement. What makes it diffent is that it only provides acceleration upon input, it does not apply air friction. It has one small form of speed limitation. 
<br>

It only applies acceleration (which is 5x slower than on ground) in the camera relative movement direction until the horizontal speed in that direction has reached the set `MAX_WALKING_SPEED`. 


### Mobility!
Here is the high mobility movements that allow Warframe players the ability to outmaneuver and dodge incoming attacks, as well as the ability to fly across maps!

#### Jump
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/Jump.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/MovementJumpDouble.gif" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

In Warframe the player can jump and double jump. One thing to note is that the double jump (which occurs when the player is in air) would be blocked after using the [Bullet Jump](#bullet-jump) movement, as it (in Warframes implementation) counts also as a double jump, just with entirely functionality.

Jumping hard sets the players Y Velocity to the Jump Velocity, this does mean that if you have a higher Y velocity than the jump that it would be reduced to the jumping speed, however this is intended.

#### Dodge Roll
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/DodgeRoll.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/MovementRollForward.gif" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

Dodge Rolling in Warframe is used for dodging incoming attacks, it can be performed on the ground and while mid-air. From what I understood is that it forces the player character to move in the given input direction for around ~0.6s, covering a distance of 10 meters.

<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
<summary><b>Click here to view : Dodge OnEnter</b></summary>

```cpp {title = "Dodge.cpp"}
void Dodge::OnEnter([[maybe_unused]]int aPreviousState, PlayerStateContext& aContext)
{
    auto moveDirection = aContext.cameraRelativeInputDirection;
    auto flattenedMoveDirection = Vector3f{moveDirection.x, 0.f, moveDirection.z};
    
    if (flattenedMoveDirection.LengthSqr() <= 0.f)
    {
        flattenedMoveDirection = Engine::GetGameplayEngine().GetActiveScene()->GetComponent<Component::Transform>(aContext.entity)->GetMatrix().GetForward().GetNormalized();
        flattenedMoveDirection.y = 0.f;
    }
    
    flattenedMoveDirection.Normalize();
    *aContext.currentVelocity = DODGE_ROLL_VELOCITY * flattenedMoveDirection;
    
    myRollDuration = {};
    myWantsToJumpAtEnd = false;
    
    auto* animationSys = RatTrap::Engine::GetGameplayEngine().GetSystem<RatTrap::AnimationSystem>();
    animationSys->SetNextAnimation(aContext.entity, "Dodge"_id);
}
```

</details>

The actual velocity set is performed in the `OnEnter`, hard setting the current horizontal velocity to the flattened input direction. If there is no directional input, it uses the model transforms forward as the directional vector. 

In Warframe the player is unable to cancel the dodge roll until it is completed, with the exception of performing a [Bullet Jump](#bullet-jump). When the roll timer, `myRollDuration`, has passed the set `DODGE_ROLL_DURATION` it will transition to one of the [basic movement states](#basics). However if the player has inputted a jump input during the dodge roll (and hasn't double jumped), the jump will be queued and performed at the end of the dodge roll.

In Warframe there are varients that hold the player rotation, such as the "Sidespring", which is the same as the above but has a shorter roll distance. My implementation always causes the player model to rotate in the dodge direction unless the player is dodging.

#### Slide
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/Slide.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/MovementJumpKick.gif" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

Sliding is an essential movement action within Warframe, allowing players to utilits different sloped to build / maintain speed. That being said I was unable to fully replicate the sliding action. I was able to replicate the "Jump Kick" portion of it where the player is able to jump then gain a forward boost. The player does slide on the ground, but doesnt properly utilise ground friction and gravity.

<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
<summary><b>Click here to view : Slide OnEnter</b></summary>

```cpp {tile = "Slide.cpp"}
void Slide::OnEnter([[maybe_unused]]int aPreviousState, PlayerStateContext& aContext)
{
    if (*aContext.antiSpamTime >= RatFrameConstants::Input::ANTI_SPAM_TIMING)
    {
        *aContext.antiSpamTime = 0.f;
        RatTrap::Vector3f directionalSlideBoost = aContext.cameraRelativeInputDirection.GetNormalized() * RatFrameConstants::Movement::SLIDE_VELOCITY;
        
        aContext.currentVelocity->x += directionalSlideBoost.x;
        aContext.currentVelocity->z += directionalSlideBoost.z;
    }
}
```
</details>
<br>
<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
<summary><b>Click here to view : Slide FixedUpdate </b></summary>

```cpp {tile = "Slide.cpp"}
void Slide::FixedUpdate(float aFixedDeltaTime, PlayerStateContext& aContext)
{
    myCurrentHorizontalSpeed = RatTrap::Vector3(aContext.currentVelocity->x, 0.f, aContext.currentVelocity->z).Length();
    
    if (aContext.isOnGround )
    {
        if (myCurrentHorizontalSpeed > 0.1f)
        {
            float drop = RatFrameConstants::Movement::SLIDE_DEACCELERATION * aFixedDeltaTime;
            float currentDeacceleration = std::max(0.00000001f, myCurrentHorizontalSpeed - drop) / myCurrentHorizontalSpeed;
        
            aContext.currentVelocity->x *= currentDeacceleration;
            aContext.currentVelocity->z *= currentDeacceleration;
        }
    } 
    
    CheckFixedUpdateTransitions(aContext);
}
```
</details>

 How the player linear Velocity is handled is what I would describe as "movement controller authoritative", meaning that the movement controller determines what velocity the collider should have rather than the controller inputting a desired velocity. This does get the job done, however the fatal flaw that made implementing a proper Slide is the fact that I don't listen (((WIP))) 

#### Bullet Jump
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/BulletJump.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/MovementJumpBullet.gif" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

WANANNANANANANAAA

### Air Gliding and Gravity Handling
<div style="display: flex; gap: 10px; align-items: flex-start;">
  <figure style="width: 50%; margin: 0;">
    <img src="/img/AirGlide.webp" alt="Specialisation" style="width: 100%;">
    <figcaption>Specialisation</figcaption>
  </figure>
  <figure style="width: 50%; margin: 0;">
    <img src="/img/MovementAimGlide.gif" alt="Warframe" style="width: 100%;">
    <figcaption>Warframe</figcaption>
  </figure>
</div>

FLOATING THROUGH DA AAAAIR


## Summary

I am quite happy with the end result even though there is plenty of room for improvement. Writing this portolio page has acted like a code review between me and my passed self, which has allowed me to realise several potential improvements and unaccounted edgecases that I will learn from and take into account the next time I create a character controller.
<br>

This was the first time I implemented a Third Person Movement Controller and I was fascinated with the small edge cases and details that Warframe accounts for. Each time I finished implementing something, I discovered more and more edge case handling and small details in Warframe that revealed hints of how different parts are implemented. Quite frankly it was awesome, testing my way forward and speculating on how each action and state blends together, then trying to apply the idea of how it is potentially implemented into a movement state, adjusting values, then noticing something new that you didn't notice when comparing to the real thing, and then rinse and repeat.
<br>

One thing I wish I definetely had more time to implement my [Wish Features](#wish-features). The biggest of them being the "Forced Input Release" and "Slide Taking into Account Slopes". 
- **Forced Input Release:** Block new input updates of a certain action until the player releases and re-presses the button.
  - Would make state transitions become more definitive and improve the amount of control the player has, as the input itself would stop input transitions from occuring. 
  - *Example: If the player holds down a directional input and the crouch button to perform a slide, the player can continously hold these buttons to repeatadly [Slide](#slide)*

- **Slide Taking into Account Slopes:** Properly implement [Slide](#slide) so that sliding takes slopes into account, allowing players to use the environment to their advantage and maintain momentum.
  - This one I defintely wished I was able to implement but could never seem to get it to work, this came down partially time constraint as well as lacking the knowledge of how to go about implementing it. This could be me not utilising Jolt Physics properly, as velocity is Player authoratative rather than Collider authoratative, each movement state telling what velocity should be in but ignoring what the collider wants to do. This is definitely a lesson I will have to experiment with in future implementations. 
 
**But to put this specialisation in 5 simple words : It was challenging and Fun!**


