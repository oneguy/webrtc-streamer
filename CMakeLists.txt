cmake_minimum_required (VERSION 3.0.0)
project (webrtc-streamer)

set (WEBRTCROOT "${CMAKE_CURRENT_SOURCE_DIR}/../webrtc" CACHE STRING "WEBRTC root directory")
set (WEBRTCDESKTOPCAPTURE "ON" CACHE STRING "WEBRTC Desktop capture")
if(NOT CMAKE_BUILD_TYPE)
set (CMAKE_BUILD_TYPE "Release")
endif()
MESSAGE("CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}\nWEBRTCROOT = ${WEBRTCROOT}\nWEBRTCDESKTOPCAPTURE= ${WEBRTCDESKTOPCAPTURE}\nCMAKE_CXX_COMPILER_ID=${CMAKE_CXX_COMPILER_ID}")

set (CMAKE_POSITION_INDEPENDENT_CODE ON)
set (CMAKE_CXX_STANDARD 14)
set (CMAKE_CXX_EXTENSIONS OFF)

# build version identifier
find_package(Git)
if(GIT_FOUND)
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} submodule update --init)
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} describe --tags --always --dirty OUTPUT_VARIABLE PROJECTVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    set (VERSION "${PROJECTVERSION}/${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")
    
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C civetweb describe --tags --always --dirty OUTPUT_VARIABLE CIVETVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    set (VERSION "${VERSION} civetweb@${CIVETVERSION}")

    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C ${WEBRTCROOT}/src describe --tags --always --dirty OUTPUT_VARIABLE WEBRTCVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    set (VERSION "${VERSION} webrtc@${WEBRTCVERSION}")

    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C live555helper describe --tags --always --dirty OUTPUT_VARIABLE LIVEVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    set (VERSION "${VERSION} live555helper@${LIVEVERSION}")
endif()
add_definitions(-DVERSION=\"${VERSION}\")

# set CMAKE_FIND_ROOT_PATH to search package in WebRTC sysroot
if(CMAKE_SYSTEM_PROCESSOR MATCHES "armv.*")
  set (CMAKE_FIND_ROOT_PATH ${WEBRTCROOT}/src/build/linux/debian_sid_arm-sysroot)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
  set (CMAKE_FIND_ROOT_PATH ${WEBRTCROOT}/src/build/linux/debian_sid_arm64-sysroot)
endif()
MESSAGE("CMAKE_FIND_ROOT_PATH = ${CMAKE_FIND_ROOT_PATH}")

# alsa ?
find_package(ALSA QUIET)
MESSAGE("ALSA_FOUND = ${ALSA_FOUND}")

# pulse ?
find_package(PulseAudio QUIET)
MESSAGE("PulseAudio_FOUND = ${PulseAudio_FOUND}")

# webrtc build
if (NOT EXISTS ${WEBRTCROOT}/src/out/${CMAKE_BUILD_TYPE}/args.gn)
  set (WEBRTCARGS rtc_include_tests=false\nrtc_enable_protobuf=false\nrtc_build_examples=false\nrtc_build_tools=false\ntreat_warnings_as_errors=false\nrtc_enable_libevent=false\nrtc_build_libevent=false\n)
  set (WEBRTCARGS use_custom_libcxx=false\n${WEBRTCARGS})
  # debug/release
  if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set (WEBRTCARGS is_debug=false\n${WEBRTCARGS})
  else()
    set (WEBRTCARGS is_debug=true\n${WEBRTCARGS})
  endif()

  # enable H264 support
  set (WEBRTCARGS is_chrome_branded=true\n${WEBRTCARGS})
  if (CMAKE_SYSTEM_PROCESSOR MATCHES "armv6.*")
    set(ENV{PATH} "${WEBRTCROOT}/src/third_party/llvm-build/Release+Asserts/bin:$ENV{PATH}")
    file(READ ${WEBRTCROOT}/src/third_party/ffmpeg/chromium/scripts/build_ffmpeg.py filecontent)
    string(REPLACE "--enable-armv6t2" "--disable-armv6t2" filecontent "${filecontent}")
    string(REPLACE "--enable-thumb" "--disable-thumb" filecontent "${filecontent}")
    string(REPLACE "--extra-cflags=-march=armv7-a" "--extra-cflags=-march=armv6" filecontent "${filecontent}")
    file(WRITE ${WEBRTCROOT}/src/third_party/ffmpeg/chromium/scripts/build_ffmpeg.py "${filecontent}")
    EXECUTE_PROCESS(WORKING_DIRECTORY ${WEBRTCROOT}/src/third_party/ffmpeg COMMAND python2 chromium/scripts/build_ffmpeg.py linux arm --config-only)
    EXECUTE_PROCESS(WORKING_DIRECTORY ${WEBRTCROOT}/src/third_party/ffmpeg COMMAND chromium/scripts/copy_config.sh)
  endif()

  # sound support
  if (NOT WIN32)
    if (NOT PulseAudio_FOUND)
      set (WEBRTCARGS rtc_include_pulse_audio=false\n${WEBRTCARGS})
    endif()
    if (NOT ALSA_FOUND)
      set (WEBRTCARGS rtc_include_internal_audio_device=false\n${WEBRTCARGS})
    endif()
  endif()

  # compilation mode depending on target
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "armv6.*")
    set (WEBRTCARGS target_cpu="arm"\narm_version=6\narm_float_abi="hard"\nenable_libaom=false\n${WEBRTCARGS})
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "armv.*")
    set (WEBRTCARGS target_cpu="arm"\n${WEBRTCARGS})
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
    set (WEBRTCARGS target_cpu="arm64"\n${WEBRTCARGS})
  endif()
  if (DEFINED CMAKE_FIND_ROOT_PATH)
    set (WEBRTCARGS use_sysroot=true\n${WEBRTCARGS})
  else()
    set (WEBRTCARGS use_sysroot=false\n${WEBRTCARGS})
  endif()
  set (WEBRTCARGS is_clang=true\n${WEBRTCARGS})

  # screen capture support
  find_package(PkgConfig QUIET)
  pkg_check_modules(GTK3 QUIET gtk+-3.0)
  MESSAGE("GTK_FOUND = ${GTK3_FOUND}")
  if(NOT GTK3_FOUND OR (WEBRTCDESKTOPCAPTURE STREQUAL "OFF"))
    set (WEBRTCARGS rtc_use_x11=false\nrtc_use_pipewire=false\n${WEBRTCARGS})
  endif()

  # write conf file
  FILE(WRITE ${WEBRTCROOT}/src/out/${CMAKE_BUILD_TYPE}/args.gn ${WEBRTCARGS})
