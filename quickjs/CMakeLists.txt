cmake_minimum_required(VERSION 3.9.4)
project(QUICKJS C)

# Build type: "Debug" or "Release"
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(COMPILE_FLAGS ${COMPILE_FLAGS} -g)
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(COMPILE_FLAGS ${COMPILE_FLAGS} -O3)
endif()

set(C_STANDARD 17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(MI_OVERRIDE OFF)

# interprocedural optimization (IPO/LTO)
include(CheckIPOSupported)
check_ipo_supported(RESULT supported OUTPUT error)
if( supported AND NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
   set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

include_directories(${PROJECT_SOURCE_DIR})
include_directories(${PROJECT_SOURCE_DIR}/include)
include_directories(${PROJECT_SOURCE_DIR}/vendor/mimalloc/include)
add_subdirectory(src)
add_subdirectory(vendor/mimalloc)

set(COMPILE_FLAGS -Wall -MMD -Wno-array-bounds -Wno-format-truncation)

add_compile_options(${COMPILE_FLAGS})
add_compile_definitions(
    _GNU_SOURCE
    CONFIG_BIGNUM
    CONFIG_VERSION=${QUICKJS_VERSION}
)

set(QJSC_CONFIG -DCONFIG_PREFIX="/usr/local" -DCONFIG_LTO)
set(QJSC_EXE "${EXECUTABLE_OUTPUT_PATH}/qjsc")
set(QJS_CONFIG ${QJSC_CONFIG} -DCONFIG_CC="clang")

add_executable(qjsc qjsc.c quickjs-libc.c)

# Add -lm if on Linux to handle math.h
# See: https://askubuntu.com/a/332919
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    list(APPEND LINK_LIBRARIES "m")
endif()

target_link_libraries(qjsc quickjs ${LINK_LIBRARIES} mimalloc-static)
target_compile_definitions(qjsc PUBLIC ${QJSC_CONFIG})

add_custom_command(
    TARGET qjsc POST_BUILD
    COMMAND ${QJSC_EXE} -c -o ${PROJECT_SOURCE_DIR}/repl.c -m ${PROJECT_SOURCE_DIR}/repl.js
    COMMAND ${QJSC_EXE} -fbignum -c -o ${PROJECT_SOURCE_DIR}/qjscalc.c ${PROJECT_SOURCE_DIR}/qjscalc.js
)

add_executable(qjs qjs.c quickjs-libc.c repl.c qjscalc.c)
target_link_libraries(qjs quickjs ${LINK_LIBRARIES} mimalloc-static)

add_executable(run-test262 run-test262.c quickjs-libc.c)
target_link_libraries(run-test262 quickjs ${LINK_LIBRARIES} mimalloc-static)
