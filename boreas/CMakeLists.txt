# SPDX-FileCopyrightText: 2020-2023 Greenbone AG
#
# SPDX-License-Identifier: GPL-2.0-or-later

include(FindPkgConfig)

if(NOT PKG_CONFIG_FOUND)
  message(FATAL_ERROR "pkg-config executable not found. Aborting.")
endif(NOT PKG_CONFIG_FOUND)

## Dependency checks

message(STATUS "Looking for libnet...")
find_library(NET net)
message(STATUS "Looking for net... ${NET}")
if(NOT NET)
  message(SEND_ERROR "The net library is required.")
endif(NOT NET)
message(STATUS "Looking for libnet-config...")
find_program(LIBNET_CONFIG libnet-config)

if(LIBNET_CONFIG)
  message(STATUS "Looking for libnet-config... ${LIBNET_CONFIG}")
  execute_process(
    COMMAND libnet-config --libs
    OUTPUT_VARIABLE LIBNET_LDFLAGS
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  execute_process(
    COMMAND libnet-config --cflags
    OUTPUT_VARIABLE LIBNET_CFLAGS
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
else(LIBNET_CONFIG)
  message(STATUS "libnet-config not found, using defaults...")
  set(LIBNET_LDFLAGS "-L/usr/lib -lnet")
  set(LIBNET_CFLAGS "-I/usr/include")
endif(LIBNET_CONFIG)

message(STATUS "Looking for pcap...")
find_library(PCAP pcap)
message(STATUS "Looking for pcap... ${PCAP}")
if(NOT PCAP)
  message(SEND_ERROR "The pcap library is required.")
endif(NOT PCAP)

message(STATUS "Looking for pcap-config...")
find_program(PCAP_CONFIG pcap-config)

if(PCAP_CONFIG)
  message(STATUS "Looking for pcap-config... ${PCAP_CONFIG}")
  execute_process(
    COMMAND pcap-config --libs
    OUTPUT_VARIABLE PCAP_LDFLAGS
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  string(STRIP "${PCAP_LDFLAGS}" PCAP_LDFLAGS)
  execute_process(
    COMMAND pcap-config --cflags
    OUTPUT_VARIABLE PCAP_CFLAGS
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
else(PCAP_CONFIG)
  message(STATUS "pcap-config not found, using defaults...")
  set(PCAP_LDFLAGS "-L/usr/lib -lpcap")
  set(PCAP_CFLAGS "-I/usr/include")
endif(PCAP_CONFIG)

find_package(Threads)

pkg_check_modules(GLIB REQUIRED glib-2.0>=2.42)

include_directories(${GLIB_INCLUDE_DIRS})

set(
  FILES
  alivedetection.c
  arp.c
  boreas_error.c
  boreas_io.c
  cli.c
  ping.c
  sniffer.c
  util.c
)

set(
  HEADERS
  alivedetection.h
  arp.h
  boreas_error.h
  boreas_io.h
  cli.h
  ping.h
  sniffer.h
  util.h
)

if(BUILD_STATIC)
  set(LIBGVM_BOREAS_NAME gvm_boreas_static)
  add_library(gvm_boreas_static STATIC ${FILES})
  set_target_properties(gvm_boreas_static PROPERTIES OUTPUT_NAME "gvm_boreas")
  set_target_properties(gvm_boreas_static PROPERTIES CLEAN_DIRECT_OUTPUT 1)
  set_target_properties(gvm_boreas_static PROPERTIES PUBLIC_HEADER "${HEADERS}")
endif(BUILD_STATIC)

if(BUILD_SHARED)
  set(LIBGVM_BOREAS_NAME gvm_boreas_shared)
  add_library(gvm_boreas_shared SHARED ${FILES})
  set_target_properties(gvm_boreas_shared PROPERTIES OUTPUT_NAME "gvm_boreas")
  set_target_properties(gvm_boreas_shared PROPERTIES CLEAN_DIRECT_OUTPUT 1)
  set_target_properties(
    gvm_boreas_shared
    PROPERTIES SOVERSION "${PROJECT_VERSION_MAJOR}"
  )
  set_target_properties(
    gvm_boreas_shared
    PROPERTIES VERSION "${CPACK_PACKAGE_VERSION}"
  )
  set_target_properties(gvm_boreas_shared PROPERTIES PUBLIC_HEADER "${HEADERS}")

  target_link_libraries(
    gvm_boreas_shared
    LINK_PRIVATE
      gvm_util_shared
      ${GLIB_LDFLAGS}
      ${LIBNET_LDFLAGS}
      ${PCAP_LDFLAGS}
      ${LINKER_HARDENING_FLAGS}
      ${CMAKE_THREAD_LIBS_INIT}
  )
endif(BUILD_SHARED)

set(LIBGVM_BOREAS_NAME ${LIBGVM_BOREAS_NAME} PARENT_SCOPE)

## Tests

if(BUILD_TESTS)
  add_unit_test(
    boreas-alivedetection-test
    alivedetection_tests.c
    gvm_boreas_shared
    gvm_base_shared
    gvm_util_shared
    ${GLIB_LDFLAGS}
    ${PCAP_LDFLAGS}
    ${LIBNET_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
  )
  add_unit_test(
    boreas-error-test
    boreas_error_tests.c
    ${GLIB_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
  )
  add_unit_test(
    boreas-io-test
    boreas_io_tests.c
    gvm_boreas_shared
    gvm_base_shared
    gvm_util_shared
    ${PCAP_LDFLAGS}
    ${LIBNET_LDFLAGS}
    ${GLIB_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
  )
  add_unit_test(
    boreas-cli-test
    cli_tests.c
    gvm_boreas_shared
    gvm_base_shared
    gvm_util_shared
    ${GLIB_LDFLAGS}
    ${PCAP_LDFLAGS}
    ${LIBNET_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
  )
  add_unit_test(
    boreas-ping-test
    ping_tests.c
    gvm_boreas_shared
    gvm_base_shared
    ${GLIB_LDFLAGS}
    ${PCAP_LDFLAGS}
    ${LIBNET_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
  )
  add_unit_test(
    boreas-sniffer-test
    sniffer_tests.c
    gvm_boreas_shared
    gvm_base_shared
    gvm_util_shared
    ${PCAP_LDFLAGS}
    ${LIBNET_LDFLAGS}
    ${GLIB_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
  )

  set(UTIL_TEST_LINKER_WRAP_OPTIONS "-Wl,-wrap,socket,-wrap,setsockopt")
  add_unit_test(
    util-test
    util_tests.c
    gvm_base_shared
    ${GLIB_LDFLAGS}
    ${LINKER_HARDENING_FLAGS}
    ${CMAKE_THREAD_LIBS_INIT}
    ${UTIL_TEST_LINKER_WRAP_OPTIONS}
  )
endif(BUILD_TESTS)

## Install

configure_file(libgvm_boreas.pc.in ${CMAKE_BINARY_DIR}/libgvm_boreas.pc @ONLY)

install(
  FILES ${CMAKE_BINARY_DIR}/libgvm_boreas.pc
  DESTINATION ${LIBDIR}/pkgconfig
)

if(BUILD_STATIC)
  install(
    TARGETS gvm_boreas_static
    RUNTIME DESTINATION ${BINDIR}
    ARCHIVE DESTINATION ${LIBDIR}
    PUBLIC_HEADER DESTINATION "${INCLUDEDIR}/gvm/boreas"
  )
endif(BUILD_STATIC)

if(BUILD_SHARED)
  install(
    TARGETS gvm_boreas_shared
    RUNTIME DESTINATION ${BINDIR}
    LIBRARY DESTINATION ${LIBDIR}
    ARCHIVE DESTINATION ${LIBDIR}
    PUBLIC_HEADER DESTINATION "${INCLUDEDIR}/gvm/boreas"
  )
endif(BUILD_SHARED)

## End
