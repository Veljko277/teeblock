cmake_minimum_required(VERSION 3.19)

file(STRINGS src/game/version.h VERSION_LINE
  LIMIT_COUNT 1
  REGEX GAME_RELEASE_VERSION
)

if(VERSION_LINE MATCHES "\"([0-9]+)\\.([0-9]+)\\.([0-9]+)\"")
  set(VERSION_MAJOR ${CMAKE_MATCH_1})
  set(VERSION_MINOR ${CMAKE_MATCH_2})
  set(VERSION_PATCH ${CMAKE_MATCH_3})
elseif(VERSION_LINE MATCHES "\"([0-9]+)\\.([0-9]+)\"")
  set(VERSION_MAJOR ${CMAKE_MATCH_1})
  set(VERSION_MINOR ${CMAKE_MATCH_2})
  set(VERSION_PATCH "0")
else()
  message(FATAL_ERROR "Couldn't parse version from src/game/version.h")
endif()

if(POLICY CMP0048)
  cmake_policy(SET CMP0048 NEW)
  if(VERSION_PATCH STREQUAL "0")
    project(teeworlds VERSION ${VERSION_MAJOR}.${VERSION_MINOR})
  else()
    project(teeworlds VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})
  endif()
else()
  project(teeworlds)
  set(PROJECT_VERSION_MAJOR ${VERSION_MAJOR})
  set(PROJECT_VERSION_MINOR ${VERSION_MINOR})
  set(PROJECT_VERSION_PATCH ${VERSION_PATCH})
  if(VERSION_PATCH STREQUAL "0")
    set(PROJECT_VERSION ${VERSION_MAJOR}.${VERSION_MINOR})
  else()
    set(PROJECT_VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})
  endif()
endif()

set(ORIGINAL_CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH})
set(ORIGINAL_CMAKE_REQUIRED_INCLUDES ${CMAKE_REQUIRED_INCLUDES})
set(ORIGINAL_CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES})
set(OWN_CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/cmake)
set(CMAKE_MODULE_PATH ${OWN_CMAKE_MODULE_PATH})

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(TARGET_BITS "64")
elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
  set(TARGET_BITS "32")
endif()

set(TARGET_OS "linux")

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(CheckSymbolExists)

check_symbol_exists(__i386 "" TARGET_ARCH_X86_i386)
if(TARGET_ARCH_X86_i386)
  set(TARGET_ARCH x86)
else()
  set(TARGET_ARCH)
endif()

set(AUTO_DEPENDENCIES_DEFAULT ON)
option(PREFER_BUNDLED_LIBS "Prefer bundled libraries over system libraries" ${AUTO_DEPENDENCIES_DEFAULT})
option(DEV "Don't generate stuff necessary for packaging" OFF)

# Set the default build type to Release
if(NOT(CMAKE_BUILD_TYPE))
  if(DEV)
    set(CMAKE_BUILD_TYPE Release)
  else()
    set(CMAKE_BUILD_TYPE Debug)
  endif()
endif()

set(DBG $<OR:$<CONFIG:Debug>,$<CONFIG:RelWithDebInfo>>)

set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS
  src/game/version.h
)

set(SERVER_EXECUTABLE server CACHE STRING "Name of the built server executable")

#########################################################################
# Compiler flags                                                        #
#########################################################################

function(add_c_compiler_flag_if_supported VARIABLE FLAG)
  if(ARGC GREATER 2)
    set(CHECKED_FLAG "${ARGV2}")
  else()
    set(CHECKED_FLAG "${FLAG}")
  endif()
  string(REGEX REPLACE "[^A-Za-z0-9]" "_" CONFIG_VARIABLE "FLAG_SUPPORTED${CHECKED_FLAG}")
  check_c_compiler_flag("${CHECKED_FLAG}" ${CONFIG_VARIABLE})
  if(${CONFIG_VARIABLE})
    if(${VARIABLE})
      set("${VARIABLE}" "${${VARIABLE}};${FLAG}" PARENT_SCOPE)
    else()
      set("${VARIABLE}" "${FLAG}" PARENT_SCOPE)
    endif()
  endif()
endfunction()

# Protect the stack pointer.
# -fstack-protector-all doesn't work on MinGW.
add_c_compiler_flag_if_supported(OUR_FLAGS -fstack-protector-all)

# Inaccurate floating point numbers cause problems on mingw-w64-gcc when
# compiling for x86, might cause problems elsewhere. So don't store floats
# in registers but keep them at higher accuracy.

