> [Check the repository](https://gitlab.com/qonfucius/minesweeper-tutorial)

# Tile Map Generation

Let's generate the minesweeper base tile map and set up our plugin.

Create  a `components` module with a `coordinates.rs` file and a `resources` module with `tile.rs` and `tilemap.rs` files in `board_plugin`:

```
├── Cargo.toml
└── src
    ├── components
    │   ├── coordinates.rs
    │   └── mod.rs
    ├── lib.rs
    └── resources
        ├── mod.rs
        ├── tile.rs
        └── tile_map.rs
```

## Components

To manage tiles and coordinates we are going to make our first component, `Coordinates`:

```rust
// coordinates.rs
use std::fmt::{self, Display, Formatter};
use std::ops::{Add, Sub};
use bevy::prelude::Component;

#[cfg_attr(feature = "debug", derive(bevy_inspector_egui::Inspectable))]
#[derive(Debug, Default, Copy, Clone, Ord, PartialOrd, Eq, PartialEq, Hash, Component)]
pub struct Coordinates {
    pub x: u16,
    pub y: u16,
}

// We want to be able to make coordinates sums..
impl Add for Coordinates {
    type Output = Self;

    fn add(self, rhs: Self) -> Self::Output {
        Self {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

// ..and subtractions
impl Sub for Coordinates {
    type Output = Self;

    fn sub(self, rhs: Self) -> Self::Output {
        Self {
            x: self.x.saturating_sub(rhs.x),
            y: self.y.saturating_sub(rhs.y),
        }
    }
}

impl Display for Coordinates {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

The `Coordinate` struct contains unsigned numeric values representing both axis of the board.
We add the`Display` implementation as a good practice, and the `Add` and `Sub` implementation to allow numeric operations.

Notice the use of `saturating_sub` to avoid **panic** if the subtraction result is negative.

We add the `Inspectable` derive through our `debug` feature gate. This trait will make our component display correctly in the inspector GUI.

> Why do a component here ?

We won't use `Coordinates` as component yet, but we will in future steps. This illustrates one of bevy aspects: *anything can be a component* if you derive `Component`.
We also added a bunch of derive attributes which will be useful in the future.

## Resources

### Tile

Let's declare our tiles:

```rust
// tile.rs
#[cfg(feature = "debug")]
use colored::Colorize;

/// Enum describing a Minesweeper tile
#[derive(Debug, Copy, Clone, Eq, PartialEq)]
pub enum Tile {
    /// Is a bomb
    Bomb,
    /// Is a bomb neighbor
    BombNeighbor(u8),
    /// Empty tile
    Empty,
}

impl Tile {
    /// Is the tile a bomb?
    pub const fn is_bomb(&self) -> bool {
        matches!(self, Self::Bomb)
    }

    #[cfg(feature = "debug")]
    pub fn console_output(&self) -> String {
        format!(
            "{}",
            match self {
                Tile::Bomb => "*".bright_red(),
                Tile::BombNeighbor(v) => match v {
                    1 => "1".cyan(),
                    2 => "2".green(),
                    3 => "3".yellow(),
                    _ => v.to_string().red(),
                },
                Tile::Empty => " ".normal(),
            }
        )
    }
}
```

We use an *enum* to avoid a complex struct, and add the `console_output` method which makes use of our optional `colorize` crate.

> Why this is not a component?

We could use `Tile` as a component but as we will see in future steps we want maximum flexibility, which means:
- **bomb** tiles will have a specific component
- **bomb neighbor** tiles will also have a specific component

**Queries** (`Query<>`) can only filter through component presence or absence (we call this *query artifacts*), so using directly our `Tile` struct would
not allow our systems to use **queries** targeting directly **bombs** for example, as all tiles would be queried.

### Tile Map

Let's make our tile map generator:

#### Empty map

```rust
// tile_map.rs
use crate::resources::tile::Tile;
use std::ops::{Deref, DerefMut};

/// Base tile map
#[derive(Debug, Clone)]
pub struct TileMap {
    bomb_count: u16,
    height: u16,
    width: u16,
    map: Vec<Vec<Tile>>,
}

impl TileMap {
    /// Generates an empty map
    pub fn empty(width: u16, height: u16) -> Self {
        let map = (0..height)
            .into_iter()
            .map(|_| (0..width).into_iter().map(|_| Tile::Empty).collect())
            .collect();
        Self {
            bomb_count: 0,
            height,
            width,
            map,
        }
    }

    #[cfg(feature = "debug")]
    pub fn console_output(&self) -> String {
        let mut buffer = format!(
            "Map ({}, {}) with {} bombs:\n",
            self.width, self.height, self.bomb_count
        );
        let line: String = (0..=(self.width + 1)).into_iter().map(|_| '-').collect();
        buffer = format!("{}{}\n", buffer, line);
        for line in self.iter().rev() {
            buffer = format!("{}|", buffer);
            for tile in line.iter() {
                buffer = format!("{}{}", buffer, tile.console_output());
            }
            buffer = format!("{}|\n", buffer);
        }
        format!("{}{}", buffer, line)
    }

    // Getter for `width`
    pub fn width(&self) -> u16 {
        self.width
    }

    // Getter for `height`
    pub fn height(&self) -> u16 {
        self.height
    }

    // Getter for `bomb_count`
    pub fn bomb_count(&self) -> u16 {
        self.bomb_count
    }
}

impl Deref for TileMap {
    type Target = Vec<Vec<Tile>>;

    fn deref(&self) -> &Self::Target {
        &self.map
    }
}

impl DerefMut for TileMap {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.map
    }
}
```

Our tile map has every generation options we need:
- `width` and `height` setting the dimensions and the number of tiles
- `bomb_count` setting the amount mines
- `map` a double 2D array of `Tile`

> Why use Vec<> and not slices?

I tried to do something like `[[Tile; WIDTH]; HEIGHT]` making use of rust 1.52 feature of generic *consts*, but I found it to get really messy.
If you find a clean way to do it I'd gladly accept a pull request !

Now we have:
- an `empty` method building a tile map of `Tile::Empty`
- a `console_output` method to print the tile map in the console
- a `Deref` and `DerefMut` implementation towards our 2D vector

#### Bombs and neighbors

Let's declare an array of 2D delta coordinates:

```rust
// tile_map.rs
/// Delta coordinates for all 8 square neighbors
const SQUARE_COORDINATES: [(i8, i8); 8] = [
    // Bottom left
    (-1, -1),
    // Bottom
    (0, -1),
    // Bottom right
    (1, -1),
    // Left
    (-1, 0),
    // Right
    (1, 0),
    // Top Left
    (-1, 1),
    // Top
    (0, 1),
    // Top right
    (1, 1),
];
```

These tuples define the delta coordinates of the 8 tiles in a square around any tile:

```
*--------*-------*-------*
| -1, 1  | 0, 1  | 1, 1  |
|--------|-------|-------|
| -1, 0  | tile  | 1, 0  |
|--------|-------|-------|
| -1, -1 | 0, -1 | 1, -1 |
*--------*-------*-------*
```

We can make use of it by adding a method to retrieve neighbor tiles:

```rust
// tile_map.rs
use crate::components::Coordinates;

 pub fn safe_square_at(&self, coordinates: Coordinates) -> impl Iterator<Item = Coordinates> {
        SQUARE_COORDINATES
            .iter()
            .copied()
            .map(move |tuple| coordinates + tuple)
    }
