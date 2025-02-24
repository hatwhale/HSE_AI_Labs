### Project creation

Create Third Person C++ project

### Pickup actor creation

In this section we will create basic class to represent pickups in our game.

Steps to create new actor:
* Create new C++ class called `Pickup` that is based on standard class `Actor`
* Open the code editor to implement the class
* Note UCLASS and GENERATED_BODY macros - they allow interaction between Code and UE4 Editor

Add boolean flat that signifies whether pickup is active

```c++
// Is true if pickup is active and false otherwise.
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Pickup)
bool bIsActive;
```

UPROPERTY allows us to use the field inside game editor and in blueprints:
* EditAnywhere - Indicates that this property can be edited via property windows, archetypes and instances within the Editor.
* BlueprintReadWrite - This property can be read or written from a blueprint.

Add sphere component that will handle collisions

```c++
// Component for handling collisions.
UPROPERTY(VisibleDefaultsOnly, BlueprintReadOnly, Category = Pickup)
USphereComponent* BaseCollisionComponent;
```

* BlueprintReadOnly - This property can be read by blueprints, but not modified.

Add static mesh component that will represent pickup in real world

```c++
// Component that represents pickup in the real world.
UPROPERTY(VisibleDefaultsOnly, BlueprintReadOnly, Category = Pickup)
UStaticMeshComponent* PickupMesh;
```

* VisibleDefaultsOnly - Indicates that this property is only visible in property windows for archetypes, and cannot be edited.

Declare function that will be called on pick up
```c++
// Called when pickup is collected.
UFUNCTION(BlueprintNativeEvent)
void OnPickedUp();
```

* BlueprintNativeEvent - This function is designed to be overridden by a Blueprint, but also has a native implementation. Provide a body named [FunctionName]_Implementation instead of [FunctionName]; the autogenerated code will include a thunk that calls the implementation method when necessary.

Initialize pickup fields:
* Pickup should be active by default
* Create collision component and set the root component to it
* Create pickup mesh component, turn on physics for it, attach it to root component

```c++
// Sets default values
APickup::APickup()
{
    // Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
    PrimaryActorTick.bCanEverTick = true;

    // Pickup is active by default.
    bIsActive = true;

    // Create collision component.
    BaseCollisionComponent = CreateDefaultSubobject<USphereComponent>(TEXT("RootComponent"));

    // Set collision component as a root component.
    RootComponent = BaseCollisionComponent;

    // Create pickup mesh component.
    PickupMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("PickupMesh"));

    // Turn on physics (e.g. gravity, collisions).
    PickupMesh->SetSimulatePhysics(true);

    // Attach mesh to root component.
    PickupMesh->SetupAttachment(RootComponent);
}
```

Implement OnPickedUp function

```c++
void APickup::OnPickedUp_Implementation()
{
    // There is no default behavior for Pickup base class.
}
```

Note the implementation suffix - it is a base implementation of the method that can be extended in blueprints.

### Battery class creation

In this section we will create battery pickup class that will represent powerup that can be collected by player to obtain energy.

Start by creating new C++ class `BatteryPickup` based on `Pickup`.

Implicitly declare public constructor to be able to change its implementation
```c++
public:
    ABatteryPickup();
```

Create `PowerLevel` field to represent battery power

```c++
// The ammount of power stored in the battery.
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Power)
float PoweLevel;
```

Override base class `OnPickedUp` method

```c++
// Override base class method implementation to customize the battery behavior.
void OnPickedUp_Implementation() override;
```

Initialize battery power level
```c++
ABatteryPickup::ABatteryPickup()
{
    // Set default power level for a battery.
    PowerLevel = 150.0f;
}
```

Override pickup implementation to destroy the battery after pickup
```c++
void ABatteryPickup::OnPickedUp_Implementation()
{
    // Call parent implementation.
    Super::OnPickedUp_Implementation();
    // Destroy the battery after pickup.
    Destroy();
}
```

### Enabling the character to collect pickups

In this section we will implement character pickup collection and implement battery effects.

First we start by adding a mapping for collecting a pickup. For this go to Edit -> Project settings -> Engine -> Input -> Bindings.
There you can see two types of mappings:
* Action Mapping - used for discrete events (e.g jumping)
* Axis Mapping - used for continous events (e.g. movement, camera direction)

Add new action mapping called `CollectPickups` and bind it to `E` key on keyboard.
* Example: https://docs.unrealengine.com/latest/INT/Gameplay/Input/

