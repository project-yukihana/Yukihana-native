cmake_minimum_required(VERSION 3.15)
cmake_policy(SET CMP0091 NEW)
cmake_policy(SET CMP0042 NEW)

project("nt_native")

set(VCPKG_ROOT $ENV{VCPKG_ROOT})
message("VCPKG_ROOT: " ${VCPKG_ROOT})
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CXX_STANDARD 17)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
set(CMAKE_VERBOSE_MAKEFILE ON)
set(PKG_TOOL pnpm)
set(TARGET_ARCH x64)
if(MSVC)
    set(VCPKG_TARGET_TRIPLET x64-windows-static)
    # set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG "${CMAKE_BINARY_DIR}/Debug")
    # set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/Release")
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()
set(CMAKE_TOOLCHAIN_FILE "${VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "Vcpkg toolchain file")
include("${VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake")

message("LD_LIBRARY_PATH:" ${LD_LIBRARY_PATH})

find_package(unofficial-sqlite3 CONFIG REQUIRED)
find_package(Protobuf CONFIG REQUIRED)
find_package(SqliteOrm CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)

find_program(PNPM_EXECUTABLE pnpm)
if(NOT PNPM_EXECUTABLE)
    message(FATAL_ERROR "pnpm not found!")
endif()
if(WIN32)
    set(PNPM_EXECUTABLE "${PNPM_EXECUTABLE}.CMD")
endif()

# message("安装 Node 包")
# execute_process(COMMAND
#         ${PKG_TOOL} install
#         WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
# )
message("CMAKE_CXX_FLAGS:${CMAKE_CXX_FLAGS}")
message("获取 cmake-js 配置")
message(${PROJECT_SOURCE_DIR})
execute_process(COMMAND
    ${PNPM_EXECUTABLE} --silent cmake-js --version
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE CMAKE_JS_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(COMMAND
    ${PNPM_EXECUTABLE} cmake-js print-cmakejs-src --arch ${TARGET_ARCH} --silent -l error
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE CMAKE_JS_SRC
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
message("获取 CMAKE_JS_SRC 配置: ${CMAKE_JS_SRC}")

message("PNPM_EXECUTABLE: " ${PNPM_EXECUTABLE})
execute_process(COMMAND
    ${PNPM_EXECUTABLE} cmake-js print-cmakejs-include --silent -l error
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE CMAKE_JS_INC
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
# string(REPLACE ";" "\;" CMAKE_JS_INC "${CMAKE_JS_INC}")
message("CMAKE_JS_INC:" ${CMAKE_JS_INC})
foreach(item IN LISTS CMAKE_JS_INC)
    message("include: ${item}")
    include_directories(${item})
endforeach()

message("获取 node-addon-api 配置")
execute_process(COMMAND
    node -p "require('node-addon-api').include"
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE NODE_ADDON_API_DIR
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
message("node-addon-api 路径:${NODE_ADDON_API_DIR}")

message("CMAKE_JS_INC: " ${CMAKE_JS_INC})
message("CMAKE_JS_SRC: " ${CMAKE_JS_SRC})
message("CMAKE_JS_LIB: " ${CMAKE_JS_LIB})

include_directories(${CMAKE_JS_SRC})
include_directories(${NODE_ADDON_API_DIR})

set(SUBHOOK_TESTS OFF)
set(SUBHOOK_STATIC ON)
set(SUBHOOK_INSTALL OFF)
add_subdirectory(thirds/subhook)

set(MODULE_NAME nt_native)
add_library(${MODULE_NAME}
    SHARED
    src/include/sqlite3/official/sqlite3.hh
    src/include/convert.hh
    src/include/hook.hh
    src/include/linux_hook.hh
    src/include/windows_hook.hh
    src/include/db/group_msg_table.hh
    src/proto/message.pb.h
    src/main.cc
    src/hook/hook.cc
    src/hook/linux_hook.cc
    src/hook/windows_hook.cc
    src/proto/message.pb.cc
    src/db/group_msg_table.cc
    src/proto/message.pb.cc
    src/yukihana/handle.cc
    src/yukihana/convert.cc
    src/include/db/model/group_msg_table.hh
    ${CMAKE_JS_SRC}
    )
if (WIN32)
    message("生成 QQ.lib")
    execute_process(COMMAND
        ${CMAKE_AR} /def:QQ.def /out:QQ.lib /machine:x86 ${CMAKE_STATIC_LINKER_FLAGS}
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/thirds/external
        COMMAND_ERROR_IS_FATAL ANY
    )
    execute_process(COMMAND
        ${CMAKE_AR} /def:QQ64.def /out:QQ64.lib /machine:x64 ${CMAKE_STATIC_LINKER_FLAGS}
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/thirds/external
        COMMAND_ERROR_IS_FATAL ANY
    )
    target_link_libraries(${MODULE_NAME} ${PROJECT_SOURCE_DIR}/thirds/external/QQ.lib)
    target_link_libraries(${MODULE_NAME} ${PROJECT_SOURCE_DIR}/thirds/external/QQ64.lib)
endif()
# target_link_libraries(${MODULE_NAME} PRIVATE unofficial::sqlite3::sqlite3)
if (WIN32)
    target_link_libraries(${MODULE_NAME} subhook)
    target_link_libraries(${MODULE_NAME} protobuf::libprotoc protobuf::libprotobuf protobuf::libprotobuf-lite)
    target_link_libraries(${MODULE_NAME} sqlite_orm::sqlite_orm)
    target_link_libraries(${MODULE_NAME} spdlog::spdlog)
else()
    target_link_libraries(${MODULE_NAME} PRIVATE subhook)
    target_link_libraries(${MODULE_NAME} PRIVATE protobuf::libprotoc protobuf::libprotobuf protobuf::libprotobuf-lite)
    target_link_libraries(${MODULE_NAME} PRIVATE sqlite_orm::sqlite_orm)
    target_link_libraries(${MODULE_NAME} PRIVATE spdlog::spdlog)
endif()
target_link_libraries(${MODULE_NAME} ${CMAKE_JS_LIB})

target_compile_definitions(${MODULE_NAME} PRIVATE $<$<CONFIG:Debug>:NATIVE_DEBUG=1> $<$<CONFIG:Debug>:SQLITE_DEBUG>)
set_target_properties(${MODULE_NAME} PROPERTIES PREFIX "" SUFFIX ".node")

################test##################

add_executable(test_sqlite
        test/test_sqlite3.cc
)
target_link_libraries(test_sqlite PRIVATE unofficial::sqlite3::sqlite3)
# target_link_libraries(test_sqlite PRIVATE sqlite3)

add_executable(test_protobuf
    test/test_protobuf.cc
    ./src/proto/message.pb.cc
)
# target_link_libraries(test_sqlite PRIVATE unofficial::sqlite3::sqlite3)
target_link_libraries(test_protobuf PRIVATE protobuf::libprotoc protobuf::libprotobuf protobuf::libprotobuf-lite)

add_executable(test_orm
        test/test_orm.cc
        src/hook/hook.cc
        src/hook/linux_hook.cc
        src/yukihana/handle.cc
        src/db/group_msg_table.cc
        src/proto/message.pb.cc
)
# target_link_libraries(test_sqlite PRIVATE unofficial::sqlite3::sqlite3)
target_link_libraries(test_orm PRIVATE sqlite_orm::sqlite_orm)
target_link_libraries(test_orm PRIVATE subhook)
target_link_libraries(test_orm PRIVATE spdlog::spdlog)
target_link_libraries(test_orm PRIVATE protobuf::libprotoc protobuf::libprotobuf protobuf::libprotobuf-lite)


add_executable(test
        test/test.cc
)