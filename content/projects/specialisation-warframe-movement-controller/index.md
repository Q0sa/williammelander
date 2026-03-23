+++
date = '2026-03-23'
draft = true
title = 'Specialisation: Warframe Movement Controller'

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


## Player Overview
Within the RatTrap Engine we define code that move and modify entities as Behaviors. I called the Player behavior RatFrameBehavior as a combination of the name of our engine and Warframe. 

### Namespace Constants
Before diving into how things are actually stored within the class itself there is an important implementation decision that I made. All constant variables that don't change under runtime are stored as constexpr's within a namespace called `RatFrameConstants`. 

It does mean that these variables are globally accessible, however it drastically reduces the amount of getters needed to access how fast the player should jump, how long it takes to crouch, aim speed, etc. It also has the benifit of keeping player related constant variables all in the same place for quick adjustments. In passed game projects it has also allowed my programmer group members to easily access player constant values.

For this specialisation, most of the values are approximations from estimated distances and time to reach the useable values, as well as checking different Warframe forum posts where players discuss the differing speeds of each playable character.
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

### Header Overview
```cpp{title = "RatFrameBehaviour.h"}
class RatFrameBehaviour : public RatTrap::ECS::Behavior<RatFrameBehaviour>
{
public:
    RatFrameBehaviour(RatTrap::ECS::Entity aEntity, RatTrap::ECS::SceneData* aSceneData)
        : Behavior(aEntity, aSceneData) {}
    
    static constexpr const char* BehaviorName = "RatFrame";
    
    void Awake() override;
    
    void Update(float aDeltaTime) override;
    void FixedUpdate(float aFixedDeltaTime) override;
    
private:
    
    //Mobility Blocking
    bool myHasDoubleJumped{};
    bool myHasBulletJumped{};
    
    //Input States that are passed to movement states
    bool myIsAiming{};
    bool myWantsToJump{};
    bool myWantsToCrouch{};
    bool myWantsToSprint{};
    
    float mySprintHoldTime{};
    float mySpamTime{};
    
    //Physics related
    float myBulletAirTime{};
    RatTrap::Vector3f myCurrentVelocity{};
    
    //Camera and model related
    RatTrap::Vector3f myCameraRelativeInputDirection{};
    RatTrap::Quatf myDesiredModelRotation{};
    
    //Motion Controller
    PlayerStateContext myPlayerStateContext{};
    BehaviorMachineState<PlayerStateContext>* myCurrentState{nullptr};
    std::unordered_map<RatFrameMovementState, std::unique_ptr<BehaviorMachineState<PlayerStateContext>>> myStates{};
    
    //Awake Functions
    void SetUpCharacterCollider();
    void CreateAndInitStateMachine();
    
    //Pre State Update Functions
    void ProcessInput(float aDeltaTime, RatTrap::ThirdPersonCamera* aThirdPersonCamera);
    void HandleModelRotation(float aDeltaTime, const RatTrap::Vector3f& aCameraForward, RatTrap::Quatf& aPlayerRotation);
    
    //State Machine handling
    void ReconstructPlayerStateContext();
    void RunActiveStateUpdate(float aDeltaTime);
    
    //Fixed Update Helper
    void ApplyGravity(float aFixedDeltaTime);
};

```

Above is the header, since this `RatFrameBehaviour` class is mostly movement based I made the decision of making the class itself the Motion Controller state machine handler. This is why the  `myCurrentState` and `myStates` are stored in the header rather than a seperate class. If this were to be used as an actual player class where multiple state machines would run simoultainously I would create a dedicated parent class for their handling.

In that scenario I would have moved variables and state function calls such as `myCurrentState`, `myStates`, `CreateAndInitStateMachine`, `RunActiveStateUpdate`, and `ApplyGravity` to a seperate, dedicated MotionController class. 

### Awake
```cpp{title = "RatFrameBehaviour.cpp"}
void RatFrameBehaviour::Awake()
{
    SetUpCharacterCollider();
    CreateAndInitStateMachine();
}
```

The Awake function is straight forward, first I set up the Jolt character collider and then the Motion State Machine. 

The character collider is a simple pill collider that uses an offset to ensure the players transform is at the foot of the collider. The main reason behind this is to ensure that the camera always has a consistent positional reference point that won't change if the collider changes height or shape.
<details>
<summary><b>Click here to view : SetUpCharacterCollider()</b></summary>

```cpp{title = "RatFrameBehaviour.cpp"}
void RatFrameBehaviour::SetUpCharacterCollider()
{
    JPH::CharacterVirtualSettings settings{};
    
    //set default collider shape
    settings.mShape = ShapeFactory::CreateCapsuleShape(ColliderDimensions::STANDING_CYLINDER_COLLIDER_HALF_HEIGHT,  
                                                       ColliderDimensions::COLLISION_RADIUS);
    
    auto* phys = Engine::GetGameplayEngine().GetSystem<PhysicsSystem>();
    
    phys->CreateCharacter(myEntity, //Who the virtual character belongs to
         settings, // often only used for setting initial shape
         GetComponent<Component::Transform>()->Position(),  //Spawn Position
         Quaternionf{}, //Default rotation
         true); //True to indicate it's a player
    
    //Offset so that the objects transform is at the feet of the collider rather than the center of mass of the collider
    //This is so that camera height is consistent even if the collider shape changes
    phys->GetCharacter(myEntity)->SetShapeOffset({0.f, ToJolt(ColliderDimensions::STANDING_HEIGHT * 0.5f), 0.f});
}
```
</details>
<br>

The function below is how the movement states are created and stored. Each movement state is a seperate child class to seperate responsibility as well as given a movement state ID so that (if needed) they are able to check what the previous state is. Each state also stores what state transitions are valid, similarly to an animation tree. 

The main reason I did this was to quickly visualise what transitions each movement state is allowed/supposed to do. <br>
[Read more about `BehaviourMachineState` here!](#templated-states)

<details>
<summary><b>Click here to view : CreateAndInitStateMachine()</b></summary>

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


### Update / Fixed Update

The Update is not only the Player update but also the Motion State Machines update. It first processes input from the input mapper which then is stored locally//  


## Templated States
### State Context
### Valid Transition Storage

## Movement States
### Basics
#### Idle
#### NormalOnGround
#### CrouchOnGround
#### NormalInAir

### Mobility!
#### Jump
#### Dodge Roll
#### Slide
#### Bullet Jump
#### Air Gliding

## Simple Third Person Camera
### Crouch and Shoulder Swap
### Aim Zooming

## Summary