if(TARGET_ARCH STREQUAL "x86")
  add_c_compiler_flag_if_supported(OUR_FLAGS -ffloat-store)
endif()
add_c_compiler_flag_if_supported(OUR_FLAGS_OWN -Wall)
if(CMAKE_VERSION VERSION_GREATER 3.3 OR CMAKE_VERSION VERSION_EQUAL 3.3)
  add_c_compiler_flag_if_supported(OUR_FLAGS_OWN
    $<$<COMPILE_LANGUAGE:C>:-Wdeclaration-after-statement>
    -Wdeclaration-after-statement
  )
endif()
add_c_compiler_flag_if_supported(OUR_FLAGS_OWN -Wextra)
add_c_compiler_flag_if_supported(OUR_FLAGS_OWN -Wno-unused-parameter)
add_c_compiler_flag_if_supported(OUR_FLAGS_OWN -Wno-missing-field-initializers)
add_c_compiler_flag_if_supported(OUR_FLAGS_OWN -Wformat=2) # Warn about format strings.
add_c_compiler_flag_if_supported(OUR_FLAGS_DEP -Wno-implicit-function-declaration)

if(CMAKE_GENERATOR STREQUAL "Ninja")
  add_c_compiler_flag_if_supported(OUR_FLAGS -fdiagnostics-color=always)
  add_c_compiler_flag_if_supported(OUR_FLAGS -fcolor-diagnostics)
endif()

check_c_compiler_flag("-O2;-Wp,-Werror;-D_FORTIFY_SOURCE=2" DEFINE_FORTIFY_SOURCE) # Some distributions define _FORTIFY_SOURCE by themselves.

########################################################################
# COMMON FUNCTIONS                                                     #
########################################################################

# Modified set_glob function: it now only performs the glob and sorts the result.
# It no longer takes manual file lists and does not warn about discrepancies.
function(set_glob VAR GLOBBING DIRECTORY) # ...
  file(${GLOBBING} GLOB_RESULT "${PROJECT_SOURCE_DIR}/${DIRECTORY}/*.c" "${PROJECT_SOURCE_DIR}/${DIRECTORY}/*.cpp")
  list(SORT GLOB_RESULT)
  set(${VAR} ${GLOB_RESULT} PARENT_SCOPE)
endfunction()

#########################################################################
# INITIALIZE TARGET LISTS                                               #
#########################################################################

set(TARGETS_OWN)
set(TARGETS_DEP)
set(TARGETS_LINK) # Targets with a linking stage.

#########################################################################
# DEPENDENCIES                                                          #
#########################################################################

function(set_extra_dirs_lib VARIABLE NAME)
  set("PATHS_${VARIABLE}_LIBDIR" PARENT_SCOPE)
  set("HINTS_${VARIABLE}_LIBDIR" PARENT_SCOPE)
  if(PREFER_BUNDLED_LIBS)
    set(TYPE HINTS)
  else()
    set(TYPE PATHS)
  endif()
  if(TARGET_BITS AND TARGET_OS)
    set(DIR "other/${NAME}/${TARGET_OS}/lib${TARGET_BITS}")
    set("${TYPE}_${VARIABLE}_LIBDIR" "${DIR}" PARENT_SCOPE)
    set("EXTRA_${VARIABLE}_LIBDIR" "${DIR}" PARENT_SCOPE)
  endif()
endfunction()

function(set_extra_dirs_include VARIABLE NAME LIBRARY)
  set("PATHS_${VARIABLE}_INCLUDEDIR" PARENT_SCOPE)
  set("HINTS_${VARIABLE}_INCLUDEDIR" PARENT_SCOPE)
  is_bundled(IS_BUNDLED "${LIBRARY}")
  if(IS_BUNDLED)
    set("HINTS_${VARIABLE}_INCLUDEDIR" "other/${NAME}/include" "other/${NAME}/include/${TARGET_OS}" PARENT_SCOPE)
  endif()
endfunction()

if(CMAKE_CROSSCOMPILING)
  set(CROSSCOMPILING_NO_CMAKE_SYSTEM_PATH NO_CMAKE_SYSTEM_PATH)
else()
  set(CROSSCOMPILING_NO_CMAKE_SYSTEM_PATH)
endif()

