---
title: Uptodate Docker
layout: docs
sidebar: ci
permalink: /uptodate/
---

# Uptodate Docker

If you have a matrix of Docker builds to do, or if you are already using [uptodate](https://github.com/vsoch/uptodate) to generate a matrix
of builds, we have provided here a more streamlined way to generate a matrix
of builds and then build from it.

## 1. Overview of Steps

For this short tutorial, we will be creating a GitHub action that builds an ubuntu
container across a matrix of versions. You should [read the uptodate](https://vsoch.github.io/uptodate)
docs for more complex examples. These steps include:

1. Generating a Dockerfile with one or more build arguments, "args"
2. Creating an uptodate.yaml file that maps out the specific versions in your matrix
3. Adding the workflow with the action here!

And you can see the complete [Example Workflow](https://github.com/rse-ops/ci/blob/main/uptodate/examples/uptodate-matrix-build.yaml) here
along with the [uptodate.yaml](https://github.com/rse-ops/ci/blob/main/uptodate/uptodate.yaml) and [Dockerfile](https://github.com/rse-ops/ci/blob/main/uptodate/Dockerfile) that are used to populate the build matrix!


## 2. Create a Dockerfile

You'll first want to create a Dockerfile. Typically this means something like the following:

```
ARG ubuntu_version=22.04
FROM ubuntu:${ubuntu_version}

RUN apt-get update && apt-get install -y cowsay
ENTRYPOINT ["cowsay"]
```

For the above, we have represented versions and anything we might want to treat as a variable as build arguments.
As an example, the first build argument, `ubuntu_version` is how we will choose a base image. You can add other build
arguments to further custom the logic of your Dockerfile.

## 3. uptodate.yaml

The uptodate.yaml is going to populate your build matrix. An example is provided below
for our ubuntu container. 

```yaml
dockerbuild:

  build_args:
    ubuntu_version:
      key: ubuntu
      versions:
       - "18.04"
       - "20.04"
       - "22.04"
```

In the above, the build arg `ubuntu_version` will be populated with each of the listed
versions. The key `ubuntu` will be used to populate the container name.
You can then follow the example provided for your workflow. The name of the container
will match to the relative path of where you run your build. For example:

```console
# Root is here, in the action the inputs-> root variable below
docker/

   # This will be <registry>/<repo>/ubuntu:<tag>
   ubuntu/Dockerfile
```
And the tags are derived from the various variables in your matrix. E.g., for ubuntu:18.04 in this repository 
using the uptodate.yaml above and recipe below, you'd get `ghcr.io/rse-ops/uptodate:ubuntu-18.04`.

## 4. Create GitHub Action

Once you have your Dockerfile and uptodate.yaml, you can add your workflow! Notice that our root working directory
is uptodate.

```yaml
name: Container Build Matrices

on: 
  pull_request: []
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  generate:
    name: Generate Build Matrix
    runs-on: ubuntu-latest
    outputs:
      matrix: {% raw %}${{ steps.matrix.outputs.matrix }}{% endraw %}
      is_empty: {% raw %}${{ steps.matrix.outputs.is_empty }}{% endraw %}
    steps:
    - uses: rse-ops/ci/uptodate/matrix@main
      id: matrix
      with:
        root: uptodate
        
  build:
    needs:
      - generate
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        result: {% raw %}${{ fromJson(needs.generate.outputs.matrix) }}{% endraw %}

    if: {% raw %}${{ needs.generate.outputs.is_empty == 'false' }}{% endraw %}
    name: "Build {% raw %}${{ matrix.result.container_name }}"{% endraw %}
    steps:
    - uses: rse-ops/ci/uptodate/build@main
      with:
        repo: rse-ops
        registry: ghcr.io
        deploy: "${{ github.event_name != 'pull_request' }}"
        token: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}
        container_name: {% raw %}${{ matrix.result.container_name }}{% endraw %}
        command_prefix: {% raw %}${{ matrix.result.command_prefix }}{% endraw %}
        dockerfile: {% raw %}${{ matrix.result.filename }}{% endraw %}
```


## 5. Full Example

For the full example, see the [uptodate.yaml](https://github.com/rse-ops/ci/blob/main/uptodate/uptodate.yaml) and matching [Dockerfile](https://github.com/rse-ops/ci/blob/main/uptodate/Dockerfile)
in this directory, and the matching [test-uptodate.yaml](https://github.com/rse-ops/ci/blob/main/.github/workflows/test-uptodate.yaml) that uses it.
