cmake_minimum_required(VERSION 3.10)
project(ros_workspace_shell NONE)

# Generates the same setup files in the root of the install prefix as
# ament_cmake does, from the copies of the ament_package prefix level
# templates in templates/, and the environment of the prefix as a dsv file
# that those setup files read, so that neither building nor using this
# package needs any ament package.

if(WIN32)
  message(FATAL_ERROR "ros_workspace_shell supports Linux and macOS only")
endif()

# REP 159: the install prefixes of the distributions this one extends with
# binary_import, nearest first. bloom passes them when releasing into such a
# distribution; a root distribution leaves the list empty.
set(ROS_WORKSPACE_PARENT_PREFIX_PATH "" CACHE STRING
  "Install prefixes of the binary_import parent distributions, nearest first")

find_package(Python3 REQUIRED COMPONENTS Interpreter)
# default interpreter written into the prefix level local_setup files
set(ament_package_PYTHON_EXECUTABLE "${Python3_EXECUTABLE}")
set(PYTHON_INSTALL_DIR
  "lib/python${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR}/site-packages")

# enable C language so that a trycompile can determine what the
# anticipated libdir will be.
enable_language(C)
include(GNUInstallDirs)

set(templates "${CMAKE_CURRENT_SOURCE_DIR}/templates")
set(gen "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(share "share/${PROJECT_NAME}")

# Copy a file into the build tree, expanding it when it ends in .in, and
# return the path of the result in dest_var.
function(_stage_file src dest_dir dest_var)
  get_filename_component(name "${src}" NAME)
  if(name MATCHES "^(.*)\\.in$")
    set(name "${CMAKE_MATCH_1}")
    configure_file("${src}" "${dest_dir}/${name}" @ONLY)
  else()
    configure_file("${src}" "${dest_dir}/${name}" COPYONLY)
  endif()
  set(${dest_var} "${dest_dir}/${name}" PARENT_SCOPE)
endfunction()

# --- environment of the prefix
# The setup files run _local_setup_util.py, which reads
# share/<package>/package.dsv for every package of the prefix and applies
# its lines. ament packages list shell scripts there, each with a dsv file
# next to it, and package level local_setup scripts; the setup files only
# read the dsv files, so this package writes the lines directly, in the
# order ament_cmake sources its hooks.
if(APPLE)
  set(LIBRARY_PATH_ENV_VAR "DYLD_LIBRARY_PATH")
else()
  set(LIBRARY_PATH_ENV_VAR "LD_LIBRARY_PATH")
endif()
set(package_dsv "prepend-non-duplicate;${LIBRARY_PATH_ENV_VAR};lib\n")
if(NOT CMAKE_INSTALL_LIBDIR STREQUAL "lib")
  # multiarch library directory, prepended after lib so that it comes first
  string(APPEND package_dsv
    "prepend-non-duplicate;${LIBRARY_PATH_ENV_VAR};${CMAKE_INSTALL_LIBDIR}\n")
endif()
string(APPEND package_dsv
  "prepend-non-duplicate-if-exists;PATH;bin\n"
  "prepend-non-duplicate;PYTHONPATH;${PYTHON_INSTALL_DIR}\n")
file(WRITE "${gen}/package.dsv" "${package_dsv}")
install(FILES "${gen}/package.dsv" package.xml DESTINATION "${share}")

# --- resource index entries read by the prefix level setup files
if(ROS_WORKSPACE_PARENT_PREFIX_PATH)
  # REP 159: record the parents as this package's parent_prefix_path
  # resource, which the prefix level setup files read to source each
  # parent's local_setup first.
  string(REPLACE ";" ":" parent_prefix_path "${ROS_WORKSPACE_PARENT_PREFIX_PATH}")
  set(SKIP_PARENT_PREFIX_PATH "")
else()
  set(parent_prefix_path "")
  # skip using ament_index/resource_index/parent_prefix_path
  # if for Debian packages it is known that there are no underlays
  # note the template only checks for an empty (use) / non-empty (skip) string
  set(SKIP_PARENT_PREFIX_PATH "SKIP_PARENT_PREFIX_PATH")
endif()
set(index "share/ament_index/resource_index")
file(WRITE "${gen}/index/packages/${PROJECT_NAME}" "")
file(WRITE "${gen}/index/package_run_dependencies/${PROJECT_NAME}" "")
file(WRITE "${gen}/index/parent_prefix_path/${PROJECT_NAME}" "${parent_prefix_path}")
foreach(resource packages package_run_dependencies parent_prefix_path)
  install(FILES "${gen}/index/${resource}/${PROJECT_NAME}"
    DESTINATION "${index}/${resource}")
endforeach()

# --- setup files in the root of the install prefix
foreach(name
    local_setup.bash local_setup.fish.in local_setup.sh.in _local_setup_util.py
    local_setup.zsh setup.bash setup.fish setup.sh.in setup.zsh)
  _stage_file("${templates}/prefix_level/${name}" "${gen}/prefix" staged)
  install(FILES "${staged}" DESTINATION .)
endforeach()
