cmake_minimum_required(VERSION 3.16)

project(geodesic_examples
  VERSION 1.0
  LANGUAGES CXX)

# ===== Options =====
option(BUILD_EXAMPLES "Build example executables (example0, example1)" ON)
option(BUILD_MATLAB_API "Build legacy geodesic_matlab_api (Windows/Boost only)" OFF)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../bin")
foreach(cfg DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
  set("CMAKE_RUNTIME_OUTPUT_DIRECTORY_${cfg}" "${CMAKE_BINARY_DIR}/../bin")
endforeach()

# ===== C++ standard & warnings =====
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(MSVC)
  add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
  add_compile_options(/W4 /permissive- /wd4996 /Zc:strictStrings-)
else()
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# ===== Header-only interface 'library' (for include path) =====
add_library(geodesic_headers INTERFACE)
target_include_directories(geodesic_headers INTERFACE
  ${CMAKE_CURRENT_SOURCE_DIR})

# ===== Data files (copy next to the built executables) =====
set(GEODESIC_DATA_FILES
  hedgehog_mesh.txt
  flat_triangular_mesh.txt)

foreach(f IN LISTS GEODESIC_DATA_FILES)
  if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${f}")
    configure_file(${f} ${f} COPYONLY)
  endif()
endforeach()

# ===== Examples =====
if(BUILD_EXAMPLES)
  add_executable(example0 src/example0.cpp)
  target_link_libraries(example0 PRIVATE geodesic_headers)

  add_executable(example1 src/example1.cpp)
  target_link_libraries(example1 PRIVATE geodesic_headers)

  # macOS の実行バンドルではなく通常のCLIとして扱う
  if(APPLE)
    set_target_properties(example0 example1 PROPERTIES
      MACOSX_BUNDLE OFF
      XCODE_ATTRIBUTE_CODE_SIGNING_ALLOWED "NO")
  endif()
endif()

# ===== (Optional) Legacy MATLAB API target =====
# 注意:
# - ソースは Windows 向けに __declspec(dllexport) と <boost\shared_ptr.hpp> の
#   バックスラッシュ表記を含むため、基本的に Windows + Boost でのみビルド可能です。
# - 他OSでビルドする場合はソース側の `#include <boost\shared_ptr.hpp>` に
#   スラッシュへ修正する等のパッチが必要です。
if(BUILD_MATLAB_API)
  if(WIN32)
    # Boostはヘッダのみ(shared_ptr)なので find_package は省略可能。
    # 既にインストール済みの Boost の include パスが必要なら:
    # set(BOOST_ROOT "C:/local/boost_1_84_0") などを事前に設定してください。
    add_library(geodesic_matlab_api SHARED geodesic_matlab_api.cpp)
    target_link_libraries(geodesic_matlab_api PRIVATE geodesic_headers)
    # DLL の出力名など調整したければここで set_target_properties を設定
  else()
    message(FATAL_ERROR "BUILD_MATLAB_API is only supported on Windows unless you patch the source.")
  endif()
endif()

# ===== Install (optional) =====
include(GNUInstallDirs)
if(BUILD_EXAMPLES)
  install(TARGETS example0 example1 RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
  install(FILES ${GEODESIC_DATA_FILES} DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

set(GEODESIC_DATA_FILES
  "${CMAKE_SOURCE_DIR}/src/hedgehog_mesh.txt"
  "${CMAKE_SOURCE_DIR}/src/flat_triangular_mesh.txt"
)

foreach(t IN ITEMS example0 example1)
  foreach(f IN LISTS GEODESIC_DATA_FILES)
    if(EXISTS "${f}")
      add_custom_command(TARGET ${t} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
          "${f}"
          "$<TARGET_FILE_DIR:${t}>/"
      )
    endif()
  endforeach()
endforeach()