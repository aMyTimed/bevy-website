+++
title = "Resources are global singletons"
weight = 3
template = "book-section.html"
page_template = "book-section.html"
+++

Not all data is stored in the entity-component data storage.
**Resources** are the blanket term for data that lives outside of entities and their components, but is still accessible by Bevy apps.
Each resource has a unique type (you can only have one resource of each type), and can be accessed in systems by adding either {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="Res" no_mod = "true")}} (for immutable read-only access) or {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="ResMut" no_mod = "true")}} (for mutable read-write access) as a system parameter.

In effect, resources serve as well-behaved **global singletons**.
They are accessible by any system in the app, and because you can only have one resource of each type, are unique.
You might want to use resources for:

- storing events
- holding configuration data
- storing simple global game state that you only need a single copy of (like a game's store)
- interoperating with other libraries
- providing faster or supplementary look-up data structures for entities (such as indexes or graphs)
- storing reference-counted handles to assets (so then they're not unloaded when the last entity using it is despawned)

## Creating resources

Unlike components, resources do not need their own trait implementation: you can use any new or existing {{rust_type(type="keyword" crate="std" name="static" no_mod = "true")}} Rust type as a resource (if it does not implement the {{rust_type(type="trait" crate="std" mod = "marker" name="Send" no_mod = "true")}} and {{rust_type(type="trait" crate="std" mod = "marker" name="Sync" no_mod = "true")}} traits, you'll need a {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="NonSend" no_mod = "true")}}  resource instead).

Like entities and their component data, resources are stored in your {{rust_type(type="struct" crate="bevy" mod = "app" name="App" no_mod = "true")}}'s {{rust_type(type="struct" crate="bevy_ecs" name="World")}} struct.
Resources are typically added statically, via {{rust_type(type="struct" crate="bevy" mod = "app" name="App" method = "insert_resource" no_mod = "true")}} or {{rust_type(type="struct" crate="bevy" mod = "app" name="App" method = "init_resource" no_mod = "true")}}.

{{rust_type(type="struct" crate="bevy" mod = "app" name="App" method = "insert_resource" no_mod = "true")}} is used when you want to set the value of a resource manually, while {{rust_type(type="struct" crate="bevy" mod = "app" name="App" method = "init_resource" no_mod = "true")}} is used when you want to automatically initialize the resources value using the {{rust_type(type="trait" crate="std" mod = "default" name="Default" no_mod = "true")}} or {{rust_type(type="struct" crate="bevy" mod = "ecs/world" name="FromWorld" method = "from_world" no_mod = "true")}} trait.

```rust
use bevy::prelude::*;

// Resources can be tuple structs
// Default can be derived for many simple resources,
// with the default value of most numeric types being 0
#[derive(Default)]
struct Score(u64);

// Resources can be ordinary namedstructs
struct PlayerSupplies {
    gold: u64,
    wood: u64,
}

// The Default trait can be manually implemented to control initial values
impl Default for PlayerSupplies {
    fn default() -> Self {
        PlayerSupplies {
            gold: 400,
            wood: 200,
        }
    }
}

// Resources can be enums
enum Turn {
    Allied,
    Enemy
}

fn main(){
    // Resources are typically inserted using AppBuilder methods
    App::build()
    // Uses the default() value provided by the derived Default trait
    .init_resource::<Score>()
    // Uses the default() value provided by the manual impl of the Default trait
    .init_resource::<PlayerSupplies>()
    // Uses the specific manual value of Turn::Allied to insert a resource of type Turn
    .insert_resource(Turn::Allied)
    // Sets the value of the standard Bevy resource `WindowDescriptor`,
    // leaving the unspecified fields as their default value
    .insert_resource(WindowDescriptor {
        title: "I am a window!".to_string(),
        width: 500.,
        height: 300.,
        vsync: true,
        ..Default::default()
    })
    .add_plugins(DefaultPlugins)
    .run()
}
```

In rare cases, you may need to add a resource later, once other parts of the world exist to ensure proper initialization.
For that, we can use the equivalent methods on {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="Commands" no_mod = "true")}}.
You can add, overwrite and even {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="Commands" no_mod = "true", method = "remove_resource")}} dynamically in this way.

