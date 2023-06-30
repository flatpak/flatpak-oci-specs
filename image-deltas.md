# Image Deltas

The traditional model for container images is to download the entire new image for every update.
This works pretty well for the traditional usage of containers for servers in a datacenter,
but can be problematical when using images for other usages,
such as storing desktop applications,
or for storing applications and OS images for IOT devices.
In these cases, bandwidth can be constrained or expensive.

Using *deltas* between layers can hugely decrease the necessary bandwidth.

Some properties that are desirable for a delta solution include:

* Deltas should be distributed by the registry mechanism
  and used transparently by clients when available.
* Applying updates should not require a *smart server* -
  deltas should be blobs suitable for distribution on a CDN.
* Delta compression should work on a wide variety of image types,
  producing small updates for typical code and configuration changes.
* It should be possible to verify that a delta
  produces the same result as downloading the full image -
  it should not be necessary to separately sign or trust deltas.
* It should be possible to create deltas or remove deltas at any point -
  they should not have to be created as part of the image creation process.

The delta update mechanism specified here works as follows:
a client that wants to update to a new version of an image
first downloads the image manifest for the target image.
It then obtains from the registry a *delta manifest*
that contains pointers to *layer deltas*
that can be used to reconstruct layers in the target image
from layers the client might have available locally.
If an appropriate layer delta is found, it is downloaded,
and the target layer is reconstructed, otherwise the target layer is downloaded from scratch.

## Layer Deltas

Layers in an OCI or Docker image are [Tar](https://en.wikipedia.org/wiki/Tar_(computing)) archives,
and are typically compressed with
[gzip](https://tools.ietf.org/html/rfc1952) or [zstd](https://tools.ietf.org/html/rfc8478).
Reconstructing the bytes of a compressed layer is generally impossible,
because it depends on exact choices made during compression,
however, this is luckily not necessary -
a container image also contains checksums of the uncompressed layers -
the [DiffID](https://github.com/opencontainers/image-spec/blob/master/config.md#layer-diffid).

[tar-diff](https://github.com/containers/tar-diff/)
([spec](https://github.com/containers/tar-diff/blob/main/file-format.md))
is a file format for describing a delta from one uncompressed tar file to another uncompressed file.
The reconstructed tar file is checksummed to compare with the DiffID in the source tar file,
and unpacked into the local filesystem for use.

The delta compression in tar-diff is based on [bsdiff](http://www.daemonology.net/bsdiff/)
and produces good results for executable binaries
without requiring any architecture-specific knowledge.

## Locating the delta manifest

One way to use an delta manifest is to store it as an
[OCI Artifact](https://github.com/opencontainers/artifacts) in an OCI registry.

All delta manifests for a repository are referenced by a single
[OCI image index](https://github.com/opencontainers/image-spec/blob/master/image-index.md).
Each manifest in this index has the MIME type `vnd.redhat.delta.manifest.v1+json`
and has the annotation `io.github.containers.delta.target`,
which is the digest of the image manifest that it corresponds to.
This image index is tagged as `_deltaindex`.

There may be alternative ways to locate an image delta manifest not specified here.
If the manifest is not stored in a registry,
then the optional `urls` property of the layer descriptor
can be used to point to the delta contents.

## Delta Manifest

The delta manifest is an
[OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/master/manifest.md).
The configuration object must have the MIME type `application/vnd.redhat.delta.config.v1+json`,
and must be an empty JSON object ('{}').

The currently defined layer MIME type is:

* `application/vnd.tar-diff`: a [tar-diff](https://github.com/containers/tar-diff/)

Each layer must have the following annotations:

* `io.github.containers.delta.from`:
  REQUIRED digest of a layer that this delta describes a change *from*
* `io.github.containers.delta.to`:
  REQUIRED digest of a layer in the target manifest that this delta describes a change *to*

Additionally the toplevel annotations for the delta manifest has thest annotations:

* `io.github.containers.delta.target`:
  REQUIRED digest of the image manifest that the delta manifest targets
(same as recorded in the delta index)

## Example Delta Manifest

``` json
{
  "schemaVersion": 1,
  "config": {
    "mediaType": "application/vnd.redhat.delta.config.v1+json",
    "size": 2,
    "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a"
  },
  "annotations": {
        "io.github.containers.delta.target": "sha256:<image_manifest_hash>",
  },
  "layers": [
    {
      "mediaType": "application/vnd.tar-diff",
      "size": 12345,
      "digest": "sha256:<hash>",
      "urls": ["https://deltas.example.com/sha256/<hash>"]
      "annotations": {
        "io.github.containers.delta.from": "sha256:<source_hash1>",
        "io.github.containers.delta.to": "sha256:<target_hash1>"
      }
    },
    {
      "mediaType": "application/vnd.tar-diff",
      "size": 12345,
      "digest": "sha256:<hash>",
      "annotations": {
        "io.github.containers.delta.from": "sha256:<source_hash2>",
        "io.github.containers.delta.to": "sha256:<target_hash1>"
      }
    },
    {
      "mediaType": "application/vnd.tar-diff",
      "size": 12345,
      "digest": "sha256:<hash>",
      "annotations": {
        "io.github.containers.delta.from": "sha256:<source_hash3>",
        "io.github.containers.delta.to": "sha256:<target_hash2>"
      }
    }
  ]
}
```

## Reconstructing a layer

For each layer in the target image manifest,
the client should examine each layer in the delta manifest, and find layers:

* That have a delta MIME type that the client understands
* Where the `io.github.containers.delta.to` annotation matches the target layer's
* Where the `io.github.containers.delta.from` annotation
  matches the a layer whose content is available locally

If a client finds multiple such layers,
it should generally pick the *smallest* layer based on the `size` property.

Then the client downloads the delta layer,
and applies the the diff to the source layer content,
producing a new uncompressed layer.
While uncompressing the content the client MUST compute the DiffID for the layer,
and check that it matches the DiffID for this layer
found in the target image's configuration object,
unless the client has some other way to verify the contents of the layer. (1)

If no matching delta layers are found, or reconstruction fails,
the client downloads the layer from the target image manifest.

## Notes

**(1)**: In some cases, the client might be able to verify the reconstructed unpacked
without computing the DiffID.
For example, if an [ostree](https://ostree.readthedocs.io/en/latest/)
is converted into a container image,
then storing the `dirtree` and `dirmeta` checksums in annotations in the image manifest
would provide an alternate method of verifying the reconstructed layer.
In such cases, the DiffID computation can be skipped.

## Locating the delta manifest (old version)

Given an image manifest with digest `<algo>`:`<hash>`,
the delta manifest will be tagged in the same repository as the manifest as `deltas-<hash[0:12]>` -
e.g., the delta manifest for the image manifest
`sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7`
is tagged as `deltas-b5b2b2c507a0`.

*Rationale*: using the full digest would result in unmanageably long tags.
Generally, an attacker will not have the ability to inject random tags into a repository.
If they did, this will be caught at the layer verification step.
So, it's just necessary to make collisions unlikely.
If a repository had 10,000 manifests of interest to download,
12 hex digits gives a chance of accidental collision of 1 in 56 million.
