# Flatpak OCI Specifications

This repository includes specifications related to storing Flatpaks as
[OCI images](https://github.com/opencontainers/distribution-spec/), and
downloading Flatpaks via the
[OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/)

This **ARE NOT** official specifications of the Open Container Initiative.
The specifications are fairly general and were writen to be useful for images other than Flatpaks,
but are currently only used within the Flatpak ecosystem.

Included specifications are:

**[Registry Index](registry-index)**
A protocol and index format for flexibly querying images stored in a repository.

**[Image Deltas](image-deltas)**
A mechanism for efficiently downloading a new version of an image
when the client already has a previous version.

## Authors

These specifications were originally created by
Owen Taylor <<otaylor@redhat.com>> and Alexander Larsson <<alexl@redhat.com>>.

## Contributing

Specifications should be clean according to
[markdownlint](https://github.com/DavidAnson/markdownlint)
([vscode extension](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint))
and line-wrapped to 100 columns, using [semantic line breaks](https://sembr.org/).

The specification and code is licensed under the Apache 2.0 license found
in the [LICENSE](LICENSE) file of this repository.
