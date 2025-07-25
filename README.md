<div align="center">

# Xilem

**An experimental Rust architecture for reactive UI**

[![Linebender Zulip chat.](https://img.shields.io/badge/Xi%20Zulip-%23xilem-blue?logo=Zulip)](https://xi.zulipchat.com/#narrow/stream/354396-xilem)
[![GitHub Actions CI status.](https://img.shields.io/github/actions/workflow/status/linebender/xilem/ci.yml?logo=github&label=CI)](https://github.com/linebender/xilem/actions)
[![Dependency staleness status.](https://deps.rs/repo/github/linebender/xilem/status.svg)](https://deps.rs/repo/github/linebender/xilem)
[![Apache 2.0 license.](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](#license)
[![Documentation build status.](https://img.shields.io/docsrs/xilem.svg)](https://docs.rs/xilem)

</div>

This repo contains an experimental architecture, implemented with Masonry.
At a very high level, it combines ideas from Flutter, SwiftUI, and Elm.
Like all of these, it uses lightweight view objects, diffing them to provide minimal updates to a retained UI.
Like SwiftUI, it is strongly typed.

## Project structure

This diagram gives an idea what the Xilem project is built on:

![Xilem project layers](docs/assets/xilem-layers.svg)

On a very coarse level, Xilem is built directly on top of xilem_core and Masonry, both of which are crates in this repository.

Then Masonry is built on top of:

- **winit** for window creation.
- **Vello and wgpu** for 2D graphics.
- **Parley** for [the text stack](https://github.com/linebender/parley#the-Parley-text-stack).
- **AccessKit** for plugging into accessibility APIs.

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for details.

<!--- TODO: This needs a serious refactor.
This section should not be in the main README. -->

### Overall program flow

> **Warning:**
>
> This README is a bit out of date. To understand more of what's going on, please read the blog post, [Xilem: an architecture for UI in Rust].

Like Elm, the app logic contains *centralized state.*
On each cycle (meaning, roughly, on each high-level UI interaction such as a button click), the framework calls a closure, giving it mutable access to the app state, and the return value is a *view tree.*
This view tree is fairly short-lived; it is used to render the UI, possibly dispatch some events, and be used as a reference for *diffing* by the next cycle, at which point it is dropped.

We'll use the standard counter example.
Here the state is a single integer, and the view tree is a column containing two buttons.

```rust
fn app_logic(data: &mut u32) -> impl View<u32, (), Element = impl Widget> {
    Column::new((
        Button::new(format!("count: {}", data), |data| *data += 1),
        Button::new("reset", |data| *data = 0),
    ))
}
```

These are all just vanilla data structures.
The next step is diffing or reconciling against a previous version, now a standard technique.
The result is an *element tree.*
Each node type in the view tree has a corresponding element as an associated type.
The `build` method on a view node creates the element, and the `rebuild` method diffs against the previous version (for example, if the string changes) and updates the element.
There's also an associated state tree, not actually needed in this simple example, but would be used for memoization.

The closures are the interesting part.
When they're run, they take a mutable reference to the app data.

### Components

A major goal is to support React-like components, where modules that build UI for some fragment of the overall app state are composed together.

```rust
struct AppData {
    count: u32,
}

fn count_button(count: &mut u32) -> impl View<u32, (), Element = impl Widget> {
    Button::new(format!("count: {}", count), |data| *data += 1)
}

fn app_logic(data: &mut AppData) -> impl View<AppData, (), Element = impl Widget> {
    lens(count_button, data, |data| &mut data.count)
}
```

This `lens` node should be quite familiar to existing Druid users, and is also very similar to the [Html.map] node in Elm.
Note that in this case the data presented to the child component to render, and the mutable app state available in callbacks is the same, but that is not necessarily the case.

### Memoization

In the simplest case, the app builds the entire view tree, which is diffed against the previous tree, only to find that most of it hasn't changed.

When a subtree is a pure function of some data, as is the case for the button above, it makes sense to *memoize.*
The data is compared to the previous version, and only when it's changed is the view tree build.
The signature of the memoize node is nearly identical to [Html.lazy] in Elm:

```rust
fn app_logic(data: &mut AppData) -> impl View<AppData, (), Element = impl Widget> {
    Memoize::new(data.count, |count| {
        Button::new(format!("count: {}", count), |data: &mut AppData| {
            data.count += 1
        })
    }),
}
```

The current code uses a `PartialEq` bound, but in practice I think it might be much more useful to use pointer equality on `Rc` and `Arc`.

I anticipate it will also be possible to do dirty tracking manually - the app logic can set a dirty flag when a subtree needs re-rendering.

### Optional type erasure

By default, view nodes are strongly typed.
The type of a container includes the types of its children (through the `ViewTuple` trait), so for a large tree the type can become quite large.
In addition, such types don't make for easy dynamic reconfiguration of the UI.
SwiftUI has exactly this issue, and provides [AnyView] as the solution.
Ours is more or less identical.

The type erasure of View nodes is not an easy trick, as the trait has two associated types and the `rebuild` method takes the previous view as a `&Self` typed parameter.
Nonetheless, it is possible.
(As far as I know, Olivier Faure was the first to demonstrate this technique, in [Panoramix], but I'm happy to be further enlightened)

## Prerequisites

### Linux and BSD

You need to have installed `pkg-config`, `clang`, and the development packages of `wayland`, `libxkbcommon`, `libxcb`, and `vulkan-loader`.
Most distributions have `pkg-config` installed by default.

To install the remaining packages on Fedora, run:

```sh
sudo dnf install clang wayland-devel libxkbcommon-x11-devel libxcb-devel vulkan-loader-devel
```

To install the remaining packages on Debian or Ubuntu, run:

```sh
sudo apt-get install clang libwayland-dev libxkbcommon-x11-dev libvulkan-dev
```

## Recommended Cargo Config

The Xilem repository includes a lot of projects and examples, most of them pulling a lot of dependencies.

If you contribute to Xilem on Linux or macOS, we recommend using [`split-debuginfo`](https://doc.rust-lang.org/cargo/reference/profiles.html#split-debuginfo) in your [`.cargo/config.toml`](https://doc.rust-lang.org/cargo/reference/config.html#hierarchical-structure) file to reduce the size of the `target/` folder:

```toml
[profile.dev]
# One debuginfo file per dependency, to reduce file size of tests/examples.
# Note that this value is not supported on Windows.
# See https://doc.rust-lang.org/cargo/reference/profiles.html#split-debuginfo
split-debuginfo="unpacked"
```

## Minimum supported Rust Version (MSRV)

This version of Xilem has been verified to compile with **Rust 1.88** and later.

Future versions of Xilem might increase the Rust version requirement.
It will not be treated as a breaking change and as such can even happen with small patch releases.

## Community

Discussion of Xilem development happens in the [Linebender Zulip](https://xi.zulipchat.com/), specifically the [#xilem channel](https://xi.zulipchat.com/#narrow/stream/354396-xilem).
All public content can be read without logging in.

Contributions are welcome by pull request.
The [Rust code of conduct] applies.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache 2.0 license, shall be licensed as noted in the [License](#license) section, without any additional terms or conditions.

## License

Licensed under the Apache License, Version 2.0 ([LICENSE](LICENSE) or <http://www.apache.org/licenses/LICENSE-2.0>)

Some files used for examples are under different licenses:

- The font file (`RobotoFlex-Subset.ttf`) in `xilem/resources/fonts/roboto_flex/` is licensed solely as documented in that folder (and is not licensed under the Apache License, Version 2.0).
- The data file (`status.csv`) in `xilem/resources/data/http_cats_status/` is licensed solely as documented in that folder (and is not licensed under the Apache License, Version 2.0).
- The data file (`emoji.csv`) in `xilem/resources/data/emoji_names/` is licensed solely as documented in that folder (and is not licensed under the Apache License, Version 2.0).

[Html.lazy]: https://guide.elm-lang.org/optimization/lazy.html
[Html map]: https://package.elm-lang.org/packages/elm/html/latest/Html#map
[Rc::make_mut]: https://doc.rust-lang.org/std/rc/struct.Rc.html#method.make_mut
[AnyView]: https://developer.apple.com/documentation/swiftui/anyview
[Panoramix]: https://github.com/PoignardAzur/panoramix
[Xilem: an architecture for UI in Rust]: https://raphlinus.github.io/rust/gui/2022/05/07/ui-architecture.html
[xkbcommon]: https://github.com/xkbcommon/libxkbcommon
[Rust code of conduct]: https://www.rust-lang.org/policies/code-of-conduct