Next we are going to implement pickups collection for our character.
All edits happen in file `Lesson_1Character.h`.
First we start with the public section of character interface.

To represent an area around character where pickups can be collected we add new component
```c++
// Virtual sphere around the characted that determines the pickups that can be collected now.
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Power)
USphereComponent* CollectionSphere;
```

We would also like to store current character power and speed
```c++
// Current power level of the character.
UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = Power)
float PowerLevel;

// Degree of influence of power on the character speed.
UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = Power)
float SpeedFactor;

// Base speed of the character.
UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = Power)
float BaseSpeed;
```

Finally, we need to override `Tick` function to control character speed when its power level changes
```c++
// Override the base function to implement character specific logic.
virtual void Tick(float DeltaSeconds) override;
```

Now we move to the private interface.

Lets declare a function that will be called when user invokes our `CollectPickups` mapping
```c++
// Collects all batteries that are in collection radius of the player.
UFUNCTION(BlueprintCallable, Category = Power)
void CollectBatteries();
```

To decouple batteries collection from the actual powerup we will use helper function that will be available from the blueprints
```c++
// Applies collected battery power to update character state.
UFUNCTION(BlueprintImplementableEvent, Category = Power)
void PowerUp(float BatteryPower);
```

We can now implement declared functions in `Lesson_1Character.cpp` file.

First we initialize default values for character properties in `ALesson_1Character` constructor
```c++
// Set initial power level of the character.
PowerLevel = 2000.0f;

// Set the power-speed multiplier and base speed of the character.
SpeedFactor = 0.75f;
BaseSpeed = 10.0f;
```

Then we create and attach the collection sphere to the root component of the character
```c++
// Create collection sphere and set it's default radius.
CollectionSphere = CreateDefaultSubobject<USphereComponent>(TEXT("CollectionSphere"));
CollectionSphere->SetupAttachment(RootComponent);
CollectionSphere->SetSphereRadius(200.0f);
```

We are finished with constructor and now move to `ALesson_1Character::SetupPlayerInputComponent` method to bind our `CollectPickups` mapping to actual method
```c++
// Bind CollectPickups key mapping to call CollectBatteries function.
InputComponent->BindAction("CollectPickups", IE_Pressed, this, &ALesson_1Character::CollectBatteries);
```

We implement our custom `Tick` method that will control character speed based on it's power level
```c++
void ALesson_1Character::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);

    // Character speed is bounded with BaseSpeed from below and linearly depends on the power level.
    GetCharacterMovement()->MaxWalkSpeed = SpeedFactor * PowerLevel + BaseSpeed;
}
```

The last but very important function that we need to implement is `CollectBatteries` that will
iterate over objects that are collectable by character (that overlap with `CollectionSphere`),
pick up all the neighboring batteries and apply their energy to power up the character
```c++
void ALesson_1Character::CollectBatteries()
{
    // Stores total collected power from one sweep of CollectBatteries call.
    float BatteryPower = 0.0f;

    // First we get all the actors that are close enough to the character (overlap with CollectionSphere).
    TArray<AActor*> CollectedActors;
    CollectionSphere->GetOverlappingActors(CollectedActors);
    for (int i = 0; i < CollectedActors.Num(); ++i) {
        // We iterate over every neighboring actor and check,
        // whether it is a battery and it is still active.
        auto* TestBattery = Cast<ABatteryPickup>(CollectedActors[i]);
        if (TestBattery && !TestBattery->IsPendingKill() && TestBattery->bIsActive) {
            // We found a battery and we collect it's power and deactivate it.
            BatteryPower += TestBattery->PowerLevel;
            TestBattery->bIsActive = false;
            TestBattery->OnPickedUp();
        }
    }

    // Finally, if we managed to collect anything, we power up the character by the collected ammount.
    if (BatteryPower > 0.0f) {
        PowerLevel += BatteryPower;
        PowerUp(BatteryPower);
    }
}
```

### Creating the batteries real-world representation

Our next step would be adding actual instances of batteries to the real world and testing `CollectBatteries` function.

First we need to create a new blueprint that will allow us to customize C++ `BatteryPickup` class.
Go to Content browser in UE4 editor and create new Blueprint Class called `BP_BatteryPickup` with `BatteryPickup` as a parent class.
* Example: https://docs.unrealengine.com/latest/INT/Engine/Blueprints/UserGuide/Types/ClassBlueprint/Creation/

