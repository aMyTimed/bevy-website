+++
title = "Detecting changes"
weight = 6
template = "book-section.html"
page_template = "book-section.html"
+++

Bevy allows you to respond to the addition of or changes to specific component types using the [`Added`] and [`Changed`] query filters.
These are incredibly useful, allowing you to:

- automatically complete initialization of components
- keep data in sync
- save on work by only operating on entities where the relevant data has been updated
- automatically respond to changes in the data of UI elements

A component will be marked as "added" on a per-system basis if it has been added to that entity (via component insertion or being spawned on an entity) since the last time that system ran.
Similarly, a component will be marked as "changed" on a per-system basis if it has been added or mutably dereferenced since the last time that system ran.
As you ([almost always](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)) need to mutably dereference data out of its wrapper in order to mutate it, this gives an accurate (and fast!) indication that the data may have changed.

Change detection works for resources too, using the exact same internal mechanisms!
Use the [`is_changed()`] and [`is_added()`] methods on resources (or individual components) to check if they've been added or changed since the last time this system ran.

The [`Added`] and [`Changed`] types are query filters, restricting the list of entities returned.
They are used in the second type parameter of our [`Query`] data types.

Let's get familiar with them by examining a few snippets of gameplay code.
You might use an addition-tracking system to automatically change the difficulty of your game's enemies:

```rust
// Enums can be used directly as both resources and components
enum Difficulty {
    Easy,
    Medium,
    Hard
}

impl Difficulty {
    fn modifier(&self) -> f32 {
        const EASY_DIFFICULTY_MODIFIER: f32 = 0.5;
        const HARD_DIFFICULTY_MODIFIER: f32 = 2.0;

        match *self {
            Difficulty::Easy => EASY_DIFFICULTY_MODIFIER,
            Difficulty::Normal => 1.0,
            Difficulty::Hard => HARD_DIFFICULTY_MODIFIER,
        }
    }
}

#[derive(Component)]
struct Life(f32);

// This will detect all new entities that spawned with the `Life` component, or entities who just had that component added
fn difficulty_adjusting_system(difficulty: Res<Difficulty>, query: Query<&mut Life, Added<Life>>){
    let modifier = difficulty.modifier();

    // Each relevant entity will always be affected exactly once by this system
    for life in query.iter_mut(){
        // We can't just change the values that our entities spawn with,
        // because the modifier can change at run-time
        // and we don't want to duplicate this code everywhere
        life.0 *= modifier;
    } 
}
```

Of course, if our difficulty changes in the middle of the game, existing entities won't be modified correctly!
Let's fix that, with a demonstration of resource change detection.

```rust
// We need to keep track of the old difficulty, so we can reverse the changes easily
struct OldDifficulty(Difficulty);

// We have to be sure that this system runs *after* difficulty_changed_system
// so then we have the correct cached value of Difficulty when we update enemy stats
fn old_difficulty_system(difficulty: Res<Difficulty>, mut old_difficulty: ResMut<OldDifficulty>){
    // Here, we're checking whether the `Difficulty` resource's value has changed
    // By checking if difficulty has changed, we can avoid constantly rewriting this value
    if difficulty.is_changed(){
        old_difficulty.0 = difficulty;
    }
}

fn difficulty_changed_system(difficulty: Res<Difficulty>, old_difficulty: Res<OldDifficulty> query: Query<&mut Life>){
    if difficulty.is_changed(){
        // Reverse the previous difficulty adjustment
        let old_modifier = old_difficulty.0.modifier();
        // Apply the new multiplier
        let new_modifer = difficulty.modifier();
        let net_modifier = new_modifier / old_modifier;

        // Apply the final change to every entity with `Life`!
        for life in query.iter_mut(){
            life.0 *= net_modifier;
        } 
    }
}
```

[`Changed`] can be used with precisely the same syntax, and is incredibly useful for avoiding repeated work.
For example, you might use it to efficiently check whether units are in a particular area each frame:

```rust
/// Detects enemies that have reached the entrance in a tower defense game
fn enemy_escape_system(query: Query<&Transform, (With<Enemy>, Changed<Transform>>>, exit: Res<Exit>, lives: ResMut<Lives>){
    for enemy_transform in query.iter(){
        if exit.in_range(transform) {
            lives.0 -= 1;
        }
    }
}
```

[`Added`]: https://docs.rs/bevy/latest/bevy/ecs/query/struct.Added.html
[`Changed`]: https://docs.rs/bevy/latest/bevy/ecs/query/struct.Changed.html
[`is_added`]: https://docs.rs/bevy/latest/bevy/ecs/system/struct.Res.html#method.is_added
[`is_changed`]: https://docs.rs/bevy/latest/bevy/ecs/system/struct.Res.html#method.is_changed
[`Query`]: https://docs.rs/bevy/latest/bevy/ecs/system/struct.Query.html

### The details of change detection

Change detection in Bevy works via a custom implementation of the [`DerefMut`] trait of our mutable [smart pointer](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) wrappers (such as [`Mut`] and [`ResMut`]).
As a result:

1. Changes won't be flagged when you use [interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html). You should probably manually flag the data as having changed using the [`set_changed()`] method when you do this to prevent tricky bugs in other consumers of your component.
2. Changes will be flagged whenever you mutably access a component or resource, even if you don't change its value. Only mutably dereference the data if you know you're going to change it to avoid false positives caused in this way.

[`DerefMut`]: https://doc.rust-lang.org/std/ops/trait.DerefMut.html
[`Mut`]: https://docs.rs/bevy/latest/bevy/ecs/world/struct.Mut.html
[`ResMut`]: https://docs.rs/bevy/latest/bevy/ecs/system/struct.ResMut.html
[`set_changed()`]: https://docs.rs/bevy/latest/bevy/ecs/world/struct.Mut.html#method.set_changed

## Removal detection

We can watch for the removal of components too, although the mechanisms are quite different.
[`RemovedComponents`] is a system parameter, returning an iterator of all entity identifiers from whom components of that type were removed during the last frame.
This includes entities with that component who were despawned.
There are three important caveats here:

1. While [`Added`] and [`Changed`] operate on the basis of "since the last time this system ran", removal detection only works for the last frame. This distinction is very important if your system does not run every frame due to states or run criteria.
2. [`RemovedComponents`] does not return the *value* of the component that was removed, due to performance concerns. If you need this, you will need to include your own change-detecting system to cache the values before they are removed.
3. The [`RemovedComponents`] system parameter is not a query filter. As a result, you will have to manually compare it to the results of a query if you only need to handle some subset of component removal events.

Despite [these caveats](https://github.com/bevyengine/bevy/issues/2148), removal detection can be quite useful for handling complex cleanup and toggling behavior through the addition and removal of components.

[`RemovedComponents`]: https://docs.rs/bevy/latest/bevy/prelude/struct.RemovedComponents.html
