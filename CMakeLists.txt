cmake_minimum_required (VERSION 3.0)
project (s2n)
##header files
file(GLOB API_HEADERS
     "api/*.h"
)

file(GLOB CRYPTO_HEADERS
    "crypto/*.h"
)

file(GLOB ERROR_HEADERS
    "error/*.h"
)

file(GLOB STUFFER_HEADERS
    "stuffer/*.h"
)

file(GLOB TLS_HEADERS
    "tls/*.h"
)

file(GLOB UTILS_HEADERS
    "utils/*.h"
)
 
 ##source files
 file(GLOB CRYPTO_SRC
     "crypto/*.c"
 )

file(GLOB ERROR_SRC
    "error/*.c"
)

file(GLOB STUFFER_SRC
    "stuffer/*.c"
)

file(GLOB TLS_SRC
    "tls/*.c"
)

file(GLOB UTILS_SRC
    "utils/*.c"
)

##be nice to visual studio users
if(MSVC)
    source_group("Header Files\\s2n\\api" FILES ${API_HEADERS})
    source_group("Header Files\\s2n\\crypto" FILES ${CRYPTO_HEADERS})
    source_group("Header Files\\s2n\\error" FILES ${ERROR_HEADERS})
    source_group("Header Files\\s2n\\stuffer" FILES ${STUFFER_HEADERS})
    source_group("Header Files\\s2n\\tls" FILES ${TLS_HEADERS})
    source_group("Header Files\\s2n\\utils" FILES ${UTILS_HEADERS})

    source_group("Source Files\\crypto" FILES ${CRYPTO_SRC})
    source_group("Source Files\\error" FILES ${ERROR_SRC})
    source_group("Source Files\\stuffer" FILES ${STUFFER_SRC})
    source_group("Source Files\\tls" FILES ${TLS_SRC})
    source_group("Source Files\\utils" FILES ${UTILS_SRC})
endif()

if(APPLE)
    set(OS_LIBS c pthread)        
elseif(CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
    set(OS_LIBS thr)
elseif(CMAKE_SYSTEM_NAME STREQUAL "NetBSD")
    set(OS_LIBS pthread)
else()
    set(OS_LIBS pthread dl rt)
endif()
 
file(GLOB S2N_HEADERS
    ${API_HEADERS}
    ${CRYPTO_HEADERS}
    ${ERROR_HEADERS}
    ${STUFFER_HEADERS}
    ${TLS_HEADERS}
    ${UTILS_HEADERS}
 )
 
 file(GLOB S2N_SRC
    ${CRYPTO_SRC}
    ${ERROR_SRC}
    ${STUFFER_SRC}
    ${TLS_SRC}
    ${UTILS_SRC}
 )

add_library(s2n ${S2N_HEADERS} ${S2N_SRC})
set_target_properties(s2n PROPERTIES LINKER_LANGUAGE C)

set(CMAKE_C_FLAGS_DEBUGOPT "")

target_compile_options(s2n PRIVATE -pedantic -std=c99 -Wall -Werror -Wimplicit -Wunused -Wcomment -Wchar-subscripts -Wuninitialized -Wshadow -Wcast-qual -Wcast-align -Wwrite-strings -Wno-deprecated-declarations -Wno-unknown-pragmas -Wformat-security)
target_compile_definitions(s2n PRIVATE -D_POSIX_C_SOURCE=200809L -D_FORTIFY_SOURCE=2)
target_compile_options(s2n PUBLIC -fPIC)    

if(NO_STACK_PROTECTOR)
    target_compile_options(s2n PRIVATE -Wstack-protector -fstack-protector-all)
endif()

if(S2N_UNSAFE_FUZZING_MODE)
    target_compile_options(s2n PRIVATE -fsanitize-coverage=trace-pc-guard -fsanitize=address,undefined,leak)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules)

find_package(LibCrypto REQUIRED)
target_link_libraries(s2n PRIVATE LibCrypto::Crypto PRIVATE ${OS_LIBS})

target_include_directories(s2n PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)
target_include_directories(s2n PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/api> $<INSTALL_INTERFACE:include>)

file(GLOB TESTLIB_SRC "tests/testlib/*.c")
file(GLOB TESTLIB_HEADERS "tests/testlib/*.h")

add_library(testss2n ${TESTLIB_HEADERS} ${TESTLIB_SRC})
target_include_directories(testss2n PRIVATE tests)
target_compile_options(testss2n PRIVATE -std=c99)
target_link_libraries(testss2n PUBLIC s2n)

#run unit tests
file (GLOB TEST_LD_PRELOAD "tests/LD_PRELOAD/*.c")
add_library(allocator_overrides SHARED ${TEST_LD_PRELOAD})

include(CTest)
enable_testing()

file(GLOB UNITTESTS_SRC "tests/unit/*.c")
    foreach(test_case ${UNITTESTS_SRC})
    string(REGEX REPLACE ".+\\/(.+)\\.c" "\\1" test_case_name ${test_case})
    add_executable(${test_case_name} ${test_case})
    target_link_libraries(${test_case_name} PRIVATE testss2n PRIVATE m pthread)
    target_include_directories(${test_case_name} PRIVATE api)
    target_include_directories(${test_case_name} PRIVATE ./)
    target_include_directories(${test_case_name} PRIVATE tests)
    target_compile_options(${test_case_name} PRIVATE -Wno-implicit-function-declaration -std=c99)
    add_test(NAME ${test_case_name} COMMAND $<TARGET_FILE:${test_case_name}> WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/unit)
   
    set_property(
    TEST
        ${test_case_name}
    PROPERTY
        ENVIRONMENT LD_PRELOAD=$<TARGET_FILE:allocator_overrides>)

endforeach(test_case)

#install the s2n files
install (FILES ${API_HEADERS} DESTINATION "include/")

install (
         TARGETS s2n
         EXPORT "${PROJECT_NAME}-config"
         ARCHIVE DESTINATION lib
         LIBRARY DESTINATION lib
         COMPONENT library     
)

export(TARGETS ${export_targets} FILE "s2n-config.cmake")
#install the cmake finder
install(EXPORT s2n-config DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/s2n/cmake/")