Double-click on the created blueprint to edit it's settings.
To add visual representation of the Battery we need to edit `PickupMesh` component of the BP_BatteryPickup. It can be found in `Components` tab.
Choose the component and edit `Static Mesh` to point to a desired mesh.
You can with some simple mesh that is included into the default assets, for example `SM_Rock` from the StarterContent.
* Example: https://docs.unrealengine.com/latest/INT/Engine/Blueprints/UserGuide/Components/index.html

Last step is to setup collisions profile for the specified mesh. For this double-click the Static Mesh and go to `Collision` menu.
* Example: https://docs.unrealengine.com/latest/INT/Engine/Physics/Collision/HowTo/index.html

After this drag a `BP_BatteryPickup` object and add it to the level. You should see new object with specified Static Mesh appearning in the level.
Go to Play mode and try to collect the Battery by reaching it at pressing `E` key.

### Implementing random battery spawning

In this section we will create a volume that will spawn batteries at random locations of the map.

We start by creating new C++ class `SpawnVolume` that will represent a box that will spawn batteries at random periods of time and drop them on the map.
First lets declare all needed fields in file `SpawnVolume.h`.
In the public section we will need the set of default methods
```c++
// Sets default values for this actor's properties
ASpawnVolume();

// Called when the game starts or when spawned
virtual void BeginPlay() override;

// Called every frame
virtual void Tick( float DeltaSeconds ) override;
```

Our volume will be described with the box where the actual spawning should happen
and with the type of `Pickup` that should be spawned
```c++
// Box that represents the volume where random spawning will happen.
UPROPERTY(VisibleInstanceOnly, Category = Spawning)
UBoxComponent* WhereToSpawn;

// Type of pickup that will be spawned in the volume.
UPROPERTY(EditAnywhere, Category = Spawning)
TSubclassOf<class APickup> WhatToSpawn;
```

We would also like to control the periodicity of spawning by chosing it as a random number in specified interval [Low, High]
```c++
// Minimum spawning delay.
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Spawning)
float SpawnDelayRangeLow;

// Maximum spawning delay.
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Spawning)
float SpawnDelayRangeHigh;
```

Finally we declare the helper function that will generate random point in the box
```c++
// Returns a random point that is located in box specified by WhereToSpawn.
UFUNCTION(BlueprintPure, Category = Spawning)
FVector GetRandomPointInVolume();
```

Moving to private section we start with a helper function for generating random spawn delay and a variable to store generated value
```c++
// Returns a random float in the interval [SpawnDelayRangeLow, SpawnDelayRangeHigh].
float GetRandomSpawnDelay();

// Stores spawn delay for current episode.
float SpawnDelay;
```

We also need to track how much time have passed since the last spawn event to be able to decide whether we need to invoke spawn now
```c++
// Timer for the spawn of the pickup.
float SpawnTime;
```

The implementation of actual spawn event will happen in function `SpawnPickup` that will be invoked in `Tick` with some periodicity
```c++
// Spawns pickup at random location.
void SpawnPickup();
```

We continue by implementing declared methods in `SpawnVolume.cpp`.

Start with the constuctor or `SpawnVolume`, create box component and set default values to `SpawnDelayRange` properties
```c++
// Sets default values
ASpawnVolume::ASpawnVolume()
{
 	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;

    // Create box component and make it the root of the actor.
    WhereToSpawn = CreateDefaultSubobject<UBoxComponent>(TEXT("WhereToSpawn"));
    RootComponent = WhereToSpawn;

    // Set default values to SpawnDelayRange.
    SpawnDelayRangeLow = 1.0f;
    SpawnDelayRangeHigh = 4.5f;

    // Generate first spawn delay from the specified range.
    SpawnDelay = GetRandomSpawnDelay();
}
```

Then implement the `Tick` function to monitor current time and invoke `SpawnPickup` method when `SpawnDelay` time passes
```c++
// Called every frame
void ASpawnVolume::Tick( float DeltaTime )
{
    // We don't need to spawn anything if the volume spawning is disabled.
    if (!bSpawningEnabled) {
        return;
    }

	Super::Tick( DeltaTime );

    // Increment current spawn wait time.
    SpawnTime += DeltaTime;
    // If it is larger than specified spawn delay proceed to spawning an object.
    bool bShouldSpawn = SpawnTime > SpawnDelay;
    if (bShouldSpawn) {
        // Do actual spawning.
        SpawnPickup();
        // Decrement wait time to account for overflows.
        SpawnTime -= SpawnDelay;
        // Generate new spawn delay that will be used for the next spawning.
        SpawnDelay = GetRandomSpawnDelay();
    }
}
```

