---
title: "Docker Layers Immutability"
date: 2025-09-19T22:18:21+01:00
author: "Aymane BOUMAAZA"
draft: false
description: "docker-layers-immutability"
tags:
 - TIL
 - docker
---

Today I learned why removing files from a docker image doesnt always reduce image size (at least the way I was doing it).

**TL;DR: Immutability.**

Commands like `COPY`, `RUN`, and `ADD` in a `Dockerfile` define the creation of a new [image layer](https://docs.docker.com/get-started/docker-concepts/building-images/understanding-image-layers/). These layers are immutable, meaning that changes in filesystem done in a layer cannot be changed, they become "frozen". When trying to do a "change" in a previous layer in a `Dockerfile`, a new layer gets created on top of the already existing ones. When building the image, layers get stacked on top of each other and form a new and unified view, **the docker image** (which is immutable, too).

Docker uses a [union filesystem](https://en.wikipedia.org/wiki/UnionFS) in images to manage this layered structure, which copies changes from each layer (lowerdir) and merges them in a final layer (upperdir, or the final image).

{{< figure src="/img/docker_layers.png" align="center" alt="Docker layers union filesystem." caption="Docker layers union filesystem" border="#f8f4f0" >}}

Docker uses the [OverlayFS](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/) union filesystem by default (via the `overlay2` storage driver), in this filesystem, deletions do not really delete files and directories, but a special "whiteout" file is created to indicate that the file or directory is -marked as- deleted (remember, layers are immutable). Therefore my attemps to remove uneeded files in later layers did not have any effect on the image size -at least a positive one-.

Here's how I've been trying to remove files from my docker images:

```dockerfile
FROM alpine

# Download a big file into the image
RUN wget https://ash-speed.hetzner.com/100MB.bin -O /app/my_very_large_file

# Do something with the file
RUN do-something-with-my-very-large-file

# Remove it in a later layer
RUN rm /app/my_very_large_file

CMD ["my", "command"]
```

The `COPY` command creates a layer where `/my_very_large_file` is added. That layer is then **immutable ("frozen")** and kept in the image history.

When the `RUN rm` executes, it creates a new layer on top of the previous one where the file is **marked as deleted**. However, the original layer with the file still exists in the final image.

{{< figure src="/img/docker_build_rm_before.png" align="center" alt="Docker image still large after removing file, and file does not exist in the container" caption="Docker image still large after removing file, and file does not exist in the container" border="#f8f4f0" >}}

This means the file continues to consume space, even though you can't see it inside the container. That's why removing large files in later layers doesn't reduce the final image size.

A common example of trying to free space in a docker image is to remove the apt cache in the following way:

```dockerfile
FROM ubuntu:jammy

# Update the apt cache
RUN apt-get update

# Install a package
RUN apt-get install -y --no-install-recommends some-package

# Remove the apt cache
RUN rm -rf /var/lib/apt/lists/*
```

This will not reduce the image size, the apt cache is still in the image, but will not be visible in the container.

The right way to remove files from a docker image is to do it in the same layer where the file is added.

```dockerfile
FROM ubuntu:jammy

# Update the apt cache
RUN apt-get update

# Install a package and remove the apt cache in the same layer
RUN apt-get install -y --no-install-recommends some-package \
    && rm -rf /var/lib/apt/lists/*
```