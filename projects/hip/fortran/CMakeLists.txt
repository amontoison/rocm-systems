# Copyright (c) Advanced Micro Devices, Inc., or its affiliates.
#
# SPDX-License-Identifier: MIT

# Fortran bindings for HIP.
#
# hip.F90 is GENERATED from the HIP C headers by the rocm-fortran generator and
# committed next to the headers it binds, so a header change and its binding
# travel in the same pull request. Do not edit it by hand.
#
# One self-contained module, `hip`: the interfaces, the enum constants, the
# derived types, the opaque handle types and the hipCheck helper. It compiles
# against nothing, so this directory does not depend on the HIP C target; the
# runtime is needed only to LINK, and is carried as the link interface.
#
# HIP is the one ROCm project whose headers live apart from its build: the
# headers are here, the runtime that implements them is built in projects/clr.
# The bindings belong with the headers, so clr adds this directory through
# HIP_COMMON_DIR.

cmake_minimum_required(VERSION 3.19)

# Standalone, this directory is its own project, so the bindings can be built
# and packaged alone. In-tree, clr's project() is already in effect.
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  project(hip-fortran LANGUAGES NONE)
endif()

# ON by default but guarded: a site with no Fortran compiler must still build
# the C library. "${UNIX}" rather than ON because ROCm ships no Fortran compiler
# on Windows: TheRock adds flang only in the non-WIN32 branch, so the amdflang
# preference below finds nothing, the probe falls through to a stray gfortran,
# and enable_language(Fortran) then fails to identify it. clr adds this
# directory under CLR_BUILD_HIP, so the guard matters from the first build.
option(BUILD_FORTRAN_BINDINGS "Build the HIP Fortran bindings" "${UNIX}")

# The bindings' own test suite, and the Fortran clients where a library has
# them: one switch rather than two. Guarded like the bindings, so asking for
# them when the bindings were not built is a skip rather than an error.
option(BUILD_FORTRAN_CLIENTS "Build the HIP Fortran clients and binding tests" ON)

# Both options are ONE cache entry shared by every library in the monorepo, so
# -DBUILD_FORTRAN_BINDINGS=OFF from the top turns the whole Fortran track off.
# Each library also honours a name of its own, which wins when it is set.
if(DEFINED HIP_BUILD_FORTRAN_BINDINGS)
  set(BUILD_FORTRAN_BINDINGS ${HIP_BUILD_FORTRAN_BINDINGS})
endif()
if(DEFINED HIP_BUILD_FORTRAN_CLIENTS)
  set(BUILD_FORTRAN_CLIENTS ${HIP_BUILD_FORTRAN_CLIENTS})
endif()

# Which array-argument overloads the module exposes. The two upper tiers cannot
# legally coexist: an assumed-rank dummy is not distinguishable by rank from the
# per-rank specifics, so assumed-rank REPLACES the per-rank variants rather than
# adding to them.
#
#   none           the raw type(c_ptr) surface only (Fortran 2003)
#   assumed-shape  + the per-rank array overloads (Fortran 2008), the default,
#                  and what the bindings shipped in ROCm are built with
#   assumed-rank   + the single dimension(..) overload per routine (Fortran 2018)
#
# For HIP, none and assumed-shape select the SAME module: the typed memory
# families are the only routines here taking array arguments, and hipfort has
# always shipped their per-rank overloads unguarded, so the generator emits them
# as the plain #else of USE_ASSUMED_RANK with no macro to switch them off.
# Verified by preprocessing hip.F90 four ways with amdflang -cpp -E: no macro
# and USE_ASSUMED_SHAPE=1 hash identically, USE_ASSUMED_RANK=1 and both-defined
# hash identically, and the groups differ. none is accepted rather than rejected
# because this cache entry is shared with every other ROCm Fortran binding,
# where the three tiers really are distinct.
set(FORTRAN_ARRAY_INTERFACES "assumed-shape" CACHE STRING
    "Array-argument overloads to expose: none, assumed-shape, or assumed-rank")
