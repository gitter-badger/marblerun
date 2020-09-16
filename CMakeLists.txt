# Copyright (c) Edgeless Systems GmbH.
# Licensed under the MIT License.

cmake_minimum_required(VERSION 3.11)
find_package(OpenEnclave CONFIG REQUIRED)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS OFF)

if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug)
endif ()

add_custom_target(coordinatorlib
  ertgo build -buildmode=c-archive
  -o libcoordinator.a
  ${CMAKE_SOURCE_DIR}/cmd/coordinator
)

add_custom_target(coordinator-noenclave ALL
  go build ${CMAKE_SOURCE_DIR}/cmd/coordinator-noenclave)

# Use the edger8r to generate C bindings from the EDL file.
add_custom_command(
  OUTPUT emain_t.c emain_u.c
  DEPENDS enclave/emain.edl
  COMMAND openenclave::oeedger8r ${CMAKE_SOURCE_DIR}/enclave/emain.edl --search-path
          ${OE_INCLUDEDIR} --search-path ${OE_INCLUDEDIR}/openenclave/edl/sgx)

add_library(emain enclave/emain.cpp emain_t.c)
target_include_directories(emain PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(emain PRIVATE openenclave::oe_includes)

add_executable(enclave enclave/emain.cpp emain_t.c)
target_include_directories(enclave PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
add_dependencies(enclave coordinatorlib)

target_link_libraries(enclave
  emain
  openenclave::oeenclave
  openenclave::oehostepoll
  openenclave::oehostfs
  openenclave::oehostsock
  ${CMAKE_BINARY_DIR}/libcoordinator.a
  crypto
  openenclave::ertlibc
  openenclave::oecustomfs
 )

# Generate key
add_custom_command(
  OUTPUT private.pem public.pem
  COMMAND openssl genrsa -out private.pem -3 3072
  COMMAND openssl rsa -in private.pem -pubout -out public.pem)

# Sign enclave
add_custom_command(
  OUTPUT enclave.signed
  DEPENDS enclave enclave/enclave.conf private.pem
  COMMAND openenclave::oesign sign -e $<TARGET_FILE:enclave> -c
          ${CMAKE_SOURCE_DIR}/enclave/enclave.conf -k private.pem)

add_custom_target(sign ALL DEPENDS enclave.signed)

# Build the host
add_executable(coordinator enclave/host.cpp emain_u.c)
target_include_directories(coordinator PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(coordinator openenclave::oehost)