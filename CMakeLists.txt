cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0063 NEW)

project(flecs-hub LANGUAGES C)

option(FLECS_STATIC "Build static flecs lib" ON)
option(FLECS_SHARED "Build shared flecs lib" ON)
option(FLECS_PIC "Compile static flecs lib with position independent code (PIC)" ON)

include(FetchContent)

FetchContent_Declare(
  flecs
  GIT_REPOSITORY https://github.com/SanderMertens/flecs
  GIT_TAG        master
)
FetchContent_Declare(
  cglm
  GIT_REPOSITORY https://github.com/recp/cglm
  GIT_TAG        master
)
FetchContent_MakeAvailable(flecs cglm)



add_library(cglm::static ALIAS cglm)

include(${flecs_SOURCE_DIR}/cmake/target_default_compile_warnings.cmake)
include(${flecs_SOURCE_DIR}/cmake/target_default_compile_options.cmake)

# Automatically generate the same folder structure in Visual Studio as we have on disk
macro(GroupSources curdir)
	file(GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/${curdir} ${PROJECT_SOURCE_DIR}/${curdir}/*)
	foreach(child ${children})
		if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${curdir}/${child})
			GroupSources(${curdir}/${child})
		else()
			string(REPLACE "/" "\\" groupname ${curdir})
            source_group(${groupname} FILES ${PROJECT_SOURCE_DIR}/${curdir}/${child})
		endif()
	endforeach()
endmacro()

function(add_flecs_target TARGET CONFIG SRC)
    add_library(${TARGET} ${CONFIG} ${SRC})
    string(REPLACE "_" "::" TARGET_UNDER ${TARGET})
    add_library(${TARGET_UNDER} ALIAS ${TARGET})

    target_default_compile_options_c(${TARGET})
    target_default_compile_warnings_c(${TARGET})

    if(WIN32)
        target_link_libraries(${TARGET} PUBLIC wsock32 ws2_32)
    endif()
    if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
        target_link_libraries(${TARGET} PUBLIC pthread)
    endif()

    if(FLECS_PIC)
        set_property(TARGET ${TARGET} PROPERTY POSITION_INDEPENDENT_CODE ON)
    endif()

    list(APPEND FLECS_HUB_TARGETS ${TARGET})
endfunction()


function(flecs_add_hub_project)
    cmake_parse_arguments(
        PARSED_ARGS
        ""
        "PROJECT;EXTRASRC"
        "DEPS"
        ${ARGN}
    )
    
    string(REPLACE "-" "_" PROJECT_UNDER ${PARSED_ARGS_PROJECT})

    file(GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/${PARSED_ARGS_PROJECT}/. ${PROJECT_SOURCE_DIR}/${PARSED_ARGS_PROJECT}/./*)
    foreach(child ${children})
	    if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${PARSED_ARGS_PROJECT}/${curdir}/${child})
		    GroupSources(${child})
	    endif()
    endforeach()


    file(GLOB_RECURSE INC ${PARSED_ARGS_PROJECT}/include/*.h ${project}/include/*.hpp)
    file(GLOB_RECURSE SRC ${PARSED_ARGS_PROJECT}/src/*.c)
    LIST(APPEND SRC ${PARSED_ARGS_EXTRASRC})
    message(Project " (" ${PARSED_ARGS_PROJECT} ") = " ${PARSED_ARGS_DEPS} " + " ${SRC})
    message("Extra Source: " ${PARSED_ARGS_EXTRASRC})

    if(FLECS_SHARED)
        add_flecs_target(${PROJECT_UNDER} SHARED "${SRC}")
        target_compile_definitions(${PROJECT_UNDER} PRIVATE ${PROJECT_UNDER}_EXPORTS=1)
        message(Setting " " ${PROJECT_UNDER}_EXPORTS=1)
        target_include_directories(${PROJECT_UNDER} PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${PARSED_ARGS_PROJECT}/include>
            $<INSTALL_INTERFACE:include>)
        target_include_directories(${PROJECT_UNDER} PRIVATE ${PROJECT_SOURCE_DIR}/include)
        target_link_libraries(${PROJECT_UNDER} PUBLIC flecs)
        target_link_libraries(${PROJECT_UNDER} PRIVATE ${PARSED_ARGS_DEPS})
    endif()

    if(FLECS_STATIC)
        set(STATIC_LIBS "")
        foreach(lib ${PARSED_ARGS_DEPS})
            list(APPEND  STATIC_LIBS ${lib}::static)
        endforeach()
        add_flecs_target(${PROJECT_UNDER}_static STATIC "${SRC}")
        target_compile_definitions(${PROJECT_UNDER}_static PUBLIC ${PROJECT_UNDER}_STATIC)
        target_include_directories(${PROJECT_UNDER}_static PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${PARSED_ARGS_PROJECT}/include>
            $<INSTALL_INTERFACE:include>)
        target_include_directories(${PROJECT_UNDER}_static PRIVATE ${PROJECT_SOURCE_DIR}/include)
        target_link_libraries(${PROJECT_UNDER}_static PUBLIC flecs_static)
        target_link_libraries(${PROJECT_UNDER}_static PRIVATE ${STATIC_LIBS})

    endif()

    include_directories(${PROJECT_SOURCE_DIR}/${PARSED_ARGS_PROJECT}/include) # Needed because bake uses #include <>
endfunction()

set(FLECS_HUB_TARGETS "")

include_directories(${cglm_SOURCE_DIR}/include) # Needed because bake uses #include <>



flecs_add_hub_project(PROJECT flecs-components-physics)
flecs_add_hub_project(PROJECT flecs-components-cglm DEPS cglm)
flecs_add_hub_project(PROJECT flecs-components-transform DEPS flecs::components::cglm cglm)
flecs_add_hub_project(PROJECT flecs-components-gui DEPS flecs::components::graphics flecs::components::transform flecs::components::cglm)
flecs_add_hub_project(PROJECT flecs-components-graphics DEPS flecs::components::cglm cglm)
flecs_add_hub_project(PROJECT flecs-components-geometry DEPS flecs::components::graphics flecs::components::cglm cglm)
flecs_add_hub_project(PROJECT flecs-components-input DEPS flecs::components::graphics)
flecs_add_hub_project(PROJECT flecs-game DEPS
    flecs::components::transform
    flecs::components::physics
    flecs::components::input
    flecs::components::graphics
    flecs::systems::physics
)

flecs_add_hub_project(PROJECT flecs-systems-transform DEPS flecs::components::transform cglm)
flecs_add_hub_project(PROJECT flecs-systems-physics DEPS flecs::components::geometry flecs::components::transform flecs::components::physics)
set(SOKOL_IMPL_FILE ${PROJECT_SOURCE_DIR}/sokol.c)
if (APPLE)
    set(SOKOL_IMPL_FILE ${PROJECT_SOURCE_DIR}/sokol.m)
endif()
flecs_add_hub_project(PROJECT flecs-systems-sokol DEPS
    flecs::components::input
    flecs::game
    flecs::components::transform
    flecs::components::geometry
    flecs::components::graphics
    flecs::systems::transform
    flecs::components::gui
    EXTRASRC ${SOKOL_IMPL_FILE}
)

if (APPLE)
    target_link_libraries(flecs_systems_sokol PUBLIC "-framework CoreFoundation" "-framework Cocoa" "-framework OpenGL")
endif()

message(STATUS "Targets: ${FLECS_TARGETS}")

add_executable(city ${PROJECT_SOURCE_DIR}/city/src/main.c ${PROJECT_SOURCE_DIR}/city/src/module.c)
target_include_directories(city PRIVATE ${PROJECT_SOURCE_DIR}/city/include)
target_link_libraries(city PUBLIC
    flecs
    flecs::game
    flecs::systems::sokol
    flecs::components::geometry
    flecs::components::physics
    flecs::components::graphics
    flecs::components::transform
    flecs::components::cglm
    flecs::components::gui
    flecs::systems::transform
    flecs::systems::physics
    
)