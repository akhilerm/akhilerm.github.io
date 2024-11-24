---
title: "Understanding OCI Image Spec"
date: 2022-05-21T16:14:55+05:30
draft: false
categories:
- Tech
- Blog
tags:
- oci
- container
---

As part of understanding the [OCI image spec](https://github.com/opencontainers/image-spec), I found that a good
explanation of the image spec will help others. Explaining a few things here, which I found a little bit difficult to
grasp while reading the [spec](https://github.com/opencontainers/image-spec/blob/main/spec.md).

### Image Format

The spec defines an OCI image. This image spec is almost similar to Docker V2.2 image schema format. Image services
like containerd can pull this image, and then create a bundle (more on this as I learn about it).

First, let me explain some of the basics that is used in the image spec so that it is easier to understand once we move
on to higher level concepts.

##### MediaType
This field is used in the spec, so as to identify what an object corresponds to. For example, `"application/vnd.oci.image.manifest.v1+json"`,
is used to identify an image manifest. The complete list of mediatypes can be found [here](https://github.com/opencontainers/image-spec/blob/main/specs-go/v1/mediatype.go).

##### Digest
The field represents the digest of the content. Normally it will be a sha256 sum. It can be said as a content addressable
storage form. Let me explain what is a content addressable form, suppose the following is the content of a file
```
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1485,
      "digest": "sha256:46331d942d6350436f64e614d75725f6de3bb5c63e266e236e04389820a234c4"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 3208,
         "digest": "sha256:7050e35b49f5e348c4809f5eff915842962cb813f32062d3bbdd35c750dd7d01"
      }
   ]
}
```
and the `sha256` sum of this content is `432f982638b3aefab73cc58ab28f5c16e96fdb504e8c134fc58dff4bae8bf338`. Then this 
content will be stored in a file with name `432f982638b3aefab73cc58ab28f5c16e96fdb504e8c134fc58dff4bae8bf338`. 
```
root@ubuntuvmw:# sha256sum -b 432f982638b3aefab73cc58ab28f5c16e96fdb504e8c134fc58dff4bae8bf338 
432f982638b3aefab73cc58ab28f5c16e96fdb504e8c134fc58dff4bae8bf338 *432f982638b3aefab73cc58ab28f5c16e96fdb504e8c134fc58dff4bae8bf338
```
So we can  identify the file by its content. The file can contain image configuration or sometimes be gzip'ed files.
The full path of the file will be `blob/sha256/<sha256>`, where `blob` means this is a binary blob, `sha256` is the 
algorithm used to generate the digest and then the actual file name.
```
root@ubuntuvmw:/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256# ls -l
total 61788
-r--r--r-- 1 root root 17067249 May 19 17:53 00fcd775d5c6fe8b686bc4cc1c604548a75981a5f4972c552ddf02dd3a592b12
-r--r--r-- 1 root root     3069 May 19 17:53 08b57ba4cbdadace2ff536b8b05808100a54e844215321be541d06e6829c5f64
-r--r--r-- 1 root root     1386 May 19 17:53 0c099543f270a7dded82d2efbdb6ad543959950507228be027225ab95950f2f4
-r--r--r-- 1 root root     2562 May  9 16:56 10d7d58d5ebd2a652f4d93fdd86da8f265f5318c6a73cc5b6a9798ff6d2b2e67
-r--r--r-- 1 root root     1184 May 19 17:51 162d762e6ae2b5e4a938679d2756481d95a3f6b37d45287dbed8b301a091d3c9
-r--r--r-- 1 root root 27169393 May 19 17:51 185e8a4c100571f111d924b5d4399d89f163bf95d71ce2c6a33f656a66c52f0a
-r--r--r-- 1 root root     1363 May 10 10:04 3d674d3f3c0fce3bf2552e02564dc37e1556c0cf68d70a0cb6e143216a1eb510
-r--r--r-- 1 root root      949 May 18 11:07 3e4ac559bd32e387adb8b509b1bbc07f6fff76f78ab77f4e73c8022800044565
-r--r--r-- 1 root root      525 May  9 16:56 432f982638b3aefab73cc58ab28f5c16e96fdb504e8c134fc58dff4bae8bf338
-r--r--r-- 1 root root     1390 May 10 10:04 51f76c0d1bb7de1d4df4afc178599c001c6055f6218589257b6538f7502702be
-r--r--r-- 1 root root      740 May 17 20:08 5401e20b0410d156b3c4f2827a2b335687119cf1fb231455ff5656c2d73941cb
-r--r--r-- 1 root root     1386 May 17 20:19 55d31f3af94c797b65b310569803cacc1c9f4a34bf61afcdc8138f89345c8308
-r--r--r-- 1 root root 18964190 May 19 17:51 5652aee039c525b5a4324443551f5f137d285641af75ee005594c08e901ca0c3
-r--r--r-- 1 root root      741 May 19 17:53 602204011e2f3366bfdd441003998a8f066c81ca9b90493b135d6c2c278f1439
-r--r--r-- 1 root root      949 May 17 20:19 84921482d941f71aa8a8bd399a54d0956a81057332a67ac01e4de04c8c4e57da
-r--r--r-- 1 root root     1386 May 17 20:27 9c2b1dd085f4249cab11cd4d536fd97ecf89a0b752d79a676cff97f3c2889413
-r--r--r-- 1 root root     1386 May 18 11:07 be917c6d0bc015a245741f71966c6a0b1d08bf8b48da6387cad5e7c53c7067ae
-r--r--r-- 1 root root     3546 May 19 17:51 f3118e629e524f427f525ab7b73d78eaecc1a919edb7a0a886d7fafbbb1d6bd1
-r--r--r-- 1 root root      740 May 17 20:27 f901cb1ae59bcd7e4f8d0243037801c80736f4e00a587e670c0f8e9468e09f08
```
This is the list of all blobs in my directory, each of this can be an image manifest, or root filesystems.

More about the digest implementation in golang can be found [here](https://github.com/opencontainers/go-digest/blob/master/digest.go)

##### Descriptors
An OCI image is composed of various types of contents(index, manifest, layers) arranged in Directed Acyclic Graph(DAG). 
The references between this different types of content are described using a Content Descriptor. The main components
are:
- `mediaType`: It represents what type of content this descriptor refers to. It can be an image index, manifest or
tar archives
- `digest`: It represents the digest of the content. This digest can be used to access the content from the storage.
- `size`: The size of the content in bytes.

Once you have the above the items, you can parse any content. For example:
```
{
    "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
    "digest": "sha256:552d1f2373af9bfe12033568ebbfb0ccbb0de11279f9a415a29207e264d7f4d9",
    "size": 2711827
}
```
means that use the file named `552d1f2373af9bfe12033568ebbfb0ccbb0de11279f9a415a29207e264d7f4d9` from the blob and parse
it as a gzip tar archive


Once the above basic concepts are understood we can look how an image is organized.
![Image](/images/oci-image22052022.png)

- Image Index: Contains the various manifests for the different images. For example. `linux/amd64` and `linux/arm64` will
be separate manifests, but will be available under the same index. Depending on the platform requested, the corresponding
manifest will be loaded. Manifests will be referenced via content descriptors from an index.
- Manifest: Contains the configuration and various layers in the image. Config and Layers will be referenced via content
descriptor from the manifest
- Config: The config contains the various configuration data like: rootfs, exposed ports, environment variables.
- Layers: The actual image layers. These layers are applied one by one, on top of the rootfs to create the actual image
filesystem. The content descriptor from layers will be pointing to tar/gzip archives on the storage for the image
filesytem.


NOTE: This document should only be considered as a primer for reading the image-spec, so that you dont get confused and
wont have to jump between links when reading the image spec.
