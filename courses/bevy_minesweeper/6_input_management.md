> [Check the repository](https://gitlab.com/qonfucius/minesweeper-tutorial)

# Input Management

We have a glorious board, but we can't interact with it, let's handle some input !

## Bounds

To detect mouse input inside our board we will use common gamedev type called `Bounds`. It is strangely missing from bevy so we'll code a simple version for our plugin in `board_plugin/src/bounds.rs`

```rust
// bounds.rs
use bevy::prelude::Vec2;

#[derive(Debug, Copy, Clone)]
pub struct Bounds2 {
    pub position: Vec2,
    pub size: Vec2,
}

impl Bounds2 {
    pub fn in_bounds(&self, coords: Vec2) -> bool {
        coords.x >= self.position.x
            && coords.y >= self.position.y
            && coords.x <= self.position.x + self.size.x
            && coords.y <= self.position.y + self.size.y
    }
}
```
Ths structure defines a 2D rectangle and can check if coordinates are contained in its extents.

Connect the file to `board_plugin/src/lib.rs`:

```rust
// lib.rs
mod bounds;
```

## The Board resource

The tile map we generate in our `create_board` startup system is lost after that system, we need to put it in a *resource* for it to last.
We also need to store our board `Bounds` for input detection.

Let's create a `board.rs` in our `resources` folder:

```rust
// mod.rs
// ..
pub use board_options::*;

mod board;
```

```rust
// board.rs
use crate::bounds::Bounds2;
use crate::{Coordinates, TileMap};
use bevy::prelude::*;

#[derive(Debug)]
pub struct Board {
    pub tile_map: TileMap,
    pub bounds: Bounds2,
    pub tile_size: f32,
}

impl Board {
    /// Translates a mouse position to board coordinates
    pub fn mouse_position(&self, window: &Window, position: Vec2) -> Option<Coordinates> {
        // Window to world space
        let window_size = Vec2::new(window.width(), window.height());
        let position = position - window_size / 2.;

        // Bounds check
        if !self.bounds.in_bounds(position) {
            return None;
        }
        // World space to board space
        let coordinates = position - self.bounds.position;
        Some(Coordinates {
            x: (coordinates.x / self.tile_size) as u16,
            y: (coordinates.y / self.tile_size) as u16,
        })
    }
}
```

Our `Board` resource stores a `TileMap`, the board `Bounds` and a `tile_size` which is the size of individual square tiles.

We provide a method converting mouse position to our own coordinate system. This computation seems strange because unlike our entities **world space** where the origin
is at the center of the screen (based on camera position), the **window space** origin is on the bottom left.

So we have to transform the mouse position so that it matches our **world space**, check the bounds and then convert the coordinates into a tile coordinate.

Now we defined our resource, we need to register it at the end of our `create_board` startup system

```rust
// lib.rs
use bounds::Bounds2;
use resources::Board;
use bevy::math::Vec3Swizzles;

// ..

// We add the main resource of the game, the board
        commands.insert_resource(Board {
            tile_map,
            bounds: Bounds2 {
                position: board_position.xy(),
                size: board_size,
            },
            tile_size,
        });
// ..
```

The `Board` is now available for any *system*.

## Input system

We can now create or first regular *system* which will check every frame for a mouse click event.

Let's create a `systems` module in our board plugin with an `input.rs` file.

> Small hierarchy recap:
> ```
> ├── Cargo.lock
> ├── Cargo.toml
> ├── assets
> ├── board_plugin
> │    ├── Cargo.toml
> │    └── src
> │         ├── bounds.rs
> │         ├── components
> │         │    ├── bomb.rs
> │         │    ├── bomb_neighbor.rs
> │         │    ├── coordinates.rs
> │         │    ├── mod.rs
> │         │    ├── uncover.rs
> │         ├── lib.rs
> │         ├── resources
> │         │    ├── board.rs
> │         │    ├── board_options.rs
> │         │    ├── mod.rs
> │         │    ├── tile.rs
> │         │    └── tile_map.rs
> │         └── systems
> │              ├── input.rs
> │              └── mod.rs
> ├── src
> │    └── main.rs
> ```

Don't forget to connect the `systems` module in `lib.rs`:
```rust
mod systems;
```
and the `input` module in `systems/mod.rs`:
```rust
pub mod input;
```

Let's define our input system !

```rust
// input.rs
use crate::Board;
use bevy::input::{mouse::MouseButtonInput, ElementState};
use bevy::log;
use bevy::prelude::*;

pub fn input_handling(
    windows: Res<Windows>,
    board: Res<Board>,
    mut button_evr: EventReader<MouseButtonInput>,
) {
    let window = windows.get_primary().unwrap();

    for event in button_evr.iter() {
        if let ElementState::Pressed = event.state {
            let position = window.cursor_position();
            if let Some(pos) = position {
                log::trace!("Mouse button pressed: {:?} at {}", event.button, pos);
                let tile_coordinates = board.mouse_position(window, pos);
                if let Some(coordinates) = tile_coordinates {
                    match event.button {
                        MouseButton::Left => {
                            log::info!("Trying to uncover tile on {}", coordinates);
                            // TODO: generate an event
                        }
                        MouseButton::Right => {
                            log::info!("Trying to mark tile on {}", coordinates);
                            // TODO: generate an event
                        }
                        _ => (),
                    }
                }
            }
        }
    }
}
```

This function is our input *system*, it takes three arguments:
- a `Windows` resource
- our own `Board` resource
- a `MouseButtonInput` event reader

We iterate throught the event reader to retrieve every event, keeping only `Pressed` events.
We retrieve the mouse position and use our `Board` to convert the mouse position into tile coordinates
and then we log the action (uncover or mark) according to the mouse button.

> So if we press an other button we will still perform the conversion?

Yes, we could check the buttons first to optimize a bit but it would make the code less clear for a tutorial.

We can now register our system in our `BoardPlugin::build()` method:

```rust
// lib.rs
// ..
//    app.add_startup_system(Self::create_board)
        .add_system(systems::input::input_handling);
// ..
```

Running the app you can now use your left and right click buttons on the window and notice that:
- If you click on the board it logs the coordinates and the action
- If you click outside the board or with another button, nothing happens

---
Author: Félix de Maneville
Follow me on [Twitter](https://twitter.com/ManevilleF)