Implement `SpawnPickup` by generating random point and spawning an object there.
```c++
void ASpawnVolume::SpawnPickup()
{
    // If no Pickup is specified, ignore SpawnPickup.
    if (!WhatToSpawn) {
        return;
    }

    UWorld* const World = GetWorld();
    // If the world is not available, ignore SpawnPickup.
    if (!World) {
        return;
    }

    // Specify spawn parameters.
    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = this;
    SpawnParams.Instigator = Instigator;

    // Generate random location and random rotation for the spawned object.
    FVector SpawnLocation = GetRandomPointInVolume();
    FRotator SpawnRotation;
    SpawnRotation.Yaw = FMath::FRand() * 360.f;
    SpawnRotation.Pitch = FMath::FRand() * 360.f;
    SpawnRotation.Roll = FMath::FRand() * 360.f;

    // Spawn the Pickup in random location in the world.
    APickup* const SpawnedPickup = World->SpawnActor<APickup>(WhatToSpawn, SpawnLocation, SpawnRotation, SpawnParams);
}
```

Implement function to generate random spawn delay
```c++
float ASpawnVolume::GetRandomSpawnDelay()
{
    // Generate random float from interval [SpawnDelayRangeLow, SpawnDelayRangeHigh].
    return FMath::FRandRange(SpawnDelayRangeLow, SpawnDelayRangeHigh);
}
```

Finally implement the function to generate random point in the volume
```c++
FVector ASpawnVolume::GetRandomPointInVolume()
{
    FVector RandomLocation;
    float MinX, MinY, MinZ;
    float MaxX, MaxY, MaxZ;

    FVector Origin;
    FVector BoxExtent;

    // Get the SpawnVolume's origin and box extent
    Origin = WhereToSpawn->Bounds.Origin;
    BoxExtent = WhereToSpawn->Bounds.BoxExtent;

    // Calculate the minimum X, Y, and Z
    MinX = Origin.X - BoxExtent.X / 2.f;
    MinY = Origin.Y - BoxExtent.Y / 2.f;
    MinZ = Origin.Z - BoxExtent.Z / 2.f;

    // Calculate the maximum X, Y, and Z
    MaxX = Origin.X + BoxExtent.X / 2.f;
    MaxY = Origin.Y + BoxExtent.Y / 2.f;
    MaxZ = Origin.Z + BoxExtent.Z / 2.f;

    // The random spawn location will fall between the min and max X, Y, and Z
    RandomLocation.X = FMath::FRandRange(MinX, MaxX);
    RandomLocation.Y = FMath::FRandRange(MinY, MaxY);
    RandomLocation.Z = FMath::FRandRange(MinZ, MaxZ);

    // Return the random spawn location
    return RandomLocation;
}
```

Now our `SpawnVolume` is ready to be added to the world. Create it by dragging C++ class into the level.
You will definitely want to change the Location and Scale of `WhereToSpawn` box of `SpawnVolume`.
This can be done in `Details` tab of the created object.
* Example: https://wiki.unrealengine.com/Introduction_to_the_UE4_Editor_-_6_-_Moving_Objects
* Example: https://wiki.unrealengine.com/Introduction_to_the_UE4_Editor_-_8_-_Scaling_Objects

You also need to set `WhatToSpawn` property of the `SpawnVolume` to be `BP_BatteryPickup` 
Run Play and verify that batteries are indeed spawned!

### Implement game modes

In this section we will implement game modes: game will end when the player reaches zero energy level.
At this point spawning will stop and player camera will be freezed.

We start in file `Lesson_1GameMode.h`.
First we need a way to store current state of the game, for this we define the following enumeration outside of the `GameMode` class:
```c++
// Enum to represent current game state.
enum class ELesson_1PlayState : short
{
    EPlaying,  // set when the game is active
    EGameOver, // set when the game is over
    EUnknown   // default value
};
```

And then add corresponding methods to game strategy to store and change current game state:
```c++
public:
    // Returns current game state.
    ELesson_1PlayState GetCurrentState() const;

    // Sets game state to new value.
    void SetCurrentState(ELesson_1PlayState NewState);

private:
    // Stores current game state.
    ELesson_1PlayState CurrentState;

    // Handles game state changes.
    void HandleNewState(ELesson_1PlayState NewState);
```

