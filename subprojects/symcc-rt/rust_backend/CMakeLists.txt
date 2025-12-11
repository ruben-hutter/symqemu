# This file is part of SymCC.
#
# SymCC is free software: you can redistribute it and/or modify it under the
# terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or (at your option) any later
# version.
#
# SymCC is distributed in the hope that it will be useful, but WITHOUT ANY
# WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR
# A PARTICULAR PURPOSE. See the GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License along with
# SymCC. If not, see <https://www.gnu.org/licenses/>.

# Create both static and shared libraries with SymQEMU-compatible names
add_library(SymCCRtObj OBJECT
  ${SHARED_RUNTIME_SOURCES}
  Runtime.cpp)

set_property(TARGET SymCCRtObj PROPERTY POSITION_INDEPENDENT_CODE 1)

add_library(SymCCRtShared SHARED $<TARGET_OBJECTS:SymCCRtObj>)
add_library(SymCCRtStatic STATIC $<TARGET_OBJECTS:SymCCRtObj>)

# Also create SymRuntime alias for compatibility
add_library(SymRuntime ALIAS SymCCRtStatic)

target_include_directories(SymCCRtObj PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/..)

set_target_properties(SymCCRtObj PROPERTIES COMPILE_FLAGS "-Werror")

# Link with LibAFL's Rust runtime which provides the _rsym_* functions
# The rust_backend is just a C++ wrapper, the actual implementation is in Rust
# User must provide LIBAFL_RUNTIME_DIR pointing to the directory containing libSymRuntime.so
if(DEFINED ENV{LIBAFL_RUNTIME_DIR})
  message(STATUS "Linking rust_backend with LibAFL runtime from: $ENV{LIBAFL_RUNTIME_DIR}")
  set(LIBAFL_SYMRUNTIME_PATH "$ENV{LIBAFL_RUNTIME_DIR}/libSymRuntime.so")
  if(EXISTS "${LIBAFL_SYMRUNTIME_PATH}")
    message(STATUS "Found LibAFL runtime: ${LIBAFL_SYMRUNTIME_PATH}")
    # Use link options instead of target_link_libraries to pass the full .so path
    # This prevents Meson from trying to add -lSymRuntime
    target_link_options(SymCCRtShared PRIVATE "${LIBAFL_SYMRUNTIME_PATH}")
    # Also need to set RPATH so the library can be found at runtime
    set_target_properties(SymCCRtShared PROPERTIES 
      BUILD_RPATH "$ENV{LIBAFL_RUNTIME_DIR}"
      INSTALL_RPATH "$ENV{LIBAFL_RUNTIME_DIR}")
  else()
    message(FATAL_ERROR "LibAFL runtime not found at: ${LIBAFL_SYMRUNTIME_PATH}")
  endif()
  # Note: SymCCRtStatic cannot link with the dynamic Rust runtime
  # It's provided for compatibility but won't work without manual linking
else()
  message(FATAL_ERROR "LIBAFL_RUNTIME_DIR not set. You must build LibAFL's runtime and set this env var. Example: export LIBAFL_RUNTIME_DIR=/path/to/runtime/target/release")
endif()

INSTALL(TARGETS SymCCRtShared SymCCRtStatic)