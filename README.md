# Typst Flatten
A npm package to flatten Typst projects into a single `main.typ` file.

## How are Typst modules handled
It is not as straightforward as one might think, in a sense we need to recreate part of the compiler module logic in javascript.

Below there is an overview on the challenges of handling modules in Typst.

### Simple `.typ` imports
For local Typst files we need to distinguish among the following cases:

**Case A - Import module**
```typ
#import "lib.typ"
```
Variables and functions from `"lib.typ"` will need to accessed using `lib.foo` or `lib.bar()`.

**Case B - Import variables and functions**
```typ
#import "lib.typ": foo, bar
```
Variables and functions get imported specifically and they can be used directly.

**Case C - Import everything**
```typ
#import "lib.typ": *
```
All variables and functions get imported and can be used directly.

**Case D - Import and rename module**
```typ
#import "lib.typ" as library
```
Renames the module. Then a variable will be used like `library.foo`.

**Case E - Import and rename variables and functions**
```typ
#import "lib.typ": foobar as fb, foo, bar
```
Imports a specific set of variables and functions, some of which can be renamed. All of them can be used directly (using their new names).

### Problems that arise
Having to carefully rename every variable and function for the `main.typ` output is not the main problem.

The main problem is potential collisions. For example if a module uses:
```typ
#import "lib_A.typ" as lib
#let y = lib.foo(x)
```
And the other one uses
```typ
#import "lib_B.typ": some-function  as foo
#let y = foo(x)
```
When mergin the modules in a single file we might end up with a lot of collissions and compile errors.

What we might do is not handle this at all and hope for the best. In most cases this won't happen, in the cases where this happens our tool can't be used. To actually do it properly we would properly need to use the same parser and evaluator the typst compiler uses internally.

### Handle relative paths

We also need to be able to resolve any relative import and construct from there an internal dependency tree.

```typ
#import "../uncle.typ"
#import "sibling.typ"
#import "./sibling.typ"
#import "../uncle-folder/cousin.typ"
#import "/project-root-child.typ"

```

### Typst Universe packages
Besides, we have the imports of packages from the Typst Universe.

One could think it's as simple as to fetch them all across files and put them at the top of the `main.typ` output file.

However these packages can also have the cases mentioined above, so they can be renamed and some of their variables or functions can also be renamed.

### Set rules
A module B might import a function from module A, and then use a set rule for that function. 

Besides set  rules might be scoped. But this is so hard to handle properly that we'll just hope for the best.

What we'll do is not treat set rules (copy paste them wherever they are).

### Show rules
This is probably the hardest part of it all. Show rules change the whole document, they allow overwritting the look or behaviour of existing functions, both typst invernal functions (like `underline`) and other module's custom functions (like `custom-table`). 

Show rules can also be scoped (for example if used inside a conditional only take effect inside the conditional.

But again, this adds so much complexity that we just hope for the best and  don't handle them at all.

###  Include imports
Besides importing a module, we can `include` it, which evaluates a file and inputs it's conents directly.

This is not the same as saying it copy pastes the source code, it can't get tricky but for most cases the use of `include` will be that, a simple copy paste so our implementation will be the simplest and most straightforward hoping for the best.

### Custom Plugins
Web Assembly custom plugins are not commonly used but they are also supported.

Instead of doing this:
```typ
#let myplugin = plugin("hello.wasm")
```
We pass the WASM plugin bytes inline:
```typ
#let myplugin = plugin(base64.decode("AGFzbQE..."))
```

## How are other file imports handled
- SVG images get converted to inline SVG images.
  ```typ
  #image(width: 40%, bytes("<svg xmlns=\"http://www.w3.org/2000/svg\"><path fill=\"purple\" d=\"M19.1835\"/></svg>"))
  ```
  _Note: With proper escaping of Typst special characters._
- JPG and PNG images get converted to a base64 inline string (using `.readAsDataURL()` from the browser's native `FileReader` API), and decoded using the Typst Universe `based` package (`#import "@preview/based:0.2.0": base64`).
  ```typ
  #image(base64.decode("/9j/4AAQSkZJRgABAQEASABIAAD/4QAC/9sAhAAIBgYHB..."))
  ```
  _Note: Obviously preserving the attributes the original image had like `width`, `heigh`, `alt`, etc._
- Data files like `.csv`, `.json`, `.yml`, `.toml`, `.cbor` or `.xml` get used exactly as they would but their contents are added inline using the `bytes` function before each corresponding data reading function.
  ```typ
  #let person = json(bytes("{\"name\": \"Martí\", \"age\": 23}"))
  Hi, my name #person.name, I'm #person.age years old.
  ```
  ```typ
  #let root = xml(bytes("
  <person id=\"1124\">
    <name>Martí</name>
    <age>30</age>
  </person>
  ")).at(0)
  #let id = root.attrs.id
  #let name = root.children.at(1).children.at(0)
  #let age = root.children.at(3).children.at(0)
  The user #id is called #name and is #age years old.
  ```
  _In other words, the function  `csv()` expects a path string, searches the file, reads it's bytes and then decodes it to Typst. But if you use `csv(bytes())` it expects the file's contents directly._
- Any other files like `.txt` or files with no extension (eg. `LICENSE`) that were originally read using the `read` function, get read with the native's browser `File` API which uses UTF-8 by default like Typst. If the original `read` function has as argument `encoding: none` we use `ArrayBuffer` and `Unit8Array`. And then the contents simply get placed inline.
  
  For example if the `read` function was used like this...
  ```typ
  #raw(read("README.txt"), block: true)
  ```
  ...now the file's contents get placed inline.
  ```typ
  #raw("ABOUT THIS PROJECT\nThis is a very special project...", block: true)
  ```
  If we had `encoding: none` being used...
  ```typ
  #read("README.txt", encoding: none)
  ```
  ...now the bytes get placed inline using the `bytes` function.
  ```typ
  #bytes("ABOUT THIS PROJECT\nThis is a very special project...")

## Architecture
###  Virtual filesystem
We can create a virtual filesystem interface, something like:
```js
interface TypstFileSystem {
  readText(path: string): Promise<string>;
  readBytes(path: string): Promise<Uint8Array>;
  writeText(path: string, content: string): Promise<void>;
  exists(path: string): Promise<boolean>;
  resolve(from: string, target: string): string;
}
```
And then make it agnostic so our npm package can work both with node.js and in the browser.

### Which parser to use
We could write our own Typst parser, or we can use an npm package like `codemirror-lang-typst` (native Lezer parser), we'll have to see which fits best.

### What's the best approach
Maybe using Rust and the official crates would be a more robust solution, but can we still be sure that it would work reliablly on all cases? If not the migration to Rust + Web Assembly is not worth it.

## License
MIT
