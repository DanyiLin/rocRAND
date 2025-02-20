# MIT License
#
# Copyright (c) 2022 Advanced Micro Devices, Inc. All rights reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

# Tests

set(INSTALL_TEST_FILE "${CMAKE_CURRENT_BINARY_DIR}/install_CTestTestfile.cmake")
file(WRITE "${INSTALL_TEST_FILE}"
[=[
# This is a test file generated by rocRAND for install time.
# It differs slightly from the default testfile, and you may encounter issues because of that.
]=]
)

function(add_relative_test test_name test_target)
    get_target_property(EXE_PATH ${test_target} RUNTIME_OUTPUT_DIRECTORY)
    if(EXE_PATH STREQUAL "EXE_PATH-NOTFOUND")
        set(EXE_PATH ".")
    endif()
    get_filename_component(EXE_PATH "${EXE_PATH}" ABSOLUTE BASE_DIR "${CMAKE_CURRENT_BINARY_DIR}")
    get_target_property(EXE_NAME ${test_target} RUNTIME_OUTPUT_NAME)
    if(EXE_NAME STREQUAL "EXE_NAME-NOTFOUND")
        get_target_property(EXE_NAME ${test_target} OUTPUT_NAME)
        if(EXE_NAME STREQUAL "EXE_NAME-NOTFOUND")
            set(EXE_NAME "${test_target}")
        endif()
    endif()
    file(RELATIVE_PATH rel_path "${CMAKE_CURRENT_BINARY_DIR}" "${EXE_PATH}/${EXE_NAME}")
    add_test(NAME "${test_name}" COMMAND "./${rel_path}")
    file(APPEND "${INSTALL_TEST_FILE}" "add_test(${test_name} \"../${EXE_NAME}\")\n")
endfunction()

# Get rocRAND tests source files
file(GLOB rocRAND_TEST_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

# Build rocRAND tests
foreach(test_src ${rocRAND_TEST_SRCS})
  get_filename_component(test_name ${test_src} NAME_WE)
  # MT19937 and Threefry4x64 need to be fixed on HIP-CPU
  set(DISABLED_TESTS)
  list(APPEND DISABLED_TESTS "test_rocrand_mt19937_prng")
  list(APPEND DISABLED_TESTS "test_rocrand_kernel_threefry4x64_20")
  list(APPEND DISABLED_TESTS "test_rocrand_threefry4x64_20_prng")
  if(NOT USE_HIP_CPU OR NOT ${test_name} IN_LIST DISABLED_TESTS)
    add_executable(${test_name} ${test_src})
    # nvcc/CUDA
    if(HIP_COMPILER STREQUAL "nvcc")
      set_source_files_properties(${test_name}
        PROPERTIES
          LANGUAGE CUDA
          CUDA_STANDARD 14
      )
      set(CUDA_HOST_COMPILER ${CMAKE_CXX_COMPILER})
    endif()
    target_include_directories(${test_name}
        PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/library/src>
    )
    if(TARGET GTest::GTest)
        target_link_libraries(${test_name}
            GTest::GTest
            GTest::Main
        )
    else()
        target_link_libraries(${test_name}
            GTest::gtest
            GTest::gtest_main
        )
    endif()
    target_link_libraries(${test_name}
        rocrand
    )
    if(USE_HIP_CPU)
        target_link_libraries(${test_name}
            Threads::Threads
            hip_cpu_rt::hip_cpu_rt
        )
        if(STL_DEPENDS_ON_TBB)
            target_link_libraries(${test_name} TBB::tbb)
        endif()
    else()
        if(HIP_COMPILER STREQUAL "hcc" OR HIP_COMPILER STREQUAL "clang")
            target_link_libraries(${test_name} hip::device)
        endif()
    endif()
    set_target_properties(
        ${test_name}
        PROPERTIES
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/test"
    )
    add_relative_test(${test_name} ${test_name})
    rocm_install(TARGETS ${test_name} COMPONENT tests)
    if (WIN32 AND NOT DEFINED DLLS_COPIED_TESTS)
      set(DLLS_COPIED_TESTS "YES")
      set(DLLS_COPIED_TESTS ${DLLS_COPIED_TESTS} PARENT_SCOPE)
      # for now adding in all .dll as dependency chain is not cmake based on win32
      file( GLOB third_party_dlls
      LIST_DIRECTORIES ON
      CONFIGURE_DEPENDS
      ${HIP_DIR}/bin/*.dll
      ${CMAKE_SOURCE_DIR}/rtest.*
      )
      foreach( file_i ${third_party_dlls})
        add_custom_command( TARGET ${test_name} POST_BUILD COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different ${file_i} ${PROJECT_BINARY_DIR}/test )
      endforeach( file_i )
    endif()
  endif()
endforeach()

# Fortran Wrapper Tests
if(BUILD_FORTRAN_WRAPPER)
    add_subdirectory(fortran)
endif()

# Checks for simple linkage problems
add_subdirectory(linkage)

rocm_install(
    FILES "${INSTALL_TEST_FILE}"
    DESTINATION "${CMAKE_INSTALL_BINDIR}/${PROJECT_NAME}"
    COMPONENT tests
    RENAME "CTestTestfile.cmake"
)