We also need to override game mode public method `BeginPlay()` that is called when the game starts
```c++
// Initializes game logic when game starts.
virtual void BeginPlay() override;
```

And add the code that will ensure decay of the character power
```c++
// Override the method to decrease character power on every tick.
virtual void Tick(float DeltaSeconds) override;

// Defines how much the power of the character will be drained with time.
UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = Power)
float DecayRate;
```

Finally, let's add auxiliary private field that will help us to freeze battery spawning when the game is over
```c++
// Stores all SpawnVolume actors on the level to allow fast access.
TArray<ASpawnVolume*> SpawnVolumeActors;
```

We also need to add forward declaration of `ASpawnVolume` class outside of `GameMode` class to be able to store pointers to it
```c++
class ASpawnVolume;
```

Now we move to `Lesson_1GameMode.cpp` to implement declared methods.

First we initialize the default value of decay rate in the constructor of `GameMode`
```c++
ALesson_1GameMode::ALesson_1GameMode()
{
    ...
    // Initialize default value of decay rate.
    DecayRate = 0.5f;
}
```

Next we need to implement the `Tick` function to decay player power level or move the game to finish state when player power is too low
```c++
void ALesson_1GameMode::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);

    ALesson_1Character* MyCharacter = Cast<ALesson_1Character>(UGameplayStatics::GetPlayerPawn(this, 0));
    if (MyCharacter->PowerLevel > 0.05) {
        MyCharacter->PowerLevel = FMath::FInterpTo(MyCharacter->PowerLevel, 0, DeltaSeconds, DecayRate);
    } else {
        SetCurrentState(ELesson_1PlayState::EGameOver);
    }
}
```

When the game is started we need to remember all SpawnVolume actors and to transition the game into `EPlaying` state
```c++
void ALesson_1GameMode::BeginPlay()
{
    // Don't forget to call parent BeginPlay() method.
    Super::BeginPlay();

    // Find all SpawnVolume actors.
    TArray<AActor*> FoundActors;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), ASpawnVolume::StaticClass(), FoundActors);
    for (auto Actor : FoundActors) {
        ASpawnVolume* SpawnVolumeActor = Cast<ASpawnVolume>(Actor);
        // If the actor indeed belongs to SpawnVolume, remember it.
        if (SpawnVolumeActor) {
            SpawnVolumeActors.Add(SpawnVolumeActor);
        }
    }
    // Transition the game into playing state.
    SetCurrentState(ELesson_1PlayState::EPlaying);
}
```

We also implement auxiliary methods to get and set game state
```c++
ELesson_1PlayState ALesson_1GameMode::GetCurrentState() const
{
    return CurrentState;
}

void ALesson_1GameMode::SetCurrentState(ELesson_1PlayState NewState)
{
    CurrentState = NewState;
    // Invoke the actions associated with transitioning to new state.
    HandleNewState(CurrentState);
}
```

Finally we describe how transition to each state happens.
When we start the game we want to activate all SpawnVolumes to start spawning batteries.
When game finishes we turn off all SpawnVolumes and take control from the player by putting the camera into cinematic mode.
```c++
void ALesson_1GameMode::HandleNewState(ELesson_1PlayState NewState)
{
    switch (NewState) {
        case ELesson_1PlayState::EPlaying:
        {
            // Turn on all spawn volumes to start creating new batteries.
            for (ASpawnVolume* Volume : SpawnVolumeActors) {
                Volume->EnableSpawning();
            }
            break;
        }
        case ELesson_1PlayState::EGameOver:
        {
            // When the game is finished turn off all spawn volumes.
            for (ASpawnVolume* Volume : SpawnVolumeActors) {
                Volume->DisableSpawning();
            }
            // Take control from the player and put camera into cinematic mode.
            APlayerController* PlayerController = UGameplayStatics::GetPlayerController(this, 0);
            PlayerController->SetCinematicMode(true, true, true);
            break;
        }
        case ELesson_1PlayState::EUnknown:
        default:
            break;
    }
}
```

Note that we used unimplemented functions `EnableSpawning` and `DisableSpawning` of `SpawnVolume`.
Let's go and fix this!
We start by editing `SpawnVolume.h` and adding two public methods and a private field
```c++
public:
    // Enables spawning.
    void EnableSpawning();

    // Disables spawning.
    void DisableSpawning();

private:
    // True if spawning is enabled, false otherwise.
    bool bSpawningEnabled;
```

