ARG containerbase
ARG containertag
FROM ${containerbase}:${containertag} as base

ARG cxx_compiler="g++"
ARG enable_openmp="On"
ARG enable_cuda="Off"
ARG enable_tests="On"
ARG version

ENV cxx_compiler=${cxx_compiler}
ENV enable_openmp=${enable_openmp}
ENV enable_cuda=${enable_cuda}
ENV enable_tests=${enable_tests}

ENV GTEST_COLOR=1
RUN git clone https://github.com/LLNL/Umpire /code

WORKDIR /code
RUN git submodule init && git submodule update
RUN cmake -B build -S . -DUMPIRE_ENABLE_C=On -DENABLE_COVERAGE=On -DCMAKE_BUILD_TYPE=Debug -DUMPIRE_ENABLE_DEVELOPER_DEFAULTS=On -DCMAKE_CXX_COMPILER=${cxx_compiler} -DENABLE_OPENMP=${enable_openmp} -DENABLE_TESTS=${enable_tests}
RUN cmake --build build
RUN cd build && ctest -T test --output-on-failure
