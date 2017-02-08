#########################################################################################
#
# CMakeLists.txt for CliffMetrics
#
#########################################################################################
# Set CMake defaults
cmake_minimum_required(VERSION 2.8.12)


#########################################################################################
# Detect the environment, and set up compiler-independent stuff
if(UNIX)
   message("Creating Unix Makefile for CliffMetrics")
endif(UNIX)
if(WIN32)
   message("Creating Windows Makefile for CliffMetrics")
endif(WIN32)
if(CYGWIN)
   message("Creating Cygwin Makefile for CliffMetrics")
endif(CYGWIN)
if(MINGW)
   message("Creating MinGW Makefile for CliffMetrics")
endif(MINGW)

# Set up some c++ and architecture defaults
enable_language(CXX)
set(FIND_LIBRARY_USE_LIB64_PATHS true)

# Set up project-specific defaults
project(CLIFF)
file(GLOB CLIFF_SOURCE_FILES *.cpp)
set(CLIFF_EXECUTABLE cliff)
set(CLIFF_INSTALL_DIR "$ENV{HOME}/projects/CliffMetrics")                # May be changed by the user

if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE Debug)                                 # If not specified, assume a Debug build
endif(NOT CMAKE_BUILD_TYPE)

if(CMAKE_BUILD_TYPE MATCHES Debug)
   set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_DEBUG})
   set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CLIFF_BINARY_DIR}/Debug CACHE PATH "Directory for Debug builds" FORCE)
   # For Debug builds on any platform, add the model's "internal" debug flag (used for extra logging) etc.
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_DEBUG")
endif(CMAKE_BUILD_TYPE MATCHES Debug)

if(CMAKE_BUILD_TYPE MATCHES Release)
   set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_RELEASE})
   set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CLIFF_BINARY_DIR}/Release CACHE PATH "Directory for Release builds" FORCE)
endif(CMAKE_BUILD_TYPE MATCHES Release)

if(CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)
   set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_RELWITHDEBINFO})
   set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CLIFF_BINARY_DIR}/RelWithDebInfo CACHE PATH "Directory for RelWithDebInfo builds" FORCE)
endif(CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)

if(CMAKE_BUILD_TYPE MATCHES MinSizeRel)
   set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_MINSIZEREL})
   set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CLIFF_BINARY_DIR}/MinSizeRel CACHE PATH "Directory for MinSizeRel builds" FORCE)
endif(CMAKE_BUILD_TYPE MATCHES MinSizeRel)

# Set up compiler-independent flags and build directories for the various build types
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCPU -Dx86")          # TODO Need to change this if different CPU architecture


#########################################################################################
# Detect the compiler and set compiler-specific flags etc. TODO this isn't finished
message("Checking compiler")

if(CMAKE_COMPILER_IS_GNUCC)
   message("C++ compiler is ${CMAKE_CXX_COMPILER_ID} g++ ${CMAKE_CXX_COMPILER_VERSION}")

   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -W -Wall -Wpointer-arith -Wcast-qual -Wcast-align -Wwrite-strings -Wredundant-decls")

   # For Debug builds under gcc
   if(CMAKE_BUILD_TYPE MATCHES Debug)
      if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "4.8")
         # Undefined behaviour sanitizer introduced with gcc 4.9, see http://developerblog.redhat.com/2014/10/16/gcc-undefined-behavior-sanitizer-ubsan/
#         set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined -fsanitize=shift -fsanitize=integer-divide-by-zero -fsanitize=unreachable -fsanitize=null -fsanitize=return -fsanitize=address")
      endif(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "4.8")

      # TODO I think these were introduced in a later gcc version, need to check
      if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "5.0")
         # Undefined behaviour sanitizer introduced with gcc 4.9, see http://developerblog.redhat.com/2014/10/16/gcc-undefined-behavior-sanitizer-ubsan/
#         set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=vla-bound -fsanitize=float-divide-by-zero -fsanitize=float-cast-overflow -fsanitize=bounds -fsanitize=alignment -fsanitize=object-size -fsanitize=vptr")
         # TODO not yet working
#         set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mmpx -fcheck-pointer-bounds -fsanitize-blacklist=blacklist.txt")
      endif(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "5.0")


#   if(CMAKE_BUILD_TYPE MATCHES Debug)
#      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Weffc++")
   endif(CMAKE_BUILD_TYPE MATCHES Debug)

   # For RelWithDebInfo builds under gcc, set flag for profiling with gprof
   if(CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pg")
   endif(CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)

   # For Release builds under gcc, add an extra flag
   if(CMAKE_BUILD_TYPE MATCHES Release)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fomit-frame-pointer")
   endif(CMAKE_BUILD_TYPE MATCHES Release)

   # Finally set some gcc-specific linker flags
   set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--as-needed -Wl,--no-undefined")
endif(CMAKE_COMPILER_IS_GNUCC)

if(MSVC)
   message("Compiler is Visual Studio")

   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W3")
endif(MSVC)


#########################################################################################
# Find libraries
message("Finding libraries")

# The model cannot be built without GDAL
find_package(GDAL REQUIRED)
message(STATUS "GDAL_LIBRARIES=${GDAL_LIBRARIES}")
message(STATUS "GDAL_INCLUDE_DIRS=${GDAL_INCLUDE_DIRS}")
set(LIBS ${LIBS} ${GDAL_LIBRARIES})
set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} ${GDAL_INCLUDE_DIRS})
message(STATUS "LIBS=${LIBS}")
message(STATUS "CMAKE_INCLUDE_PATH=${CMAKE_INCLUDE_PATH}")

# Added by DFM, stolen from https://github.com/qgis/QGIS/blob/master/cmake/FindGDAL.cmake
set(GDAL_CONFIG_PREFER_PATH "$ENV{GDAL_HOME}/bin" CACHE STRING "preferred path to GDAL (gdal-config)")
set(GDAL_CONFIG_PREFER_FWTOOLS_PATH "$ENV{FWTOOLS_HOME}/bin_safe" CACHE STRING "preferred path to GDAL (gdal-config) from FWTools")
find_program(GDAL_CONFIG gdal-config
   ${GDAL_CONFIG_PREFER_PATH}
   ${GDAL_CONFIG_PREFER_FWTOOLS_PATH}
   /usr/local/bin/
   /usr/bin/
)
message("DBG GDAL_CONFIG ${GDAL_CONFIG}")

#########################################################################################
# Finally, the important bits
message("Writing Makefile")
include_directories(SYSTEM ${CMAKE_INCLUDE_PATH})
include_directories(${CLIFF_SOURCE_DIR})
add_executable(${CLIFF_EXECUTABLE} ${CLIFF_SOURCE_FILES})
target_link_libraries(${CLIFF_EXECUTABLE} ${LIBS})
install(TARGETS ${CLIFF_EXECUTABLE} RUNTIME DESTINATION ${CLIFF_INSTALL_DIR})

# Tell the user what has happened
message(STATUS "CMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}")
message(STATUS "CMAKE_EXE_LINKER_FLAGS=${CMAKE_EXE_LINKER_FLAGS}")
message(STATUS "CMAKE_RUNTIME_OUTPUT_DIRECTORY=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
message(STATUS "CMAKE_INCLUDE_PATH=${CMAKE_INCLUDE_PATH}")
message(STATUS "LIBS=${LIBS}")
message(STATUS "CLIFF_INSTALL_DIR=${CLIFF_INSTALL_DIR}")
message(STATUS "CMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}")
message(STATUS "CMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")


#########################################################################################