function(is_bundled VARIABLE PATH)
  if(PATH)
    string(FIND "${PATH}" "${PROJECT_SOURCE_DIR}" LOCAL_PATH_POS)
    if(LOCAL_PATH_POS EQUAL 0 AND TARGET_BITS AND TARGET_OS)
      set("${VARIABLE}" ON PARENT_SCOPE)
    else()
      set("${VARIABLE}" OFF PARENT_SCOPE)
    endif()
  else()
    set("${VARIABLE}" OFF PARENT_SCOPE)
  endif()
endfunction()

if(NOT CMAKE_CROSSCOMPILING)
  # Check for PkgConfig once so all the other `find_package` calls can do it
  # quietly.
  find_package(PkgConfig)
endif()
find_package(ZLIB)
find_package(Git)
find_package(PythonInterp)
find_package(Threads)

message(STATUS "******** Teeworlds ********")
message(STATUS "Target OS: ${TARGET_OS} ${TARGET_BITS}bit")
message(STATUS "Compiler: ${CMAKE_CXX_COMPILER}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

message(STATUS "Dependencies:")
function(show_dependency_status OUTPUT_NAME NAME)
  if(${NAME}_FOUND)
    if(${NAME}_BUNDLED)
      message(STATUS " * ${OUTPUT_NAME} not found (using bundled version)")
    else()
      message(STATUS " * ${OUTPUT_NAME} found")
    endif()
  else()
    message(STATUS " * ${OUTPUT_NAME} not found")
  endif()
endfunction()

show_dependency_status("PythonInterp" PYTHONINTERP)
show_dependency_status("Zlib" ZLIB)

if(NOT(PYTHONINTERP_FOUND))
  message(SEND_ERROR "You must install Python to compile Teeworlds")
endif()

#########################################################################
# COPY DATA                                                             #
#########################################################################

file(COPY maps DESTINATION .)
file(COPY autoexec.cfg DESTINATION .)

#########################################################################
# CODE GENERATION                                                       #
#########################################################################

function(chash output_file)
  add_custom_command(OUTPUT ${output_file}
    COMMAND ${PYTHON_EXECUTABLE} scripts/cmd5.py ${ARGN}
      > "${PROJECT_BINARY_DIR}/${output_file}"
    DEPENDS scripts/cmd5.py ${ARGN}
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
  )
endfunction()

function(generate_source output_file script_parameter)
  add_custom_command(OUTPUT ${output_file}
    COMMAND ${PYTHON_EXECUTABLE} scripts/compile.py ${script_parameter}
      > "${PROJECT_BINARY_DIR}/${output_file}"
    DEPENDS
      scripts/compile.py
      scripts/content.py
      scripts/datatypes.py
      scripts/network.py
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
  )
endfunction()

file(MAKE_DIRECTORY "${PROJECT_BINARY_DIR}/src/game/generated/")

chash("src/game/generated/nethash.cpp"
  "src/engine/shared/protocol.h"
  "${PROJECT_BINARY_DIR}/src/game/generated/protocol.h"
  "src/game/tuning.h"
  "src/game/gamecore.cpp"
)
generate_source("src/game/generated/protocol.cpp" "network_source")
generate_source("src/game/generated/protocol.h" "network_header")
generate_source("src/game/generated/server_data.cpp" "server_content_source")
generate_source("src/game/generated/server_data.h" "server_content_header")


#########################################################################
# SHARED                                                                #
#########################################################################

# Sources
# Now only glob for .c and .cpp files in the specified directories.
set_glob(BASE_SOURCES GLOB_RECURSE src/base)
set_glob(ENGINE_SHARED_SOURCES GLOB src/engine/shared)
set_glob(GAME_SHARED_SOURCES GLOB src/game)

# Explicitly add external library sources that are typically bundled and compiled
set(EXTERNAL_BUNDLED_SOURCES
    "${PROJECT_SOURCE_DIR}/src/engine/external/md5/md5.c"
    # Add any other bundled sources if they exist (e.g., SQLite, etc.)
)

set(ENGINE_GENERATED_SHARED "${PROJECT_BINARY_DIR}/src/game/generated/protocol.cpp" "${PROJECT_BINARY_DIR}/src/game/generated/protocol.h")
set(GAME_GENERATED_SHARED
  "${PROJECT_BINARY_DIR}/src/game/generated/nethash.cpp"
  "${PROJECT_BINARY_DIR}/src/game/generated/protocol.h"
)

set(DEPS ${DEP_MD5} ${ZLIB_DEP})

# Libraries
set(LIBS ${CMAKE_THREAD_LIBS_INIT} ${ZLIB_LIBRARIES} ${PLATFORM_LIBS})

# Targets
add_library(engine-shared EXCLUDE_FROM_ALL OBJECT ${BASE_SOURCES} ${ENGINE_SHARED_SOURCES} ${ENGINE_GENERATED_SHARED} ${EXTERNAL_BUNDLED_SOURCES})
add_library(game-shared EXCLUDE_FROM_ALL OBJECT ${GAME_SHARED_SOURCES} ${GAME_GENERATED_SHARED})
list(APPEND TARGETS_OWN engine-shared game-shared)

#########################################################################
# SERVER                                                                #
#########################################################################

# Sources
set_glob(ENGINE_SERVER_SOURCES GLOB src/engine/server)
set_glob(GAME_SERVER_SOURCES GLOB_RECURSE src/game/server)

set(GAME_GENERATED_SERVER
  "${PROJECT_BINARY_DIR}/src/game/generated/server_data.cpp"
  "${PROJECT_BINARY_DIR}/src/game/generated/server_data.h"
)
set(SERVER_SRC ${ENGINE_SERVER_SOURCES} ${GAME_SERVER_SOURCES} ${GAME_GENERATED_SERVER})

# Libraries
set(LIBS_SERVER ${LIBS})

# Target
set(TARGET_SERVER ${SERVER_EXECUTABLE})
add_executable(${TARGET_SERVER}
  ${DEPS}
  ${SERVER_SRC}
  ${SERVER_ICON}
  $<TARGET_OBJECTS:engine-shared>
  $<TARGET_OBJECTS:game-shared>
)
target_link_libraries(${TARGET_SERVER} ${LIBS_SERVER})
list(APPEND TARGETS_OWN ${TARGET_SERVER})
list(APPEND TARGETS_LINK ${TARGET_SERVER})

#########################################################################
# INSTALLATION                                                          #
#########################################################################

function(escape_regex VAR STRING)
  string(REGEX REPLACE "([][^$.+*?|()\\\\])" "\\\\\\1" ESCAPED "${STRING}")
  set(${VAR} ${ESCAPED} PARENT_SCOPE)
endfunction()

function(escape_backslashes VAR STRING)
  string(REGEX REPLACE "\\\\" "\\\\\\\\" ESCAPED "${STRING}")
  set(${VAR} ${ESCAPED} PARENT_SCOPE)
endfunction()

function(max_length VAR)
  set(MAX_LENGTH 0)
  foreach(str ${ARGN})
    string(LENGTH ${str} LENGTH)
    if(LENGTH GREATER MAX_LENGTH)
      set(MAX_LENGTH ${LENGTH})
    endif()
  endforeach()
  set(${VAR} ${MAX_LENGTH} PARENT_SCOPE)
endfunction()

# Tries to generate a list of regex that matches everything except the given
# parameters.
function(regex_inverted VAR)
  max_length(MAX_LENGTH ${ARGN})
  math(EXPR UPPER_BOUND "${MAX_LENGTH}-1")

  set(REMAINING ${ARGN})
  set(RESULT)

  foreach(i RANGE ${UPPER_BOUND})
    set(TEMP ${REMAINING})
    set(REMAINING)
    foreach(str ${TEMP})
      string(LENGTH ${str} LENGTH)
      if(i LESS LENGTH)
        list(APPEND REMAINING ${str})
      endif()
    endforeach()

    set(ADDITIONAL)
    foreach(outer ${REMAINING})
      string(SUBSTRING ${outer} 0 ${i} OUTER_PREFIX)
      set(CHARS "")
      foreach(inner ${REMAINING})
        string(SUBSTRING ${inner} 0 ${i} INNER_PREFIX)
        if(OUTER_PREFIX STREQUAL INNER_PREFIX)
          string(SUBSTRING ${inner} ${i} 1 INNER_NEXT)
          set(CHARS "${CHARS}${INNER_NEXT}")
        endif()
      endforeach()
      escape_regex(OUTER_PREFIX_ESCAPED "${OUTER_PREFIX}")

      list(APPEND ADDITIONAL "${OUTER_PREFIX_ESCAPED}([^${CHARS}]|$)")
    endforeach()
    list(REMOVE_DUPLICATES ADDITIONAL)
    list(APPEND RESULT ${ADDITIONAL})
  endforeach()
  set(${VAR} ${RESULT} PARENT_SCOPE)
endfunction()

set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
set(CPACK_GENERATOR TGZ TXZ)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)
set(CPACK_COMPONENTS_ALL portable)
set(CPACK_SOURCE_GENERATOR ZIP TGZ TBZ2 TXZ)
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
if(VERSION_PATCH STREQUAL "0")
  set(CPACK_PACKAGE_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR})
