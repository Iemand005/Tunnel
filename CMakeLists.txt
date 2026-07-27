cmake_minimum_required(VERSION 3.15)
project(Tunnel)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(KISSFFT_TEST OFF CACHE BOOL "Disable KissFFT tests" FORCE)

set(KISSFFT_STATIC ON CACHE BOOL "Build static library" FORCE)

set(FE_EXCLUDE_JOLT ON CACHE BOOL "Disable JOLT" FORCE)
set(FE_EXCLUDE_KISSFFT ON CACHE BOOL "Exclude kissfft from the build" FORCE)

set(ENGINE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../FenixEngine")

# add_subdirectory(${CMAKE_SOURCE_DIR}/../FenixEngine/src/Engine ${CMAKE_BINARY_DIR}/out/Engine)


set(USE_STATIC_MSVC_RUNTIME_LIBRARY OFF CACHE BOOL "Use dynamic MSVC runtime to match rest of project" FORCE)

if(CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)
endif()


if(NOT TARGET Engine)
    message(STATUS "Engine target not found. Looking for engine source...")
    
    if(EXISTS "${ENGINE_DIR}/CMakeLists.txt")
        add_subdirectory("${ENGINE_DIR}" FenixEngine)
    # elif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/FenixEngine/CMakeLists.txt")
    #     add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/FenixEngine" FenixEngine)
    else()
        message(FATAL_ERROR "FenixEngine not found!")
    endif()
endif()

set(IMGUI_DIR ../../../external/imgui)

file(GLOB_RECURSE IMGUI_SOURCE_FILES
	${IMGUI_DIR}/imgui.cpp
	${IMGUI_DIR}/backends/imgui_impl_sdl3.cpp
	${IMGUI_DIR}/backends/imgui_impl_sdlgpu3.cpp
	${IMGUI_DIR}/backends/imgui_impl_sdlrenderer3.cpp
	${IMGUI_DIR}/backends/imgui_impl_opengl3.cpp
	${IMGUI_DIR}/imgui_tables.cpp
	${IMGUI_DIR}/imgui_widgets.cpp
	${IMGUI_DIR}/imgui_demo.cpp
	${IMGUI_DIR}/imgui_draw.cpp
)

if(EMSCRIPTEN)
    add_executable(Tunnel
        ${IMGUI_SOURCE_FILES}
        main.cpp
    )
else()
    add_executable(Tunnel
        WIN32
        ${IMGUI_SOURCE_FILES}
        main.cpp
    )
endif()

target_include_directories(Tunnel PRIVATE
	include
	${ENGINE_DIR}
	${ENGINE_DIR}/include
	${ENGINE_DIR}/external
	${ENGINE_DIR}/external/imgui
	${ENGINE_DIR}/external/kissfft
	${ENGINE_DIR}/external/SDL/include
	${ENGINE_DIR}/src/Engine
)

target_link_libraries(Tunnel PRIVATE
	Engine
)

if(NOT WIN32 AND NOT EMSCRIPTEN)
find_package(PkgConfig REQUIRED)
pkg_check_modules(PIPEWIRE REQUIRED libpipewire-0.3)
target_include_directories(Tunnel PRIVATE ${PIPEWIRE_INCLUDE_DIRS})
target_link_libraries(Tunnel PRIVATE ${PIPEWIRE_LIBRARIES})
target_compile_options(Tunnel PRIVATE ${PIPEWIRE_CFLAGS_OTHER})

find_package(X11 REQUIRED)

target_link_libraries(Tunnel PRIVATE X11::X11)
endif()

set_property(TARGET Tunnel PROPERTY VS_DEBUGGER_WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
	
if(MSVC)
	target_compile_options(Tunnel PRIVATE $<$<CONFIG:Debug>:/Zi>)

	target_link_options(Tunnel PRIVATE $<$<CONFIG:Debug>:/DEBUG>)

	add_compile_options(/RTC1)
endif()

# set_target_properties(Tunnel PROPERTIES SUFFIX ".scr")


if(EMSCRIPTEN)
    set(CMAKE_EXECUTABLE_SUFFIX ".html")
    
    target_compile_options(Tunnel PRIVATE
        -g
        -fdebug-compilation-dir="${CMAKE_SOURCE_DIR}"
    )
    target_link_options(Tunnel PRIVATE
        -g
        -gsource-map
    )
    
    set_target_properties(Tunnel PROPERTIES
        SUFFIX ".html"
        LINK_FLAGS "-s USE_WEBGL2=1 -s FULL_ES3=1 -sGL_ENABLE_GET_PROC_ADDRESS=1 -s ALLOW_MEMORY_GROWTH=1 -s INITIAL_MEMORY=64MB -s NO_EXIT_RUNTIME=1 -s EXIT_RUNTIME=0 -s MAX_WEBGL_VERSION=2 -s MIN_WEBGL_VERSION=2 -sFORCE_FILESYSTEM=1 -sEXPORTED_RUNTIME_METHODS=FS,ccall,cwrap -sEXPORTED_FUNCTIONS=_main,_emscripten_file_dialog_callback,_malloc,_free --preload-file ${CMAKE_CURRENT_SOURCE_DIR}/resources@resources"
    )
endif()