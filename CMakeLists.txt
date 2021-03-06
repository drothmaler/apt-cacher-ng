cmake_minimum_required(VERSION 2.6.2)

set(CMAKE_LEGACY_CYGWIN_WIN32 0) # don't care for now... just expecting POSIX

# we want the old policy to support cmake 2.x
IF(POLICY CMP0056) # added in CMake 3.2
   cmake_policy (SET CMP0056 OLD)
ENDIF()
IF (POLICY CMP0053) # added in CMake 3.1
	cmake_policy (SET CMP0053 OLD) # use old-style @VAR@ expansion
ENDIF (POLICY CMP0053)

PROJECT(ACNG CXX C)

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR})
set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR})
set(CMAKE_SKIP_BUILD_RPATH on) # no -rdynamic needed ever

INCLUDE(CheckIncludeFiles) 
INCLUDE(CheckCXXSourceCompiles)
INCLUDE(CheckCXXCompilerFlag)
INCLUDE(CheckTypeSize)
INCLUDE(TestBigEndian)
INCLUDE(CheckLibraryExists)
INCLUDE(FindPkgConfig)

set(PACKAGE "apt-cacher-ng")
IF(NOT DEFINED(CMAKE_INSTALL_PREFIX))
set(CMAKE_INSTALL_PREFIX "/usr/local" CACHE STRING "Target file space")
ENDIF()
IF(NOT DEFINED(DATADIR))
	set(DATADIR "${CMAKE_INSTALL_PREFIX}/share" CACHE STRING "Shared data files directory")
ENDIF()
IF(NOT DEFINED(LIBDIR))
	set(LIBDIR "${CMAKE_INSTALL_PREFIX}/lib/${PACKAGE}" CACHE STRING "Location of ${PACKAGE} extra files")
ENDIF()
IF(NOT DEFINED(DOCDIR))
	set(DOCDIR "${DATADIR}/doc/${PACKAGE}" CACHE STRING "Location of documentation files")
ENDIF()

IF(NOT DEFINED(SYSCONFDIR))
   if("${CMAKE_INSTALL_PREFIX}" STREQUAL "/usr")
      set(SYSCONFDIR /etc)
   else()
      set(SYSCONFDIR ${CMAKE_INSTALL_PREFIX}/etc)
   endif()
ENDIF()

if(NOT DEFINED(CFGDIR))
set(CFGDIR "${SYSCONFDIR}/${PACKAGE}")
endif()

if(NOT DEFINED(SBINDIR))
set(SBINDIR "${CMAKE_INSTALL_PREFIX}/sbin")
endif()

if(NOT DEFINED(RUNDIR))
	set(RUNDIR "/run")
endif()

INCLUDE_DIRECTORIES(${CMAKE_BINARY_DIR} ${CMAKE_SOURCE_DIR} "include")

IF(NOT DEFINED(ACVERSION))
FILE(READ "${CMAKE_SOURCE_DIR}/VERSION" ACVERSION)
string(REPLACE "\n" "" ACVERSION ${ACVERSION})
#MESSAGE(STATUS "Version: ${ACVERSION}")
ENDIF()

# Various feature checks
CHECK_INCLUDE_FILES ("sys/param.h;sys/mount.h" HAVE_SYS_MOUNT_H)
CHECK_INCLUDE_FILES ("sys/vfs.h" HAVE_SYS_VFS_H)
CHECK_TYPE_SIZE(int SIZE_INT)
CHECK_TYPE_SIZE(long SIZE_LONG)
TEST_BIG_ENDIAN(WORDS_BIGENDIAN)

# those are appended later via COMPILER_FLAGS. $CXXFLAGS and CMAKE_CXX_FLAGS
# also influence it, need to be adjusted carefully by user.
SET(ACNG_CXXFLAGS_COMMON "-D_FILE_OFFSET_BITS=64")

foreach(cxxarg  -pthread -Wall -Wextra -Wno-unused-parameter -fvisibility-inlines-hidden -fvisibility=hidden)
	STRING(REGEX REPLACE "=|-|," "" optname "${cxxarg}")
  CHECK_CXX_COMPILER_FLAG("${cxxarg}" "CXX_${optname}")
  if(CXX_${optname})
     SET(ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} ${cxxarg}")
	endif()
endforeach()

foreach(linkarg -Wl,--as-needed -Wl,-O1 -Wl,--discard-all -Wl,--no-undefined -Wl,--build-id=sha1 -Wl,-fuse-ld=gold -Wl,--threads)
	STRING(REGEX REPLACE "=|-|," "" optname "${linkarg}")
	set(CMAKE_REQUIRED_FLAGS "${linkarg}")
	CHECK_CXX_COMPILER_FLAG("" "LD_${optname}")
	if(LD_${optname})
		SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${linkarg}")
	endif()
	set(CMAKE_REQUIRED_FLAGS "")