endif() 

set (WEBRTCOBJS ${WEBRTCROOT}/src/out/${CMAKE_BUILD_TYPE}/obj)

if (NOT EXISTS ${WEBRTCOBJS}/${CMAKE_STATIC_LIBRARY_PREFIX}webrtc${CMAKE_STATIC_LIBRARY_SUFFIX})
  if (WIN32)
    SET (SHELLCOMMAND cmd /c )
  endif()
  EXECUTE_PROCESS(WORKING_DIRECTORY ${WEBRTCROOT}/src/out/${CMAKE_BUILD_TYPE} COMMAND ${SHELLCOMMAND} gn gen .)
  
  SET(NINJA_TARGET webrtc rtc_json jsoncpp builtin_video_decoder_factory builtin_video_encoder_factory peerconnection p2p_server_utils task_queue default_task_queue_factory)
  EXECUTE_PROCESS(WORKING_DIRECTORY ${WEBRTCROOT}/src/out/${CMAKE_BUILD_TYPE} COMMAND ninja ${NINJA_TARGET})
endif() 


FILE(GLOB_RECURSE WEBRTBBRCOBJS ${WEBRTCOBJS}/modules/congestion_controller/bbr/*${CMAKE_C_OUTPUT_EXTENSION})
FILE(GLOB_RECURSE WEBRTJSONCPPCOBJS ${WEBRTCOBJS}/third_party/jsoncpp/jsoncpp/*${CMAKE_C_OUTPUT_EXTENSION})
FILE(GLOB_RECURSE WEBRTP2POBJ ${WEBRTCOBJS}/p2p/p2p_server_utils/*${CMAKE_C_OUTPUT_EXTENSION})
FILE(GLOB_RECURSE WEBRTTASK ${WEBRTCOBJS}/api/task_queue/*${CMAKE_C_OUTPUT_EXTENSION} ${WEBRTCOBJS}/rtc_base/rtc_task_queue_*/*${CMAKE_C_OUTPUT_EXTENSION})
SET (WEBRTCEXTRAOBJS ${WEBRTJSONCPPCOBJS} ${WEBRTBBRCOBJS} ${WEBRTP2POBJ} ${WEBRTTASK} ${WEBRTCOBJS}/rtc_base/rtc_json/json${CMAKE_C_OUTPUT_EXTENSION})	

