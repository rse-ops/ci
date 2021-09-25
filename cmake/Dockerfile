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
