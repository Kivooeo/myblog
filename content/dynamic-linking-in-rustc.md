---
title: "Dynamic Linking in Rust might be a thing!"
---

Rust’s `prefer-dynamic` flag enables dynamic linking for smaller binaries by relying on shared libraries (e.g., Rust’s standard library) at runtime. Use cases are specific, and static linking remains the default for portability

---

### How It Works  
- **Static vs. Dynamic**:  
  Static linking (default) embeds all dependencies into the binary. Dynamic linking loads libraries (e.g., `libstd`) at runtime, reducing binary size but requiring those libraries on the target system

- **Usage**:  
  Pass the flag to `rustc` directly:  
  ```
  rustc -C prefer-dynamic main.rs
  ```  
  or via Cargo (example will be below)
  
  *Note:* Only affects compatible dependencies (e.g., `libstd`). Some crates still link statically

---

### Pros 
1. **Smaller Binaries**:  
   Reduces size from MBs to KBs for tools/plugins. 
2. **Shared Systems**:  
   Multiple Rust apps on one system share a single `libstd` in memory
3. **Plugins/Mods**:  
   Facilitates lightweight, reloadable modules

---

### Cons  
- **Dependency Requirements**: Target systems must have the exact library versions (e.g., `libstd-*.so`)
- **Limited Compatibility**: Many crates lack dynamic versions
- **Version Mismatches**: Risk of crashes if library versions differ between build and target systems
- **Minor Overhead**: Runtime library loading adds negligible latency

---

### Example  
Let's build this basic code for start with just static linking
```rust
fn main() {
    println!("Hello, world");
}
```

so we run `cargo build --release` here and see this

` -a----          5/6/2025   2:50 AM         134144 f `

then, to tell cargo run `prefer-dynamic` as argument, we should write 
```toml
[build]
rustflags = ["-C", "prefer-dynamic"]
```
in `~/.cargo/config.toml`, and now... we can recompile this to watch results
running `cargo build --release` once again and see

` -a----          5/6/2025   2:55 AM          10240 f `

it's way better, agree?

### Problems you may face

so just right after compiling this thing and trying to run it, you likely we'll see something like this
```
$ ./main
./main: error while loading shared libraries: libstd-f6265b21db1f990f.so: cannot open shared object file: No such file or directory
```

this problem occurs because our binary can't find a .so stdlib files in `LD_LIBRARY_PATH`, so to fix it, we have to manually setup this, on linux you can do something like this

```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/.rustup/toolchains/<toolchain>/lib/rustlib/<toolchain>/lib/
```
or for windows you should use this

```powershell
$env:Path = "$env:Path;$env:USERPROFILE\.rustup\toolchains\<toolchain>\lib\rustlib\<toolchain>\lib"
```

this directory will contain all .so (or .rlib or .dll im not sure what windows version uses) files that you need to run the programm correctly, after adding a path to our library we can finally and most likely succesfully run our binary
```
$  ~ ./main  
Hello, world!
$  ~ ls -ls main   
20 -rwxr-xr-x 1 coder coder 17064 May  8 07:49 main
```
![Мой котик](/images/cat.gif)
---

### When to Avoid  
- **Portable Apps**: Static linking ensures no external dependencies
- **Uncontrolled Environments**: Avoid dependency issues on user machines
- **Performance-Critical Code**: Static binaries eliminate runtime linking overhead
- **LTO**: It's just not works when LTO enabled