set_property(CACHE FORTRAN_ARRAY_INTERFACES PROPERTY STRINGS
             none assumed-shape assumed-rank)

# Distinct from ROCM_HAVE_FORTRAN, which says only that a compiler exists.
# Guarded because PARENT_SCOPE at top level warns under -Wdev rather than being
# a silent no-op.
if(NOT CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(HIP_HAVE_FORTRAN_BINDINGS FALSE PARENT_SCOPE)
endif()

if(NOT BUILD_FORTRAN_BINDINGS)
  return()
endif()

# Probing is not free, so the answer is computed ONCE and published as
# ROCM_HAVE_FORTRAN -- the single name in both monorepos, since a library from
# one is regularly built inside the other and a repo-specific name would have
# each side reading a variable the other never sets. It comes from a monorepo
# root that probes on the way in, or from the shared module below: rocm-systems
# has no root CMakeLists, so its probe travels as a module. OPTIONAL, and
# guarded on the answer not already being in hand, since a deliberate
# ROCM_HAVE_FORTRAN=OFF must not be contradicted by a fresh probe.
if(NOT DEFINED ROCM_HAVE_FORTRAN)
  if(NOT DEFINED ROCM_SYSTEMS_ROOT)
    get_filename_component(ROCM_SYSTEMS_ROOT
                           "${CMAKE_CURRENT_SOURCE_DIR}/../../.." ABSOLUTE)
  endif()
  include("${ROCM_SYSTEMS_ROOT}/shared/cmake/ROCmFortran.cmake" OPTIONAL)
endif()

# The local probe is the last resort: this directory is also built extracted
# from the monorepo, where neither a root nor the shared module exists.
if(DEFINED ROCM_HAVE_FORTRAN)
  set(_have_fortran ${ROCM_HAVE_FORTRAN})
else()
  include(CheckLanguage)

  # check_language() writes the CACHE entry CMAKE_Fortran_COMPILER, which
  # rocm-cmake's ROCMChecks watches to catch a project setting a toolchain
  # variable behind the toolchain file's back. A legitimate probe is
  # indistinguishable from a violation, and under -DROCM_ERROR_TOOLCHAIN_VAR=ON
  # (which CI sets) that is a hard configure failure, so silence the checker
  # across the probe and restore it afterwards. Plain variables, so they shadow
  # the cache for this scope only; both names, because the callback escalates on
  # the error flag before it ever looks at the warn one.
  set(_rocm_warn_toolchain_var "${ROCM_WARN_TOOLCHAIN_VAR}")
  set(_rocm_error_toolchain_var "${ROCM_ERROR_TOOLCHAIN_VAR}")
  set(ROCM_WARN_TOOLCHAIN_VAR OFF)
  set(ROCM_ERROR_TOOLCHAIN_VAR OFF)

  # Prefer ROCm's own amdflang: check_language(Fortran) takes the first compiler
  # on the search path, which on a ROCm CI image is the system gfortran -- that
  # is how the bindings ended up installed under fortran/gfortran/ while the
  # documentation promises fortran/amdflang/. Look beside the clang already
  # compiling this project, then under ROCM_PATH (variable and environment)
  # and /opt/rocm, with NO_DEFAULT_PATH so a stray amdflang cannot win. An explicit
  # CMAKE_Fortran_COMPILER, or FC, always wins. check_language() is a no-op once
  # CMAKE_Fortran_COMPILER is defined, so this has to run before it.
  if(NOT CMAKE_Fortran_COMPILER AND NOT DEFINED ENV{FC})
    set(_rocm_fc_hints)
    foreach(_rocm_fc_cc "${CMAKE_CXX_COMPILER}" "${CMAKE_C_COMPILER}"
                        "${CMAKE_HIP_COMPILER}")
      if(_rocm_fc_cc)
        get_filename_component(_rocm_fc_dir "${_rocm_fc_cc}" DIRECTORY)
        list(APPEND _rocm_fc_hints "${_rocm_fc_dir}")
      endif()
    endforeach()
    foreach(_rocm_fc_root "${ROCM_PATH}" "$ENV{ROCM_PATH}" "/opt/rocm")
      if(_rocm_fc_root)
        list(APPEND _rocm_fc_hints "${_rocm_fc_root}/bin"
                                   "${_rocm_fc_root}/lib/llvm/bin")
      endif()
    endforeach()
    if(_rocm_fc_hints)
      unset(_rocm_amdflang CACHE)
      find_program(_rocm_amdflang NAMES amdflang HINTS ${_rocm_fc_hints}
                   NO_DEFAULT_PATH)
      # Confirm it runs before committing to it: a ROCm tree can carry a
      # dangling amdflang symlink, which check_language() would not catch, so
      # the failure would surface far later as a broken enable_language.
      if(_rocm_amdflang)
        execute_process(COMMAND "${_rocm_amdflang}" --version
                        RESULT_VARIABLE _rocm_fc_rc
                        OUTPUT_QUIET ERROR_QUIET)
        if(_rocm_fc_rc EQUAL 0)
          set(CMAKE_Fortran_COMPILER "${_rocm_amdflang}" CACHE FILEPATH
              "Fortran compiler used for the ROCm Fortran bindings" FORCE)
        endif()
        unset(_rocm_fc_rc)
      endif()
      unset(_rocm_amdflang CACHE)
    endif()
    unset(_rocm_fc_hints)
    unset(_rocm_fc_cc)
    unset(_rocm_fc_dir)
    unset(_rocm_fc_root)
  endif()

  check_language(Fortran)
  set(ROCM_WARN_TOOLCHAIN_VAR "${_rocm_warn_toolchain_var}")
  set(ROCM_ERROR_TOOLCHAIN_VAR "${_rocm_error_toolchain_var}")
  unset(_rocm_warn_toolchain_var)
  unset(_rocm_error_toolchain_var)

  if(CMAKE_Fortran_COMPILER)
    set(_have_fortran TRUE)
  else()
    set(_have_fortran FALSE)
  endif()
endif()

if(NOT _have_fortran)
  message(STATUS "HIP: no Fortran compiler found, skipping the Fortran bindings")
  return()
endif()

enable_language(Fortran)

# Here rather than beside the install() rules, because the install layout is
# also baked into the target's INSTALL_INTERFACE include directory, set well
# before them. Nothing must hardcode lib/: a distro with
# CMAKE_INSTALL_LIBDIR=lib64 would get a lib/fortran that does not exist.
include(GNUInstallDirs)

if(NOT FORTRAN_ARRAY_INTERFACES MATCHES "^(none|assumed-shape|assumed-rank)$")
  message(FATAL_ERROR
    "FORTRAN_ARRAY_INTERFACES must be none, assumed-shape or assumed-rank, "
    "not '${FORTRAN_ARRAY_INTERFACES}'.")
endif()

add_library(hip_fortran STATIC hip.F90)
add_library(hip::hip_fortran ALIAS hip_fortran)

# A .mod is not portable across compilers, so the artifacts install under a
# per-compiler directory and two compilers can be installed side by side. Named
# after the compiler a user actually types (amdflang, gfortran), so the -I/-L
# paths in the documentation match a raw link line. Derived from the compiler ID
# rather than the basename of CMAKE_Fortran_COMPILER, which would be wrong for
# an MPI wrapper (mpif90), gfortran-13, an absolute path or the Cray ftn.
if(CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
  set(_fc_name "gfortran")
elseif(CMAKE_Fortran_COMPILER_ID MATCHES "Flang")     # Flang, LLVMFlang
  set(_fc_name "amdflang")
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "Cray")
  set(_fc_name "ftn")
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "IntelLLVM")
  set(_fc_name "ifx")
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "Intel")
  set(_fc_name "ifort")
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "NVHPC")
  set(_fc_name "nvfortran")
