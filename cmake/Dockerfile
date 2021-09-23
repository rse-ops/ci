ARG containerbase=ghcr.io/rse-radiuss/gcc-ubuntu-20.04
FROM ${containerbase}

ARG flags=""
ENV extra_flags=${flags}

ENV GTEST_COLOR=1
RUN git clone https://github.com/LLNL/Umpire /code

WORKDIR /code
RUN git submodule init && git submodule update
RUN cmake -B build -S . -DUMPIRE_ENABLE_C=On -DENABLE_COVERAGE=On -DCMAKE_BUILD_TYPE=Debug -DUMPIRE_ENABLE_DEVELOPER_DEFAULTS=On ${extra_flags}
RUN cmake --build build
RUN cd build && ctest -T test --output-on-failure