Then in `SpawnVolume.cpp` we implement this methods
```c++
void ASpawnVolume::EnableSpawning()
{
    bSpawningEnabled = true;
}

void ASpawnVolume::DisableSpawning()
{
    bSpawningEnabled = false;
}
```

And also modify the `Tick` function to disable spawning when flag is set to false
```c++
void ASpawnVolume::Tick( float DeltaTime )
{
    // We don't need to spawn anything if the volume spawning is disabled.
    if (!bSpawningEnabled) {
        return;
    }
    ...
}
```

Finally, we will change the character code to disable battery collection when the game is over.
For this add the following lines in `Lesson_1Character.cpp` in the beginning `CollectBatteries` function
```
void ALesson_1Character::CollectBatteries()
{
    // Don't collect batteries when the game is over.
    ALesson_1GameMode* MyGameMode = Cast<ALesson_1GameMode>(UGameplayStatics::GetGameMode(this));
    if (MyGameMode->GetCurrentState() == ELesson_1PlayState::EGameOver) {
        return;
    }
    ...
}
```

And also include the following headers for C++ to resolve used classes
```c++
#include "Lesson_1GameMode.h"

#include "Kismet/GameplayStatics.h"
```

Now build and run the game to verify that the player power level decay works (player should move slower with time)
and also that the game transitions to game over state when the player becomes really slow.

### Implement HUD

In this section we will implement text interface to display current power lowel and also show Game Over message in the end of the game.

To do this we start by adding a new C++ class inherited from HUD.
Now we edit `Lesson_1HUD.h` and add the variable to store the font for the messages
```c++
// Store the font used for messages.
UPROPERTY()
UFont* HUDFont;
```

And overriding the `DrawHUD` function to support custom behavior
```c++
// Override draw function to show custom messages.
virtual void DrawHUD() override;
```

In `Lesson_1HUD.cpp` we include a set of required headers
```c++
#include "Lesson_1GameMode.h"
#include "Lesson_1Character.h"

#include "Kismet/GameplayStatics.h"

#include "Engine/Canvas.h"
#include "Engine/Font.h"
```

And then specify the constructor for our HUD to load font from the standard library
```c++
ALesson_1HUD::ALesson_1HUD()
{
    //Use the RobotoDistanceField font from the engine
    static ConstructorHelpers::FObjectFinder<UFont>HUDFontOb(TEXT("/Engine/EngineFonts/RobotoDistanceField"));
    HUDFont = HUDFontOb.Object;
}
```

We proceed with implementing `DrawHUD` function that prints character current power level and also shows "Game Over" message when the game is in final state.
```c++
void ALesson_1HUD::DrawHUD()
{
    // Get the screen dimensions.
    FVector2D ScreenDimensions = FVector2D(Canvas->SizeX, Canvas->SizeY);

    // Call to the parent versions of DrawHUD.
    Super::DrawHUD();

    // Get the character and print its power level.
    ALesson_1Character* MyCharacter = Cast<ALesson_1Character>(UGameplayStatics::GetPlayerPawn(this, 0));
    FString PowerLevelString = FString::Printf(TEXT("%10.1f"), FMath::Abs(MyCharacter->PowerLevel));
    DrawText(PowerLevelString, FColor::White, 50, 50, HUDFont);

    ALesson_1GameMode* MyGameMode = Cast<ALesson_1GameMode>(UGameplayStatics::GetGameMode(this));
    // If the game is over.
    if (MyGameMode->GetCurrentState() == ELesson_1PlayState::EGameOver)
    {
        // Create a variable for storing the size of printing Game Over.
        FVector2D GameOverSize;
        GetTextSize(TEXT("GAME OVER"), GameOverSize.X, GameOverSize.Y, HUDFont);
        DrawText(TEXT("GAME OVER"), FColor::White, (ScreenDimensions.X - GameOverSize.X) / 2.0f, (ScreenDimensions.Y - GameOverSize.Y) / 2.0f, HUDFont);
    }
}
```

To activate the HUD we need to add the following lines to the constructor in `Lesson_1GameMode.cpp`
```c++
ALesson_1GameMode::ALesson_1GameMode()
{
    ...
    // Set the default HUD class to be used in game.
    HUDClass = ALesson_1HUD::StaticClass();
    ...
}
```

To test our HUD we build and run the project and check that power level is shown on the screen and that when the game finishes we get "Game Over" message.