endforeach(linkarg)

option(USE_SSL "Use OpenSSL library for TLS and other crypto functionality" on)

IF(CMAKE_SYSTEM MATCHES "Darwin")
   SET(ACNG_CXXFLAGS_COMMON " ${ACNG_CXXFLAGS_COMMON} -D_DARWIN_C_SOURCE ")
ENDIF(CMAKE_SYSTEM MATCHES "Darwin")

IF(CMAKE_SYSTEM MATCHES "CYGWIN")
	set(USE_LTO_DEFAULT off)
	SET(ACNG_CXXFLAGS_COMMON " ${ACNG_CXXFLAGS_COMMON}  -U__STRICT_ANSI__ -DNOMINMAX ")
ELSE()
  if(CMAKE_COMPILER_IS_GNUCXX AND CMAKE_CXX_COMPILER_VERSION MATCHES "^[56789]")
     set(USE_LTO_DEFAULT on)
  endif()
ENDIF()

if(CMAKE_BUILD_TYPE MATCHES Debug)
   set(USE_LTO_DEFAULT off)
   set(ACNG_CXXFLAGS_COMMON "-DDEBUG ${ACNG_CXXFLAGS_COMMON}")
else()
   set(CMAKE_REQUIRED_FLAGS "-Wl,--gc-sections")
   CHECK_CXX_COMPILER_FLAG("-Os -fdata-sections -ffunction-sections -Wl,--gc-sections" GC_SECTIONS)
   if(GC_SECTIONS)
      SET(ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} -fdata-sections -ffunction-sections")
      SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections")
   endif()
   set(CMAKE_REQUIRED_FLAGS "")
endif()

FILE(READ test/build/testcxx11.cc CXX11_TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${CXX11_TESTSRC}" COMPILER_SUPPORTS_DEFAULT_CXX11)
if(NOT COMPILER_SUPPORTS_DEFAULT_CXX11)
   # prefer that style if possible since since Cygwin seems to have an old bug with missing definitions, http://cygwin.com/ml/cygwin/2012-04/msg00140.html
   CHECK_CXX_COMPILER_FLAG(-std=gnu++11 COMPILER_SUPPORTS_GNU11)
   if(COMPILER_SUPPORTS_GNU11)
      set(ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} -std=gnu++11")
   else()
      CHECK_CXX_COMPILER_FLAG(-std=c++11 COMPILER_SUPPORTS_CXX11)
      if(COMPILER_SUPPORTS_CXX11)
         set(ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} -std=c++11")
      else()
         CHECK_CXX_COMPILER_FLAG(-std=c++0x COMPILER_SUPPORTS_CXX0X)
         if(COMPILER_SUPPORTS_CXX0X)
            set(ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} -std=c++0x")
         else()
            message(FATAL_ERROR "!! Error: failed to configure compiler for C++11 support. For GCC, version 4.7 or newer is required.")
         endif()
      endif()
   endif()
endif()

# XXX: make this stuff not be printed later?
#if(NOT DEFINED(USE_LTO))
message("-- LTO default choice for this toolchain is: ${USE_LTO_DEFAULT}")
#endif()
option(USE_LTO "Enable Link Time Optimization (requires modern compilers)" ${USE_LTO_DEFAULT})

if(USE_LTO)
   SET(LDFLAGS_NOLTO "${CMAKE_EXE_LINKER_FLAGS}")
   SET(CMAKE_REQUIRED_FLAGS "${ACNG_CXXFLAGS_COMMON} -flto")
   SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -flto")
   CHECK_CXX_SOURCE_COMPILES("${CXX11_TESTSRC}" HAS_LTO)
   if(HAS_LTO)
      SET(ACNG_CXXFLAGS_COMMON ${CMAKE_REQUIRED_FLAGS})
   else()
      SET(CMAKE_REQUIRED_FLAGS "${ACNG_CXXFLAGS_COMMON}")
      SET(CMAKE_EXE_LINKER_FLAGS "${LDFLAGS_NOLTO}")
      message(WARNING "Link Time Optimization support broken, disabling it.")
   endif()
endif()
message("-- LTO final setting is: ${USE_LTO}")

FIND_LIBRARY(HAVE_SOCKETLIB socket) # separate socket lib looks like Solaris-like environment
if(HAVE_SOCKETLIB)
   LIST(APPEND BaseNetworkLibs socket nsl)
endif(HAVE_SOCKETLIB)

# explicit linking needed for global locks in dlcon
list(APPEND BaseNetworkLibs pthread)

SET(CMAKE_REQUIRED_LIBRARIES wrap ${BaseNetworkLibs})
FILE(READ test/build/HAVE_LIBWRAP.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_LIBWRAP)
IF(HAVE_LIBWRAP)
   LIST(APPEND ServerLibs wrap)
ELSE(HAVE_LIBWRAP)
   MESSAGE("!! libwrap development files not usable, disabling support")
   SET(HAVE_LIBWRAP)
ENDIF(HAVE_LIBWRAP)
SET(CMAKE_REQUIRED_LIBRARIES "")

if(CYGWIN)
message("!! Not using wordexp on Cygwin, not reliable")
set(HAVE_WORDEXP off)
else()
FILE(READ test/build/HAVE_WORDEXP.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_WORDEXP)
endif()

# test some methods explicitly where we want to be sure about the signatures
# and linkability

FILE(READ test/build/HAVE_GLOB.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_GLOB)

FILE(READ test/build/HAVE_FADVISE.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_FADVISE)

FILE(READ test/build/HAVE_MADVISE.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_MADVISE)

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")

   FILE(READ test/build/HAVE_LINUX_FALLOCATE.cc TESTSRC)
   CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_LINUX_FALLOCATE)

   FILE(READ test/build/HAVE_LINUX_SENDFILE.cc TESTSRC)
   CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_LINUX_SENDFILE)

   FILE(READ test/build/HAVE_LINUX_EVENTFD.cc TESTSRC)
   CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_LINUX_EVENTFD)

   FILE(READ test/build/HAVE_LINUX_SPLICE.cc TESTSRC)
   CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_LINUX_SPLICE)
