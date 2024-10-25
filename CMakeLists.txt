cmake_minimum_required(VERSION 3.20)

# Set the project name and version
project(libmodbus LANGUAGES C)

# Set the C standard
set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED True)

add_compile_options(-Wno-inconsistent-dllimport -Wno-unused-parameter -Wno-deprecated-declarations)

# Extract version number from configure.ac for dynamic versioning
set(LIBMODBUS_VERSION)
set(LIBMODBUS_VERSION_MAJOR)
set(LIBMODBUS_VERSION_MINOR)
set(LIBMODBUS_VERSION_MICRO)

# Extract version information from the configure.ac file
set(version_file_path "${CMAKE_CURRENT_SOURCE_DIR}/configure.ac")
if(EXISTS "${version_file_path}")
    file(STRINGS "${version_file_path}" version REGEX "m4_define\\(\\[libmodbus_version_(major|minor|micro)\\]")
    string(REGEX REPLACE ".*_major\\], \\[([0-9]*)\\]\\).*" "\\1" major "${version}")
    string(REGEX REPLACE ".*_minor\\], \\[([0-9]*)\\]\\).*" "\\1" minor "${version}")
    string(REGEX REPLACE ".*_micro\\], \\[([0-9]*)\\]\\).*" "\\1" micro "${version}")
    if(NOT major STREQUAL "" AND NOT minor STREQUAL "" AND NOT micro STREQUAL "")
        set(LIBMODBUS_VERSION "${major}.${minor}.${micro}")
        set(LIBMODBUS_VERSION_MAJOR ${major})
        set(LIBMODBUS_VERSION_MINOR ${minor})
        set(LIBMODBUS_VERSION_MICRO ${micro})
    endif()
endif()

# Use extracted version
message(STATUS "Building libmodbus version ${LIBMODBUS_VERSION}")

# Set default install prefix for Windows (C:/libmodbus)
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT AND WIN32)
    set(CMAKE_INSTALL_PREFIX "C:/libmodbus" CACHE PATH "Default install path" FORCE)
endif()

# Include necessary modules for checks
include(CheckIncludeFile)
include(CheckFunctionExists)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Standard C library headers
check_include_file("arpa/inet.h" HAVE_ARPA_INET_H)
check_include_file("fcntl.h" HAVE_FCNTL_H)
check_include_file("errno.h" HAVE_ERRNO_H)
check_include_file("inttypes.h" HAVE_INTTYPES_H)
check_include_file("limits.h" HAVE_LIMITS_H)
check_include_file("memory.h" HAVE_MEMORY_H)
check_include_file("netdb.h" HAVE_NETDB_H)
check_include_file("netinet/in.h" HAVE_NETINET_IN_H)
check_include_file("netinet/ip.h" HAVE_NETINET_IP_H)
check_include_file("netinet/tcp.h" HAVE_NETINET_TCP_H)
check_include_file("sys/ioctl.h" HAVE_SYS_IOCTL_H)
check_include_file("sys/socket.h" HAVE_SYS_SOCKET_H)
check_include_file("sys/stat.h" HAVE_SYS_STAT_H)
check_include_file("sys/time.h" HAVE_SYS_TIME_H)
check_include_file("sys/types.h" HAVE_SYS_TYPES_H)
check_include_file("termios.h" HAVE_TERMIOS_H)
check_include_file("time.h" HAVE_TIME_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("wchar.h" HAVE_WCHAR_H)  # Added for wide character support
check_include_file("winsock2.h" HAVE_WINSOCK2_H)  # Windows-specific

# Function checks
check_function_exists(getaddrinfo HAVE_GETADDRINFO)
check_function_exists(select HAVE_SELECT)
check_function_exists(inet_ntop HAVE_INET_NTOP)  # Added for networking support
check_function_exists(inet_pton HAVE_INET_PTON)  # Added for networking support
check_function_exists(strlcpy HAVE_STRLCPY)  # Added for safe string manipulation (macOS/BSD)
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)  # Time-related function check

# Configure the config.h file from the config.h.in template
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
)

# Generate modbus-version.h from template
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/modbus-version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/include/modbus/modbus-version.h
)

# Set include directories
set(MODBUS_INCLUDES
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}/include/modbus  # <-- Ensure this points to where modbus-version.h is generated
)

# Add the source files
set(SRC_FILES
    src/modbus.c
    src/modbus-data.c
    src/modbus-rtu.c
    src/modbus-tcp.c
)

# Add the headers
set(HEADER_FILES
    src/modbus.h
    src/modbus-private.h
    src/modbus-rtu.h
    src/modbus-tcp.h
    src/modbus-rtu-private.h
    src/modbus-tcp-private.h
)

