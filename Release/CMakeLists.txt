set(CMAKE_LEGACY_CYGWIN_WIN32 0)
cmake_minimum_required(VERSION 2.6)
project(casablanca)

enable_testing()

set(WARNINGS)
set(ANDROID_STL_FLAGS)

# Platform (not compiler) specific settings
if(UNIX) # This includes OSX
  find_package(Boost REQUIRED COMPONENTS random chrono system thread locale regex filesystem)
  find_package(Threads REQUIRED)
  if(APPLE AND NOT OPENSSL_ROOT_DIR)
    # Prefer a homebrew version of OpenSSL over the one in /usr/lib
    file(GLOB OPENSSL_ROOT_DIR /usr/local/Cellar/openssl/*)
    # Prefer the latest (make the latest one first)
    list(REVERSE OPENSSL_ROOT_DIR)
  endif()
  # This should prevent linking against the system provided 0.9.8y
  find_package(OpenSSL 1.0.0 REQUIRED)

  option(BUILD_SHARED_LIBS "Build shared Libraries." OFF)
  option(BUILD_TESTS "Build tests." OFF)
  option(BUILD_SAMPLES "Build samples." OFF)
  option(CASA_INSTALL_HEADERS "Install header files." ON)
  if(CASA_INSTALL_HEADERS)
    file(GLOB CASA_HEADERS_CPPREST include/cpprest/*.hpp include/cpprest/*.h include/cpprest/*.dat)
    install(FILES ${CASA_HEADERS_CPPREST} DESTINATION include/cpprest)
    file(GLOB CASA_HEADERS_PPLX include/pplx/*.hpp include/pplx/*.h)
    install(FILES ${CASA_HEADERS_PPLX} DESTINATION include/pplx)
    file(GLOB CASA_HEADERS_DETAILS include/cpprest/details/*.hpp include/cpprest/details/*.h)
    install(FILES ${CASA_HEADERS_DETAILS} DESTINATION include/cpprest/details)
  endif()
endif()

# Compiler (not platform) specific settings
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
  message("-- Setting gcc options")

  set(WARNINGS "-Wall -Wextra -Wunused-parameter -Wcast-align -Wcast-qual -Wconversion -Wformat=2 -Winit-self -Winvalid-pch -Wmissing-format-attribute -Wmissing-include-dirs -Wpacked -Wredundant-decls -Wunreachable-code")
  set(LINUX_SUPPRESSIONS "-Wno-deprecated -Wno-unknown-pragmas -Wno-reorder -Wno-unused-function -Wno-char-subscripts -Wno-switch -Wno-unused-but-set-parameter -Wno-unused-value -Wno-unused-local-typedefs")

  set(WARNINGS "${WARNINGS} ${LINUX_SUPPRESSIONS}")
  set(LD_FLAGS "${LD_FLAGS} -Wl,-z,defs")

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fno-strict-aliasing")
else()
  message("-- Unknown compiler, success is doubtful.")
endif()

# Reconfigure final output directory
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/Binaries)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/Binaries)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/Binaries)

# These settings can be used by the test targets
set(Casablanca_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)
set(Casablanca_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include ${Boost_INCLUDE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/libs/websocketpp)

set(Casablanca_LIBRARY ${LIB}cpprest)
set(Casablanca_LIBRARIES ${Casablanca_LIBRARY} ${Boost_LIBRARIES} ${Boost_FRAMEWORK})

# Everything in the project needs access to the casablanca include directories
include_directories(${Casablanca_INCLUDE_DIRS})

# Finally, the tests all use the same style declaration to build themselves, so we use a function
function(add_casablanca_test NAME SOURCES_VAR)
  add_library(${NAME} ${TEST_LIBRARY_TARGET_TYPE} ${${SOURCES_VAR}})
  message("-- Added test library ${NAME}")
  if (NOT TEST_LIBRARY_TARGET_TYPE STREQUAL "OBJECT")
    target_link_libraries(${NAME}
      ${LIB}httptest_utilities
      ${LIB}common_utilities
      ${LIB}unittestpp
      ${Casablanca_LIBRARIES}
      ${ANDROID_STL_FLAGS}
      )
    if (BUILD_SHARED_LIBS)
      add_test(NAME ${NAME}
        WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
        COMMAND test_runner lib${NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}
        )
    endif()
  endif()
endfunction()

add_subdirectory(src)

if(BUILD_TESTS)
  add_subdirectory(tests)
endif()

if(BUILD_SAMPLES)
  add_subdirectory(samples)
endif()
