# Registry Index Protocol and File Format

## Requests

Two endpoints are defined: `/index/static` and `/index/dynamic`.
They take the same query parameters and return the same results (except possibly for HTTP headers),
but the difference encodes a hint as to the intent of the client.

`/index/static`: Requests to this endpoint are expected to be repeated with exactly
the same parameters,
possibly by many different clients. It is useful for caches to cache the data.

`/index/dynamic`: Requests to this endpoint are constructed via user interaction
and are expected to be rarely repeated.
The server MAY choose to set headers such as `Cache-Control: no-store` on the response,
but it may also use to the hint to influence internal caching,
or may entirely ignore the hint.

The following query parameters are understood:

* `repository=<value>`: limit results to images in the given repository
* `tag=<value>`: limit results to images with a given tag,
  and  images within image lists with the given tag
* `os=<value>`: limit results to images for the given operating system.
  Values are as for [`GOOS`](https://golang.org/doc/install/source#environment).
* `architecture=<value>`: limit results to images for the given architecture.
  Values are as for [`GOARCH`](https://golang.org/doc/install/source#environment).
* `annotation:<annotation>=<value>`, `annotation:<annotation>:exists=1`:
  Limit results to images where the annotation `<annotation>` has the given value,
  or for the `:exists` form, exists with any value.
* `label:<label>=<value>`, `label:<label>:exists=1`:
  Limit results to images where the label `<label>` has the given value,
  or for the `:exists` form, exists with any value.

For parameters with `:exists` suffix,
it is undefined what happens if any value other than `1` is specified.

Multiple parameters with the same key are OR'ed together:
An image&nbsp;*I* matches a given set of query parameters *S* iff: For each *K*=*V*
in *S*: either *K*=*V* matches S or there is another *K*=*V&prime;* in *S*
with the same *K* and *K*=*V&prime;* matches *S*.

*Example*: `label:org.example.a:exists=1&label:org.example.b=X&label:org.example.b=Y`
matches images which have any `org.example.a` label,
and an `org.example.b` label with the value `X` or `Y`.

For requests to `/static`,
Clients SHOULD sort their request parameters by the URL-encoded value of the `<key>=<value>` string.
Python example:

``` python
params = [
    ('label', 'latest'),
    ('annotation:org.flatpak.metadata:exists', '1'),
    ('architecture': 'amd64')
]
quoted = [ urllib.quote_plus(k) + '=' + urllib.quote_plus(v) for k, v in params ]
url = index_server + '/index/static?' + '&'.join(sorted(quoted))
```

## Responses

The basic structure of the JSON returned from a request is:

```json
{
    "Registry": <url>,
    "Results": [ <array of repositories> ]
}
```

* **`Registry`**: URL to the registry where the images are found.
  This is interpreted with the base URL being the URL of the document
  (that is `/index/[static/dynamic]`.)
* **`Results`**: matching images, grouped by repository

### Repository

```json
{
    "Name": "some/repository",
    "Images": [ <array of images> ]
    "Lists": [ <array of image lists> ]
}
```

* **`Name`**: name of the repository containing the given image and image lists.
  (An image list is a Docker Manifest List or an OCI Image Index.)
* **`Images`**: images directly tagged in this repository that matched
* **`Image Lists`**: images within image lists that matched, grouped by image list.

### Image

```json
{
    "Tags": ["<tag>", "<tag>"],
    "Digest": "<digest>",
    "MediaType": "<media type>",
    "OS": "<os>",
    "Architecture": "<architecture>",
    "Annotations": {
        "org.example.annotations.x": "<value>"
    }
    "Labels": {
        "com.redhat.component": "<value>"
    }
}
```

* **`Tags`**: List of all tags applied to the image.
  This appears only for images appearing in the `Images` list of a repository,
  and not for images in an image List.
  All tags applied to the image SHOULD be returned,
  even if a `tag` parameter in the query limits the images returned by tag.
* **`Digest`**: The digest of the image manifest.
  The manifest can be retrieved be retrieved from the relative url
  `<registry>/v2/<name>/manifests/<digest>`.
* **`MediaType`**: `application/vnd.oci.image.manifest.v1+json` or
  `application/vnd.docker.distribution.manifest.v2+json`
* **`OS`**: The operating system that this image is for.
  Values are as for [`GOOS`](https://golang.org/doc/install/source#environment).
* **`Architecture`**: The architecture this image is for.
  Values are as for [`GOARCH`](https://golang.org/doc/install/source#environment).
* **`Annotations`**: Annotations applied to the image.
* **`Labels`**: Labels applied to the image

### Image List

```json
{
    "Tags": ["<tag>", "<tag>"],
    "Digest": "<digest>",
    "MediaType": "<media type>",
    "Images": [
        { <array of images, no tags field> },
    ]
}
```

* **`Tags`**: List of all tags applied to the image list.
  All tags applied to the image list SHOULD be returned,
  not just the tags matching a `tag` parameter in the query.
* **`Digest`**: The digest of the Docker manifest list or OCI image index.
  The contents can be retrieved from the relative url `<registry>/v2/<name>/manifests/<digest>`.
* **`MediaType`**: `application/vnd.docker.distribution.manifest.list.v2+json` or
  `application/vnd.oci.image.index.v1+json`
* **`Images`**: *Matching* images within the image list.
  For example, if an image list has images for the `i386`, `amd64`, and `arm64` architectures
  and `architecture=amd64` is present in the query, only the amd64 image will be included.

## Notes

* For images within an image list, the architecture matched by `architecture=` queries and
  returned in the JSON result is the architecture extracted from the images `config.json`,
  not the architecture in the manifest list or image index.
