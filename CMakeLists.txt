cmake_minimum_required(VERSION 3.12)
project(squirrel
  VERSION 3.2
  DESCRIPTION "The Squirrel programming language"
  HOMEPAGE_URL "http://squirrel-lang.org/"
  LANGUAGES C CXX
  )

option(DISABLE_STATIC "Avoid building/installing static libraries.")
option(LONG_OUTPUT_NAMES "Use longer names for binaries and libraries: squirrel3 (not sq).")

if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif ()

include(GNUInstallDirs)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
set(CMAKE_CXX_STANDARD 11)

if(CMAKE_COMPILER_IS_GNUCXX)
  add_compile_options(
    "$<$<COMPILE_LANGUAGE:CXX>:-fno-rtti;-fno-exceptions>"
    -fno-strict-aliasing
    -Wall
    -Wextra
    -pedantic
    -Wcast-qual
    "$<$<CONFIG:Release>:-O3>"
    "$<$<CONFIG:RelWithDebInfo>:-O3;-g>"
    "$<$<CONFIG:MinSizeRel>:-Os>"
    "$<$<CONFIG:Debug>:-pg;-pie;-gstabs;-g3;-Og>"
    )
elseif(MSVC)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

# Need to be defined here to be visible to squirrel.pc.in
if(LONG_OUTPUT_NAMES)
  set(LIBSQUIRREL_NAME squirrel3)
  set(SQSTDLIB_NAME sqstdlib3)
else()
  set(LIBSQUIRREL_NAME squirrel)
  set(SQSTDLIB_NAME sqstdlib)
endif()

add_subdirectory(squirrel)
add_subdirectory(sqstdlib)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(tgts)
  if(NOT DISABLE_DYNAMIC)
    list(APPEND tgts squirrel sqstdlib)
  endif()
  if(NOT DISABLE_STATIC)
    list(APPEND tgts squirrel_static sqstdlib_static)
  endif()
  foreach(t ${tgts})
    target_compile_definitions(${t} PUBLIC -D_SQ64)
  endforeach()
endif()

if(NOT DISABLE_DYNAMIC)
  set_target_properties(squirrel sqstdlib PROPERTIES SOVERSION 0 VERSION 0.0.0)
endif()

if(NOT SQ_DISABLE_INSTALLER AND NOT SQ_DISABLE_HEADER_INSTALLER)
  install(FILES
    include/sqconfig.h
    include/squirrel.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    COMPONENT Development
    )
  install(FILES
    include/sqstdaux.h
    include/sqstdblob.h
    include/sqstdio.h
    include/sqstdmath.h
    include/sqstdstring.h
    include/sqstdsystem.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    COMPONENT Development
    )
endif()

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}/cmake/squirrel/squirrel-config-version.cmake"
  VERSION "${squirrel_VERSION}"
  COMPATIBILITY AnyNewerVersion
  )

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/squirrel-config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}/cmake/squirrel/squirrel-config.cmake"
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/squirrel"
  )

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/squirrel.pc.in"
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
  @ONLY
)

if(NOT SQ_DISABLE_INSTALLER AND NOT SQ_DISABLE_CMAKE_INSTALLER)
  export(EXPORT squirrel
    NAMESPACE squirrel::
    FILE "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}/cmake/squirrel/squirrel-targets.cmake"
    )

  install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}/cmake/squirrel/squirrel-config-version.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}/cmake/squirrel/squirrel-config.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/squirrel"
    COMPONENT Development
    )

  # pc(5) only allows static variant to have extra flags, not an entirely new
  # set of flags. While it's uncommon, a way to do it would be to generate a
  # ${PROJECT_NAME}_static.pc file.
  if(NOT DISABLE_DYNAMIC)
    install(
      FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
      COMPONENT Development
    )
  endif()

  install(EXPORT squirrel
    NAMESPACE squirrel::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/squirrel"
    FILE "squirrel-targets.cmake"
    COMPONENT Development
    )
endif()
