PROJECT(ACNG CXX C)

FILE(GLOB SRCS "${CMAKE_CURRENT_SOURCE_DIR}/*.cc")
list(REMOVE_ITEM SRCS ${OPTSOURCES})

ADD_EXECUTABLE(apt-cacher-ng ${SRCS})
TARGET_LINK_LIBRARIES(apt-cacher-ng ${ServerLibs} ${CompLibs} ${SSL_LIB_LIST} ${lsd_LDFLAGS} ${EXTRA_LIBS_ACNG})
# I'd like this method more but apparently LINK_FLAGS is always prepended, no way to append it
#SET_TARGET_PROPERTIES(apt-cacher-ng PROPERTIES LINK_FLAGS "${lsd_LDFLAGS}")
SET_TARGET_PROPERTIES(apt-cacher-ng PROPERTIES COMPILE_FLAGS "${ACNG_CXXFLAGS_COMMON} ${lsd_CFLAGS}")
INSTALL(TARGETS apt-cacher-ng DESTINATION ${BINDIR})