else()
  # Best guess available, and still collision-free.
  get_filename_component(_fc_name "${CMAKE_Fortran_COMPILER}" NAME_WE)
  string(TOLOWER "${_fc_name}" _fc_name)
endif()

# Deliberately lossy: a .mod is tied to the compiler VERSION too, so gfortran 13
# and 14 share a directory and the second overwrites the first. Right for the
# common case of one compiler per prefix; a site with several versions of the
# SAME compiler sets this itself.
set(HIP_FORTRAN_COMPILER_DIR "${_fc_name}" CACHE STRING
    "Per-compiler subdirectory the .mod and the archive install under")

set(_moddir "${CMAKE_CURRENT_BINARY_DIR}/include")
set_target_properties(hip_fortran PROPERTIES
  Fortran_MODULE_DIRECTORY "${_moddir}"
  # The array overloads sit behind preprocessor guards.
  Fortran_PREPROCESS ON
  # Free form is the .F90 default, but stating it lets CMake emit whichever flag
  # the compiler in use expects rather than relying on the extension.
  Fortran_FORMAT FREE
  POSITION_INDEPENDENT_CODE ON)

# hip.F90 contains five bare #ifdef USE_ASSUMED_RANK regions and no
# USE_ASSUMED_SHAPE at all, for the reason given beside the option above, so
# defining USE_ASSUMED_SHAPE here would name a macro the source never tests: a
# reader would take it for the switch selecting the F2008 tier, and changing it
# would change nothing. The lower two tiers therefore define nothing.
if(FORTRAN_ARRAY_INTERFACES STREQUAL "assumed-rank")
  target_compile_definitions(hip_fortran PRIVATE USE_ASSUMED_RANK=1)
