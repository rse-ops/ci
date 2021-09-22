ARG containerbase=ghcr.io/rse-radiuss/gcc-ubuntu-20.04
FROM ${containerbase}
ARG flags=""
ENV extra_flags=${flags}
FROM ${containerbase} AS gcc
ENV GTEST_COLOR=1
RUN git clone https://github.com/LLNL/Umpire /code
WORKDIR /code
RUN git submodule init && git submodule update
#COPY . /code
RUN mkdir build && cd build && cmake -DUMPIRE_ENABLE_C=On -DENABLE_COVERAGE=On -DCMAKE_BUILD_TYPE=Debug -DUMPIRE_ENABLE_DEVELOPER_DEFAULTS=On -DCMAKE_CXX_COMPILER=g++ ${extra_flags}  ..
RUN cd build && make -j 16
RUN cd build && ctest -T test --output-on-failure
