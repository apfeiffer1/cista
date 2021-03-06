project(cista)
cmake_minimum_required(VERSION 3.5)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(cista-compile-flags
     -Wno-global-constructors
     -Wno-exit-time-destructors
     -fno-strict-aliasing
     -Weverything
     -Wno-c++98-compat
     -Wno-c++98-compat-pedantic
     -Wno-newline-eof
     -Wno-missing-prototypes
     -Wno-padded
     -Wno-double-promotion
     -Wno-undef
     -Wno-undefined-reinterpret-cast
     -Wno-float-conversion
     -Wno-return-std-move-in-c++11
     -Wno-gnu-zero-variadic-macro-arguments
     -Wno-unknown-pragmas
     -Wno-documentation-unknown-command
     -Werror
  )
endif()

option(CISTA_COVERAGE "generate coverage report" OFF)
option(CISTA_GENERATE_TO_TUPLE "generate include/cista/reflection/to_tuple.h" OFF)

add_subdirectory(tools/xxh3)
option(CISTA_XXH3 "Use experimental xxhash xxh3" FALSE)

add_library(cista INTERFACE)
if (CISTA_XXH3)
  target_link_libraries(cista INTERFACE xxh3)
  target_compile_definitions(cista INTERFACE CISTA_XXH3=1)
endif()
target_include_directories(cista INTERFACE include)
target_compile_features(cista INTERFACE cxx_std_17)

if (${CISTA_GENERATE_TO_TUPLE})
  add_subdirectory(tools/to_tuple_generator EXCLUDE_FROM_ALL)
  add_custom_target(generate_to_tuple
    COMMAND to_tuple_generator
      64  # max number of supported member fields
    > ${CMAKE_CURRENT_SOURCE_DIR}/include/cista/reflection/to_tuple.h
  )
  add_dependencies(cista generate_to_tuple)
endif()

add_subdirectory(tools/doctest EXCLUDE_FROM_ALL)

file(GLOB_RECURSE cista-include-files include/*.h*)

add_subdirectory(tools/uniter EXCLUDE_FROM_ALL)
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/cista.h
  COMMAND uniter
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/include/cista/mmap.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/cista/serialization.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/cista/reflection/comparable.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/cista/reflection/printable.h
  > ${CMAKE_CURRENT_BINARY_DIR}/cista.h
  DEPENDS ${cista-include-files}
)

file(GLOB_RECURSE cista-test-files test/*.cc)
add_executable(cista-test-single-header EXCLUDE_FROM_ALL ${cista-test-files} ${CMAKE_CURRENT_BINARY_DIR}/cista.h)
target_link_libraries(cista-test-single-header doctest)
target_include_directories(cista-test-single-header PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_compile_options(cista-test-single-header PRIVATE ${cista-compile-flags})
target_compile_definitions(cista-test-single-header PRIVATE SINGLE_HEADER)
target_compile_features(cista-test-single-header PRIVATE cxx_std_17)

add_executable(cista-test EXCLUDE_FROM_ALL ${cista-test-files})
target_compile_options(cista-test PRIVATE ${cista-compile-flags})
target_link_libraries(cista-test doctest cista)
if(CISTA_COVERAGE)
  target_compile_options(cista-test PRIVATE -fprofile-arcs -ftest-coverage)
  set_target_properties(cista-test PROPERTIES LINK_FLAGS --coverage)
endif()

add_custom_target(cista-format-check
  find
    ${CMAKE_CURRENT_SOURCE_DIR}/test
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    -type f
    (
    -name "*.cc"
    -o
    -name "*.h"
    -o
    -name "*.hpp"
    )
    -not -name "doctest.h"
    -print0
  | xargs -0 clang-format -i
  COMMAND
  git status --porcelain | xargs -I {} -0 test -z \"{}\"
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  COMMENT "Checking source code formatting"
  VERBATIM
)

add_custom_target(cista-lint
  /usr/bin/python
    ${CMAKE_CURRENT_SOURCE_DIR}/tools/run-clang-tidy.py
      -exit-on-error
      -quiet
      -clang-tidy-binary clang-tidy-7
      -header-filter "^${CMAKE_CURRENT_SOURCE_DIR}/(include/)"
      "^${CMAKE_CURRENT_SOURCE_DIR}/test/"
  COMMENT "Running clang-tidy"
  VERBATIM
)
add_dependencies(cista-lint cista-test)

add_custom_target(cista-coverage
  rm -rf *.info &&
  find . -name "*.gcda" -delete &&
  ./cista-test &&
  lcov --directory . --capture --output-file cov.info &&
  lcov -r cov.info "*usr/include/*" -o cov.info &&
  lcov -r cov.info "*doctest*" -o cov.info &&
  lcov -r cov.info "*test/*.cc" -o cov.info &&
  lcov -r cov.info "*v1*" -o cov.info
)
add_dependencies(cista-coverage cista-test)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(cista-fuzz-targets "")
  file(GLOB_RECURSE fuzz-files fuzz/*.cc)
  foreach(fuzz-file ${fuzz-files})
    get_filename_component(test-name ${fuzz-file} NAME_WE)

    add_executable(cista-fuzz-${test-name} EXCLUDE_FROM_ALL ${fuzz-file})
    target_link_libraries(cista-fuzz-${test-name} cista -fsanitize=address,undefined,fuzzer)
    target_compile_options(cista-fuzz-${test-name} PRIVATE -g -O0 -fsanitize=address,fuzzer)
    
    add_executable(cista-fuzz-${test-name}-seed EXCLUDE_FROM_ALL ${fuzz-file})
    target_link_libraries(cista-fuzz-${test-name}-seed cista)
    target_compile_definitions(cista-fuzz-${test-name}-seed PRIVATE GENERATE_SEED)

    add_custom_target(cista-fuzz-${test-name}-run
      DEPENDS
        cista-fuzz-${test-name}
        cista-fuzz-${test-name}-seed
      COMMAND
        mkdir -p fuzz-${test-name}-corpus &&
        ./cista-fuzz-${test-name}-seed ./fuzz-${test-name}-corpus/seed.bin &&
        ./cista-fuzz-${test-name} ./fuzz-${test-name}-corpus -max_total_time=120
    )

    list(APPEND cista-fuzz-targets cista-fuzz-${test-name}-run)
  endforeach()

  add_custom_target(cista-fuzz)
  add_dependencies(cista-fuzz ${cista-fuzz-targets})
endif()