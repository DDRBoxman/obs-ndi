cmake_minimum_required(VERSION 3.5)
project(obs-ndi)

set(CMAKE_PREFIX_PATH "${QTDIR}")
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)

include(external/FindLibObs.cmake)
find_package(LibObs REQUIRED)
find_package(Qt5Core REQUIRED)
find_package(Qt5Widgets REQUIRED)

set(NDISDK_DIR "NDISDK_DIR-NOTFOUND" CACHE PATH "NewTek NDI SDK")
if(NDISDK_DIR EQUAL "NDISDK_DIR-NOTFOUND")
	message(FATAL_ERROR "Could not find NewTek's NDI SDK !")
endif()

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(ARCH 64)
else()
	set(ARCH 32)
endif()

set(obs-ndi_SOURCES
	obs-ndi.cpp
	obs-ndi-source.cpp
	obs-ndi-output.cpp
	obs-ndi-filter.cpp
	Config.cpp
	forms/output-settings.cpp)

set(obs-ndi_HEADERS
	obs-ndi.h
	Config.h
	forms/output-settings.h)

add_library(obs-ndi MODULE
	${obs-ndi_SOURCES}
	${obs-ndi_HEADERS})

include_directories(
	"${LIBOBS_INCLUDE_DIR}/../UI/obs-frontend-api"
	${Qt5Core_INCLUDES}
	${Qt5Widgets_INCLUDES})

target_link_libraries(obs-ndi
	libobs
	Qt5::Core
	Qt5::Widgets)

# Windows
if(WIN32)
	set(OBS_FRONTEND_LIB "OBS_FRONTEND_LIB-NOTFOUND" CACHE FILEPATH "OBS frontend library")
	if(OBS_FRONTEND_LIB EQUAL "OBS_FRONTEND_LIB-NOTFOUND")
		message(FATAL_ERROR "Could not find OBS Frontend API\'s library !")
	endif()

	add_definitions(/O2)
	string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
	message("${CMAKE_CXX_FLAGS_RELEASE}")

	if(ARCH EQUAL 64)
		set(OBS_ARCH_NAME "64bit")
	else()
		set(OBS_ARCH_NAME "32bit")
		add_definitions(/arch:SSE2)
	endif()

	find_package(w32-pthreads REQUIRED)

	include_directories(
		"${NDISDK_DIR}/Include")

	target_link_libraries(obs-ndi
		"${OBS_FRONTEND_LIB}"
		w32-pthreads)

	set(RELEASE_DIR "${PROJECT_SOURCE_DIR}/release")
	add_custom_command(TARGET obs-ndi POST_BUILD
		COMMAND if $<CONFIG:Release>==1 (
			"${CMAKE_COMMAND}" -E make_directory
			"${RELEASE_DIR}/data/obs-plugins/obs-ndi"
			"${RELEASE_DIR}/obs-plugins/${OBS_ARCH_NAME}")

		COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy_directory
			"${PROJECT_SOURCE_DIR}/data"
			"${RELEASE_DIR}/data/obs-plugins/obs-ndi")

		COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy
			"$<TARGET_FILE:obs-ndi>"
			"${RELEASE_DIR}/obs-plugins/${OBS_ARCH_NAME}")
	)
endif()

# OSX (WARNING : needs testing)
if(APPLE)
	include_directories(
		"${NDISDK_DIR}/include")
	target_link_libraries(obs-ndi
		"${OBS_FRONTEND_LIB}")
endif()

# Linux
if(UNIX AND NOT APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -mtune=core2 -Ofast")

	if(ARCH EQUAL 64)
		set(ARCH_NAME "x86_64")
	else()
		set(ARCH_NAME "i686")
	endif()

	set_target_properties(obs-ndi PROPERTIES PREFIX "")

	include_directories(
		"${NDISDK_DIR}/include")
	target_link_libraries(obs-ndi
		obs-frontend-api)

	install(TARGETS obs-ndi
		LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib/obs-plugins)
	install(FILES data/locale/en-US.ini data/locale/fr-FR.ini
		DESTINATION "${CMAKE_INSTALL_PREFIX}/share/obs/obs-plugins/obs-ndi/locale")
endif()
