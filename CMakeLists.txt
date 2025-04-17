cmake_minimum_required(VERSION 3.28)
project(Yosys VERSION 0.19 LANGUAGES C CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Include the cmake modules directory
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

# Feature options
option(ENABLE_TCL "Enable TCL support" ON)
option(ENABLE_ABC "Enable ABC support" ON)
option(ENABLE_GLOB "Enable GLOB support" ON)
option(ENABLE_PLUGINS "Enable plugins" ON)
# Enable Readline by default unless overridden by a specific target
set(ENABLE_READLINE ON)
option(ENABLE_EDITLINE "Enable Editline support" OFF)
option(ENABLE_GHDL "Enable GHDL support" OFF)
option(ENABLE_VERIFIC "Enable Verific support" OFF)
option(ENABLE_VERIFIC_SYSTEMVERILOG "Enable Verific SystemVerilog" ON)
option(ENABLE_VERIFIC_VHDL "Enable Verific VHDL" ON)
option(ENABLE_VERIFIC_HIER_TREE "Enable Verific hierarchical tree" ON)
option(ENABLE_VERIFIC_YOSYSHQ_EXTENSIONS "Enable YosysHQ Verific extensions" OFF)
option(ENABLE_VERIFIC_EDIF "Enable Verific EDIF" OFF)
option(ENABLE_VERIFIC_LIBERTY "Enable Verific Liberty" OFF)
option(ENABLE_COVER "Enable coverage instrumentation" ON)
option(ENABLE_LIBYOSYS "Enable libyosys" OFF)
option(ENABLE_ZLIB "Enable zlib support" ON)

# Configuration options
set(CONFIG "none" CACHE STRING "Build configuration")
set_property(CACHE CONFIG PROPERTY STRINGS "none" "clang" "gcc" "wasi" "msys2-32" "msys2-64")

# If building for WASI, disable Readline
if (CONFIG STREQUAL "wasi")
    set(ENABLE_READLINE OFF)
endif()

# Additional build options
option(ENABLE_GCOV "Enable GCOV for coverage" OFF)
option(ENABLE_GPROF "Enable GPROF for profiling" OFF)
option(ENABLE_DEBUG "Enable debug mode" OFF)
option(ENABLE_LTO "Enable Link-Time Optimization (LTO)" OFF)
option(ENABLE_CCACHE "Enable CCache" OFF)
option(ENABLE_SCCACHE "Enable Sccache" OFF)
option(ENABLE_FUNCTIONAL_TESTS "Enable functional tests" OFF)
option(LINK_CURSES "Link against curses" OFF)
option(LINK_TERMCAP "Link against termcap" OFF)
option(LINK_ABC "Link against ABC" OFF)

# Generate build timestamp
string(TIMESTAMP YOSYS_DATESTAMP "%Y%m%d")

# Configure paths
set(YOSYS_DATDIR "${CMAKE_INSTALL_PREFIX}/share/yosys" CACHE PATH "Yosys data directory")
set(YOSYS_BINDIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Yosys binary directory")

# Define YOSYS_SRC equivalent to Makefile's $(dir $(firstword $(MAKEFILE_LIST)))
set(YOSYS_SRC "${CMAKE_CURRENT_SOURCE_DIR}")

# Special handling for log.cc
set_source_files_properties(
  kernel/log.cc
  PROPERTIES COMPILE_DEFINITIONS "YOSYS_SRC=\"${YOSYS_SRC}\""
)

# Find dependencies based on enabled features
if(ENABLE_TCL)
    find_package(TCL REQUIRED)
endif()

if(ENABLE_READLINE)
    find_package(Readline REQUIRED)
endif()

if(ENABLE_ZLIB)
    find_package(ZLIB REQUIRED)
endif()

# Add include directories
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
)

# Add feature-dependent include directories
if(ENABLE_TCL)
    include_directories(${TCL_INCLUDE_PATH})
endif()

# Define main Yosys library sources
file(GLOB SOURCES
    "kernel/*.cc"
    "libs/ezsat/*.cc"
    "libs/minisat/*.cc"
    "passes/*/*.cc"
)

# Create main Yosys library
add_library(libyosys STATIC ${SOURCES})

# Set target properties for the library
set_target_properties(libyosys PROPERTIES
    OUTPUT_NAME "yosys"
    PREFIX "lib"
)

# Add the critical _YOSYS_ define to all targets
add_compile_definitions(_YOSYS_)

# Set common compile definitions
target_compile_definitions(libyosys PRIVATE
    _YOSYS_VERSION="${PROJECT_VERSION}"
    _YOSYS_DATESTAMP="${YOSYS_DATESTAMP}"
    _YOSYS_DATDIR="${YOSYS_DATDIR}"
    _YOSYS_BINDIR="${YOSYS_BINDIR}"
)

# Add feature-dependent compile definitions
if(ENABLE_PLUGINS)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_PLUGINS)
endif()

