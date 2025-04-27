<div align="center">
    <img src="branding/nte-colors.svg" width="200" />
    <br />
    <h1>nte</h1>
</div>

nix template engine - takes some templates, entries and applies the templates to the entries

nte's main repository is on [my forgejo](https://git.poz.pet/poz/nte) instance

mirrors are available on [github](https://github.com/imnotpoz/nte) and [codeberg](https://codeberg.org/poz/nte), I accept contributions from anywhere

# sites written in nte
https://poz.pet

https://nixwebr.ing

https://mewoocat.github.io/

if your site (or anything else) is written in nte, let me know and I'll add you to this list

you can also use this button on your site and link to one of the repos

[<img src="branding/powered-by-nte.png">](https://git.poz.pet/poz/nte)

# examples

check `example/` for a static website written in nte

build and run it using 
```sh
nix shell nixpkgs#darkhttpd --command sh -c "nix build -L .#examples.x86_64-linux.default && darkhttpd ./result"
```
the site will be available at http://localhost:8080

the example is a cut down version of my own website

# usage

first add nte as an input in your project's flake

```nix
nte = {
  url = "git+https://git.poz.pet/poz/nte";
  # or one of the mirrors
  #url = "git+https://codeberg.org/poz/nte";
  #url = "github:imnotpoz/nte";
  inputs.nixpkgs.follows = "nixpkgs";
};
```

then use the `mkNteDerivation` wrapper over `stdenv.mkDerivation` available under
```nix
inputs.nte.functions.${system}.mkNteDerivation
```
it accepts an attrset of:
- `name`, `version`, `src` - passthrough to `stdenv.mkDerivation`
- `extraArgs` - an attrset of additional arguments passed to all entries and templates
- `entries` - a list of all entry files to be processed
- `templates` - a list of all template files to be applied
- `extraFiles` - a list of either:
    - a string containing the path to the source file - will be copied to `$out` in the `installPhase`
    - attrset of `source` and `destination`:
        - `source` - a string containing a path, if relative `$PWD` is `$src` in the `installPhase`
        - `destination` - a string containing a path, never absolute, appended to `$out` in the `installPhase`
- `preBuild` - passthrough to `stdenv.mkDerivation`
- `postBuild` - passthrough to `stdenv.mkDerivation`
- `preInstall` - passthrough to `stdenv.mkDerivation`
- `postInstall` - passthrough to `stdenv.mkDerivation`

make sure not to use nix paths in `extraFiles` if you want the names of the files to match up

example usage of the wrapper function:
```nix
mkNteDerivation {
  name = "nte-example";
  version = "0.1";
  src = ./.;

  extraArgs = {
    foo = 2137;
    bar = "dupa";
    baz = arg1: arg2: ''
      here's arg1: ${arg1}
      and here's arg2: ${arg2}
    '';
  };

  entries = [
    ./entry1.nix
    ./foo/entry2.nix
    ./foo/entry3.nix
    ./bar/entry4.nix
    ./bar/entry5.nix
    ./bar/entry6.nix
  ];

  templates = [
    ./template1.nix
    ./template2.nix
  ];

  extraFiles = [
    "./data.txt" # equivalent to { source = ./data.txt; destination = "/"; }
    { source = "./image.png"; destination = "/assets/"; }
    { source = "./image2.png"; destination = "/assets/dupa.png"; }
    { source = "./data/*"; destination = "/assets/data/"; }
    { source = fetchurl { ... }; destination = "/"; }
  ];
}
```

nte will handle creating directories if your source file structure isn't flat

if the `mkNteDerivation` wrapper isn't enough for you, you can do things the old way - by putting the output of `inputs.nte.functions.${system}.engine` in a derivation's `buildPhase`:
```nix
mkDerivation {
  # ...

  buildPhase = ''
    runHook preBuild

    ${engine {inherit extraArgs entries templates;}}

    runHook postBuild
  '';
}
```

in that case if you wish to replicate the functionality of `extraFiles` you can use the derivation's `installPhase`, manually `mkdir` the needed directories and `cp` your files into `$out`

nte offers a standard library that contains:
- `nixpkgs`
- `getEntry` - a function that gives you access to the entry's attributes
- `applyTemplate` - a function that allows you to manually apply a template to an entry
- utility functions found in [stdlib.nix](./stdlib.nix)

## templates

a template can take an arbitrary number of arguments and returns `{ name, format, output }`:

- `name` - used as a template ID for the entries
- `format` - the extension of the output file (ignored if an entry defines `file`)
- `output` - string if in a base template, entry to another template otherwise

example template:
```nix
{
  name,
  location,
  info,
  ...
}: {
  name = "greeting";
  format = "txt";

  output = ''
    Hello ${name}! Welcome to ${location}.

    Here's some more information:
    ${info}
  '';
}
```

a template's output can also be an entry to another template:
```nix
{
  name,
  location,
  date,
  time,
  ...
}: {
  name = "greeting-with-date";
  output = {
    template = "greeting";

    inherit name location;

    info = ''
      You're visiting ${location} on ${date} at ${time}!
    '';
  };
}
```
a template that's inherited from a different template also inherits its format - no need to define it again

## entries

an entry can take an arbitrary number of arguments and returns `{ template, ... }`, the `...` being the desired template's arguments (sans `extraArgs`, those are passed either way)

there's a built-in `passthrough` template, which (as the name might suggest) takes in a `format` and `output` and passes them through to the template with no changes

this is useful if you're using nte to create a single file - you won't have to create a boilerplate template

example entries (using the previous example templates):
```nix
_: {
  template = "greeting";

  name = "Jacek";
  location = "Wrocław";
  info = ''
    As of 2023, the official population of Wrocław is 674132 making it the third largest city in Poland.
  '';
}
```
an entry using the stdlib:
```nix
{
  run,
  ...
}: {
  template = "greeting-with-date";

  name = "Rafał";
  location = "Osieck";
  date = run "date +%F";
  time = run "date +%T";
}
```
if a binary isn't in `$PATH`, remember that each entry gets `pkgs`:
```nix
{
  pkgs,
  run,
  ...
}: let
  date = "${pkgs.coreutils-full}/bin/date";
in {
  # ...
  date = run "${date} +%F";
  time = run "${date} +%T";
}
```

nte by default will follow your source file structure, if you want to specify the output location yourself use `file`:
```nix
_: {
  # ...
  file = "foo/bar.txt";
}
```
in this example the output of this entry will end up at `$out/foo/bar.txt` instead of the default location - a base template's `format` will also be ignored

# thanks

[raf](https://notashelf.dev/) for helping me out with some of the nix and setting up mirrors

# license
MIT
