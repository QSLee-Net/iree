################################################################################
# Autogenerated by build_tools/bazel_to_cmake/bazel_to_cmake.py from           #
# compiler/src/iree/compiler/API/BUILD.bazel                                   #
#                                                                              #
# Use iree_cmake_extra_content from iree/build_defs.oss.bzl to add arbitrary   #
# CMake-only content.                                                          #
#                                                                              #
# To disable autogeneration for this file entirely, delete this header.        #
################################################################################

iree_add_all_subdirs()

iree_cc_library(
  NAME
    StaticImpl
  DEPS
    IREEDialectsCAPI
    MLIRCAPIDebug
    MLIRCAPIGPU
    MLIRCAPIIR
    MLIRCAPIInterfaces
    MLIRCAPILLVM
    MLIRCAPILinalg
    MLIRCAPIPDL
    MLIRCAPITransformDialect
    MLIRCAPITransformDialectTransforms
    MLIRCAPITransforms
    iree::compiler::API::Internal::CompilerDriver
    iree::compiler::API::Internal::IREECodegenDialectCAPI
    iree::compiler::API::Internal::IREECompileToolEntryPoint
    iree::compiler::API::Internal::IREEGPUDialectCAPI
    iree::compiler::API::Internal::IREEMLIRLSPServerToolEntryPoint
    iree::compiler::API::Internal::IREEOptToolEntryPoint
    iree::compiler::API::Internal::IREEReduceToolEntryPoint
    iree::compiler::API::Internal::LLDToolEntryPoint
  PUBLIC
)

### BAZEL_TO_CMAKE_PRESERVES_ALL_CONTENT_BELOW_THIS_LINE ###

################################################################################
# libIREECompiler.so building
# It's 2023. We were supposed to have sane ways to build shared libraries.
# We were promised.
#
# Anyway, the shared library must be explicitly assembled from the *object
# libraries* that make up its public interface. We could use the new
# $<LINK_LIBRARY:WHOLE_ARCHIVE> feature to bypass this old way of doing it,
# but that interacts badly with BUILD_SHARED_LIBS mode (since it would then
# not be linking the static libraries *into* a composite shared library).
#
# Therefore, we duplicate the object libs from above. They are lexically
# related:
#   1. LLVM derived object libraries are named with an "obj." prefix.
#   2. IREE derived object libraries are named with an ".objects" suffix.
#   3. Alias libraries don't work, so we need to use the "_" vs "::" form.
#
# Given this variance, we opt for simple duplication. Make sure that the list
# below is always kept in sync with the list above for StaticImpl.
#
# Further, we have to separate out the LINK_LIBRARIES dependencies of the
# OBJECTS libraries and include them manually as PRIVATE deps of the shared
# library. This involves some generator shenanigans.
#
# Finally, we allow super-projects to export additional API from our shared
# library by populating the IREE_COMPILER_API_ADDL_EXPORT_OBJECTS.
################################################################################

set(_EXPORT_OBJECT_LIBS
  obj.IREEDialectsCAPI
  obj.MLIRCAPIDebug
  obj.MLIRCAPIIR
  obj.MLIRCAPIInterfaces
  obj.MLIRCAPIGPU
  obj.MLIRCAPILLVM
  obj.MLIRCAPILinalg
  obj.MLIRCAPIPDL
  obj.MLIRCAPITransforms
  obj.MLIRCAPITransformDialect
  obj.MLIRCAPITransformDialectTransforms
  iree_compiler_API_Internal_CompilerDriver.objects
  iree_compiler_API_Internal_IREECompileToolEntryPoint.objects
  iree_compiler_API_Internal_IREECodegenDialectCAPI.objects
  iree_compiler_API_Internal_IREEGPUDialectCAPI.objects
  iree_compiler_API_Internal_IREEMLIRLSPServerToolEntryPoint.objects
  iree_compiler_API_Internal_IREEOptToolEntryPoint.objects
  iree_compiler_API_Internal_IREEReduceToolEntryPoint.objects
  iree_compiler_API_Internal_LLDToolEntryPoint.objects
)

