cmake_minimum_required(VERSION 3.0.0)
project(test_bgfx VERSION 0.1.0)

# download all submodules
find_package(Git QUIET)
if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    # update submodules as needed
    option(GIT_SUBMODULE "check submodules during build process" ON)
    if(GIT_SUBMODULE)
        message(STATUS "submodule update")
        execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
                        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                        RESULT_VARIABLE GIT_SUBMODULE_RESULT
        )
        if(NOT GIT_SUBMODULE_RESULT EQUAL "0")
            message(FATAL_ERROR "git submodule update --init --recursive failed with ${GIT_SUBMODULE_RESULT}, please checkout submodules")
        endif()
    endif()
endif()

# make sure everything is inplace
if(NOT EXISTS "${PROJECT_SOURCE_DIR}/external/bx/makefile")
    message(FATAL_ERROR "bx submodule is not downloaded! GIT_SUBMODULE is turned off or failed. please update submodules and try again")
endif()
if(NOT EXISTS "${PROJECT_SOURCE_DIR}/external/bimg/makefile")
    message(FATAL_ERROR "bimg submodule is not downloaded! GIT_SUBMODULE is turned off or failed. please update submodules and try again")
endif()
if(NOT EXISTS "${PROJECT_SOURCE_DIR}/external/bgfx/makefile")
    message(FATAL_ERROR "bgfx submodule is not downloaded! GIT_SUBMODULE is turned off or failed. please update submodules and try again")
endif()
if(NOT EXISTS "${PROJECT_SOURCE_DIR}/external/entt/CMakeLists.txt")
    message(FATAL_ERROR "entt submodule is not downloaded! GIT_SUBMODULE is turned off or failed. please update submodules and try again")
endif()

# custom make targets
add_custom_target(build_bgfx_linux_release COMMAND make linux-release64
                  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx
)
add_custom_target(build_bgfx_linux_debug COMMAND make linux-debug64
                  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx
)


include(CTest)
enable_testing()

cmake_policy(SET CMP0072 NEW)
find_package(OpenGL)
if(NOT OPENGL_FOUND)
    message(STATUS "opengl not found")
endif()

add_executable(test_bgfx main.cpp)
target_include_directories(test_bgfx PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx/include
    ${CMAKE_CURRENT_SOURCE_DIR}/external/bx/include
    ${CMAKE_CURRENT_SOURCE_DIR}/external/bimg/include
    ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx/3rdparty
    ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx/examples/common
)

if(UNIX)
    target_link_libraries(test_bgfx
        OpenGL::GLX
        X11
        pthread
        ${CMAKE_DL_LIBS}
    )


    target_link_directories(test_bgfx PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx/.build/linux64_gcc/bin/)
    if(${CMAKE_BUILD_TYPE} STREQUAL "Debug")
        message(STATUS "link against bgfx debug build")
        add_dependencies(test_bgfx build_bgfx_linux_debug)
        target_link_libraries(test_bgfx
            example-glueDebug
            example-commonDebug
            bimg_decodeDebug
        )
        target_link_libraries(test_bgfx
            bgfxDebug
            bimgDebug
            bxDebug
        )
    elseif(${CMAKE_BUILD_TYPE} STREQUAL "Release")
        message(STATUS "link against bgfx release build")
        add_dependencies(test_bgfx build_bgfx_linux_release)
        target_link_libraries(test_bgfx
            example-glueRelease
            example-commonRelease
            bimg_decodeRelease
        )
        target_link_libraries(test_bgfx
            bgfxRelease 
            bimgRelease
            bxRelease
        )
    else()
        message(FATAL_ERROR "cmake build type incorrect. current build type: ${CMAKE_BUILD_TYPE}")
    endif()
endif()

file (COPY ${CMAKE_CURRENT_SOURCE_DIR}/external/bgfx/examples/runtime/ DESTINATION ${CMAKE_CURRENT_BINARY_DIR})

set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
include(CPack)
