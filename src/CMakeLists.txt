project(PrusaSlicer-native)

add_subdirectory(admesh)
add_subdirectory(avrdude)
# boost/nowide
add_subdirectory(boost)
add_subdirectory(clipper)
add_subdirectory(miniz)
add_subdirectory(glu-libtess)
add_subdirectory(polypartition)
add_subdirectory(poly2tri)
add_subdirectory(qhull)
add_subdirectory(Shiny)
add_subdirectory(semver)

# Adding libnest2d project for bin packing...
set(LIBNEST2D_UNITTESTS ON CACHE BOOL "Force generating unittests for libnest2d")
add_subdirectory(libnest2d)

include_directories(${LIBDIR}/qhull/src)
#message(STATUS ${LIBDIR}/qhull/src)

add_subdirectory(libslic3r)

if (SLIC3R_GUI)
    add_subdirectory(imgui)

    if(WIN32)
        message(STATUS "WXWIN environment set to: $ENV{WXWIN}")
    elseif(UNIX)
        set(wxWidgets_USE_UNICODE ON)
        if(SLIC3R_STATIC)
            set(wxWidgets_USE_STATIC ON)
        else()
            set(wxWidgets_USE_STATIC OFF)
        endif()
    endif()

    if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set (wxWidgets_CONFIG_OPTIONS "--toolkit=gtk${SLIC3R_GTK}")
        if (SLIC3R_WX_STABLE)
            find_package(wxWidgets 3.0 REQUIRED COMPONENTS base core adv html gl)
        else ()
            find_package(wxWidgets 3.1 QUIET COMPONENTS base core adv html gl)

            if (NOT wxWidgets_FOUND)
                message(FATAL_ERROR "\nCould not find wxWidgets 3.1.\n"
                    "Hint: On Linux you can set -DSLIC3R_WX_STABLE=1 to use wxWidgets 3.0\n")
            endif ()
        endif ()
    else ()
        find_package(wxWidgets 3.1 REQUIRED COMPONENTS base core adv html gl)
    endif ()

    if(UNIX)
        message(STATUS "wx-config path: ${wxWidgets_CONFIG_EXECUTABLE}")
    endif()

    include(${wxWidgets_USE_FILE})

    add_subdirectory(slic3r)
endif()


# Create a slic3r executable
# Process mainfests for various platforms.
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/platform/msw/PrusaSlicer.rc.in ${CMAKE_CURRENT_BINARY_DIR}/PrusaSlicer.rc @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/platform/msw/PrusaSlicer.manifest.in ${CMAKE_CURRENT_BINARY_DIR}/PrusaSlicer.manifest @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/platform/osx/Info.plist.in ${CMAKE_CURRENT_BINARY_DIR}/Info.plist @ONLY)
if (MSVC)
    add_library(PrusaSlicer SHARED PrusaSlicer.cpp PrusaSlicer.hpp)
else ()
    add_executable(PrusaSlicer PrusaSlicer.cpp PrusaSlicer.hpp)
endif ()
if (NOT MSVC)
    # Binary name on unix like systems (OSX, Linux)
    set_target_properties(PrusaSlicer PROPERTIES OUTPUT_NAME "prusa-slicer")
endif ()

target_link_libraries(PrusaSlicer libslic3r)
if (APPLE)
#    add_compile_options(-stdlib=libc++)
#    add_definitions(-DBOOST_THREAD_DONT_USE_CHRONO -DBOOST_NO_CXX11_RVALUE_REFERENCES -DBOOST_THREAD_USES_MOVE)
    # -liconv: boost links to libiconv by default
    target_link_libraries(PrusaSlicer "-liconv -framework IOKit" "-framework CoreFoundation" -lc++)
elseif (MSVC)
    # Manifest is provided through PrusaSlicer.rc, don't generate your own.
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /MANIFEST:NO")
else ()
    # Boost on Raspberry-Pi does not link to pthreads explicitely.
    target_link_libraries(PrusaSlicer ${CMAKE_DL_LIBS} -lstdc++ Threads::Threads)
endif ()

# Add the Slic3r GUI library, libcurl, OpenGL and GLU libraries.
if (SLIC3R_GUI)
    target_link_libraries(PrusaSlicer libslic3r_gui ${wxWidgets_LIBRARIES})

    # Configure libcurl and its dependencies OpenSSL & zlib
    find_package(CURL REQUIRED)
    if (NOT MSVC)
        # Required by libcurl
        find_package(ZLIB REQUIRED)
    endif()
    target_include_directories(PrusaSlicer PRIVATE ${CURL_INCLUDE_DIRS})
    target_link_libraries(PrusaSlicer ${CURL_LIBRARIES} ${ZLIB_LIBRARIES})
    if (SLIC3R_STATIC)
        if (NOT APPLE)
            # libcurl is always linked dynamically to the system libcurl on OSX.
            # On other systems, libcurl is linked statically if SLIC3R_STATIC is set.
            target_compile_definitions(PrusaSlicer PRIVATE CURL_STATICLIB)
        endif()
        if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
            # As of now, our build system produces a statically linked libcurl,
            # which links the OpenSSL library dynamically.
            find_package(OpenSSL REQUIRED)
            message("OpenSSL include dir: ${OPENSSL_INCLUDE_DIR}")
            message("OpenSSL libraries: ${OPENSSL_LIBRARIES}")
            target_include_directories(PrusaSlicer PRIVATE ${OPENSSL_INCLUDE_DIR})
            target_link_libraries(PrusaSlicer ${OPENSSL_LIBRARIES})
        endif()
    endif()

    if (MSVC)
        # Generate debug symbols even in release mode.
        target_link_options(PrusaSlicer PUBLIC "$<$<CONFIG:RELEASE>:/DEBUG>")
        target_link_libraries(PrusaSlicer user32.lib Setupapi.lib OpenGL32.Lib GlU32.Lib)
    elseif (MINGW)
        target_link_libraries(PrusaSlicer -lopengl32)
    elseif (APPLE)
        target_link_libraries(PrusaSlicer "-framework OpenGL")
    else ()
        target_link_libraries(PrusaSlicer -ldl -lGL -lGLU)
    endif ()
