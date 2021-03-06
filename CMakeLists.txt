# NOTE:  If you installed SDL2 at an unusual place and FindSDL2 fails,
#        you can define SDL2_INCLUDE_DIR on the cmdline.  For example:
#        cmake -DSDL2_INCLUDE_DIR=/opt/SDL2/include/SDL2 [other stuff]

cmake_minimum_required(VERSION 3.0.0)
project(sdl12_compat
        VERSION 0.0.1
        LANGUAGES C)

set(SDL12_COMPAT_VERSION_STR "1.2.50")

option(SDL12TESTS "Enable to build SDL-1.2 test programs" ON)
option(SDL12DEVEL "Enable installing SDL-1.2 development headers" ON)

set(CMAKE_SKIP_RPATH TRUE)

if(APPLE)
  set(OSX_SRCS "src/SDL12_compat_objc.m")
  set_source_files_properties(${OSX_SRCS} PROPERTIES LANGUAGE C)
  # the following matches SDL-1.2 Xcode project file
  set(DYLIB_COMPAT_VERSION 1.0.0)
  set(DYLIB_CURRENT_VERSION 12.50.0)
  include_directories("/opt/X11/include")  # hack.
endif()
if(WIN32)
  set(WIN32_SRCS "src/version.rc")
endif()

set(SDL12COMPAT_SRCS
  src/SDL12_compat.c
  ${OSX_SRCS}
  ${WIN32_SRCS}
)
add_library(SDL SHARED ${SDL12COMPAT_SRCS})

include(GNUInstallDirs)
include("cmake/modules/FindSDL2.cmake")
target_include_directories(SDL PRIVATE ${SDL2_INCLUDE_DIRS})

# avoid DLL having 'lib' prefix with Windows MinGW builds
if(WIN32)
  set(CMAKE_SHARED_LIBRARY_PREFIX "")
  set_target_properties(SDL PROPERTIES COMPILE_DEFINITIONS "DLL_EXPORT")
endif()

if(UNIX AND NOT APPLE)
    set_target_properties(SDL PROPERTIES COMPILE_DEFINITIONS "_REENTRANT")
    target_link_libraries(SDL PRIVATE dl)
endif()
if(APPLE)
    set_target_properties(SDL PROPERTIES COMPILE_DEFINITIONS "_THREAD_SAFE")
    set_target_properties(SDL PROPERTIES LINK_FLAGS
            "-Wl,-compatibility_version,${DYLIB_COMPAT_VERSION} -Wl,-current_version,${DYLIB_CURRENT_VERSION}")
    target_link_libraries(SDL PRIVATE "-framework AppKit")
    set_target_properties(SDL PROPERTIES
        MACOSX_RPATH 1
        OUTPUT_NAME "SDL-1.2.0"
    )
elseif(UNIX AND NOT ANDROID)
    set_target_properties(SDL PROPERTIES
        VERSION "${SDL12_COMPAT_VERSION_STR}"
        SOVERSION "0"
        OUTPUT_NAME "SDL-1.2")
else()
    set_target_properties(SDL PROPERTIES
        VERSION "${SDL12_COMPAT_VERSION_STR}"
        SOVERSION "0"
      OUTPUT_NAME "SDL")
endif()

if(MINGW)
    set_target_properties(SDL PROPERTIES LINK_FLAGS "-nostdlib")
