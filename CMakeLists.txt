cmake_minimum_required (VERSION 2.8)
project (Fit)
 
# The version number.
set (Fit_VERSION_MAJOR 0)
set (Fit_VERSION_MINOR 4)

include(CheckCXXCompilerFlag)
enable_language(CXX)

if(CMAKE_HOST_APPLE)
  check_cxx_compiler_flag("-stdlib=libc++" COMPILER_HAS_CXX_FLAG_libcxx)
  if(COMPILER_HAS_CXX_FLAG_libcxx)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
  endif()
endif()

set(ENABLE_CXXFLAGS_TO_CHECK 
    -std=gnu++1y 
    -std=c++1y
    -std=gnu++11 
    -std=c++11
    -std=gnu++0x 
    -std=c++0x)

foreach(flag ${ENABLE_CXXFLAGS_TO_CHECK})
    string(REPLACE "-std=" "_" flag_var ${flag})
    string(REPLACE "+" "x" flag_var ${flag_var})
    check_cxx_compiler_flag("${flag}" COMPILER_HAS_CXX_FLAG${flag_var})
    if(COMPILER_HAS_CXX_FLAG${flag_var})
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${flag}")
        break()
    endif()
endforeach()

install (DIRECTORY include/fit DESTINATION include)
install (FILES include/fit.hpp DESTINATION include)
configure_file(fit.pc.in fit.pc)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/fit.pc DESTINATION lib/pkgconfig)

add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} -VV -C ${CMAKE_CFG_INTDIR})

function(add_test_executable TEST_NAME)
    add_executable (${TEST_NAME} EXCLUDE_FROM_ALL ${ARGN})
    if(WIN32)
        add_test(NAME ${TEST_NAME} WORKING_DIRECTORY ${LIBRARY_OUTPUT_PATH} COMMAND ${TEST_NAME}${CMAKE_EXECUTABLE_SUFFIX})
    else()
        add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
    endif()
    add_dependencies(check ${TEST_NAME})
    set_tests_properties(${TEST_NAME} PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
endfunction(add_test_executable)

function(create_executable NAME CONTENT)
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp "${CONTENT}")
    add_executable(${NAME} ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp)
endfunction()

function(create_test_executable NAME CONTENT)
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp "${CONTENT}")
    add_test_executable(${NAME} ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp)
endfunction()

function(add_test_header TEST_NAME HEADER)
    create_test_executable(header-include-${TEST_NAME} 
        "#include <${HEADER}>\nint main() {}\n"
    )
endfunction(add_test_header)

function(add_test_static_header TEST_NAME HEADER)
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/header-main-include-${TEST_NAME}.cpp 
        "#include <${HEADER}>\nint main() {}\n"
    )
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/header-static-include-${TEST_NAME}.cpp 
        "#include <${HEADER}>\n"
    )
    add_test_executable(header-static-include-${TEST_NAME} 
        ${CMAKE_CURRENT_BINARY_DIR}/header-main-include-${TEST_NAME}.cpp 
        ${CMAKE_CURRENT_BINARY_DIR}/header-static-include-${TEST_NAME}.cpp
    )
endfunction(add_test_static_header)

include(CTest)

include_directories(include)

file(GLOB TESTS test/*.cpp)
foreach(TEST ${TESTS})
    get_filename_component(BASE_NAME ${TEST} NAME_WE)
    add_test_executable(${BASE_NAME} ${TEST})
endforeach()
add_test_executable(static_def test/static_def/static_def.cpp test/static_def/static_def2.cpp)

file(GLOB HEADERS include/fit/*.hpp)
foreach(HEADER ${HEADERS})
    get_filename_component(BASE_NAME ${HEADER} NAME_WE)
    add_test_header(${BASE_NAME} fit/${BASE_NAME}.hpp)
endforeach()
add_test_static_header(fit fit.hpp)

function(extract_doc SOURCE OUTPUTVAR)
    file(READ ${SOURCE} CONTENT)
    string(REGEX REPLACE "(\n(/[^/][^/]|//[^/]|[^/][^/][^/])([^\n])*)" "" CONTENT "\n${CONTENT}")
    string(REPLACE "/// " "" CONTENT "${CONTENT}")
    set(${OUTPUTVAR} "${CONTENT}" PARENT_SCOPE)
endfunction()

function(build_example SECTIONCONTENT NAME)
    string(MD5 MD5_SECTION "${SECTIONCONTENT}")
    set(TARGET_NAME "example-${NAME}-${MD5_SECTION}")
    set(CONTENT "")
    foreach(LINE ${SECTIONCONTENT})
        if(LINE MATCHES "^    ")
            string(SUBSTRING ${LINE} "4" "-1" OUTPUT_LINE)
            list(APPEND CONTENT ${OUTPUT_LINE})
        endif()
    endforeach()
    string(REPLACE ";" "\n" CONTENT "${CONTENT}")
    string(REPLACE "$$__semicolon__$$" ";" CONTENT "${CONTENT}")
    if(NOT CONTENT MATCHES "int main")
        string(APPEND CONTENT "int main() {}")
    endif()
    message(STATUS "Adding example: ${TARGET_NAME}")
    create_test_executable(${TARGET_NAME} "${CONTENT}\n")
endfunction()

function(extract_example SOURCE)
    extract_doc(${SOURCE} CONTENT)
    string(REPLACE ";" "$$__semicolon__$$" CONTENT "${CONTENT}")
    string(REPLACE "\n" ";" CONTENT "${CONTENT}")

    get_filename_component(BASE_NAME ${SOURCE} NAME_WE)

    set(SECTION "")
    set(PREVLINE "")
    set(APPEND_SECTION Off)
    foreach(LINE ${CONTENT})
        if(APPEND_SECTION)
            if((LINE MATCHES "^---") OR (LINE MATCHES "^==="))
                build_example("${SECTION}" ${BASE_NAME})
                set(APPEND_SECTION Off)
                set(SECTION "")
            elseif(NOT PREVLINE MATCHES "^---")
                list(APPEND SECTION ${PREVLINE})
            endif()
        else()
            if((LINE MATCHES "^---") AND (PREVLINE MATCHES "^Example"))
                set(APPEND_SECTION One)
            endif()
        endif()
        set(PREVLINE "${LINE}")
    endforeach()
    list(APPEND SECTION ${PREVLINE})
    if(APPEND_SECTION)
        build_example("${SECTION}" ${BASE_NAME})
    endif()
endfunction()

set(BUILD_EXAMPLES off CACHE BOOL "Set this to build the examples")

if (BUILD_EXAMPLES)
    file(GLOB EXAMPLES example/*.cpp)
    foreach(EXAMPLE ${EXAMPLES})
        get_filename_component(BASE_NAME ${EXAMPLE} NAME_WE)
        add_test_executable(example-${BASE_NAME} ${EXAMPLE})
    endforeach()
    foreach(HEADER ${HEADERS})
        extract_example(${HEADER})
    endforeach()
endif()
