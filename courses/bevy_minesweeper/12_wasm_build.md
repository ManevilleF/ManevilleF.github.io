> [Check the repository](https://gitlab.com/qonfucius/minesweeper-tutorial)

# WASM Build

Let's make our project run on browser:

Let's change our `board_plugin/Cargo.toml`:

```diff
[dependencies]
- # Engine
- bevy = "0.6"

# Serialization
serde = "1.0"

# Random
rand = "0.8"

# Console Debug
colored = { version = "2.0", optional = true }
# Hierarchy inspector debug
bevy-inspector-egui = { version = "0.8", optional = true }

+ # Engine
+ [dependencies.bevy]
+ version = "0.6"
+ default-features = false
+ features = ["render"]

+ # Dependencies for WASM only
+ [target.'cfg(target_arch = "wasm32")'.dependencies.getrandom]
+ version="0.2"
+ features=["js"]
```
We only need one feature of the `bevy` dependency so we improve its declaration, and we add a new dependency only for `wasm` targets which improves `rand` on browser.

Now we edit the main `Cargo.toml`:

```diff
[dependencies]
- bevy = "0.6"
board_plugin = { path = "board_plugin" }

# Hierarchy inspector debug
bevy-inspector-egui = { version = "0.8", optional = true }


+ [dependencies.bevy]
+ version = "0.6"
+ default-features = false
+ features = ["render", "bevy_winit", "png"]

+ # Dependencies for native only.
+ [target.'cfg(not(target_arch = "wasm32"))'.dependencies.bevy]
+ version = "0.6"
+ default-features = false
+ features = ["x11"]

[workspace]
members = [
    "board_plugin"
]
```

We can also disable the default features of `bevy` and only enable the useful ones. We add the `x11` feature only for native use to avoid compilation issue for web assembly.

And.. That's it ! The app can now compile and run natively on wasm.

Let's improve a bit by adding a cargo config in `.cargo/config.toml`:

```toml
[target.wasm32-unknown-unknown]
runner = "wasm-server-runner"

[alias]
serve = "run --target wasm32-unknown-unknown"
```

Then install the runner:
`cargo install wasm-server-runner`

you may now directly execute `cargo serve` and test your app on your browser !
You may also try the [live version](https://qonfucius.gitlab.io/minesweeper-tutorial/)

---
Author: Félix de Maneville
Follow me on [Twitter](https://twitter.com/ManevilleF)