Only use {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="Commands" no_mod = "true")}} to add resources where it's needed due to the need to initialize a resource with other data from the world.
It's less clear, and like all commands, commands that insert resources are delayed and only take effect at the end of the current stage.

## Reading and writing from resources

Once our resources have been added to the app, we can read and write to them by adding systems that refer to them using the `Res` and `ResMut` system parameters.
Let's take a look at how this works in a cohesive context by building a tiny guessing game.

```rust
use bevy::prelude::*;

fn main() {
    App::build()
        .add_plugins(MinimalPlugins)
        .init_resource::<Secret>()
        .insert_resource(InputMode::Recording)
        .add_system(record_secret.label("Record Secret"))
        // We must check the secret before we record the secret,
        // otherwise the system can peek!
        .add_system(check_secret.before("Record Secret"))
        .run();
}

/// Resource to store our secret key
#[derive(Default)]
struct Secret {
    // The default value of Option<T> fields is always None
    val: Option<KeyCode>,
}

/// Resource to control the interaction mode of our game
#[derive(PartialEq, Eq)]
enum InputMode {
    /// Stores inputs in the Secret resource
    Recording,
    /// Compares inputs to the Secret resource
    Guessing,
}

/// Stores the keyboard input in our Secret resource
fn record_secret(
    mut input_mode: ResMut<InputMode>,
    mut secret: ResMut<Secret>,
    mut input: ResMut<Input<KeyCode>>,
) {
    // This system should only do work in the Recording input mode
    // Note that we need to derefence out of the ResMut smart pointer
    // using * to access the underlying InputMode data
    if *input_mode == InputMode::Recording {
        // Only display the text prompt once, when the input_mode changes
        if input_mode.is_changed() {
            println!("Press a key to store a secret to be guessed by a friend!")
        }

        // Input is stored in resources too!
        // Here, we only want one key to store as our secret,
        // so we arbitarily grab the first key in case multiple keys are pressed at once
        let maybe_keycode = input.get_just_pressed().next();

        // maybe_keycode may be None, if no key was pressed
        // We only care about handling the case where a key was pressed,
        // so we use `if let` to destructure our option
        if let Some(keycode) = maybe_keycode {
            // Storing our input in the Secret resource
            secret.val = Some(*keycode);

            // Now that we've stored a Secret, we should swap to guessing it
            // Again, we need to derefence our resource to refer to the data rather than the wrapper
            *input_mode = InputMode::Guessing;
        }
    }
}

/// Checks if the new input matches the stored secret
fn check_secret(
    // We need to use `mut` + `ResMut` for `input_mode` and `secret` since we change their values
    mut input_mode: ResMut<InputMode>,
    mut secret: ResMut<Secret>,
    // `input` only needs a Res, since we're only reading the KeyCodes that were pressed
    input: Res<Input<KeyCode>>,
) {
    if *input_mode == InputMode::Guessing {
        if input_mode.is_changed() {
            println!("Press a key to check if it matches the secret! Only one key will be checked per frame.")
        }

        let maybe_keycode = input.get_just_pressed().next();
        if let Some(keycode) = maybe_keycode {
            if Some(*keycode) == secret.val {
                println!("You've guessed the secret!");
                // Get a new secret if it was guessed successfully
                secret.val = None;
                *input_mode = InputMode::Recording;
            } else {
                println!("Nope! Try again.")
            }
        }
    }
}
```

## Singleton entity or Resource?

As discussed in [**Systems access data through queries**](../systems-queries/_index.md), [`Query::single`](https://docs.rs/bevy/latest/bevy/ecs/system/struct.Query.html#method.single) is a convenient way to get access to the data of an entity when you know that exactly one entity will be returned by a query.
So when should you use a singleton entity, and when should you use a resource?

Let's list the advantage of each, beginning with resources:

- fast and simple access model: no need for queries or unwrapping
- will not be accidentally broken by later code that modifies your entity's components or creates more matching entities
- can store data that is not thread-safe using {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="NonSend" no_mod = "true")}} resources

By contrast, singleton entities are useful because they:

