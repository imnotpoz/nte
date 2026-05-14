# changelog

## 0.1.0

- made this

## 0.2.0

- made `file` in entries optional

## 0.3.0

- set nixpkgs input as unstable by default
- added `github:nix-systems/default` as an input
- moved the following outputs:
  - `engines.${system}.default` -> `functions.${system}.engine`
- introduced `mkNteDerivation` helper function (`functions.${system}.mkNteDerivation`)

## 0.3.1

- improved engine code readability a bit
- exposed `applyTemplate` so that users can do templating inside a single file

## 0.3.2

- allowed the use of raw paths alongside attrsets in `extraFiles`

## 0.3.3

- made `applyTemplate` work on a raw entry and return a string
  and added an internal `applyTemplateFile` that works as the former did before
- added built-in `passthrough` template

## 0.3.4

- fixed an issue where `file` wasn't available to the base template
  if another one inherited from it

## 0.3.5

- added `{pre,post}{Build,Install}` as passthrough parameters to `mkNteDerivation`

## 0.3.6

- fixed the above mentioned hooks not having any effect

## 0.3.7

- added passthrough `meta` parameter to `mkNteDerivation`

## 0.3.8

- replaced `systems` input with hardcoded `lib.platforms.all`