endif()

# No line-length flag: every emitted line fits inside the 132 columns free-form
# Fortran guarantees. Measured on the committed hip.F90, the longest line is 131
# columns.

target_include_directories(hip_fortran PUBLIC
  "$<BUILD_INTERFACE:${_moddir}>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/fortran/${HIP_FORTRAN_COMPILER_DIR}>")

# Link interface only: needed to link an executable, not to build this archive.
# Three cases, and the order matters.
#
# In-tree under clr the runtime is BEING BUILT here, as the target amdhip64, and
# clr adds this directory right after hipamd so it already exists. Calling
# find_package(hip) there would be circular, and under TheRock it is fatal: the
# super-project installs a dependency provider that intercepts every
# find_package and hard-errors on a package it builds itself but which the
# caller has not declared. QUIET does not help, because the provider reports
# before the search is ever attempted.
#
# Standalone the runtime has to be looked for; testing for the target without
# ever calling find_package would leave the archive exporting no link interface
# and every consumer hitting undefined references to hipMalloc. hip-config.cmake
# reaches FindThreads, which hard-errors unless C or CXX is enabled, so C is
# enabled first. QUIET, because the bindings compile against nothing and are
# legitimately buildable with no ROCm installed.
if(NOT TARGET hip::host AND NOT TARGET amdhip64)
  if(NOT CMAKE_C_COMPILER_LOADED AND NOT CMAKE_CXX_COMPILER_LOADED)
    enable_language(C)
  endif()
  find_package(hip QUIET)
endif()

# Whether the INSTALLED link interface names hip::host is also what the config
# package has to know, and it cannot work that out for itself: it sees only the
# targets file. Its find_dependency(hip) exists solely to make hip::host exist
# before the targets file names it, so a binding built with no runtime in sight
# must not make its consumers find a package the artifact does not depend on.
if(TARGET hip::host)
  target_link_libraries(hip_fortran INTERFACE hip::host)
  set(HIP_FORTRAN_NEEDS_C_PACKAGE TRUE)
