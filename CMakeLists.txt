#-------------------------------------------------------------------------------
#	CMAKE SETTINGS
#-------------------------------------------------------------------------------

cmake_minimum_required(VERSION 2.8.12)
set(QT_MINIMUM_VERSION 5.9.0)

#-------------------------------------------------------------------------------
#	SOURCE CODE
#-------------------------------------------------------------------------------

# .h files
list(APPEND HEADER_FILES

)

# .cpp files
list(APPEND SOURCE_FILES
src/main.cpp
)

# on windows, append headers to source so they show up in visual studio
if(WIN32)
	LIST(APPEND SOURCE_FILES ${HEADER_FILES})
endif()

#-------------------------------------------------------------------------------
#	PROJECT SETTINGS
#-------------------------------------------------------------------------------

set(TARGET_NAME cmake-qt-project)

# Use git branch as project name if clones from git
find_package(Git)
if(GIT_FOUND)
	execute_process(
	  COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
	  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
	  OUTPUT_VARIABLE BRANCH
	  OUTPUT_STRIP_TRAILING_WHITESPACE
	)
	execute_process(
	  COMMAND ${GIT_EXECUTABLE} describe --abbrev=0 --tags
	  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
	  OUTPUT_VARIABLE VERSION
	  OUTPUT_STRIP_TRAILING_WHITESPACE
	)
	set(PROJECT_NAME ${TARGET_NAME}-${BRANCH})
else()
	set(PROJECT_NAME ${TARGET_NAME})
endif()

# Set the project name
PROJECT(${PROJECT_NAME})

#-------------------------------------------------------------------------------
#	Qt Settings
#-------------------------------------------------------------------------------

# Required Qt settings
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(QT_USE_QTMAIN TRUE)

# Setup the cmake prefix path so Qt can be found. Use the QTDIR variable if defined
if(NOT DEFINED ENV{QTDIR})
	MESSAGE(STATUS "QTDIR environment variable not set. Be sure to set the QTDIR
	cmake cache variable to the Qt root install location")
endif()

set(QTDIR $ENV{QTDIR} CACHE STRING "Qt install path")
list(APPEND CMAKE_PREFIX_PATH ${QTDIR})

# Find the necessary Qt5 modules
find_package(Qt5Core REQUIRED)
find_package(Qt5Gui REQUIRED)
find_package(Qt5Widgets REQUIRED)

# Test for supported Qt version
find_program(QMAKE_EXECUTABLE NAMES qmake HINTS ${QTDIR} ENV QTDIR PATH_SUFFIXES bin)
execute_process(COMMAND ${QMAKE_EXECUTABLE} -query QT_VERSION OUTPUT_VARIABLE QT_VERSION)
if(QT_VERSION LESS QT_MINIMUM_VERSION)
	MESSAGE(FATAL_ERROR "Minimum supported Qt version: ${QT_MINIMUM_VERSION}. Installed version: ${QT_VERSION}")
endif()

# find thread library 
find_package(Threads REQUIRED)

#-------------------------------------------------------------------------------
#	RESOURCES
#-------------------------------------------------------------------------------

#list(APPEND QT_RESOURCES
#resources/resources.qrc
#)
#
#qt5_add_resources(RESOURCES ${QT_RESOURCES})
#
#list(APPEND RESOURCES
#resources/resources.rc
#)

#-------------------------------------------------------------------------------
#	COMPILE FLAGS
#-------------------------------------------------------------------------------

set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 17)

if(MSVC)
	set(GUI_TYPE WIN32)
	add_compile_options(/MP)
elseif(CMAKE_COMPILER_IS_GNUCXX)
	add_compile_options(-std=c++17)
endif(MSVC)

#-------------------------------------------------------------------------------
#	CONFIGURATION
#-------------------------------------------------------------------------------

set(APPLICATION_NAME ${TARGET_NAME})
set(APPLICATION_VERSION ${VERSION})
set(APPLICATION_ORGANIZATION "Menari Softworks")
set(APPLICATION_ORGANIZATION_DOMAIN "https://github.com/nholthaus/")

configure_file(src/appinfo.h.in appinfo.h @ONLY NEWLINE_STYLE UNIX)

#-------------------------------------------------------------------------------
#	SOURCE GROUPS
#-------------------------------------------------------------------------------

source_group(generated REGULAR_EXPRESSION ".*_automoc.cpp")
source_group(resources REGULAR_EXPRESSION ".*[.][q]?rc")
source_group(source REGULAR_EXPRESSION ".*[.]cpp")
source_group(headers REGULAR_EXPRESSION ".*[.]h")
source_group(continuous-integration REGULAR_EXPRESSION ".*[.]yml")

#-------------------------------------------------------------------------------
#	ADDITIONAL LIBRARIES
#-------------------------------------------------------------------------------

list(APPEND LIBRARIES
Qt5::Core
Qt5::Gui
Qt5::Widgets
Threads::Threads
)

# Remove duplicate library dependencies. 
list(REMOVE_DUPLICATES LIBRARIES)

#-------------------------------------------------------------------------------
#	GENERATE EXECUTABLE
#-------------------------------------------------------------------------------

# Set additional include directories
include_directories(${INCLUDE_DIRS})

# Generate the executable
add_executable(${TARGET_NAME} ${GUI_TYPE} ${SOURCE_FILES} ${RESOURCES})

# link libraries
target_link_libraries(${TARGET_NAME} ${LIBRARIES})

# Use the automatically determined RPATH values
set_target_properties(${TARGET_NAME} PROPERTIES INSTALL_RPATH_USE_LINK_PATH TRUE)

INSTALL(TARGETS ${TARGET_NAME} RUNTIME DESTINATION bin)
INCLUDE(InstallRequiredSystemLibraries)

# Windows specific build steps
if(WIN32)
	# Run winddeployqt if it can be found
	find_program(WINDEPLOYQT_EXECUTABLE NAMES windeployqt HINTS ${QTDIR} ENV QTDIR PATH_SUFFIXES bin)
	add_custom_command(TARGET ${TARGET_NAME} POST_BUILD
	COMMAND ${WINDEPLOYQT_EXECUTABLE} $<TARGET_FILE:${TARGET_NAME}>)
	# Gather up all the dependencies in the bin dir
	# Lack of a trailing slash means install this directory, trailing slash means install everything in this directory.
	install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/Release/
			RUNTIME DESTINATION bin
			FILES_MATCHING
			PATTERN "_CPack*" EXCLUDE
			PATTERN "*.dll"
			)
endif()