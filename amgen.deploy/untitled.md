# Untitled

# sig-archive

Signature archive for Haskell packages

See the original propsal here:
[Package signing proposal](https://github.com/commercialhaskell/commercialhaskell/wiki/Package-signing-proposal).

See the detailed proposal here: [Package signing detailed propsal](https://github.com/commercialhaskell/commercialhaskell/wiki/Package-signing-detailed-propsal)

## How to use this service

To use this service check out the
[sig-tool](https://github.com/commercialhaskell/sig-tool) project
which uses this repository as its source of data.

## Structure

There are two kinds of things stored in this archive:

* Signatures: a signature of the format `FINGERPRINT.asc` which is a
  signature for a package.
* Mappings: a YAML file listing mappings from authors to packages;
  specifying who can sign what package.

The file structure looks like this:

    signatures/
    signatures/demo/4.1.0/0D4F46E1.asc
    signatures/demo-base/1.2.3/0D4F46E1.asc
    signatures/example/4.5.6/0D4F46E1.asc
    signatures/zot/1.2.3/0D4F46E1.asc
    signatures/zot/1.2.3/34JKA8GD.asc
    mappings/
    mappings/foo.yaml
    mappings/foo.yaml.asc
    mappings/bar.yaml
    mappings/bar.yaml.asc

## Contributing

To contribute signatures of a package, you can use
[sig-tool](https://github.com/commercialhaskell/sig-archive) to submit
signatures for any package archive you have. The typical use-case is
that you will be the author of the package, but anyone can sign a
package. It is up to the mappings files whether those signatures are
trustworthy.

To contribute authoritative mappings from signers to packages, please
open a pull request:

1. Adding a file in the `mappings` directory
   `your-name-or-organization.yaml`.
2. In the mappings file specify who you trust to sign which
   packages. See the existing files in `mappings/` for examples.
3. Include a signature `your-name-or-organization.yaml.asc` of that
   mapping file.