endif()

FILE(READ test/build/HAVE_PREAD.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_PREAD)

FILE(READ test/build/HAVE_DAEMON.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_DAEMON)

pkg_check_modules(lsd "libsystemd>=209")
if(NOT lsd_FOUND)
pkg_check_modules(lsd libsystemd-daemon)
endif()
string(REPLACE ";" " " lsd_LDFLAGS "${lsd_LDFLAGS}")
string(REPLACE ";" " " lsd_CFLAGS "${lsd_CFLAGS}")
set(HAVE_SD_NOTIFY ${lsd_FOUND})

SET(CMAKE_REQUIRED_FLAGS ${ACNG_CXXFLAGS_COMMON})
FILE(READ test/build/HAVE_MEMORY_SPTR.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_MEMORY_SPTR)
if(NOT HAVE_MEMORY_SPTR)
   message(FATAL_ERROR "Could not find a working smart pointer implementation.")
endif()

SET(CMAKE_REQUIRED_LIBRARIES dl)
FILE(READ test/build/HAVE_DLOPEN.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_DLOPEN)

# XXX: convert the compressor stuff to pkgconfig usage and print an overview of enabled/found values in one final message and collect and process cflags too

SET(CMAKE_REQUIRED_FLAGS ${ACNG_CXXFLAGS_COMMON})
#INCLUDE(FindZLIB) # broken, hangs for 10 seconds
# header check should be enough, gzip should be everywhere nowadays
#CHECK_INCLUDE_FILES("gzip.h" HAVE_ZLIB)
FIND_PATH(HAVE_ZLIB zlib.h )
if(HAVE_ZLIB)
	list(APPEND CompLibs z)
	INCLUDE_DIRECTORIES(${HAVE_ZLIB})
else(HAVE_ZLIB)
   message(FATAL_ERROR "!! apt-cacher-ng requires gzip library and development files ${HAVE_ZLIB}")
endif(HAVE_ZLIB)

INCLUDE(FindBZip2)
if (BZIP2_FOUND)
   SET(HAVE_LIBBZ2 1)
   MARK_AS_ADVANCED(HAVE_LIBBZ2)
	INCLUDE_DIRECTORIES(${BZIP2_INCLUDE_DIR})
	list(APPEND CompLibs bz2)
else (BZIP2_FOUND)
   message("!! apt-cacher-ng requires bzip2 library and development files for bz2 format support")
endif (BZIP2_FOUND)

SET(CMAKE_REQUIRED_FLAGS ${ACNG_CXXFLAGS_COMMON})
SET(CMAKE_REQUIRED_LIBRARIES lzma)
FILE(READ test/build/HAVE_LZMA.cc TESTSRC)
CHECK_CXX_SOURCE_COMPILES("${TESTSRC}" HAVE_LZMA)
IF(HAVE_LZMA)
	list(APPEND CompLibs lzma)