if(DEFINED IREE_COMPILER_API_ADDL_EXPORT_OBJECTS)
  message(STATUS "Bundling additional exports into libIREECompiler.so: ${IREE_COMPILER_API_ADDL_EXPORT_OBJECTS}")
  list(APPEND _EXPORT_OBJECT_LIBS ${IREE_COMPILER_API_ADDL_EXPORT_OBJECTS})
endif()

set(_EXPORT_OBJECT_SRCS)
set(_EXPORT_OBJECT_DEPS)
foreach(_object_lib ${_EXPORT_OBJECT_LIBS})
  # On MSVC, source level __declspec(dllexport) remain the most precise
  # way to control DLL API availability. All of the MLIR and IREE CAPI
  # are annotated this way, which approximates the same behavior we get if
  # building the overall project with visibility=hidden on POSIX.
  # Emission of these annotations is guarded by two macros in each
  # codebase: one to enable DECLSPEC emission at all and another
  # to indicate that we are building a library and the symbols should be
  # exported (vs imported). Note that with CMake controlling linking,
  # it is not necessary to emit dllimport annotations, since CMake will
  # link against the import library, which is sufficient for functions.
  # The alternative is to export everything by setting (on the SharedImpl):
  #   WINDOWS_EXPORT_ALL_SYMBOLS ON
  # This, however, produces on the order of 6-10k exported symbols, whereas
  # the focused API is 1/10 of that.
  #
  # It isn't super polite to be poking at other people's library definitions
  # like this, but MSVC is weird and given the structure of the projects, this
  # isn't so bad.
  get_target_property(_imported ${_object_lib} IMPORTED)
  if(MSVC AND NOT ${_imported})
    target_compile_definitions(${_object_lib} PRIVATE
      # See compiler/bindings/c/iree/compiler/api_support.h.
      # IREE embedded CAPI uses its own macros to enable
      # declaration specs.
      IREE_EMBED_ENABLE_WINDOWS_DLL_DECLSPEC
      IREE_EMBED_BUILDING_LIBRARY

      # See mlir-c/Support.h for the annotation selection.
      MLIR_CAPI_ENABLE_WINDOWS_DLL_DECLSPEC
      MLIR_CAPI_BUILDING_LIBRARY
    )
  endif()

  list(APPEND _EXPORT_OBJECT_SRCS "$<TARGET_OBJECTS:${_object_lib}>")
  # Ugh. CMake. GENEX_EVAL is not recursive: it evaluates one level of generator
  # expressions. There is a way to hold it just right so that GENEX_EVAL is
  # used at the call site, but it is really tricky and people can't be expected
  # to get it right. In fact, at the time of this writing, some misc usage
  # in LLVM causes a second level of generator expressions in some base
  # libraries. For good measure, we do GENEX_EVAL four times. If you get errors
  # that show like generator expressions are showing up in link lines, this is
  # the culprit. Look at the export_objects_debug.txt to confirm. Then, add
  # another level of fix upstream if you like pain.
  list(APPEND _EXPORT_OBJECT_DEPS "$<GENEX_EVAL:$<GENEX_EVAL:$<GENEX_EVAL:$<GENEX_EVAL:$<TARGET_PROPERTY:${_object_lib},LINK_LIBRARIES>>>>>")
endforeach()

# UNCOMMENT TO DEBUG WHAT IS GOING ON.
# file(GENERATE OUTPUT export_objects_debug.txt CONTENT "OBJECTS:${_EXPORT_OBJECT_SRCS}\n\nDEPS:${_EXPORT_OBJECT_DEPS}")

