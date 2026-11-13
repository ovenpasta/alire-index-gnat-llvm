# Alire index for the gnat_llvm toolchain

[GNAT-LLVM](https://github.com/ovenpasta/gnat-llvm) (GCC 16.1 front end,
LLVM 21) as a prebuilt Alire toolchain, `gnat_llvm`, for x86_64 and aarch64
Linux. It ships the compiler, clang / wasm-ld, and the WebAssembly runtimes
(with native wasm exception handling) and a native Linux runtime. No need to
build LLVM or the compiler yourself.

## 1. Add this index (once)

```sh
alr index --add git+https://github.com/ovenpasta/alire-index-gnat-llvm --name gnat_llvm
```

Your default compiler does not change: only crates that depend on
`gnat_llvm` (directly or through e.g. `adawebpack`) use it.

## 2a. Plain Ada with Emscripten (node or browser)

Full `Ada.Text_IO`, files, `Ada.Calendar` and exceptions. Needs
[emsdk](https://emscripten.org/docs/getting_started/downloads.html) 5.0.2
with `emcc` on `PATH`.

```sh
alr init --bin hello_wasm && cd hello_wasm
alr with gnat_llvm
```

In `hello_wasm.gpr`, add:

```ada
   for Target use "llvm";
   for Runtime ("Ada") use
     external ("GNAT_LLVM_PREFIX") & "/lib/gnat-llvm/wasm32/rts-wasm-emcc-eh";

   package Compiler is
      for Default_Switches ("Ada") use
        Hello_Wasm_Config.Ada_Compiler_Switches & ("--target=wasm32");
   end Compiler;

   package Builder is
      for Executable ("hello_wasm.adb") use "hello_wasm.js";
   end Builder;

   package Linker is
      for Driver use "emcc";
      for Switches ("Ada") use ("-fwasm-exceptions");
   end Linker;
```

```sh
alr build
node bin/hello_wasm.js
```

`GNAT_LLVM_PREFIX` is set by the toolchain crate. emcc does the final link,
so you get `.js` + `.wasm` (emcc options such as `-o x.html` work too).

## 2b. Browser apps with AdaWebPack (DOM, WebGL, ...)

```sh
alr init --bin my_page && cd my_page
alr with adawebpack --use=https://github.com/ovenpasta/adawebpack
```

`my_page.gpr`:

```ada
with "config/my_page_config.gpr";
with "adawebpack.gpr";

project My_Page is
   for Target use AdaWebPack'Target;
   for Runtime ("Ada") use AdaWebPack'Runtime ("Ada");
   for Source_Dirs use ("src/", "config/");
   for Object_Dir use "obj/" & My_Page_Config.Build_Profile;
   for Create_Missing_Dirs use "True";
   for Exec_Dir use "bin";
   for Main use ("my_page.adb");

   package Compiler renames AdaWebPack.Compiler;
   package Binder is
      for Switches ("Ada") use ("-z");
   end Binder;
   package Linker is
      for Switches ("Ada") use AdaWebPack.Linker_Switches;
   end Linker;
   package Builder is
      for Executable ("my_page.adb") use "my_page.wasm";
   end Builder;
end My_Page;
```

`src/my_page.adb`:

```ada
with Web.HTML.Elements;
with Web.Strings;
with Web.Window;

procedure My_Page is
   Greeting : Web.HTML.Elements.HTML_Element :=
     Web.Window.Document.Get_Element_By_Id
       (Web.Strings.To_Web_String ("greeting"));
begin
   Greeting.Set_Hidden (False);
end My_Page;
```

Build, copy the JS glue next to the wasm, and add `bin/index.html`:

```sh
alr build
alr exec -- sh -c 'cp $ADAWEBPACK_ALIRE_PREFIX/source/adawebpack.mjs bin/'
```

```html
<p id="greeting" hidden>Hello from Ada!</p>
<script type="module">
  import * as AdaWebPack from './adawebpack.mjs';
  const { instance } = await WebAssembly.instantiateStreaming(
    fetch('my_page.wasm'), { env: AdaWebPack.imports });
  AdaWebPack.initialize(instance);
  instance.exports._ada_my_page();
</script>
```

Serve `bin/` (e.g. `python3 -m http.server`) and open `index.html`.

The runtime is chosen in `alire.toml`:

```toml
[gpr-set-externals]
ADAWEBPACK_RUNTIME = "wasm-eh"   # or wasm, wasm-eh-exnref, ...
```

The `*-exnref` runtimes also need `GNAT_WASM_EH.set = "exnref"` under
`[environment]`.

## Runtimes in the toolchain

| Runtime | Use |
|---------|-----|
| `rts-wasm-eh` | standalone (no Emscripten), full exception propagation; AdaWebPack's default |
| `rts-wasm` | standalone, `No_Exception_Propagation`, smallest |
| `rts-wasm-emcc-eh`, `rts-wasm-emcc` | on Emscripten's libc: full `Text_IO`, files, `Ada.Calendar` |
| `rts-wasm-wasi-eh` | WASI (wasmtime, wasmer, node:wasi); link with `wasm-ld` and a wasi-sysroot |
| `*-exnref` | the standardized exnref exception encoding |
| `rts-native` | native Linux programs |

The [AdaWebPack examples](https://github.com/ovenpasta/adawebpack/tree/master/examples)
(SDL3, HAC, DOM demos) are Alire crates too: `cd examples/<name> && alr build`.

Known: emcc links print a harmless `function signature mismatch: strncpy`
warning.

## Releases

Each [gnat-llvm release](https://github.com/ovenpasta/gnat-llvm/releases)
attaches its manifest (`gnat_llvm-<version>.toml`), which is added here under
`gn/gnat_llvm/`. The last version digit counts toolchain builds of the same
GCC release, as for Alire's `gnat_native`.
