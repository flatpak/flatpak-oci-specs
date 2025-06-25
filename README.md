# Flatpak OCI Specifications

This repository includes specifications related to storing Flatpaks as
[OCI images](https://github.com/opencontainers/distribution-spec/), and
downloading Flatpaks via the
[OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/)

This **ARE NOT** official specifications of the Open Container Initiative.
The specifications are fairly general and were writen to be useful for images other than Flatpaks,
but are currently only used within the Flatpak ecosystem.

Included specifications are:

**[Registry Index](registry-index.md)**
A protocol and index format for flexibly querying images stored in a repository.

**[Image Deltas](image-deltas.md)**
A mechanism for efficiently downloading a new version of an image
when the client already has a previous version.

## Authors

These specifications were originally created by
Owen Taylor <<otaylor@redhat.com>> and Alexander Larsson <<alexl@redhat.com>>.

## Known implmentations

Some links below may be "permalinks" to ensure stability, but if you're
viewing them you should likely look at the most recent commit.

- [Flatpak OCI (permalink)](https://github.com/flatpak/flatpak/blob/66b038e14809bc973d46e8078a070dc32e894903/common/flatpak-oci-registry.c)
- [flatpak-indexer](https://github.com/owtaylor/flatpak-indexer)

## Contributing

Specifications should be clean according to
[markdownlint](https://github.com/DavidAnson/markdownlint)
([vscode extension](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint))
and line-wrapped to 100 columns, using [semantic line breaks](https://sembr.org/).

The specification and code is licensed under the Apache 2.0 license found
in the [LICENSE](LICENSE) file of this repository.