if(ENABLE_ABC)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_ABC)
endif()

if(ENABLE_GLOB)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_GLOB)
endif()

if(ENABLE_COVER)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_COVER)
endif()

if(ENABLE_TCL)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_TCL)
endif()

if(ENABLE_ZLIB)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_ZLIB)
endif()

if(ENABLE_DEBUG)
    target_compile_definitions(libyosys PRIVATE DEBUG)
    set(CMAKE_BUILD_TYPE Debug)
endif()

# Handle Readline/Editline
if(ENABLE_READLINE)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_READLINE)
elseif(ENABLE_EDITLINE)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_EDITLINE)
endif()

# Handle Verific
if(ENABLE_VERIFIC)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC)

    if(ENABLE_VERIFIC_SYSTEMVERILOG)
        target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC_SYSTEMVERILOG)
    endif()

    if(ENABLE_VERIFIC_VHDL)
        target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC_VHDL)
    endif()

    if(ENABLE_VERIFIC_HIER_TREE)
        target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC_HIER_TREE)
    endif()

    if(ENABLE_VERIFIC_YOSYSHQ_EXTENSIONS)
        target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC_YOSYSHQ_EXTENSIONS)
    endif()

    if(ENABLE_VERIFIC_EDIF)
        target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC_EDIF)
    endif()

    if(ENABLE_VERIFIC_LIBERTY)
        target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_VERIFIC_LIBERTY)
    endif()
endif()

# Handle GHDL
if(ENABLE_GHDL)
    target_compile_definitions(libyosys PRIVATE YOSYS_ENABLE_GHDL)
endif()

# Apply feature-dependent linking
if(ENABLE_TCL)
    target_link_libraries(libyosys PRIVATE ${TCL_LIBRARY})
endif()

if(ENABLE_ZLIB)
    target_link_libraries(libyosys PRIVATE ${ZLIB_LIBRARIES})
endif()

if(ENABLE_PLUGINS)
    target_link_libraries(libyosys PRIVATE ${CMAKE_DL_LIBS})  # Usually -ldl on Linux
endif()

if(ENABLE_ABC AND LINK_ABC)
    target_link_libraries(libyosys PRIVATE abc)
endif()

# Handle Readline/Editline linking
if(ENABLE_READLINE)
    target_link_libraries(libyosys PRIVATE ${Readline_LIBRARIES})

    if(LINK_CURSES)
        target_link_libraries(libyosys PRIVATE curses)
    endif()

    if(LINK_TERMCAP)
        target_link_libraries(libyosys PRIVATE termcap)
    endif()
elseif(ENABLE_EDITLINE)
    target_link_libraries(libyosys PRIVATE edit)
endif()

# Create Yosys executable
add_executable(yosys kernel/driver.cc)
target_link_libraries(yosys PRIVATE libyosys)

# Apply profiling with GPROF if enabled
if(ENABLE_GPROF)
    target_compile_options(libyosys PRIVATE -pg)
    target_compile_options(yosys PRIVATE -pg)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pg")
endif()

# Apply GCOV for coverage if enabled
if(ENABLE_GCOV)
    target_compile_options(libyosys PRIVATE --coverage)
    target_compile_options(yosys PRIVATE --coverage)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
endif()

# Apply LTO if enabled
if(ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT LTO_SUPPORTED)
    if(LTO_SUPPORTED)
        set_property(TARGET libyosys PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
        set_property(TARGET yosys PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
    else()
        message(WARNING "LTO is not supported by the compiler")
    endif()
endif()

# Handle CCACHE if enabled
if(ENABLE_CCACHE)
    find_program(CCACHE_PROGRAM ccache)
    if(CCACHE_PROGRAM)
        set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
        set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    endif()
endif()

# Handle SCCACHE if enabled
if(ENABLE_SCCACHE)
    find_program(SCCACHE_PROGRAM sccache)
    if(SCCACHE_PROGRAM)
        set(CMAKE_C_COMPILER_LAUNCHER "${SCCACHE_PROGRAM}")
        set(CMAKE_CXX_COMPILER_LAUNCHER "${SCCACHE_PROGRAM}")
    endif()
endif()