```
To allow the `Coordinates` + `(i8, i8)` we need to add the following in `coordinates.rs`:

```rust
// coordinates.rs
impl Add<(i8, i8)> for Coordinates {
    type Output = Self;

    fn add(self, (x, y): (i8, i8)) -> Self::Output {
        let x = ((self.x as i16) + x as i16) as u16;
        let y = ((self.y as i16) + y as i16) as u16;
        Self { x, y }
    }
}
```

Now that we can retrieve surrounding tiles we will use it to count bombs around coordinates, to fill our bomb neighbor tiles:

```rust
// tile_map.rs

pub fn is_bomb_at(&self, coordinates: Coordinates) -> bool {
    if coordinates.x >= self.width || coordinates.y >= self.height {
        return false;
    };
    self.map[coordinates.y as usize][coordinates.x as usize].is_bomb()
}

pub fn bomb_count_at(&self, coordinates: Coordinates) -> u8 {
    if self.is_bomb_at(coordinates) {
        return 0;
    }
    let res = self
         .safe_square_at(coordinates)
         .filter(|coord| self.is_bomb_at(*coord))
         .count();
    res as u8
}
```

Let's place our bombs and neighbors !

```rust
// tile_map.rs
use rand::{thread_rng, Rng};

/// Places bombs and bomb neighbor tiles
pub fn set_bombs(&mut self, bomb_count: u16) {
    self.bomb_count = bomb_count;
    let mut remaining_bombs = bomb_count;
    let mut rng = thread_rng();
    // Place bombs
    while remaining_bombs > 0 {
        let (x, y) = (
            rng.gen_range(0..self.width) as usize,
            rng.gen_range(0..self.height) as usize,
        );
        if let Tile::Empty = self[y][x] {
            self[y][x] = Tile::Bomb;
            remaining_bombs -= 1;
        }
    }
    // Place bomb neighbors
    for y in 0..self.height {
        for x in 0..self.width {
            let coords = Coordinates { x, y };
            if self.is_bomb_at(coords) {
                continue;
            }
            let num = self.bomb_count_at(coords);
            if num == 0 {
                continue;
            }
            let tile = &mut self[y as usize][x as usize];
            *tile = Tile::BombNeighbor(num);
        }
    }
}
```

Great, let's connect everything in the modules:

```rust
// board_plugin/resources/mod.rs

