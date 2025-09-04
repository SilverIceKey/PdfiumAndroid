# CMake ≥3.22 随你原项目
cmake_minimum_required(VERSION 3.22.1)
project(jniPdfium LANGUAGES C CXX)

# === 版本/编译选项（对应 APP_CPPFLAGS += -fexceptions）===
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS OFF)
# 若外层（Gradle）未指定 minSdk，可在这里兜底成 android-16
if(NOT DEFINED ANDROID_PLATFORM AND NOT DEFINED CMAKE_ANDROID_API)
    set(CMAKE_ANDROID_API 26)
endif()

# === 预编译库路径（与 mk 中 $(LOCAL_PATH)/lib/$(ARCH_PATH) 对齐）===
set(_LIBDIR "${CMAKE_CURRENT_LIST_DIR}/lib/${ANDROID_ABI}")

# aospPdfium  (libmodpdfium.so)
add_library(aospPdfium SHARED IMPORTED)
set_target_properties(aospPdfium PROPERTIES
        IMPORTED_LOCATION "${_LIBDIR}/libmodpdfium.so")

# libc++_shared1（如果你就是要随 APK 打包这个重命名的 C++ 运行库）
add_library(modcxx_shared1 SHARED IMPORTED)
set_target_properties(modcxx_shared1 PROPERTIES
        IMPORTED_LOCATION "${_LIBDIR}/libc++_shared1.so")

# libmodft2.so
add_library(modft2 SHARED IMPORTED)
set_target_properties(modft2 PROPERTIES
        IMPORTED_LOCATION "${_LIBDIR}/libmodft2.so")

# libmodpng.so
add_library(modpng SHARED IMPORTED)
set_target_properties(modpng PROPERTIES
        IMPORTED_LOCATION "${_LIBDIR}/libmodpng.so")

# === 主 JNI 动态库（对应 jniPdfium）===
add_library(${PROJECT_NAME} SHARED
        src/mainJNILib.cpp
        src/util.hpp
)

# 对应 LOCAL_CFLAGS += -DHAVE_PTHREADS 与 LOCAL_C_INCLUDES += include
target_compile_definitions(${PROJECT_NAME} PRIVATE HAVE_PTHREADS)
target_compile_options(${PROJECT_NAME} PRIVATE -fexceptions)
target_include_directories(${PROJECT_NAME} PRIVATE
        "include"
)

# === NDK 系统库（-llog -landroid -ljnigraphics）===
find_library(log-lib         log)
find_library(android-lib     android)
find_library(jnigraphics-lib jnigraphics)
# === 链接（把所有预编译 .so 和系统库都绑上）===
target_link_libraries(${PROJECT_NAME}
        aospPdfium          # libmodpdfium.so
        modft2              # libmodft2.so
        modpng              # libmodpng.so
        modcxx_shared1      # 你自带的 libc++_shared1.so（如果用 NDK 自带就删掉它）
        ${android-lib}
        ${jnigraphics-lib}
        ${log-lib}
)

# === 连接（mk 里只显式链接 aospPdfium；这里把依赖库也连上以便被打包进 APK）=