ELSE(HAVE_LZMA)
   MESSAGE("!! XZ (liblzma) not found or not working, disabling support")
   SET(HAVE_LZMA )
ENDIF(HAVE_LZMA)
SET(CMAKE_REQUIRED_LIBRARIES "")

set(HAVE_CHECKSUM on)

if(USE_SSL)
	pkg_check_modules(openssl openssl)
	if(openssl_FOUND)
		set(HAVE_SSL on)
		string(REPLACE ";" " " ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} ${openssl_CFLAGS}")
		string(REPLACE ";" " " SSL_LIB_LIST "${openssl_LDFLAGS}")
	else()
		message(WARNING "OpenSSL not found, disabling TLS connection support")
		set(HAVE_SSL off)
	endif()
endif()

if(NOT HAVE_SSL)
if(EXISTS "$ENV{TOMCRYPT_HOME}")
message("!! TOMCRYPT_HOME folder found, will use the library from there")
include_directories("$ENV{TOMCRYPT_HOME}/src/headers")
link_directories("$ENV{TOMCRYPT_HOME}")
link_libraries(tomcrypt)
set(tomcrypt_FOUND on)
else()
	pkg_check_modules(tomcrypt "libtomcrypt")
endif()

	if(tomcrypt_FOUND)
		set(HAVE_TOMCRYPT on)
		# with workaround, http://clang.debian.net/logs/2015-03-25/libtomcrypt_1.17-6_unstable_clang.log
    string(REPLACE ";" " " ACNG_CXXFLAGS_COMMON "${ACNG_CXXFLAGS_COMMON} ${tomcrypt_CFLAGS} -DLTC_NO_ROLC -DLTC_BASE64")
		string(REPLACE ";" " " SSL_LIB_LIST "${tomcrypt_LDFLAGS}")
	else()
		set(HAVE_TOMCRYPT off)
		message(WARNING "Could not find LibTomCrypt or use OpenSSL. Some important functionality will be disabled.")
		set(HAVE_CHECKSUM off)
	endif()
endif()

# -DEXTRA_LIBS_INETD=-lsupc++
# funny hack, link with gcc and avoid libstdc++/libm (since STL is not used
# there). However, it needs to be tested - the alternative linking makes
# the binary ~40kb larger, might include higher relocation costs and bigger
# chunks of unique memory while libstdc++ needs to be loaded anyway for the
# server process.
# Needs HAVE_WL_AS_NEEDED!

message("Compiler flags: ${CMAKE_CXX_FLAGS} ${ACNG_CXXFLAGS_COMMON}")
message("Linker flags: ${CMAKE_EXE_LINKER_FLAGS}")

SET(CMAKE_REQUIRED_LIBRARIES "")
SET(CMAKE_REQUIRED_FLAGS ${ACNG_CXXFLAGS_COMMON})

# I don't need -rdynamic, thanks!
SET(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")
SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")

#######################################
# all checks done, save configuration #
#######################################

CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/include/acsyscap.h.in" "${CMAKE_BINARY_DIR}/acsyscap.h")

add_subdirectory(client)
add_subdirectory(fs)
add_subdirectory(source)
add_subdirectory(conf)
add_subdirectory(systemd)

###
### Extra install rules for static files
###
if(NOT DEFINED(HTMLDIR))
		set(HTMLDIR ${DOCDIR}/html)
endif()
if(NOT DEFINED(MANDIR))
   set(MANDIR ${DATADIR}/man)
endif()
install(FILES doc/README doc/apt-cacher-ng.pdf DESTINATION ${DOCDIR})
install(DIRECTORY doc/html/ DESTINATION ${HTMLDIR}
   FILES_MATCHING PATTERN "*.*")
install(DIRECTORY doc/man/ DESTINATION ${MANDIR}/man8
   FILES_MATCHING PATTERN "*.8")
if(NOT DEFINED(AVAHIDIR))
   set(AVAHIDIR "${SYSCONFDIR}/avahi/services")
endif()
install(FILES contrib/apt-cacher-ng.service DESTINATION ${AVAHIDIR})

message("Installation settings:
PACKAGE: ${PACKAGE}
VERSION: ${ACVERSION}
CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}
DATADIR: ${DATADIR}
LIBDIR: ${LIBDIR}
DOCDIR: ${DOCDIR}
SYSCONFDIR: ${SYSCONFDIR}
CFGDIR: ${CFGDIR}
SBINDIR: ${SBINDIR}
")
