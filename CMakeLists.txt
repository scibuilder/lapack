cmake_minimum_required(VERSION 2.8)
project(LAPACK Fortran C)

option(BUILD_SHARED_LIBS "Build shared libraries" ON)

# Add the CMake directory for custon CMake modules
set(CMAKE_MODULE_PATH "${LAPACK_SOURCE_DIR}/CMAKE" ${CMAKE_MODULE_PATH})

# Configure the warning and code coverage suppression file
configure_file( 
  "${LAPACK_SOURCE_DIR}/CTestCustom.cmake.in"
  "${LAPACK_BINARY_DIR}/CTestCustom.cmake"
  COPYONLY
)

if(UNIX)
  if("${CMAKE_Fortran_COMPILER}" MATCHES "ifort")
    set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port")
  endif()
  if("${CMAKE_Fortran_COMPILER}" MATCHES "xlf")
    set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -qnosave -qstrict=none")
  endif ()
  # Delete libmtsk in linking sequence for Sun/Oracle Fortran Compiler.
  # This library is not present in the Sun package 
  # SolarisStudio12.3-linux-x86-bin
  string(REPLACE \;mtsk\; \; CMAKE_Fortran_IMPLICIT_LINK_LIBRARIES "${CMAKE_Fortran_IMPLICIT_LINK_LIBRARIES}")
endif()

# Get Python
find_package(PythonInterp)
message(STATUS "Looking for Python found - ${PYTHONINTERP_FOUND}")
if(PYTHONINTERP_FOUND)
  message(STATUS "Using Python version ${PYTHON_VERSION_STRING}")
endif()

# On Windows-GNU builds try to provide MS import libraries too.
if(BUILD_SHARED_LIBS)
  include(GNUtoMS)
endif()

if(GNUtoMS)
  set(LAPACK_GNUtoMS_IMPORT ${LAPACK_SOURCE_DIR}/CMAKE/lapack-GNUtoMS.cmake)
  set(LAPACK_GNUtoMS_INSTALL "include(\${_SELF_DIR}/lapack-GNUtoMS.cmake)")
  set(LAPACK_GNUtoMS_BUILD "include(\"${LAPACK_GNUtoMS_IMPORT}\")")
endif()

macro(lapack_install_library lib)
  install(TARGETS ${lib} EXPORT lapack-targets
    ARCHIVE DESTINATION lib${LIB_SUFFIX}
    LIBRARY DESTINATION lib${LIB_SUFFIX}
    RUNTIME DESTINATION bin
  )
  if(GNUtoMS)
    install(FILES ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}/lib${lib}.lib DESTINATION lib)
  endif()
endmacro()

# --------------------------------------------------
# Testing

enable_testing()
include(CTest)
enable_testing()
# --------------------------------------------------

# Organize output files.  On Windows this also keeps .dll files next
# to the .exe files that need them, making tests easy to run.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${LAPACK_BINARY_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${LAPACK_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${LAPACK_BINARY_DIR}/lib)

# --------------------------------------------------
# Check for any necessary platform specific compiler flags
include(CheckLAPACKCompilerFlags)
CheckLAPACKCompilerFlags()

# --------------------------------------------------
# Check second function

include(CheckTimeFunction)
set(TIME_FUNC NONE ${TIME_FUNC})
CHECK_TIME_FUNCTION(NONE TIME_FUNC)
CHECK_TIME_FUNCTION(INT_CPU_TIME TIME_FUNC)
CHECK_TIME_FUNCTION(EXT_ETIME TIME_FUNC)
CHECK_TIME_FUNCTION(EXT_ETIME_ TIME_FUNC)
CHECK_TIME_FUNCTION(INT_ETIME TIME_FUNC)
message(STATUS "--> Will use second_${TIME_FUNC}.f and dsecnd_${TIME_FUNC}.f as timing function.")

