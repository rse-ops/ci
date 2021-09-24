# Cmake CI

This is an example for how to set up a GitHub Action to test your Cmake project.

## 1. Overview of Steps

For this short tutorial, we will be doing the following steps to create a Dockerfile
and associated GitHub action:

 - getting the source into the Docker image
 - run cmake with our options from the dictionary
 - run make (or cmake --build .)
 - run ctest

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

The first argument, `containerbase` is an example of choosing a base image. Since it's an argument,
we will be able to populate it using different base images. For the default value (`ghcr.io/rse-radiuss/gcc-ubuntu-20.04:gcc-11.2.0`) you should choose the one that you want the Dockerfile to build if no build argument
is provided. This is true for any build argument (e.g., also flags).

## 3. Choose Base Images

You can choose one or more base images from the [rse-radiuss](https://rse-radiuss.github.io/docker-images/)
library. Each comes with spack pre-installed, along with a compiler/version of your choice.

## 4. Create GitHub Action

There are two possible ways to create the GitHub action, and it depends on if you
want to create all permutations of a matrix (you can use native GitHub actions) or you
want a 1:1 mapping of a matrix (e.g., in the matrix everything at index 1 builds with the
other arguments at index 1, and all build args must be the same lenth. We will discuss and show both
options here.

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
    name: "${{ matrix.containerbase }} ${{ matrix.flags }}"
    steps:
    - uses: actions/checkout@v2
    - name: Build and run testing container
      run: |

         # Since we want to build the Dockerfile in cmake, we chdir there first
         cd cmake/

         # It's useful to print the command first
         command="docker build --build-arg containerbase=ghcr.io/rse-radiuss/${{ matrix.containerbase }} --build-arg flags=${{ matrix.flags }} -t cmake-testing-container ."
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

This GitHub actions workflow is slightly longer because we need a first job to generate the matrix for us, and to check if it's empty, etc.

```yaml
on: [pull_request]

jobs:
  generate:
    name: Generate Build Matrix
    runs-on: ubuntu-latest
    outputs:
      dockerbuild_matrix: ${{ steps.dockerbuild.outputs.dockerbuild_matrix }}
      empty_matrix: ${{ steps.dockerbuild.outputs.dockerbuild_matrix_empty }}

    steps:
    - uses: actions/checkout@v2
    - name: Generate Build Matrix
      uses: vsoch/uptodate@main
      id: dockerbuild
      with: 

        # Where your Dockerfile files
        root: ./cmake

        # Build all matrix builds, and not looking for only changes or updates
        flags: "--all"
        parser: dockerbuild

    - name: View and Check Build Matrix Result
      env:
        result: ${{ steps.dockerbuild.outputs.dockerbuild_matrix }}
      run: |
        echo ${result}

  test:
    needs:
      - generate
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        result: ${{ fromJson(needs.generate.outputs.dockerbuild_matrix) }}
    if: ${{ needs.generate.outputs.empty_matrix == 'false' }}

    name: "Build ${{ matrix.result.description }}"
    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: "Build ${{ matrix.result.description }}"
      id: builder
      env:
        container: ${{ matrix.result.container_name }}
        prefix: ${{ matrix.result.command_prefix }}
        filename: ${{ matrix.result.filename }}
      run: |
        basedir=$(dirname $filename)
        printf "Base directory is ${basedir}\n"
        # Get relative path to PWD and generate dashed name from it
        cd $basedir
        echo "${prefix} -t ${container} ."
        ${prefix} -t ${container} .
```

For either approach above, you can add this file (name it something appropriate like `container-test.yaml`) to .github/workflows in your repository, and it will trigger and run the tests in parallel. It's that easy!

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

For an example, see the [uptodate.yaml](uptodate.yaml) and matching [Dockerfile](Dockerfile)
in this directory, and the matching [test-cmake.yaml](../.github/workflows/test-cmake.yaml)