else()
  set(CPACK_PACKAGE_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})
endif()
set(CPACK_SYSTEM_NAME ${CMAKE_SYSTEM_NAME})

if(TARGET_OS AND TARGET_BITS)
  # Assuming Intel here.
  if(TARGET_BITS EQUAL 32)
    set(CPACK_SYSTEM_NAME "linux_x86")
  elseif(TARGET_BITS EQUAL 64)
    set(CPACK_SYSTEM_NAME "linux_x86_64")
  endif()
endif()

set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CPACK_SYSTEM_NAME})
set(CPACK_ARCHIVE_PORTABLE_FILE_NAME ${CPACK_PACKAGE_FILE_NAME})
set(CPACK_SOURCE_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-src)
set(CPACK_SOURCE_FILES
  CMakeLists.txt
  scripts/
  src/
)
set(CPACK_SOURCE_IGNORE_FILES
  "\\\\.o$"
  "\\\\.pyc$"
  "/\\\\.git"
  "/__pycache__/"
)

regex_inverted(CPACK_SOURCE_FILES_INVERTED ${CPACK_SOURCE_FILES})
escape_regex(PROJECT_SOURCE_DIR_ESCAPED ${PROJECT_SOURCE_DIR})

foreach(str ${CPACK_SOURCE_FILES_INVERTED})
  escape_backslashes(STR_ESCAPED "${PROJECT_SOURCE_DIR_ESCAPED}/${str}")
  list(APPEND CPACK_SOURCE_IGNORE_FILES "${STR_ESCAPED}")