set(SECOND_SRC  ${LAPACK_SOURCE_DIR}/INSTALL/second_${TIME_FUNC}.f)
set(DSECOND_SRC  ${LAPACK_SOURCE_DIR}/INSTALL/dsecnd_${TIME_FUNC}.f)
set(prefix ${CMAKE_INSTALL_PREFIX})
set(libdir ${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX})
set(PKG_CONFIG_DIR ${libdir}/pkgconfig)

# --------------------------------------------------
# Precision to build
# By default all precisions are generated


# --------------------------------------------------
# Subdirectories that need to be processed

# Check the usage of the user provided BLAS libraries
if(BLAS_LIBRARIES)
  include(CheckFortranFunctionExists)
  set(CMAKE_REQUIRED_LIBRARIES ${BLAS_LIBRARIES})
  CHECK_FORTRAN_FUNCTION_EXISTS("dgemm" BLAS_FOUND)
  unset(CMAKE_REQUIRED_LIBRARIES)
  if(BLAS_FOUND)
    message(STATUS "--> BLAS supplied by user is WORKING, will use ${BLAS_LIBRARIES}.")
  else()
    message(ERROR "--> BLAS supplied by user is not WORKING, CANNOT USE ${BLAS_LIBRARIES}.")
  endif()
endif()

if(NOT DISABLE_BLIS)
  include(ExternalBLIS)
  if(HAVE_BLIS)
    set(BLAS_FOUND TRUE)
    set(BLAS_LIBRARIES ${BLIS_LIBS})
    message(STATUS "Will use BLIS via BLAS_LIBRARIES=${BLAS_LIBRARIES}")
    if(BLIS_LINK_FLAGS)
      message(STATUS "Will link BLIS using OpenMP: ${OpenMP_C_FLAGS}")
    else()
      message(STATUS "Will *NOT* link BLIS using OpenMP")
    endif()
  elseif(BUILD_BLIS)
    message(FATAL_ERROR "Demand that BLIS be built could not be fulfilled") 
  endif()
endif()

if(NOT HAVE_BLIS AND NOT DISABLE_NETLIB_BLAS)
  find_package(BLAS)
endif()

# Neither user specified or optimized BLAS libraries can be used
if(NOT BLAS_FOUND)
  message(STATUS "Using NETLIB BLAS implementation")
  add_subdirectory(BLAS)
  set(BLAS_LIBRARIES blas)
else()
  set(CMAKE_EXE_LINKER_FLAGS 
    "${CMAKE_EXE_LINKER_FLAGS} ${BLAS_LINKER_FLAGS}" 
    CACHE STRING "Linker flags for executables" FORCE)
  set(CMAKE_MODULE_LINKER_FLAGS 
    "${CMAKE_MODULE_LINKER_FLAGS} ${BLAS_LINKER_FLAGS}" 
    CACHE STRING "Linker flags for modules" FORCE)
  set(CMAKE_SHARED_LINKER_FLAGS 
    "${CMAKE_SHARED_LINKER_FLAGS} ${BLAS_LINKER_FLAGS}" 
    CACHE STRING "Linker flags for shared libs" FORCE)
endif()

option(USE_XBLAS "Build extended precision (needs XBLAS)" OFF)
if(USE_XBLAS)
  find_library(XBLAS_LIBRARY NAMES xblas)
endif()
   
message(STATUS "Using supplied NETLIB LAPACK implementation")
set(LAPACK_LIBRARIES lapack)
option(BUILD_SINGLE "Build LAPACK Single Precision" ON)
option(BUILD_DOUBLE "Build LAPACK Double Precision" ON)
option(BUILD_COMPLEX "Build LAPACK Complex Precision" ON)
option(BUILD_COMPLEX16 "Build LAPACK Double Complex Precision" ON)
add_subdirectory(SRC)

message(STATUS "BUILD TESTING : ${BUILD_TESTING}" )
if(BUILD_TESTING)
  add_subdirectory(TESTING)
endif()

# --------------------------------------------------
# LAPACKE
option(LAPACKE "Build LAPACKE" OFF)