elseif(TARGET amdhip64)
  # In-tree the runtime is a real target, and clr already exports it from TWO
  # sets (hip-targets and hip-lang-targets). CMake refuses to export a target
  # whose link interface names one exported more than once, so naming amdhip64
  # unconditionally fails the generate step with "not in this export set, but in
  # multiple other export sets". So link it for the BUILD only and let the
  # INSTALLED interface name hip::host, as the standalone branch above does.
  target_link_libraries(hip_fortran INTERFACE
    $<BUILD_INTERFACE:amdhip64>
    $<INSTALL_INTERFACE:hip::host>)
  set(HIP_FORTRAN_NEEDS_C_PACKAGE TRUE)
else()
  set(HIP_FORTRAN_NEEDS_C_PACKAGE FALSE)
  message(STATUS
    "HIP: the runtime was not found, so the Fortran bindings carry no link "
    "interface; a consumer will have to link it (-lamdhip64) itself.")
endif()

if(NOT CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(HIP_HAVE_FORTRAN_BINDINGS TRUE PARENT_SCOPE)
endif()

# The binding tests are GPU runtime executables, so in-tree they must not come
# along with a plain library build: 29 test binaries linking a GPU runtime is
# not what -DCLR_BUILD_HIP=ON asked for, and test/CMakeLists.txt additionally
# shells out to rocm_agent_enumerator AT CONFIGURE TIME to pick an offload arch.
# Every other ROCm Fortran binding gates its tests on its own project's client
# switch; clr has none -- the BUILD_TESTS it carries is scoped inside opencl/.
#
# So the condition is BUILD_TESTING, the name include(CTest) defines. clr never
# includes CTest, so in-tree it is undefined and the tests stay off until asked
# for with -DBUILD_TESTING=ON, while a super-project that turned testing on for
# the whole build gets them without a HIP-specific flag.
# HIP_BUILD_FORTRAN_CLIENTS still wins, because the option block above promises
# that a per-library name overrides the shared one. Set as a plain variable,
# because the cache entry is one switch shared by every library and must not be
# written back, and placed past both early returns, since a configure that left
# earlier would announce a skip for the wrong reason.
if(NOT CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR
   AND BUILD_FORTRAN_CLIENTS
   AND NOT (BUILD_TESTING OR HIP_BUILD_FORTRAN_CLIENTS))
  message(STATUS
    "HIP: testing is not enabled, skipping the Fortran binding tests. Pass "
    "-DBUILD_TESTING=ON or -DHIP_BUILD_FORTRAN_CLIENTS=ON to build them.")
  set(BUILD_FORTRAN_CLIENTS OFF)
endif()

if(BUILD_FORTRAN_CLIENTS)
  # Enough for a STANDALONE build, but NOT for an in-tree one: CMake writes the
  # chain of CTestTestfile.cmake that ctest walks down from a build root only
  # when testing was enabled at the TOP level. Otherwise the tests build and
  # register, and `ctest` at the parent build root reports "Total Tests: 0".
  enable_testing()
  add_subdirectory(test)
endif()

include(CMakePackageConfigHelpers)

# lib/cmake/hip-fortran, the directory find_package(hip-fortran) searches
# natively, so the documented spelling resolves with no forwarder. Putting it
# beside the C package in lib/cmake/hip would only pay off if hip-config.cmake
# grew an include() of it, which would mean editing the file EVERY HIP consumer
# loads. rocRAND is the cautionary precedent: its rocrand-config.cmake carries
# exactly such an include with no OPTIONAL, so find_package(rocrand) fails
# outright on a prefix where the bindings are not installed.
set(_fortran_pkgdir "${CMAKE_INSTALL_LIBDIR}/cmake/hip-fortran")

# dev is the component clr's CPack packages HIP's headers and config under.
set(HIP_FORTRAN_INSTALL_COMPONENT "dev" CACHE STRING
    "Install component of the HIP Fortran bindings")

install(TARGETS hip_fortran
        EXPORT hip-fortran-targets
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}/fortran/${HIP_FORTRAN_COMPILER_DIR}"
                COMPONENT ${HIP_FORTRAN_INSTALL_COMPONENT})
