cmake_minimum_required(VERSION 3.0.0)

# * https://github.com/hunter-packages/gate
# * https://github.com/ruslo/hunter
include("${CMAKE_CURRENT_LIST_DIR}/cmake/HunterGate.cmake")

HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.19.51.tar.gz"
    SHA1 "d238dc1dd4db83e45a592f96fdb95d17c688600a"
)

project(GLFM VERSION 1.0.0)

option(GLFM_BUILD_EXAMPLE "Build the GLFM example" OFF)
option(GLFM_INSTALL "Install the targets" ON)

set(GLFM_HEADERS include/glfm.h)

if (CMAKE_SYSTEM_NAME MATCHES "Emscripten")
    set(GLFM_SRC src/glfm_platform.h src/glfm_platform_emscripten.c)
elseif (CMAKE_SYSTEM_NAME MATCHES "Android")
    set(GLFM_SRC src/glfm_platform.h src/glfm_platform_android.c ${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c)
    # Set NDEBUG for android_native_app_glue to remove some superfluous logging
    set_source_files_properties(${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c PROPERTIES COMPILE_FLAGS "-DNDEBUG")
else() # Assume iOS
    set(IOS TRUE)
    set(GLFM_SRC src/glfm_platform.h src/glfm_platform_ios.m)
    set(CMAKE_OSX_SYSROOT "iphoneos")
    set(GLFM_COMPILE_FLAGS "-Wno-objc-interface-ivars -Wno-objc-missing-property-synthesis -Wno-direct-ivar-access")
endif()

add_library(glfm ${GLFM_SRC} ${GLFM_HEADERS})

target_include_directories(glfm
  PUBLIC
  "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
  PRIVATE
  "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
)

source_group(include FILES ${GLFM_HEADERS})
source_group(src FILES ${GLFM_SRC})

if (CMAKE_C_COMPILER_ID MATCHES "Clang")
    set_target_properties(glfm PROPERTIES COMPILE_FLAGS "-Weverything -Wwrite-strings -Wno-padded -Wno-covered-switch-default ${GLFM_COMPILE_FLAGS}")
elseif (CMAKE_C_COMPILER_ID MATCHES "GNU")
    set_target_properties(glfm PROPERTIES COMPILE_FLAGS "-Wall -Wextra -Wwrite-strings ${GLFM_COMPILE_FLAGS}")
elseif (CMAKE_C_COMPILER_ID MATCHES "MSVC")
    set_target_properties(glfm PROPERTIES COMPILE_FLAGS "/Wall ${GLFM_COMPILE_FLAGS}")
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Android")

    include (GenerateExportHeader)
    generate_export_header(glfm)
    set(glfm_export_header "${CMAKE_CURRENT_BINARY_DIR}/src/glfm_export.hpp")
    generate_export_header(glfm EXPORT_FILE_NAME "${glfm_export_header}")

    find_package(android_log REQUIRED)
    find_package(android REQUIRED)
    find_package(egl REQUIRED)
    find_package(gles2 REQUIRED)

    target_link_libraries(glfm PUBLIC android_log::android_log android::android egl::egl gles2::gles2)
    target_include_directories(glfm
      PRIVATE
      "$<BUILD_INTERFACE:${ANDROID_NDK}/sources/android/native_app_glue>"
      "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/src>"
      )

elseif (IOS)
    find_package(foundation REQUIRED)
    find_package(coregraphics REQUIRED)
    find_package(uikit REQUIRED)
    find_package(opengles REQUIRED)
    find_package(quartzcore REQUIRED)

    target_link_libraries(
        glfw
        PUBLIC
        foundation::foundation
        coregraphics::coregraphics
        uikit::uikit
        opengles::opengles
        quartzcore::quartzcore
    )
    set_target_properties(glfm PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/GLFM.build/lib # For Archiving
        XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET 8.0
        XCODE_ATTRIBUTE_CLANG_ENABLE_OBJC_ARC YES
    )
endif()

if (GLFM_BUILD_EXAMPLE)
  add_subdirectory(example)
endif()

###############
### Install ###
###############

if(NOT GLFM_INSTALL)
  return()
endif()

### Install ###
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Note: use 'targets_export_name'
configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in"
    "${project_config}"
     INSTALL_DESTINATION "${config_install_dir}"
)

install(
    TARGETS glfm
    EXPORT "${targets_export_name}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

install(
    FILES ${GLFM_HEADERS}
    DESTINATION "${include_install_dir}"
)

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
  )

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
  )