- can easily share behavior and data types with other entities through systems that operate on their components
- can be extended and contracted at run time by adding or removing components
- have more granular change detection: operating on a per component basis rather than the entire object
- allows you to fetch only the data you immediately need, rather than the entire resource struct

Overall, resources are a good default for one-off demands: they're clear and very ergonomic to access.
You should turn to singleton entities when you want to share behavior with other entities (i.e. a singleton entity for the player is almost always going to be superior to a monolithic `Player` resource), or for when you want to be able to extend or modify behavior dynamically during gameplay.

## Complex resource initialization from world state

Sometimes you may need to initialize resources in more complex ways, depending on data from the {{rust_type(type="struct" crate="bevy_ecs" name="World")}} at large.
For this, we can use the {{rust_type(type="trait" crate="bevy" mod = "ecs/world" name="FromWorld" no_mod = "true")}} trait, which allows you to create a new copy of the type that it's implemented automatically from the world.

Ordinarily, the {{rust_type(type="trait" crate="std" mod = "default" name="Default" no_mod = "true")}} trait is used to handle resource initialization, due to the blanket implementation of {{rust_type(type="struct" crate="bevy" mod = "ecs/world" name="FromWorld" no_mod = "true")}} for `T: Default`.
Note that you cannot manually implement {{rust_type(type="struct" crate="bevy" mod = "ecs/world" name="FromWorld" no_mod = "true")}} on a type that has the {{rust_type(type="trait" crate="std" mod = "default" name="Default" no_mod = "true")}} trait, as Rust forbids conflicting implementations of the same trait.

{{rust_type(type="struct" crate="bevy" mod = "ecs/world" name="FromWorld" no_mod = "true")}} is commonly used in asset loading to automatically create handles for simple assets, and its use in this case is demonstrated in the [section on loading assets](../../assets/loading-assets/_index.md).
For advice on how to work with the {{rust_type(type="struct" crate="bevy_ecs" name="World")}} exposed by the {{rust_type(type="struct" crate="bevy" mod = "ecs/world" name="FromWorld" method = "from_world" no_mod = "true")}} method, see the section on [exclusive world access](../exclusive-world-access/_index.md).

## Optional Resources

Sometimes, a resource may not exist by the time a regularly scheduled system is called.
We can handle both the case where it exists and the case where it doesn't by requesting `Option<Res<T>>` (or the appropriate underlying resource wrapper) as a system parameter, and then branching on the resulting option returned.

Here's a quick runnable example:

```rust
use bevy::prelude::*;

fn main() {
    App::build()
        .add_plugins(DefaultPlugins)
        .add_system(countdown)
        .run()
}

struct Countdown {
    time_remaining: u8,
}

// `countdown` does not need to be marked as `mut` here,
// as destructuring (like via `match`) does not require mutation
// Only the internal data (`validated_countdown`) needs to be marked as `mut`
fn countdown(
    countdown: Option<ResMut<Countdown>>,
    mut commands: Commands,
    mut app_exit: EventWriter<AppExit>,
) {
    match countdown {
        // Resources can be inserted at runtime using commands
        None => commands.insert_resource(Countdown { time_remaining: 10 }),
        Some(mut validated_countdown) => {
            info!("{} ticks remaining!", validated_countdown.time_remaining);
            if validated_countdown.time_remaining > 1 {
                validated_countdown.time_remaining -= 1;
            } else {
                info!("Ka-BOOM!");
                app_exit.send(AppExit);
            }
        }
    }
}
```

## Resources that are not thread-safe

Non-send resources are used to store data that do not meet the `Send + Sync` trait bounds: they cannot be sent safely across threads.
Their use cases are typically quite advanced and tend to involve interfacing with external libraries for things like audio or networking.
{{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="NonSend" no_mod = "true")}} and {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="NonSendMut" no_mod = "true")}} can be directly substituted for {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="Res" no_mod = "true")}}
and {{rust_type(type="struct" crate="bevy" mod = "ecs/system" name="ResMut" no_mod = "true")}} in any system's parameters.

The inclusion of one or more non-send resources in your system will force that system to run on the main thread,
rather than being automatically scheduled to the first available thread.
This ensures that the data is always accessed from the same thread, no matter what combination of not-thread safe objects you may need to access.