install(DIRECTORY "${_moddir}/"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/fortran/${HIP_FORTRAN_COMPILER_DIR}"
        COMPONENT ${HIP_FORTRAN_INSTALL_COMPONENT}
        FILES_MATCHING PATTERN "*.mod")
install(EXPORT hip-fortran-targets
        FILE hip-fortran-targets.cmake
        NAMESPACE hip::
        DESTINATION "${_fortran_pkgdir}"
        COMPONENT ${HIP_FORTRAN_INSTALL_COMPONENT})

# Kept out of the template so neither side spells lib/ or include/ literally: a
# literal lib/fortran is wrong the moment CMAKE_INSTALL_LIBDIR is lib64.
set(HIP_FORTRAN_LIBDIR     "${CMAKE_INSTALL_LIBDIR}/fortran")
set(HIP_FORTRAN_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/fortran")

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/hip-fortran-config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/hip-fortran-config.cmake"
  INSTALL_DESTINATION "${_fortran_pkgdir}"
  # @PACKAGE_INIT@ rewrites each of these into an absolute path computed from
  # where the config file was FOUND, so the package survives relocation and gets
  # lib64 right. HIP_FORTRAN_COMPILER_DIR stays a plain @-substitution: it is a
  # bare directory NAME, and PATH_VARS would resolve it under the prefix.
  PATH_VARS HIP_FORTRAN_LIBDIR HIP_FORTRAN_INCLUDEDIR)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/hip-fortran-config.cmake"
        DESTINATION "${_fortran_pkgdir}"
        COMPONENT ${HIP_FORTRAN_INSTALL_COMPONENT})

# Read from the VERSION file in the source tree, the same one hipamd parses, and
# NOT from hip_VERSION: that is set by find_package(hip) above and reports
# whatever HIP is INSTALLED. Components are read as a list rather than captured
# by regex, so a legitimate 0 in any position is not a failed match.
set(_fortran_version "")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../VERSION")
  file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/../VERSION" _hip_vlist REGEX "^[0-9]+")
  list(LENGTH _hip_vlist _hip_vn)
  if(_hip_vn GREATER_EQUAL 3)
    list(GET _hip_vlist 0 1 2 _hip_v)
    list(JOIN _hip_v "." _fortran_version)
  endif()
  unset(_hip_vlist)
endif()

if(_fortran_version)
  write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/hip-fortran-config-version.cmake"
    VERSION "${_fortran_version}"
    COMPATIBILITY SameMajorVersion)
  # Beside the config it belongs to, and only there: CMake looks for the version
  # file NEXT TO the <pkg>-config.cmake it resolved.
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/hip-fortran-config-version.cmake"
          DESTINATION "${_fortran_pkgdir}"
          COMPONENT ${HIP_FORTRAN_INSTALL_COMPONENT})
else()
  # Better none than a wrong one: a version file claiming the wrong number makes
  # a versioned find_package succeed and link the wrong .mod.
  message(STATUS
    "HIP: could not determine the version from the VERSION file, so no "
    "hip-fortran-config-version.cmake is installed and a versioned "
    "find_package(hip-fortran) will not match.")
endif()

# The generated source ships too, so a user on a compiler ROCm ships no .mod for
# can rebuild the bindings. Under share/ and not include/, because every entry
# under include/fortran/ is a compiler name and the config enumerates that
# directory to report which compilers are installed.
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/hip.F90"
        DESTINATION "${CMAKE_INSTALL_DATADIR}/hip/fortran"
        COMPONENT ${HIP_FORTRAN_INSTALL_COMPONENT})
