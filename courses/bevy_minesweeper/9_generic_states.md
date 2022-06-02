> [Check the repository](https://gitlab.com/qonfucius/minesweeper-tutorial)

# Generic States

Our board plugin is customizable through `BoardOptions`, but pour app can't interact with it.
We need to make our `BoardPlugin` generic, to allow control through [*states*](https://bevy-cheatbook.github.io/programming/states.html).

## Plugin

Lets edit our plugin structure:

```rust
// lib.rs
use bevy::ecs::schedule::StateData;

pub struct BoardPlugin<T> {
    pub running_state: T,
}

impl<T: StateData> Plugin for BoardPlugin<T> {
    fn build(&self, app: &mut App) {
        // ..
    }
}

impl<T> BoardPlugin<T> {
    // ..
}
```

Our plugin cannot know what *state* the app using it has defined, it needs to be generic.

We can now change our systems structure to take in account this `running_state`:


```rust
// lib.rs
// ..
fn build(&self, app: &mut App) {
    // When the running states comes into the stack we load a board
        app.add_system_set(
            SystemSet::on_enter(self.running_state.clone()).with_system(Self::create_board),
        )
        // We handle input and trigger events only if the state is active
        .add_system_set(
            SystemSet::on_update(self.running_state.clone())
                .with_system(systems::input::input_handling)
                .with_system(systems::uncover::trigger_event_handler),
        )
        // We handle uncovering even if the state is inactive
        .add_system_set(
            SystemSet::on_in_stack_update(self.running_state.clone())
                .with_system(systems::uncover::uncover_tiles),
        )
        .add_event::<TileTriggerEvent>();
}
```

Bevy's states are in a *stack*:
- if a state is at the top of the stack it is considered *active*
- if a state is in the stack but not at the top it is considered *inactive*
- if a state leaves the stack it is considered *exited*
- if a state enters the stack it is considered *entered*

So what did we do here:
- Since we now constrain systems with state conditions, everything is a `SystemSet`
- Instead of a `startup_system` we call our `setup_board` system when our state *enters* the stack
- We handle our input, and the trigger events only if the state is *active* (we allow for paused states)
- The uncovering system should not be paused, so we run it if the state is in the stack, *active* or not.

With this configuration the app using the plugin can have menus or other stuff and trigger our board generation with the `running_state`.
But we need to be able to clean up the board in case the states *exits* the stack.

For that the `Board` resource should have a reference to its own `Entity` to despawn it with all its children:

```diff
// board.rs
// ..
#[derive(Debug)]
pub struct Board {
    pub tile_map: TileMap,
    pub bounds: Bounds2,
    pub tile_size: f32,
    pub covered_tiles: HashMap<Coordinates, Entity>,
+    pub entity: Entity,
}
// ..
```

Let's edit our `create_board` system to retrieve the entity:

```rust
// lib.rs
fn create_board(
    // ..
) {
    // .. 
    let board_entity = commands
    //        .spawn()
    //        .insert(Name::new("Board"))
    // ..
    .id();
    // ..
    commands.insert_resource(Board {
            // ..
            entity: board_entity,
        })

}
```

Now we can register a cleaning system for our plugin:

```rust
// lib.rs

// ..
fn build(&self, app: &mut App) {
    //..
    .add_system_set(
        SystemSet::on_exit(self.running_state.clone())
            .with_system(Self::cleanup_board),
    )
    // ..
}

impl<T> BoardPlugin<T> {
    // ..
    fn cleanup_board(board: Res<Board>, mut commands: Commands) {
        commands.entity(board.entity).despawn_recursive();
        commands.remove_resource::<Board>();
    }
}
```

> What about all the tiles, the texts, the sprites, the covers, etc ?

Since we spawned every board entity as children to our `board_entity`, using `despawn_recursive` will also despawn its children:
- the background
- the tiles
- the tile texts
- the tile sprites
- the tile covers
- etc.

## App

Let's define some basic states:

```rust
// main.rs
#[derive(Debug, Clone, Eq, PartialEq, Hash)]
pub enum AppState {
    InGame,
    Out,
}

fn main() {
    // ..
    .add_state(AppState::InGame)
    .add_plugin(BoardPlugin {
        running_state: AppState::InGame,
    })
}
```

If we run the app now, nothing has changed, but if we edit the states we can completely control our board systems:

```rust
// main.rs
use bevy::log;

fn main() {
    // ..
    // State handling
    .add_system(state_handler);
    // ..
}

fn state_handler(mut state: ResMut<State<AppState>>, keys: Res<Input<KeyCode>>) {
    if keys.just_pressed(KeyCode::C) {
        log::debug!("clearing detected");
        if state.current() == &AppState::InGame {
            log::info!("clearing game");
            state.set(AppState::Out).unwrap();
        }
    }
    if keys.just_pressed(KeyCode::G) {
        log::debug!("loading detected");
        if state.current() == &AppState::Out {
            log::info!("loading game");
            state.set(AppState::InGame).unwrap();
        }
    }
}
```

Everything should be familiar here,
- `state` is wrapped in `ResMut<>`, because states are handled like any resource but with an additional wrapper: `State<>`
- `keys` is an `Input<>` argument, allowing to check for keyboard interaction using `KeyCode` (it can be used with `MouseButton` for mouse interaction)

Now pressing **C** should cleanup the board entirely, and pressing **G** should generate a new board.

## Exercise

States can be tricky, so it's good to practice using it.

Implement the following features:

1. When I press *Escape* the game pauses and I can't interact with the board, if I press *Escape* again the game resumes.
2. When I press *G* a new board generates, without having to press *C* first.

Give me your answers on Twitter at [@ManevilleF](https://twitter.com/ManevilleF)

---
Author: Félix de Maneville
Follow me on [Twitter](https://twitter.com/ManevilleF)