endforeach()

set(CMAKE_INSTALL_DEFAULT_COMPONENT_NAME ${PROJECT_NAME})

set(CPACK_TARGETS
  ${TARGET_SERVER}
)
set(CPACK_DIRS data)
set(CPACK_FILES
  license.txt
  storage.cfg
  ${COPY_FILES}
)

if(NOT DEV)
  install(DIRECTORY data DESTINATION share/${PROJECT_NAME} COMPONENT data)
  install(TARGETS ${TARGET_SERVER} DESTINATION bin COMPONENT server)
endif()

if(DEV)
  # Don't generate CPack targets.
elseif(CMAKE_VERSION VERSION_LESS 3.6 OR CMAKE_VERSION VERSION_EQUAL 3.6)
  message(WARNING "Cannot create CPack targets, CMake version too old. Use CMake 3.6 or newer.")
else()
  set(EXTRA_ARGS DESTINATION ${CPACK_PACKAGE_FILE_NAME} COMPONENT portable EXCLUDE_FROM_ALL)
  install(TARGETS ${CPACK_TARGETS} ${EXTRA_ARGS})
  install(DIRECTORY ${CPACK_DIRS} ${EXTRA_ARGS})
  install(FILES ${CPACK_FILES} ${EXTRA_ARGS})
endif()