# Build both shared and static versions of the library
add_library(modbus_shared SHARED ${SRC_FILES})
add_library(modbus_static STATIC ${SRC_FILES})

# Ensure DLL export for Windows
set_target_properties(modbus_shared PROPERTIES 
    OUTPUT_NAME "libmodbus"
    WINDOWS_EXPORT_ALL_SYMBOLS ON
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
)

set_target_properties(modbus_static PROPERTIES 
    OUTPUT_NAME "libmodbus_static"
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
)

# Compiler options
if (CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    set(COMMON_COMPILE_OPTIONS -Wall -Wextra -Wpedantic -O2)
elseif (MSVC)
    set(COMMON_COMPILE_OPTIONS /W4)
endif()

# Apply options to the libraries
target_compile_options(modbus_shared PRIVATE ${COMMON_COMPILE_OPTIONS})
target_compile_options(modbus_static PRIVATE ${COMMON_COMPILE_OPTIONS})

# Define per-target compile definitions
if(WIN32)
    set(WINDOWS_DEFINES
        _WIN32_WINNT=0x0600
        WINVER=0x0600
        OS_WIN32
    )
    target_compile_definitions(modbus_shared PRIVATE ${WINDOWS_DEFINES})
    target_compile_definitions(modbus_static PRIVATE ${WINDOWS_DEFINES})
endif()

# Include directories with generator expressions
target_include_directories(modbus_shared PUBLIC
    $<BUILD_INTERFACE:${MODBUS_INCLUDES}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_include_directories(modbus_static PUBLIC
    $<BUILD_INTERFACE:${MODBUS_INCLUDES}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Link platform-specific libraries
if(WIN32)
    target_link_libraries(modbus_shared PUBLIC ws2_32)
    target_link_libraries(modbus_static PUBLIC ws2_32)
elseif(UNIX)
    target_link_libraries(modbus_shared PUBLIC m)
    target_link_libraries(modbus_static PUBLIC m)
endif()

# Install the shared and static libraries
install(TARGETS modbus_shared modbus_static
    EXPORT libmodbusTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/modbus
)

# Install the headers into include/modbus under the install prefix
install(FILES ${HEADER_FILES}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/modbus
)

# Install generated header modbus-version.h
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/modbus/modbus-version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/modbus
)

# Generate and install the libmodbus.pc file for pkg-config
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/libmodbus.pc.in
    ${CMAKE_CURRENT_BINARY_DIR}/libmodbus.pc
    @ONLY
)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libmodbus.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

# Generate the libmodbusConfigVersion.cmake file
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/libmodbusConfigVersion.cmake"
    VERSION ${LIBMODBUS_VERSION}
    COMPATIBILITY AnyNewerVersion
)

# Configure the libmodbusConfig.cmake file
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/libmodbusConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/libmodbusConfig.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/libmodbus"
)

# Install the config files
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/libmodbusConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/libmodbusConfigVersion.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/libmodbus"
)

# Install the exported targets
install(EXPORT libmodbusTargets
    FILE libmodbusTargets.cmake
    NAMESPACE libmodbus::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/libmodbus"
)

# Optionally, build the test programs
option(BUILD_TESTS "Build the tests" ON)

if(BUILD_TESTS)
    # Enable testing
    enable_testing()

    # Add the test client
    add_executable(test-client tests/unit-test-client.c)
    target_include_directories(test-client PRIVATE ${MODBUS_INCLUDES} ${CMAKE_CURRENT_BINARY_DIR})
    target_link_libraries(test-client PRIVATE modbus_shared)

    # Register test-client with CTest
    add_test(NAME test-client COMMAND test-client)

    # Add the test server
    add_executable(test-server tests/unit-test-server.c)
    target_include_directories(test-server PRIVATE ${MODBUS_INCLUDES} ${CMAKE_CURRENT_BINARY_DIR})
    target_link_libraries(test-server PRIVATE modbus_shared)

    # Register test-server with CTest
    add_test(NAME test-server COMMAND test-server)

    # Configure unit-test.h
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/tests/unit-test.h.in
        ${CMAKE_CURRENT_BINARY_DIR}/unit-test.h
    )

    target_include_directories(test-client PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
    target_include_directories(test-server PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
endif()

set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
set(CPACK_PACKAGE_VERSION ${LIBMODBUS_VERSION})
set(CPACK_GENERATOR "ZIP")

include(CPack)

# Output additional config information
message(STATUS "Configuring ${PROJECT_NAME} version ${LIBMODBUS_VERSION}")
message(STATUS "Build tests: ${BUILD_TESTS}")
message(STATUS "Installation path: ${CMAKE_INSTALL_PREFIX}")