pub(crate) mod tile;
pub(crate) mod tile_map;
```

```rust
// board_plugin/components/mod.rs
pub use coordinates::Coordinates;

mod coordinates;
```

## Plugin

We have our tile map let's test it in our plugin:

```rust
// lib.rs
pub mod components;
pub mod resources;

use bevy::log;
use bevy::prelude::*;
use resources::tile_map::TileMap;

pub struct BoardPlugin;

impl Plugin for BoardPlugin {
    fn build(&self, app: &mut App) {
        app.add_startup_system(Self::create_board);
        log::info!("Loaded Board Plugin");
    }
}

impl BoardPlugin {
    /// System to generate the complete board
    pub fn create_board() {
        let mut tile_map = TileMap::empty(20, 20);
        tile_map.set_bombs(40);
        #[cfg(feature = "debug")]
        log::info!("{}", tile_map.console_output());
    }
}
```

What's happening here should look familiar, as we implement `Plugin` for our `BoardPlugin` we get access to our `App`.
We then register a simple startup **system** to generate our new tile map and print it.

We need to register our plugin to our `main.rs`

```rust
// main.rs
use board_plugin::BoardPlugin;

// ..
.add_plugin(BoardPlugin)
```

and let's run the app: `cargo run --features debug`

We now have our tile map printed in the console:

```
2022-02-21T09:24:05.748340Z  INFO board_plugin: Loaded Board Plugin
2022-02-21T09:24:05.917041Z  INFO board_plugin: Map (20, 20) with 40 bombs:
----------------------
|      111  1*1      |
|      1*211111 111  |
|111   223*1    1*1  |
|1*1   1*211   1221  |
|111   111     1*1   |
|121211    111 111   |
|*2*2*1    1*211     |
|121211  11212*21    |
|111     1*1 13*31   |
|1*1     111  2**21  |
|222         1234*2  |
|2*2         1*23*211|
|2*2         123*222*|
|111     1221 1*211*2|
|   111  1**1 111 111|
|   1*1  1221   1221 |
| 11322111   1111**1 |
| 1*3*11*1  12*11221 |
|123*21222  1*21 111 |
|1*211 1*1  111  1*1 |
----------------------
```

---
Author: Félix de Maneville
Follow me on [Twitter](https://twitter.com/ManevilleF)
