---
layout: post
title: "The sweet marriage of Rust and Unity"
category: technology
comments: true
---

Or setting up Continuous Delivery for Unity game with rust native plugin.

Contents:
* TOC
{:toc}

# Game-dev & Rust?

Development of games in **Rust** is in quite raw state. There are tools, eg. [Amethyst] or [gfx-rs], however, there's still plenty of work that needs to be done, before we'd be able to prototype something quickly or deliver a full fledged experience to a player.  
I've started thinking, why not use existing solutions and just integrate **Rust** into them, for that sweet performance boost and great experience during development.  
Game engine im most familiar with mould be **Unity**. I looked into whether *can I easily make this chimera multi-platform* and *can it support continuous delivery*? The answer was *yes* - it can.

# Unity and continuous delivery

There's well known option for making *in the cloud* builds of unity apps: Unity's **Cloud Builds** service. But theres a catch. Or two. It's not free and it didn't support custom steps last time I've checked. An alternative was needed. Here, to the rescue came github actions. They are fairly new, however, they have ever growing community of collaborators, that produce open source plugins, and what's most important in my case: there was defined unity build template.  

[unity-actions] allow to build and test projects on push. There are even actions responsible for requesting license server.   

### The yaml:

- this part does not need any special explanation.  

```yaml
name: Build my Game

on: push
```

- we must provide unity license - in my case it was *Unity Personal* one. I'll go deeper into where to get it in next section.  

{% raw %}
```yaml
env:
  UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
```
{% endraw %}

- first thing we checkout our repo and create cache for out Library folder, for the speedup of consecutive executions.  

```yaml
jobs:
  build:
    name: Build for Windows
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
        with:
          lfs: true
      - uses: actions/cache@v1.1.0
        with:
          path: Library
          key: Library
```

- then we run - first tests using `webbertakken/unity-test-runner` action, followed by a build for desired `targetPlatform` using `webbertakken/unity-builder`. Simple, isn't it?

> As of writing this article, I wasn't able to setup those runs with 2020 beta, due to segmentation fault in license verifier. When setting up your own job, keep that in mind.

> It is important that editors version that's assigned to `secrets.UNITY_LICENSE` is matching with the version we are making build on. I've seen notes that license files should work between versions but, well, that wasn't my case.

```yaml
      - name: Run tests
        uses: webbertakken/unity-test-runner@v1.3
        with:
          unityVersion: 2019.3.5f1
      - name: Build project
        uses: webbertakken/unity-builder@v0.10
        with:
          unityVersion: 2019.3.5f1
          targetPlatform: StandaloneWindows64 
```

- last thing, let's upload our build somewhere. We don't need to prepare release, so an artifact will be enough.

```yaml
      - uses: actions/upload-artifact@v1
        with:
          name: StandaloneWindows64
          path: build
```

# Getting the Unity license file

Pro users have it easier, cause all they have to do is fill *UNITY_SERIAL*, *UNITY_EMAIL* and *UNITY_PASSWORD* variables in repository secrets. For personal edition users, we need to request special *.ulf* file directly from Unity's website.  
First of all, we need license request file. It can be retrieved from **Unity Hub** in the settings, but it may be that It'll output your first license from when you've created given account. For me that was edition 2017 - it didn't work with my target editor.  
In order to overcome such an issue, I've created a dummy action on my github using https://github.com/marketplace/actions/unity-request-activation-file. This generated me *.alf* license request file with desired version. Then I've gone to https://license.unity3d.com/manual and was able to retrieve correct *.ulf* license file. I've then pasted it's contents to new secret in my repository under name `UNITY_LICENSE`. The build started working.  

# The native library

With game building nicely on Github's cloud we can switch our attention to getting it running with our rust native plugin. We create new library with `cargo new --lib`, we open Cargo.toml and add
```toml
[lib]
crate-type = ["cdylib"]
```
a crate type, so cargo will know how to build our lib when we run `cargo build --release`. We need a shared library, which is a `dll` on Windows, `dylib` on MacOS and `so` on Linux.  

Our code may look whatever, but for the functions we'll use outside must follow certain pattern:
```rust
#[no_mangle]
pub extern fn test() -> i32 {
    42
}
```
`extern` keyword tells the compiler that this function will be used outside and `#[no_mangle]` disables name mangling (rust usually adds a lot of useful information about the function to it's name, when it compiles a library, however, from outside, we'd have to guess what would be exact name we want to import).  
Let's place this code into *lib.rs* and setup a job, so github would compile it for us before adding as a *Plugin* to Unity.  
For that, as a base, I used [mean-bean-ci-template].
```yaml
lib_windows:
  runs-on: windows-latest
  needs: install-cross
  steps:
    - uses: actions/checkout@v2
      with:
        depth: 50
    - run: chmod +x ci/*
    - run: ci/set_rust_version.bash stable x86_64-pc-windows-msvc
    - run: cargo build --target x86_64-pc-windows-msvc --all-features --release
    - uses: actions/upload-artifact@v1
      with:
        name: library-x86_64-pc-windows-msvc.dll
        path: target/x86_64-pc-windows-msvc/release/library.dll
```
this required me to copy the folder *ci* from the [XAMPPRocky's repo][mean-bean-ci-template] into mine.  

One last thing - this artifact must be placed in *Assets/Plugins* directory in Unity Project. For that, we must add a step to unity build job:
```yaml
- name: Download lib
  uses: actions/download-artifact@v1
  with:
    name: library-x86_64-pc-windows-msvc.dll
    path: Assets/Plugins/
```

and we are almost done.

# Back to Unity - DllImport

Now we must head back to Unity, in main scene of our example project create an object and attach a script to it. Let's make it more fun - let our object be UI Text. In it's script we must first import our rust library. We do so by writing:
```csharp
[DllImport("library")]
private static extern Int32 test();
```
> when building for iOS we'd use library named **__Internal** instead. That's double underscore.

after this operation, we can just call `test()` from our code as normal function. Eg.:
```csharp
private void Start() {
  this.GetComponent<Text>().text = test().ToString();
}
```

viola, when ran, our project should put **42** in that text field (taken we compile our lib and place it in *Plugins* folder on our local machine (we can always push to github and download artifact for testing)).

# An expanded solution

The following contains compilation for multiple platforms, without tests, but adding these would be quite easy.

<style type="text/css">
  .gist-file
  .gist-data {max-height: 300px}
</style>
<script src="https://gist.github.com/luke-biel/ae708a4b0fcba9eafb27382971720499.js"></script>


[Amethyst]: https://github.com/amethyst/amethyst
[gfx-rs]: https://github.com/gfx-rs/gfx
[unity-actions]: https://github.com/webbertakken/unity-actions
[mean-bean-ci-template]: https://github.com/XAMPPRocky/mean-bean-ci-template