# LAPACKE has also the interface to some routines from tmglib,
# if LAPACKE_WITH_TMG is selected, we need to add those routines to LAPACKE
option(LAPACKE_WITH_TMG "Build LAPACKE with tmglib routines" OFF)
if(LAPACKE_WITH_TMG)
  set(LAPACKE ON)
  if(NOT BUILD_TESTING)
    add_subdirectory(TESTING/MATGEN)
  endif()
endif()

if(LAPACKE)
  add_subdirectory(lapacke)
endif()

# --------------------------------------------------
# CPACK Packaging 

set(CPACK_PACKAGE_NAME "LAPACK")
set(CPACK_PACKAGE_VENDOR "University of Tennessee, Univ. of California Berkeley, Univ. of Colorado Denver and NAG Ltd")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "LAPACK- Linear Algebra Package")
set(LAPACK_VERSION 3.4.2)
set(CPACK_PACKAGE_VERSION_MAJOR 3)
set(CPACK_PACKAGE_VERSION_MINOR 4)
set(CPACK_PACKAGE_VERSION_PATCH 2)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "LAPACK")
if(WIN32 AND NOT UNIX)
  # There is a bug in NSI that does not handle full unix paths properly. Make
  # sure there is at least one set of four (4) backlasshes.
  set(CPACK_NSIS_HELP_LINK "http:\\\\\\\\http://icl.cs.utk.edu/lapack-forum")
  set(CPACK_NSIS_URL_INFO_ABOUT "http:\\\\\\\\www.netlib.org/lapack")
  set(CPACK_NSIS_CONTACT "lapack@eecs.utk.edu")
  set(CPACK_NSIS_MODIFY_PATH ON)
  set(CPACK_NSIS_DISPLAY_NAME "LAPACK-${LAPACK_VERSION}")
  set(CPACK_PACKAGE_RELOCATABLE "true")
else()
  set(CPACK_GENERATOR "TGZ")
  set(CPACK_SOURCE_GENERATOR TGZ)
  set(CPACK_SOURCE_PACKAGE_FILE_NAME "lapack-${LAPACK_VERSION}" )
  set(CPACK_SOURCE_IGNORE_FILES ~$ .svn ${CPACK_SOURCE_IGNORE_FILES} )
endif()
include(CPack)

if(NOT BLAS_FOUND)
  set(ALL_TARGETS ${ALL_TARGETS} blas)
endif()

if(NOT LATESTLAPACK_FOUND)
  set(ALL_TARGETS ${ALL_TARGETS} lapack)
endif()

if(BUILD_TESTING OR LAPACKE_WITH_TMG)
  set(ALL_TARGETS ${ALL_TARGETS} tmglib)
endif()

if(LAPACKE)
  set(ALL_TARGETS ${ALL_TARGETS} lapacke)
endif()

export(TARGETS ${ALL_TARGETS} FILE lapack-targets.cmake)

configure_file(${LAPACK_SOURCE_DIR}/CMAKE/lapack-config-version.cmake.in
  ${LAPACK_BINARY_DIR}/lapack-config-version.cmake @ONLY)
configure_file(${LAPACK_SOURCE_DIR}/CMAKE/lapack-config-build.cmake.in
  ${LAPACK_BINARY_DIR}/lapack-config.cmake @ONLY)


configure_file(${CMAKE_CURRENT_SOURCE_DIR}/lapack.pc.in ${CMAKE_CURRENT_BINARY_DIR}/lapack.pc)
  install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/lapack.pc
  DESTINATION ${PKG_CONFIG_DIR}
   )

configure_file(${LAPACK_SOURCE_DIR}/CMAKE/lapack-config-install.cmake.in
  ${LAPACK_BINARY_DIR}/CMakeFiles/lapack-config.cmake @ONLY)
install(FILES
  ${LAPACK_GNUtoMS_IMPORT}
  ${LAPACK_BINARY_DIR}/CMakeFiles/lapack-config.cmake
  ${LAPACK_BINARY_DIR}/lapack-config-version.cmake
  DESTINATION lib/cmake/lapack-${LAPACK_VERSION}
  )

install(EXPORT lapack-targets DESTINATION lib/cmake/lapack-${LAPACK_VERSION})