endif ()

# On Windows, a shim application is required to produce a console / non console version of the Slic3r application.
# Also the shim may load the Mesa software OpenGL renderer if the default renderer does not support OpenGL 2.0 and higher.
if (MSVC)
    add_executable(PrusaSlicer_app_gui WIN32 PrusaSlicer_app_msvc.cpp ${CMAKE_CURRENT_BINARY_DIR}/PrusaSlicer.rc)
    # Generate debug symbols even in release mode.
    target_link_options(PrusaSlicer_app_gui PUBLIC "$<$<CONFIG:RELEASE>:/DEBUG>")
    target_compile_definitions(PrusaSlicer_app_gui PRIVATE -DSLIC3R_WRAPPER_NOCONSOLE)
    add_dependencies(PrusaSlicer_app_gui PrusaSlicer)
    set_target_properties(PrusaSlicer_app_gui PROPERTIES OUTPUT_NAME "prusa-slicer")
    target_include_directories(PrusaSlicer_app_gui SYSTEM PUBLIC ${Boost_INCLUDE_DIRS})

    add_executable(PrusaSlicer_app_console PrusaSlicer_app_msvc.cpp ${CMAKE_CURRENT_BINARY_DIR}/PrusaSlicer.rc)
    # Generate debug symbols even in release mode.
    target_link_options(PrusaSlicer_app_console PUBLIC "$<$<CONFIG:RELEASE>:/DEBUG>")
    target_compile_definitions(PrusaSlicer_app_console PRIVATE -DSLIC3R_WRAPPER_CONSOLE)
    add_dependencies(PrusaSlicer_app_console PrusaSlicer)
    set_target_properties(PrusaSlicer_app_console PROPERTIES OUTPUT_NAME "prusa-slicer-console")
    target_include_directories(PrusaSlicer_app_console SYSTEM PUBLIC ${Boost_INCLUDE_DIRS})
endif ()

# Link the resources dir to where Slic3r GUI expects it
if (MSVC)
    if (CMAKE_CONFIGURATION_TYPES)
        foreach (CONF ${CMAKE_CONFIGURATION_TYPES})
            file(TO_NATIVE_PATH "${CMAKE_CURRENT_BINARY_DIR}/${CONF}" WIN_CONF_OUTPUT_DIR)
            file(TO_NATIVE_PATH "${CMAKE_CURRENT_BINARY_DIR}/${CONF}/resources" WIN_RESOURCES_SYMLINK)
            add_custom_command(TARGET PrusaSlicer POST_BUILD
                COMMAND if exist "${WIN_CONF_OUTPUT_DIR}" "("
                        if not exist "${WIN_RESOURCES_SYMLINK}" "("
                            mklink /J "${WIN_RESOURCES_SYMLINK}" "${SLIC3R_RESOURCES_DIR_WIN}"
                        ")"
                    ")"
                COMMENT "Symlinking the resources directory into the build tree"
                VERBATIM
            )
        endforeach ()
    else ()
        file(TO_NATIVE_PATH "${CMAKE_CURRENT_BINARY_DIR}/resources" WIN_RESOURCES_SYMLINK)
        add_custom_command(TARGET PrusaSlicer POST_BUILD
            COMMAND if not exist "${WIN_RESOURCES_SYMLINK}" "(" mklink /J "${WIN_RESOURCES_SYMLINK}" "${SLIC3R_RESOURCES_DIR_WIN}" ")"
            COMMENT "Symlinking the resources directory into the build tree"
            VERBATIM
        )
    endif ()
elseif (XCODE)
    # Because of Debug/Release/etc. configurations (similar to MSVC) the slic3r binary is located in an extra level
    add_custom_command(TARGET PrusaSlicer POST_BUILD
        COMMAND ln -sf "${SLIC3R_RESOURCES_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/resources"
        COMMENT "Symlinking the resources directory into the build tree"
        VERBATIM
    )
else ()
    add_custom_command(TARGET PrusaSlicer POST_BUILD
        COMMAND ln -sf "${SLIC3R_RESOURCES_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/../resources"
        COMMENT "Symlinking the resources directory into the build tree"
        VERBATIM
    )
endif()

# Slic3r binary install target
if (WIN32)
    install(TARGETS PrusaSlicer RUNTIME DESTINATION "${CMAKE_INSTALL_PREFIX}")
    if (MSVC)
        install(TARGETS PrusaSlicer_app_gui RUNTIME DESTINATION "${CMAKE_INSTALL_PREFIX}")
        install(TARGETS PrusaSlicer_app_console RUNTIME DESTINATION "${CMAKE_INSTALL_PREFIX}")
    endif ()
else ()
    install(TARGETS PrusaSlicer RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
endif ()
