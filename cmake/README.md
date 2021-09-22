# Cmake CI

The Cmake CI is set up to support RADIUSS projects that use cmake. By default, we:

 - get the source into the image
 - run cmake with our options from the dictionary
 - run make (or cmake --build .)
 - run ctest

## Usage

The GitHub action can be used as follows:

```yaml
on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    name: Test Cmake Container
    steps:
      - uses: actions/checkout@v2
      - id: tester
        uses: rse-radiuss/ci/cmake@main
        with:
          # relative path when cd to cmake
          dockerfile: Dockerfile
          context: .
          workdir: cmake

      - run: echo ${{ steps.tester.outputs.return_code }}
        shell: bash
```
