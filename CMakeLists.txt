cmake_minimum_required(VERSION 3.21)

project(wxFormBuilder VERSION 3.9.0 LANGUAGES CXX)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

if(PROJECT_IS_TOP_LEVEL)
  set(buildTypes "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
  set(defaultBuildType "Debug")
  get_property(isMultiConfig GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
  if(NOT isMultiConfig)
    if(NOT CMAKE_BUILD_TYPE)
      message(STATUS "Setting build type to '${defaultBuildType}' as none was specified.")
      set(CMAKE_BUILD_TYPE "${defaultBuildType}" CACHE STRING "Choose the type of build." FORCE)
      set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "${buildTypes}")
    elseif(NOT CMAKE_BUILD_TYPE IN_LIST buildTypes)
      message(FATAL_ERROR "Unknown build type: '${CMAKE_BUILD_TYPE}'")
    endif()
  endif()
endif()

if(NOT DEFINED CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
elseif(CMAKE_CXX_STANDARD LESS 17)
  message(FATAL_ERROR "The CMAKE_CXX_STANDARD value needs to be at least 17, current value: ${CMAKE_CXX_STANDARD}")
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if (NOT DEFINED CMAKE_CXX_EXTENSIONS)
  set(CMAKE_CXX_EXTENSIONS OFF)
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)
if(APPLE)
  # TODO: The current plugin loader code requires this extension
  set(CMAKE_SHARED_MODULE_SUFFIX ".dylib")
endif()

if(MSVC OR MINGW AND NOT MSYS)
  set(wxfbWxWidgetsMonoDefault ON)
else()
  set(wxfbWxWidgetsMonoDefault OFF)
endif()
option(WXFB_WXWIDGETS_MONO "Use monolithic variant of wxWidgets." ${wxfbWxWidgetsMonoDefault})
option(WXFB_WXWIDGETS_SHARED "Use shared variant of wxWidgets." ON)
option(WXFB_TICPP_SHARED "Use shared variant of TiCPP." OFF)

set(wxfbWxWidgetsVersionMin 3.1.2)
# BUILD_SHARED_LIBS does only affect win32 search style, wxWidgets_USE_STATIC does only affect unix search style
if(DEFINED BUILD_SHARED_LIBS)
  set(backup_BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS})
endif()
set(BUILD_SHARED_LIBS ${WXFB_WXWIDGETS_SHARED})
if(WXFB_WXWIDGETS_SHARED)
  set(wxWidgets_USE_STATIC OFF)
else()
  set(wxWidgets_USE_STATIC ON)
endif()
if(WXFB_WXWIDGETS_MONO)
  find_package(wxWidgets ${wxfbWxWidgetsVersionMin} REQUIRED mono scintilla)
else()
  # TODO: OPTIONAL_COMPONENTS seems to be broken currently (at least on Linux), so make media a required component
  find_package(wxWidgets ${wxfbWxWidgetsVersionMin} REQUIRED media xrc richtext propgrid ribbon stc html aui xml net core base)
endif()
include(${wxWidgets_USE_FILE})
if(DEFINED backup_BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS ${backup_BUILD_SHARED_LIBS})
  unset(backup_BUILD_SHARED_LIBS)
else()
  unset(BUILD_SHARED_LIBS)
endif()
# TODO: Some Boost packages do not include the CMake config files so config mode with the headers component can't be used
find_package(Boost 1.74.0 REQUIRED)

# Adjust paths to match wxStandardPaths implementation
include(GNUInstallDirs)
if(WIN32)
  set(CMAKE_INSTALL_BINDIR .)
  set(CMAKE_INSTALL_LIBDIR .)
  set(CMAKE_INSTALL_DATADIR .)
elseif(APPLE)
  set(CMAKE_INSTALL_DATADIR wxFormBuilder.app/Contents/SharedSupport)
else()
  set(CMAKE_INSTALL_DATADIR ${CMAKE_INSTALL_DATADIR}/wxformbuilder)
endif()
include(cmake/plugins.cmake)

add_subdirectory(src)
add_subdirectory(plugins)
add_subdirectory(sdk/plugin_interface)
if(DEFINED BUILD_SHARED_LIBS)
  set(backup_BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS})
endif()
set(BUILD_SHARED_LIBS ${WXFB_TICPP_SHARED})
add_subdirectory(subprojects/ticpp)
if(DEFINED backup_BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS ${backup_BUILD_SHARED_LIBS})
  unset(backup_BUILD_SHARED_LIBS)
else()
  unset(BUILD_SHARED_LIBS)
endif()

if(APPLE)
  # TODO: Is there any possibility to determine the output filenames of the plugins to create a list that can be inserted literal?
  # TODO: The fixup_bundle command erases all present RPATH information. Since it fixes the loader paths of all libraries and none
  #       of them loads libraries dynamically, it is sufficient to add an RPATH only to the PlugIns directory which contains the
  #       libraries that get loaded dynamically.
  install(
    CODE
    "
      include(BundleUtilities)
      file(GLOB moduleDependencies LIST_DIRECTORIES false \"\${CMAKE_INSTALL_PREFIX}/wxFormBuilder.app/Contents/PlugIns/*${CMAKE_SHARED_MODULE_SUFFIX}\")
      fixup_bundle(\"\${CMAKE_INSTALL_PREFIX}/wxFormBuilder.app\" \"\${moduleDependencies}\" \"\")

      find_program(install_name_tool_cmd NAMES install_name_tool REQUIRED)
      execute_process(COMMAND \"\${install_name_tool_cmd}\" -add_rpath @executable_path/../PlugIns \"\${CMAKE_INSTALL_PREFIX}/wxFormBuilder.app/Contents/MacOS/wxFormBuilder\")

      find_program(codesign_cmd NAMES codesign REQUIRED)
      mark_as_advanced(codesign_cmd)
      execute_process(COMMAND \"\${codesign_cmd}\" -s - -f --deep --verbose \"\${CMAKE_INSTALL_PREFIX}/wxFormBuilder.app\")
    "
  )
endif()