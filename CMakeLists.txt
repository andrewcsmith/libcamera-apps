cmake_minimum_required(VERSION 3.6)

if (NOT EXISTS ${CMAKE_BINARY_DIR}/CMakeCache.txt)
  if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
    message(STATUS "No previous build - default to Release build")
  endif()
endif()

set (CMAKE_EXPORT_COMPILE_COMMANDS ON)
set (CMAKE_CXX_STANDARD 17)
set (CMAKE CXX_FLAGS "-Wall -Wextra -pedantic -Wno-unused-parameter -faligned-new")
add_definitions(-Wfatal-errors)
add_definitions(-Wno-psabi)
add_definitions(-DBOOST_LOG_DYN_LINK)

IF (NOT ENABLE_COMPILE_FLAGS_FOR_TARGET)
  execute_process(COMMAND dpkg-architecture -qDEB_HOST_ARCH
    OUTPUT_VARIABLE ENABLE_COMPILE_FLAGS_FOR_TARGET OUTPUT_STRIP_TRAILING_WHITESPACE)
  if ("${ENABLE_COMPILE_FLAGS_FOR_TARGET}" STREQUAL "armhf")
    execute_process(COMMAND grep neon /proc/cpuinfo
      OUTPUT_VARIABLE NEON_PRESENT OUTPUT_STRIP_TRAILING_WHITESPACE)
    if (NOT NEON_PRESENT)
      set(ENABLE_COMPILE_FLAGS_FOR_TARGET "none")
    endif()
  endif()
endif()
message(STATUS "Optimising for platform: ${ENABLE_COMPILE_FLAGS_FOR_TARGET}")
if ("${ENABLE_COMPILE_FLAGS_FOR_TARGET}" STREQUAL "armhf")
  add_definitions(-mfpu=neon-fp-armv8 -ftree-vectorize)
elseif ("${ENABLE_COMPILE_FLAGS_FOR_TARGET}" STREQUAL "arm64")
  add_definitions(-ftree-vectorize)
endif()

project(common)
add_library(common output.cpp file_output.cpp net_output.cpp circular_output.cpp egl_preview.cpp drm_preview.cpp jpeg.cpp)

project(encoders)
add_library(encoders encoder.cpp null_encoder.cpp h264_encoder.cpp mjpeg_encoder.cpp)

project(libcamera-still)

find_package(PkgConfig REQUIRED)
pkg_check_modules(CAMERA REQUIRED camera)
pkg_check_modules(LIBDRM REQUIRED libdrm)

include_directories(. "${CAMERA_INCLUDE_DIRS}" "${LIBDRM_INCLUDE_DIRS}")

find_package( Boost REQUIRED COMPONENTS program_options )

find_library(EXIF_LIBRARY exif REQUIRED)
find_library(JPEG_LIBRARY jpeg REQUIRED)
find_library(TIFF_LIBRARY tiff REQUIRED)
find_library(PNG_LIBRARY png REQUIRED)
find_library(LIBCAMERA_LIBRARY libcamera.so REQUIRED)
message(STATUS "${LIBCAMERA_LIBRARY}")
find_package(X11 REQUIRED)
message(STATUS "${X11_INCLUDE_DIR}")
find_library(EPOXY_LIBRARY libepoxy.so REQUIRED)
message(STATUS "${EPOXY_LIBRARY}")
find_library(DRM_LIBRARY libdrm.so REQUIRED)
message(STATUS "${DRM_LIBRARY}")

add_executable(libcamera-still libcamera_still.cpp yuv.cpp dng.cpp png.cpp bmp.cpp)
target_link_libraries(libcamera-still common "${LIBCAMERA_LIBRARY}" exif ${Boost_LIBRARIES} jpeg tiff png pthread "${X11_LIBRARIES}" "${EPOXY_LIBRARY}" "${DRM_LIBRARY}")

project(libcamera-vid)

add_executable(libcamera-vid libcamera_vid.cpp)
target_link_libraries(libcamera-vid encoders common "${LIBCAMERA_LIBRARY}" exif ${Boost_LIBRARIES} pthread jpeg "${X11_LIBRARIES}" "${EPOXY_LIBRARY}" "${DRM_LIBRARY}")

project(libcamera-hello)

add_executable(libcamera-hello libcamera_hello.cpp)
target_link_libraries(libcamera-hello common "${LIBCAMERA_LIBRARY}" exif ${Boost_LIBRARIES} jpeg pthread "${X11_LIBRARIES}" "${EPOXY_LIBRARY}" "${DRM_LIBRARY}")

project(libcamera-raw)

add_executable(libcamera-raw libcamera_raw.cpp)
target_link_libraries(libcamera-raw encoders common "${LIBCAMERA_LIBRARY}" exif jpeg ${Boost_LIBRARIES} pthread "${X11_LIBRARIES}" "${EPOXY_LIBRARY}" "${DRM_LIBRARY}")

project(libcamera-jpeg)

add_executable(libcamera-jpeg libcamera_jpeg.cpp)
target_link_libraries(libcamera-jpeg common "${LIBCAMERA_LIBRARY}" exif jpeg ${Boost_LIBRARIES} pthread "${X11_LIBRARIES}" "${EPOXY_LIBRARY}" "${DRM_LIBRARY}")

install(TARGETS common encoders LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
install(TARGETS libcamera-still libcamera-vid libcamera-hello libcamera-raw libcamera-jpeg RUNTIME DESTINATION bin)