foreach(ext zip tar.gz tar.xz)
  set(TAR_MODE c)
  set(TAR_EXTRA_ARGS)
  string(REPLACE . _ EXT_SLUG ${ext})

  set(TMPDIR pack_${CPACK_PACKAGE_FILE_NAME}_${EXT_SLUG}/${CPACK_PACKAGE_FILE_NAME})

  set(COPY_FILE_COMMANDS)
  set(COPY_DIR_COMMANDS)
  set(COPY_TARGET_COMMANDS)
  set(STRIP_TARGET_COMMANDS)
  foreach(file ${CPACK_FILES})
    list(APPEND COPY_FILE_COMMANDS COMMAND ${CMAKE_COMMAND} -E copy ${PROJECT_SOURCE_DIR}/${file} ${TMPDIR}/)
  endforeach()
  foreach(dir ${CPACK_DIRS})
    list(APPEND COPY_DIR_COMMANDS COMMAND ${CMAKE_COMMAND} -E copy_directory ${PROJECT_SOURCE_DIR}/${dir} ${TMPDIR}/${dir})
  endforeach()
  foreach(target ${CPACK_TARGETS})
    list(APPEND COPY_TARGET_COMMANDS COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${target}> ${TMPDIR}/)
  endforeach()

  if(ext STREQUAL zip)
    set(TAR_EXTRA_ARGS --format=zip)
  elseif(ext STREQUAL tar.gz)
    set(TAR_MODE cz)
  elseif(ext STREQUAL tar.xz)
    set(TAR_MODE cJ)
  endif()
  add_custom_command(OUTPUT ${CPACK_PACKAGE_FILE_NAME}.${ext}
    COMMAND ${CMAKE_COMMAND} -E remove_directory ${TMPDIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${TMPDIR}
    ${COPY_FILE_COMMANDS}
    ${COPY_DIR_COMMANDS}
    ${COPY_TARGET_COMMANDS}
    ${STRIP_TARGET_COMMANDS}
    COMMAND ${CMAKE_COMMAND} -E chdir pack_${CPACK_PACKAGE_FILE_NAME}_${EXT_SLUG} ${CMAKE_COMMAND} -E tar ${TAR_MODE} ../${CPACK_PACKAGE_FILE_NAME}.${ext} ${TAR_EXTRA_ARGS} -- ${CPACK_PACKAGE_FILE_NAME}/
    DEPENDS ${CPACK_TARGETS}
  )
  add_custom_target(package_${EXT_SLUG} DEPENDS ${CPACK_PACKAGE_FILE_NAME}.${ext})
  list(APPEND PACKAGE_TARGETS package_${EXT_SLUG})
endforeach()

set(PACKAGE_DEFAULT tar_xz)
add_custom_target(package_default DEPENDS package_${PACKAGE_DEFAULT})

add_custom_target(package_all DEPENDS ${PACKAGE_TARGETS})

# Unset these variables, they might do something in the future of CPack.
unset(CPACK_SOURCE_FILES)
unset(CPACK_SOURCE_FILES_INVERTED)
unset(CPACK_TARGETS)
unset(CPACK_DIRS)
unset(CPACK_FILES)

include(CPack)

#########################################################################
# COMPILER-SPECIFICS                                                    #
#########################################################################

# In the future (CMake 3.8.0+), use source_group(TREE ...)
macro(source_group_tree dir)
  file(GLOB ents RELATIVE ${PROJECT_SOURCE_DIR}/${dir} ${PROJECT_SOURCE_DIR}/${dir}/*)
  foreach(ent ${ents})
    if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${dir}/${ent})
      source_group_tree(${dir}/${ent})
    else()
      string(REPLACE "/" "\\" group ${dir})
      source_group(${group} FILES ${PROJECT_SOURCE_DIR}/${dir}/${ent})
    endif()
  endforeach()
endmacro()
source_group_tree(src)

set(TARGETS ${TARGETS_OWN} ${TARGETS_DEP})

foreach(target ${TARGETS})
  if(OUR_FLAGS)
    target_compile_options(${target} PRIVATE ${OUR_FLAGS})
  endif()
  if(DEFINE_FORTIFY_SOURCE)
    target_compile_definitions(${target} PRIVATE $<$<NOT:$<CONFIG:Debug>>:_FORTIFY_SOURCE=2>) # Detect some buffer overflows.
  endif()
endforeach()

foreach(target ${TARGETS_LINK})
  if((MINGW) AND PREFER_BUNDLED_LIBS)
    # Statically link the standard libraries with on MinGW/Linux so we don't
    # have to ship them as DLLs.
    target_link_libraries(${target} -static-libgcc)
    target_link_libraries(${target} -static-libstdc++)
    if(MINGW)
      # Link pthread library statically instead of dynamically.
      # Solution from https://stackoverflow.com/a/28001261.
      target_link_libraries(${target} -Wl,-Bstatic -lstdc++ -lpthread -Wl,-Bdynamic)
    endif()
  endif()
endforeach()

foreach(target ${TARGETS_OWN})
  if(OUR_FLAGS_OWN)
    target_compile_options(${target} PRIVATE ${OUR_FLAGS_OWN})
  endif()
  target_include_directories(${target} PRIVATE ${PROJECT_BINARY_DIR}/src)
  target_include_directories(${target} PRIVATE src)
  target_compile_definitions(${target} PRIVATE $<$<CONFIG:Debug>:CONF_DEBUG>)
  target_include_directories(${target} PRIVATE ${CURL_INCLUDE_DIRS})
  target_include_directories(${target} PRIVATE ${ZLIB_INCLUDE_DIRS})
endforeach()

foreach(target ${TARGETS_DEP})
  if(OUR_FLAGS_DEP)
    target_compile_options(${target} PRIVATE ${OUR_FLAGS_DEP})
  endif()
endforeach()