endif()
if(MSVC)
    # Don't try to link with the default set of libraries.
    set_target_properties(SDL PROPERTIES COMPILE_FLAGS "/GS-")
    set_target_properties(SDL PROPERTIES LINK_FLAGS "/NODEFAULTLIB")
    # Make sure /RTC1 is disabled: (from SDL2 CMake)
    foreach(flag_var
      CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
      CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
      string(REGEX REPLACE "/RTC(su|[1su])" "" ${flag_var} "${${flag_var}}")
    endforeach(flag_var)
endif()


if(SDL12TESTS)
    if(NOT (WIN32 OR APPLE OR CYGWIN OR HAIKU OR BEOS))
      find_library(MATH_LIBRARY m)
    endif()

    macro(test_program _NAME _SRCS)
        add_executable(${_NAME} ${_SRCS})
        target_include_directories(${_NAME} PRIVATE "include/SDL")
        target_link_libraries(${_NAME} SDL)
        set_target_properties(${_NAME} PROPERTIES COMPILE_DEFINITIONS "HAVE_OPENGL")
        if(MATH_LIBRARY)
          target_link_libraries(${_NAME} ${MATH_LIBRARY})
        endif()
    endmacro()

    test_program(checkkeys "test/checkkeys.c")
    test_program(graywin "test/graywin.c")
    test_program(loopwave "test/loopwave.c")
    test_program(testalpha "test/testalpha.c")
    test_program(testbitmap "test/testbitmap.c")
    test_program(testblitspeed "test/testblitspeed.c")
    test_program(testcdrom "test/testcdrom.c")
    test_program(testcursor "test/testcursor.c")
    test_program(testdyngl "test/testdyngl.c")
    test_program(testerror "test/testerror.c")
    test_program(testfile "test/testfile.c")
    test_program(testgamma "test/testgamma.c")
    test_program(testgl "test/testgl.c")
    test_program(testthread "test/testhread.c")
    test_program(testiconv "test/testiconv.c")
    test_program(testjoystick "test/testjoystick.c")
    test_program(testkeys "test/testkeys.c")
    test_program(testloadso "test/testloadso.c")
    test_program(testlock "test/testlock.c")
    test_program(testoverlay "test/testoverlay.c")
    test_program(testoverlay2 "test/testoverlay2.c")
    test_program(testpalette "test/testpalette.c")
    test_program(testplatform "test/testplatform.c")
    test_program(testsem "test/testsem.c")
    test_program(testsprite "test/testsprite.c")
    test_program(testtimer "test/testtimer.c")
    test_program(testver "test/testver.c")
    test_program(testvidinfo "test/testvidinfo.c")
    test_program(testwin "test/testwin.c")
    test_program(testwm "test/testwm.c")
    test_program(threadwin "test/threadwin.c")
    test_program(torturethread "test/torturethread.c")

    if(APPLE)
        target_link_libraries(testgl "-framework OpenGL")
        set_target_properties(testgl PROPERTIES COMPILE_DEFINITIONS "GL_SILENCE_DEPRECATION=1")
    else()
        target_link_libraries(testgl "GL")
    endif()

    foreach(fname "icon.bmp" "moose.dat" "picture.xbm" "sail.bmp" "sample.bmp" "sample.wav" "utf8.txt")
        file(COPY "${CMAKE_SOURCE_DIR}/test/${fname}" DESTINATION "${CMAKE_BINARY_DIR}")
    endforeach(fname)
endif()

install(TARGETS SDL
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(SDL12DEVEL)
  install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )
  configure_file(sdl12_compat.pc.in sdl12_compat.pc @ONLY)
  install(FILES ${CMAKE_BINARY_DIR}/sdl12_compat.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  )

  # !!! FIXME: lots of these sdl-config vars probably need more customization.

  # !!! FIXME: do we _want_ static builds?
  set(ENABLE_STATIC_TRUE "")
  set(ENABLE_STATIC_FALSE "#")
  set(ENABLE_SHARED_TRUE "")
  set(ENABLE_SHARED_FALSE "#")

  set(SDL_VERSION "${SDL12_COMPAT_VERSION_STR}")
  set(SDL_CFLAGS "-D_GNU_SOURCE=1 -D_REENTRANT")
  set(SDL_RLD_FLAGS "")  # !!! FIXME: this forces rpath, which we might want?
  set(SDL_LIBS "-lSDL")
  set(SDL_STATIC_LIBS "-lm -ldl -lpthread")
  set(prefix ${CMAKE_INSTALL_PREFIX})
  set(exec_prefix "\${prefix}")
  set(libdir "\${exec_prefix}/lib${LIB_SUFFIX}")
  set(bindir "\${exec_prefix}/bin")
  set(includedir "\${prefix}/include")
  configure_file("${CMAKE_SOURCE_DIR}/sdl-config.in" "${CMAKE_BINARY_DIR}/sdl-config" @ONLY)
  install(PROGRAMS "${CMAKE_BINARY_DIR}/sdl-config" DESTINATION bin)
endif()