# project target
FILE(GLOB SOURCE src/*.cpp)
add_executable (${CMAKE_PROJECT_NAME} ${SOURCE} ${WEBRTCEXTRAOBJS})
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE inc) 

# webrtc 
set (WEBRTCINCLUDE ${WEBRTCROOT}/src ${WEBRTCROOT}/src/third_party/abseil-cpp ${WEBRTCROOT}/src/third_party/jsoncpp/source/include  ${WEBRTCROOT}/src/third_party/jsoncpp/generated ${WEBRTCROOT}/src/third_party/libyuv/include)
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${WEBRTCINCLUDE})
target_link_libraries (${CMAKE_PROJECT_NAME} ${WEBRTCOBJS}/${CMAKE_STATIC_LIBRARY_PREFIX}webrtc${CMAKE_STATIC_LIBRARY_SUFFIX}) 
add_definitions(-DHAVE_JPEG)

# thread
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package (Threads)
target_link_libraries (${CMAKE_PROJECT_NAME} Threads::Threads) 

# civetweb
add_library (civetweb STATIC civetweb/src/civetweb.c civetweb/src/CivetServer.cpp)	
add_definitions(-DOPENSSL_API_1_1)
target_link_libraries (${CMAKE_PROJECT_NAME} civetweb) 
target_include_directories(civetweb PUBLIC civetweb/include) 

# compiler specific
if (WIN32)
    # overide compilation flags
    set(CompilerFlags CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_C_FLAGS_RELEASE)
    foreach(CompilerFlag ${CompilerFlags})
      string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
    endforeach()

    # getopt
    add_library (getopt getopt/getopt.c)
    target_include_directories(getopt PUBLIC getopt) 
    target_link_libraries (${CMAKE_PROJECT_NAME} getopt)
    # webrtc
    add_definitions(-DWEBRTC_WIN -EHsc -DNOMINMAX -D__PRETTY_FUNCTION__=__FUNCTION__ -D_HAS_ITERATOR_DEBUGGING=0)
    add_definitions(-DUSE_X11 -DHAVE_SOUND -D_WINSOCKAPI_)
    target_link_libraries (${CMAKE_PROJECT_NAME} secur32 dmoguids wmcodecdspuuid strmiids msdmo winmm dxgi d3d11)
else()
    # libv4l2cpp
    add_definitions(-DHAVE_V4L2)
    aux_source_directory(libv4l2cpp/src LIBSRC_FILES)
    include_directories("libv4l2cpp/inc")
    add_library(libv4l2cpp STATIC ${LIBSRC_FILES})
    target_link_libraries (${PROJECT_NAME} libv4l2cpp) 

    # webrtc
    add_definitions(-DWEBRTC_POSIX -fno-rtti)
    if (CMAKE_SYSTEM_PROCESSOR MATCHES "armv6.*")
        add_definitions(-marm -march=armv6 -mfpu=vfp -mfloat-abi=hard)
    endif()
    if (EXISTS ${WEBRTCROOT}/src/out/${CMAKE_BUILD_TYPE}/obj/modules/desktop_capture/desktop_capture.ninja)
        add_definitions(-DUSE_X11)
        target_link_libraries (${CMAKE_PROJECT_NAME} X11 Xext Xdamage Xfixes Xcomposite Xrandr Xtst)
    endif()
    # civetweb
    add_definitions(-DCRYPTO_LIB="libcrypto.so.1.1" -DSSL_LIB="libssl.so.1.1")
    target_link_libraries (${CMAKE_PROJECT_NAME} dl)
    #alsa
    if (ALSA_FOUND)
        add_definitions(-DHAVE_SOUND)
    endif()
endif()

# prometheus 
include(GenerateExportHeader)
include(GNUInstallDirs)
set(PROJECT_NAME prometheus-cpp)
add_subdirectory(prometheus-cpp/core EXCLUDE_FROM_ALL)
target_link_libraries (${CMAKE_PROJECT_NAME} core)  
target_include_directories(${CMAKE_PROJECT_NAME} PUBLIC prometheus-cpp/core/include)

# live555helper 
add_subdirectory(live555helper EXCLUDE_FROM_ALL)
target_link_libraries (${CMAKE_PROJECT_NAME} liblive555helper) 
add_definitions(-DHAVE_LIVE555 -DNO_OPENSSL=1)

# static link of libatomic on armv6l
if (CMAKE_SYSTEM_PROCESSOR MATCHES "armv6.*")
    target_link_libraries (${CMAKE_PROJECT_NAME} -Wl,-Bstatic -latomic -Wl,-Bdynamic)
endif()
# static link of stdc++ if available
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-static-libgcc" CXX_SUPPORTS_STATIC_LIBGCC)
if (CXX_SUPPORTS_STATIC_LIBGCC) 
  target_link_libraries (${CMAKE_PROJECT_NAME} -static-libgcc)
endif()    
CHECK_CXX_COMPILER_FLAG("-static-libstdc++" CXX_SUPPORTS_STATIC_LIBSTDCPP)
if (CXX_SUPPORTS_STATIC_LIBSTDCPP) 
  target_link_libraries (${CMAKE_PROJECT_NAME} -static-libstdc++)
endif()    

#cpack
install (TARGETS ${CMAKE_PROJECT_NAME} RUNTIME DESTINATION .)
install (DIRECTORY html DESTINATION .)
install (FILES config.json DESTINATION .)
install (FILES Procfile DESTINATION .)
SET(CPACK_GENERATOR "TGZ")
SET(CPACK_SYSTEM_NAME ${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_BUILD_TYPE})
SET(CPACK_PACKAGE_CONTACT "michel.promonet@free.fr") 
if(PROJECTVERSION)
    SET(CPACK_PACKAGE_VERSION "${PROJECTVERSION}")
endif()
INCLUDE(CPack)
