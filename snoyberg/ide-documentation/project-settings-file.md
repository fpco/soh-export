# Project settings file

Every project in the FP Haskell Center IDE includes a metadata file containing various project settings. This file is stored as `.project-settings.yaml` and, as you may have guessed, uses the YAML file format. Most of the time, you'll never see this file, since the IDE provides access to its contents via a GUI. You can view the file by pushing your project to a remote Git repository and looking at the contents there.

There is one time when you will encounter this file in the IDE: when it is invalid. This can happen because:

1. During a merge resolution, the file ended up in an invalid state.
2. Someone manually modified the file in a remote Git repository, and you have just pulled in the change.

When this occurs, you will be required to manually resolve problems in this file before the IDE will be fully functional. Usually, such resolution is a simple change. However, to be certain this can be done properly, you need to know the file format. This file will explain the different sections of the file. It assumes basic knowledge of YAML. If you've never used YAML before, you may want to [have a quick look at Wikipedia](http://en.wikipedia.org/wiki/YAML#Features).

The YAML file is represented with a top level map, with the following sections:

## version

Indicates the version number of the YAML file format. The version described in this document is 1.

```yaml
version: 1
```

## extensions

`extensions` provides a key/value mapping between language extensions, and whether they are enabled.

```yaml
extensions:
    OverloadedStrings: true
    ImplicitPrelude: false
```

## cabal-file

This setting is available for future use, though we currently ignore it. It indicates whether or not a cabal file should be auto-generated for your project on commit and, if so, the file it should be saved in.

## module-templates

Provides a template for newly created modules, where the special string `MODULE_NAME` will be replaced with the new module name.

```yaml
module-template: ! 'module MODULE_NAME where

import SomeCommonModule
'
```

## modules

Indicates the modules available in your project, the file path they are located, and (optionally) the target type, either web or console.

```yaml
modules:
  Util:
    filename: src/Util.hs
  WebGui:
    filename: src/WebGui.hs
    target-type: web
  TestSuite:
    filename: test/TestSuite.hs
    target-type: console
```