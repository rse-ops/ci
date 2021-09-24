cmake_minimum_required(3.14)

project(rse-radiuss-cmake-ci)

option(ENABLE_TESTS "Enable tests" Off)

enable_testing()

add_executable(test.exe test.cpp)

if (ENABLE_TESTS)
  add_test(NAME test.exe COMMAND test.exe)
endif ()