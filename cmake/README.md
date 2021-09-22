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

The GitHub action is fairly simple, and you likely want the trigger to be on a pull request to test changes.
You can use the example below as a template, and remove comments as needed.

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

You can add this file (name it something appropriate like `container-test.yaml`) to .github/workflows
in your repository, and it will trigger and run the tests in parallel. It's that easy!
