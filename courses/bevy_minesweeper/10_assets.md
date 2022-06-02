> [Check the repository](https://gitlab.com/qonfucius/minesweeper-tutorial)

# Assets

We have great board configuration with our `BoardOptions` resource but we hard coded every color, texture and fonts.
Let's create a new configuration resource in `board_assets.rs` for our `board_plugin`:

```rust
// board_assets.rs
use bevy::prelude::*;
use bevy::render::texture::DEFAULT_IMAGE_HANDLE;

/// Material of a `Sprite` with a texture and color
#[derive(Debug, Clone)]
pub struct SpriteMaterial {
    pub color: Color,
    pub texture: Handle<Image>,
}

impl Default for SpriteMaterial {
    fn default() -> Self {
        Self {
            color: Color::WHITE,
            texture: DEFAULT_IMAGE_HANDLE.typed(),
        }
    }
}

/// Assets for the board. Must be used as a resource.
///
/// Use the loader for partial setup
#[derive(Debug, Clone)]
pub struct BoardAssets {
    /// Label
    pub label: String,
    ///
    pub board_material: SpriteMaterial,
    ///
    pub tile_material: SpriteMaterial,
    ///
    pub covered_tile_material: SpriteMaterial,
    ///
    pub bomb_counter_font: Handle<Font>,
    ///
    pub bomb_counter_colors: Vec<Color>,
    ///
    pub flag_material: SpriteMaterial,
    ///
    pub bomb_material: SpriteMaterial,
}

impl BoardAssets {
    /// Default bomb counter color set
    pub fn default_colors() -> Vec<Color> {
        vec![
            Color::WHITE,
            Color::GREEN,
            Color::YELLOW,
            Color::ORANGE,
            Color::PURPLE,
        ]
    }

    /// Safely retrieves the color matching a bomb counter
    pub fn bomb_counter_color(&self, counter: u8) -> Color {
        let counter = counter.saturating_sub(1) as usize;
        match self.bomb_counter_colors.get(counter) {
            Some(c) => *c,
            None => match self.bomb_counter_colors.last() {
                None => Color::WHITE,
                Some(c) => *c,
            },
        }
    }
}
```

Declare the module in `resources/mod.rs`:
```rust
// mod.rs
// ..
pub use board_assets::*;
mod board_assets;
```

This new resource will store every visual data we need and  allow customization.

We also added a `bomb_counter_colors` field to customize the bomb neighbor text colors and made a utility `bomb_counter_color` method to retrieve it.

> What is this `DEFAULT_IMAGE_HANDLE` constant value?

We copy the way `SpriteBundle` handles its default texture using the same hard coded `Handle<Image>` for a white texture.
Now that we have the option for custom textures for everything from the tiles to the board background we will enable every `texture` field we omitted.

## Plugin

Let's use our now resource in our `create_board` system in our `board_plugin`:

```diff
// lib.rs
+ use resources::BoardAssets;
// ..

    pub fn create_board(
        mut commands: Commands,
        board_options: Option<Res<BoardOptions>>,
+       board_assets: Res<BoardAssets>,
        window: Res<WindowDescriptor>,
-       asset_server: Res<AssetServer>,
    ) {
        // ..
-     let font = asset_server.load("fonts/pixeled.ttf");
-     let bomb_image = asset_server.load("sprites/bomb.png");
      // ..

      // Board background sprite:
      parent
                    .spawn_bundle(SpriteBundle {
                        sprite: Sprite {
-                            color: Color::WHITE
+                            color: board_assets.board_material.color,
                            custom_size: Some(board_size),
                            ..Default::default()
                        },
+                       texture: board_assets.board_material.texture.clone(),
                        transform: Transform::from_xyz(board_size.x / 2., board_size.y / 2., 0.),
                        ..Default::default()
                    })
        // ..
        Self::spawn_tiles(
                    parent,
                    &tile_map,
                    tile_size,
                    options.tile_padding,
-                   Color::GRAY,
-                   bomb_image,
-                   font,
-                   Color::DARK_GRAY,
+                   &board_assets,
                    &mut covered_tiles,
                    &mut safe_start,
                );
        // ..
    }
```

We remove the `asset_server` argument.

> Why is `board_assets` not optional?

Making it optional is not easy because bevy doesn't provide a default font `Handle`. It would require advanced engine manipulation like using `FromWorld` and `Assets` implementations and using a hard coded font or font path.

> But `Handle` implements `Default`

Indeed but then either the app will panic when trying to print out text or nothing will show up.

---- 

Our `spawn_tiles` and `bomb_count_text_bundle` functions should be cleaned up as well:

```diff
// lib.rs

fn spawn_tiles(
        parent: &mut ChildBuilder,
        tile_map: &TileMap,
        size: f32,
        padding: f32,
-       color: Color,
-       bomb_image: Handle<Image>,
-       font: Handle<Font>,
-       covered_tile_color: Color,
+       board_assets: &BoardAssets,
        covered_tiles: &mut HashMap<Coordinates, Entity>,
        safe_start_entity: &mut Option<Entity>,
    ) {
        // ..
        // Tile sprite
        cmd.insert_bundle(SpriteBundle {
                    sprite: Sprite {
-                       color
+                       color: board_assets.tile_material.color,
                        custom_size: Some(Vec2::splat(size - padding)),
                        ..Default::default()
                    },
                    transform: Transform::from_xyz(
                        (x as f32 * size) + (size / 2.),
                        (y as f32 * size) + (size / 2.),
                        1.,
                    ),
+                   texture: board_assets.tile_material.texture.clone(),
                    ..Default::default()
                })
                // ..
                // Tile Cover
           let entity = parent
                        .spawn_bundle(SpriteBundle {
                            sprite: Sprite {
                                custom_size: Some(Vec2::splat(size - padding)),
-                               color: covered_tile_color,
+                               color: board_assets.covered_tile_material.color,
                                ..Default::default()
                            },
+                           texture: board_assets.covered_tile_material.texture.clone(),
                            transform: Transform::from_xyz(0., 0., 2.),
                            ..Default::default()
                        })
                        .insert(Name::new("Tile Cover"))
                        .id();
                // ..
                // Bomb neighbor text
                parent.spawn_bundle(Self::bomb_count_text_bundle(
                                *v,
-                               font.clone(),
+                               board_assets,
                                size - padding,
                            ));
}

fn bomb_count_text_bundle(
        count: u8,
-       font: Handle<Font>,        
+       board_assets: &BoardAssets,
        size: f32,
    ) -> Text2dBundle {
        // We retrieve the text and the correct color
-       let (text, color) = (
-           count.to_string(),
-           match count {
-               1 => Color::WHITE,
-               2 => Color::GREEN,
-               3 => Color::YELLOW,
-               4 => Color::ORANGE,
-               _ => Color::PURPLE,
-           },
-       );
+       let color = board_assets.bomb_counter_color(count);
        // We generate a text bundle
        Text2dBundle {
            text: Text {
                sections: vec![TextSection {
-                   value: text,
+                   value: count.to_string(),
                    style: TextStyle {
                        color,
-                       font,
+                       font: board_assets.bomb_counter_font.clone(),
                        font_size: size,
                    },
                }],
     // ..           
```

We now use only our `BoardAssets` resource for every visual element of the board.

## App

We need to set a `BoardAssets` resource, but we have an issue. Loading our assets must be in a *system*, here a *startup system*, but we need to do it **before** our plugin launches its `setup_board` system or it will panic.

So let's prevent this situation by setting our state to `Out`:

```diff
// main.rs

fn main() {
    // ..
-   .add_state(AppState::InGame)
+   .add_state(AppState::Out)
    // ..
}
```

and registering a `setup_board` startup system, and moving the previous board setup into it

```diff
// main.rs

fn main() {
    // ..
-   app.insert_resource(BoardOptions {
-       map_size: (20, 20),
-       bomb_count: 40,
-       tile_padding: 3.0,
-       safe_start: true,
-       ..Default::default()
-   })
    // ..
+    .add_startup_system(setup_board)
    // ..
}
```
We can declare the new system:

```rust
// main.rs
use board_plugin::resources::{BoardAssets, SpriteMaterial};

// ..
fn setup_board(
    mut commands: Commands,
    mut state: ResMut<State<AppState>>,
    asset_server: Res<AssetServer>,
) {
    // Board plugin options
    commands.insert_resource(BoardOptions {
        map_size: (20, 20),
        bomb_count: 40,
        tile_padding: 1.,
        safe_start: true,
        ..Default::default()
    });
    // Board assets
    commands.insert_resource(BoardAssets {
        label: "Default".to_string(),
        board_material: SpriteMaterial {
            color: Color::WHITE,
            ..Default::default()
        },
        tile_material: SpriteMaterial {
            color: Color::DARK_GRAY,
            ..Default::default()
        },
        covered_tile_material: SpriteMaterial {
            color: Color::GRAY,
            ..Default::default()
        },
        bomb_counter_font: asset_server.load("fonts/pixeled.ttf"),
        bomb_counter_colors: BoardAssets::default_colors(),
        flag_material: SpriteMaterial {
            texture: asset_server.load("sprites/flag.png"),
            color: Color::WHITE,
        },
        bomb_material: SpriteMaterial {
            texture: asset_server.load("sprites/bomb.png"),
            color: Color::WHITE,
        },
    });
    // Plugin activation
    state.set(AppState::InGame).unwrap();
}
```

Using the generic state system we set up in the [previous part](./8_states.md) we control when we want the plugin to launch.
Here, we want it to launch *after* we loaded our assets and set up the `BoardAssets` resource.
That's why we first set our *state* to `Out` and set it to `InGame` once our assets are ready.

Our plugin is now completely modular and has zero hard coded values, everything from the board size to the tile colors can be customized.

> Can we edit the theme at runtime?

Yes ! the `BoardAssets` resource is available to every system,
but everything that is not a `Handle` won't be applied until the next generation. For a more dynamic system you can check my plugin [bevy_sprite_material](https://github.com/ManevilleF/bevy_sprite_material).

---
Author: Félix de Maneville
Follow me on [Twitter](https://twitter.com/ManevilleF)