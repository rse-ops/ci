---
title: CMake
layout: page-two-col
parent: Continuous Integration
active: Continuous Integration
permalink: /cmake/
---

# CMake CI

{:.no_toc}

This is an example for how to set up a GitHub Action to test your CMake project.

* TOC
{:toc}


## 1. Overview of Steps

For this short tutorial, we will be creating a GitHub action that builds and tests
a CMake project.  These steps generally include:

 - getting the source into the Docker image
 - run cmake with our options from the dictionary
 - run make (or cmake --build .)
 - run ctest

However the way that you create your build matrix will vary depending on your needs!
Keep reading for a detailed tutorial, or jump into the code examples provided:

 - [GitHub Actions Basic (without helper actions)](https://github.com/rse-radiuss/ci/blob/main/cmake/examples/cmake-build-test-basic.yaml)
 - [Automated Matrix Generation and Test/Build](https://github.com/rse-radiuss/ci/blob/main/cmake/examples/cmake-build-test-uptodate-full.yaml) using and [uptodate.yaml](https://github.com/rse-radiuss/ci/blob/main/cmake/uptodate.yaml) and matching [Dockerfile](https://github.com/rse-radiuss/ci/blob/main/cmake/Dockerfile)
 - [Automated Matrix Generation and Manual Test/Build](https://github.com/rse-radiuss/ci/blob/main/cmake/examples/cmake-build-test-uptodate.yaml) with same [uptodate.yaml](https://github.com/rse-radiuss/ci/blob/main/cmake/uptodate.yaml) and [Dockerfile](https://github.com/rse-radiuss/ci/blob/main/cmake/Dockerfile)

We are hoping to have a tool to make these yaml recipes easier to generate, stay tuned!

## 2. Create a Dockerfile

You'll first want to create a Dockerfile. Typically this means something like the following:

```
ARG containerbase=ghcr.io/rse-radiuss/gcc-ubuntu-20.04:gcc-11.2.0
FROM ${containerbase}

# Any extra flags you can provide as a build argument
ARG flags=""
ENV extra_flags=${flags}

FROM ${containerbase} AS gcc
ENV GTEST_COLOR=1

# Get your source code into the image
WORKDIR /code
COPY . /code

# Run make or cmake build
RUN mkdir build && cd build && cmake -DUMPIRE_ENABLE_C=On -DENABLE_COVERAGE=On -DCMAKE_BUILD_TYPE=Debug -DUMPIRE_ENABLE_DEVELOPER_DEFAULTS=On -DCMAKE_CXX_COMPILER=g++ ${extra_flags}  ..
RUN cd build && make -j 16

# Run ctest
RUN cd build && ctest -T test --output-on-failure
```

For the above, we have represented versions and anything we might want to treat as a variable as build arguments.
As an example, the first build argument, `containerbase` is how we will choose a base image. Since it's an argument,
we will be able to populate it using different base images. For the default value (`ghcr.io/rse-radiuss/gcc-ubuntu-20.04:gcc-11.2.0`) you should choose the one that you want the Dockerfile to build if no build argument
is provided. This is true for any build argument (e.g., also flags).

### Choose Base Images

Speaking of base images, you can choose one or more base images from the [rse-radiuss](https://rse-radiuss.github.io/docker-images/) library. 
Each comes with spack pre-installed, along with a compiler/version of your choice.

## 3. Create GitHub Action

There are two possible ways to create the GitHub action, and it depends on if you
want to create all permutations of a matrix (you can use a native GitHub actions matrix) or you
want a 1:1 mapping of a matrix (a column-wise matrix where everything at index 1 builds with the
other arguments at index 1, and all build args must be the same length)). We will discuss and show both options here.

### GitHub Action Matrix

The GitHub actions matrix will build all permutations of the matrix. E.g.,:

```yaml
 - letter: [A, B, C]
 - number: [1, 2, 3]
```

will build:

```
A1 A2 A3 B1 B2 B3 C1 C2 C3
```

So you can use it if this kind of combination is possible for your builds. This approach is fairly simple because no additional actions or dependencies are required. You likely want the trigger to be on a pull request to test changes.  Here is an example:

```yaml
on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:

      # Failing fast means if one job fails, we cancel the rest
      fail-fast: true

      # Here is our build matrix. We will build each entry in container with each entry in flags, for a total of 3x3=9
      matrix:
        containerbase: ["gcc-ubuntu-20.04:gcc-11.2.0", "clang-ubuntu-20.04:llvm-12.0.0", "cuda-ubuntu-20.04:cuda-11.4.0"]
        flags: ["-DENABLE_OPENMP=On", "-DENABLE_OPENMP=On", "-DENABLE_CUDA=On"]

    # This is how to reference a variable in the matrix
    name: "{% raw %}${{ matrix.containerbase }} ${{ matrix.flags }}{% endraw %}"
    steps:
    - uses: actions/checkout@v2
    - name: Build and run testing container
      run: |

         # Since we want to build the Dockerfile in cmake, we chdir there first
         cd cmake/

         # It's useful to print the command first
         command="docker build --build-arg containerbase=ghcr.io/rse-radiuss/{% raw %}${{ matrix.containerbase }}{% endraw %} --build-arg flags={% raw %}${{ matrix.flags }}{% endraw %} -t cmake-testing-container ."
         printf "${command}\n"
         ${command}
```

To reiterate what was said above, for the matrix shown above, we will generate a build for every permutation, meaning 3 container bases (`containerbase`) by 3 flags for a total of 3x3=9 builds. This only works if you can use every combination. If not, you likely want an [uptodate](https://github.com/vsoch/uptodate) matrix, shown next.


### Column-wise Matrix

Let's say that you have build args (like a container base and then compile flags) and you cannot build all combinations together. To adjust our previous example, with this matrix strategy

```yaml
 - letter: [A, B, C]
 - number: [1, 2, 3]
```

will build:

```
A1 B2 C3
```

#### uptodate.yaml

This GitHub actions workflow is slightly longer because we will first need to generate an uptodate.yaml file that defines one or more build args to be used as a matrix as shown above. For example this uptodate.yaml:

```yaml
dockerbuild:
  matrix:
    containerbase:
       - ghcr.io/rse-radiuss/gcc-ubuntu-20.04
       - ghcr.io/rse-radiuss/clang-ubuntu-20.04
    containertag:
       - gcc-8.1.0
       - llvm-10.0.0
    cxx_compiler:
       - g++
       - clang++
    enable_tests:
       - "On"
       - "On"
```

matched to this Dockerfile:

```dockerfile
ARG containerbase
ARG containertag
FROM ${containerbase}:${containertag} as base

ARG cxx_compiler="g++"
ARG enable_tests="On"
ARG version

ENV cxx_compiler=${cxx_compiler}
ENV enable_tests=${enable_tests}

ENV GTEST_COLOR=1

COPY . /code
WORKDIR /code
RUN cmake -B build -S . -DCMAKE_CXX_COMPILER=${cxx_compiler} -DENABLE_TESTS=${enable_tests}
RUN cmake --build build
RUN cd build && ctest -T test --output-on-failure
```

Will generate the following two docker builds:

```bash
docker build -f Dockerfile --build-arg cxx_compiler=g++ --build-arg enable_tests=On --build-arg containerbase=ghcr.io/rse-radiuss/gcc-ubuntu-20.04 --build-arg containertag=gcc-8.1.0 cmake
docker build -f Dockerfile --build-arg cxx_compiler=clang++ --build-arg enable_tests=On --build-arg containerbase=ghcr.io/rse-radiuss/clang-ubuntu-20.04 --build-arg containertag=llvm-10.0.0 cmake
```

You can be creative about how you break apart your build args and map them to the Dockerfile - the example above might be a bit excessive for your use case. Logical steps to generating this are:

1. Start with a Dockerfile that builds and tests your software
2. Figure out where you can use variables (typically base images, versions, or flags)
3. Write out those variables as build args. To get into your Dockerfile, the `ARG` needs to be mapped to an `ENV` (environment) variable before it can be referenced as such
4. Write down those exact pairings and names of build args in your uptodate.yaml matrix, as shown in the example. A variable named `var` will map to `ARG var`. If necessary you can set a default with `ARG var=<default>`

If you wanted to test this locally, you can build [uptodate](https://vsoch.github.io/uptodate/docs/#/user-guide/user-guide?id=install) and run:

```bash
$ uptodate dockerbuild --all ./folder
```

Where `folder` corresponds to the directory with your Dockerfile and uptodate.yaml.

#### uptodate GitHub Action

Uptodate comes with its own GitHub action to run the command above, and map it into a GitHub matrix that will
then be pushed out to multiple parallel jobs. You can see this example [here]()

```yaml
on: [pull_request]

jobs:
  generate:
    name: Generate Build Matrix
    runs-on: ubuntu-latest
    outputs:
      dockerbuild_matrix: {% raw %}${{ steps.dockerbuild.outputs.dockerbuild_matrix }}{% endraw %}
      empty_matrix: {% raw %}${{ steps.dockerbuild.outputs.dockerbuild_matrix_empty }}{% endraw %}

    steps:
    - uses: actions/checkout@v2
    - name: Generate Build Matrix
      uses: vsoch/uptodate@main
      id: dockerbuild
      with: 
        root: ./cmake    # Where your Dockerfile and uptodate.yaml live        
        flags: "--all"   # Build all matrix builds, and not looking for only changes or updates
        parser: dockerbuild

    - name: View and Check Build Matrix Result
      env:
        result: {% raw %}${{ steps.dockerbuild.outputs.dockerbuild_matrix }}{% endraw %}
      run: |
        echo ${result}

  test:
    needs:
      - generate
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        result: {% raw %}${{ fromJson(needs.generate.outputs.dockerbuild_matrix) }}{% endraw %}
    if: {% raw %}${{ needs.generate.outputs.empty_matrix == 'false' }}{% endraw %}

    name: "{% raw %}${{ matrix.result.description }}{% endraw %}"
    steps:

    - name: Build and Test
      uses: rse-radiuss/ci/cmake@main
```

If you need to customize the specific build, you can expand the action to not use the `rse-radiuss/cmake` build steps, and
instead write your own!


<details>

<summary>A Full GitHub Workflow Example</summary>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code>
on: [pull_request]

jobs:
  generate:
    name: Generate Build Matrix
    runs-on: ubuntu-latest
    outputs:
      dockerbuild_matrix: {% raw %}${{ steps.dockerbuild.outputs.dockerbuild_matrix }}{% endraw %}
      empty_matrix: {% raw %}${{ steps.dockerbuild.outputs.dockerbuild_matrix_empty }}{% endraw %}

    steps:
    - uses: actions/checkout@v2
    - name: Generate Build Matrix
      uses: vsoch/uptodate@main
      id: dockerbuild
      with: 
        root: ./cmake    # Where your Dockerfile and uptodate.yaml live        
        flags: "--all"   # Build all matrix builds, and not looking for only changes or updates
        parser: dockerbuild

    - name: View and Check Build Matrix Result
      env:
        result: {% raw %}${{ steps.dockerbuild.outputs.dockerbuild_matrix }}{% endraw %}
      run: |
        echo ${result}

  test:
    needs:
      - generate
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        result: {% raw %}${{ fromJson(needs.generate.outputs.dockerbuild_matrix) }}{% endraw %}
    if: {% raw %}${{ needs.generate.outputs.empty_matrix == 'false' }}{% endraw %}

    name: "{% raw %}${{ matrix.result.description }}{% endraw %}"
    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: "{% raw %}${{ matrix.result.description }}{% endraw %}"
      id: builder
      env:
        container: {% raw %}${{ matrix.result.container_name }}{% endraw %}
        prefix: {% raw %}${{ matrix.result.command_prefix }}{% endraw %}
        filename: {% raw %}${{ matrix.result.filename }}{% endraw %}
      run: |
        basedir=$(dirname $filename)
        printf "Base directory is ${basedir}\n"
        # Get relative path to PWD and generate dashed name from it
        cd $basedir
        echo "${prefix} -t ${container} ."
        ${prefix} -t ${container} .
</code></pre></div></div>

</details>

<br>

For either approach above, you can add this file (name it something appropriate like `container-test.yaml`) to .github/workflows in your repository, and it will trigger and run the tests in parallel. We are also working on templates to make this easier for you to do, and will update the documentation here when that is done.

## 5. Optimizations

If it's the case that your build runs out of room, you can add this step directly before building
to (usually) make enough additional room:

```yaml
    - name: Make Space For Build
      run: |
          sudo rm -rf /usr/share/dotnet
          sudo rm -rf /opt/ghc
```


## 6. Example

For an example, see the [uptodate.yaml](https://github.com/rse-radiuss/ci/blob/main/cmake/uptodate.yaml) and matching [Dockerfile](https://github.com/rse-radiuss/ci/blob/main/cmake/Dockerfile)
in this directory, and the matching [test-cmake.yaml](https://github.com/rse-radiuss/ci/blob/main/.github/workflows/test-cmake.yaml).