# Disable .so.0 style naming/linking. In order to be consistent across platforms
# and bindings, we will embed a major version in the library name when it is time.
# OS packagers can re-enable this or take other steps to get canonical shared
# library structures.
set(CMAKE_PLATFORM_NO_VERSIONED_SONAME ON)

iree_cc_library(
  SHARED
  NAME
    SharedImpl
  SRCS
     # Force this to be a regular library with an empty source file.
     empty.c
  LINKOPTS
    # Linux uses a linker script to version symbols.
    $<$<PLATFORM_ID:Linux>:-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/api_version.ld>
  DEPS
    iree::compiler::bindings::c::headers
)

# Make sure that re-link if extra linker inputs change.
set_target_properties(iree_compiler_API_SharedImpl PROPERTIES
  LINK_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/api_version.ld
)

# We need to add the object sources directly to the SHARED sub-target, not
# the .objects sub-target, which is what would happen if added to SRCS.
target_sources(iree_compiler_API_SharedImpl PRIVATE
  ${_EXPORT_OBJECT_SRCS}
)

# Explicitly link deps as PRIVATE (the default rule uses PUBLIC) so that usage
# requirements do not cascade: The shared library is intended to be used as an
# isolated boundary for C consumers.
target_link_libraries(iree_compiler_API_SharedImpl PRIVATE
  ${_EXPORT_OBJECT_DEPS}
)

# If not using sanitizers, ask linkers to error on undefined symbols.
# For shared libraries, using sanitizers implies undefined symbols on Unix-like.
# Also disabled on Apple (unsupported in XCode 15 default linker).
if(NOT IREE_ENABLE_ASAN
    AND NOT IREE_ENABLE_MSAN
    AND NOT IREE_ENABLE_TSAN
    AND NOT IREE_ENABLE_UBSAN
    AND NOT APPLE)
  target_link_options(iree_compiler_API_SharedImpl PRIVATE
    $<$<PLATFORM_ID:Linux>:-Wl,--no-undefined>
    $<$<PLATFORM_ID:Darwin>:-Wl,-undefined,error>
  )
endif()

set_target_properties(
  iree_compiler_API_SharedImpl
  PROPERTIES
    OUTPUT_NAME "IREECompiler"
    VERSION "0"
    SOVERSION "0"
)

# We can install shared libraries and export them without exporting their
# deps. This wouldn't work if this were a static library as we'd need to
# untangle the exporting of all of LLVM.
iree_install_targets(
  TARGETS
    iree_compiler_API_SharedImpl
  COMPONENT
    IREERuntimeLibraries-Compiler
  EXPORT_SET
    Compiler
)

# Signal the main shared library/dll as part of the bundle of files that
# must be included with compiler dylibs.
# What we really want to be using here is the TARGET_LINKER_FILE_NAME
# generator expression. But since this is used in places where target
# dependent generator expressions are not legal, we guesstimate it with
# platform specific logic.
set(_target_linker_file_name "${CMAKE_SHARED_LIBRARY_PREFIX}IREECompiler${CMAKE_SHARED_LIBRARY_SUFFIX}")
set_property(GLOBAL APPEND PROPERTY
  IREE_COMPILER_DYLIB_RELPATHS "${_target_linker_file_name}")

install(
  TARGETS iree_compiler_API_SharedImpl
  COMPONENT Compiler
)

# The bazel side uses a "Impl" library alias to generically represent
# static or shared. Provide a corresponding one here.
# When in BUILD_SHARED_LIBS mode, we do CMake level linking against the
# non-shared library, since strict layering is enforced there. The mondo
# shared library can still be generated and used dynamically but is not
# generally depended on from a normal C++ link.

if(IREE_LINK_COMPILER_SHARED_LIBRARY AND NOT BUILD_SHARED_LIBS)
iree_cc_library(
  NAME
    Impl
  DEPS
    ::SharedImpl
)
else()
iree_cc_library(
  NAME
    Impl
  DEPS
    ::StaticImpl
)
endif()
