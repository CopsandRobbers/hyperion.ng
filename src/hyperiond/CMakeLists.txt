if (APPLE)
	project(Hyperion)
else()
	project(hyperiond)
endif()

if (NOT CMAKE_VERSION VERSION_LESS "3.12")
	find_package(Python3 COMPONENTS Interpreter Development REQUIRED)
	include_directories(${Python3_INCLUDE_DIRS} ${Python3_INCLUDE_DIRS}/..)
else()
	find_package (PythonLibs ${PYTHON_VERSION_STRING} EXACT) # Maps PythonLibs to the PythonInterp version of the main cmake
	include_directories(${PYTHON_INCLUDE_DIRS} ${PYTHON_INCLUDE_DIRS}/..)
endif()

find_package(Qt5Widgets REQUIRED)

# generate windows .rc file for this binary
if (WIN32)
	include(${CMAKE_SOURCE_DIR}/cmake/win/win_rc.cmake)
	generate_win_rc_file(${PROJECT_NAME})
endif(WIN32)

# include resource files for macos bundle (copy LICENSE file and correct line breaks)
if (APPLE)
	configure_file(${CMAKE_SOURCE_DIR}/LICENSE ${CMAKE_BINARY_DIR}/LICENSE COPYONLY)
	execute_process(COMMAND bash -c "perl -pi -e 's/\n/\r/g' ${CMAKE_BINARY_DIR}/LICENSE")
	set(BUNDLE_RESOURCE_FILES ${CMAKE_SOURCE_DIR}/cmake/osxbundle/Hyperion.icns ${CMAKE_BINARY_DIR}/LICENSE)
	set_source_files_properties(${BUNDLE_RESOURCE_FILES} PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
endif(APPLE)

add_executable(${PROJECT_NAME}
	console.h
	hyperiond.h
	systray.h
	hyperiond.cpp
	systray.cpp
	main.cpp
	${hyperiond_WIN_RC_PATH}
	${BUNDLE_RESOURCE_FILES}
)

# promote hyperiond as GUI app
if (WIN32)
	target_link_options(${PROJECT_NAME} PUBLIC /SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup)
endif(WIN32)

target_link_libraries(${PROJECT_NAME}
	commandline
	hyperion
	effectengine
	jsonserver
	flatbufserver
	protoserver
	webserver
	ssdp
	database
	python
	resources
	Qt5::Widgets
)

if (NOT CMAKE_VERSION VERSION_LESS "3.12")
	target_link_libraries( ${PROJECT_NAME} ${Python3_LIBRARIES} )
else()
	target_link_libraries( ${PROJECT_NAME} ${PYTHON_LIBRARIES} )
endif()

if (ENABLE_AVAHI)
	target_link_libraries(${PROJECT_NAME} bonjour)
endif (ENABLE_AVAHI)

if (ENABLE_AMLOGIC)
	target_link_libraries(${PROJECT_NAME}
		Qt5::Core
		pcre16 dl z
	)
endif(ENABLE_AMLOGIC)

if (ENABLE_DISPMANX)
	IF ( "${PLATFORM}" MATCHES rpi)
		find_package(BCM REQUIRED)
		include_directories(${BCM_INCLUDE_DIRS})
	ELSE()
		SET(BCM_INCLUDE_DIRS "")
		SET(BCM_LIBRARIES "")
	ENDIF()
	target_link_libraries(${PROJECT_NAME} dispmanx-grabber)
endif (ENABLE_DISPMANX)

if (ENABLE_FB)
	target_link_libraries(${PROJECT_NAME} framebuffer-grabber)
endif (ENABLE_FB)

if (ENABLE_OSX)
	target_link_libraries(${PROJECT_NAME} osx-grabber)
endif (ENABLE_OSX)

if (ENABLE_V4L2)
	target_link_libraries(${PROJECT_NAME} v4l2-grabber)
endif ()

if (ENABLE_MF)
	target_link_libraries(${PROJECT_NAME} mf-grabber)
endif (ENABLE_MF)

if (ENABLE_AMLOGIC)
	target_link_libraries(${PROJECT_NAME} amlogic-grabber)
endif (ENABLE_AMLOGIC)

if (ENABLE_X11)
	if(APPLE)
		include_directories("/opt/X11/include")
	endif(APPLE)
	target_link_libraries(${PROJECT_NAME} x11-grabber)
endif (ENABLE_X11)

if (ENABLE_XCB)
	target_link_libraries(${PROJECT_NAME} xcb-grabber)
endif (ENABLE_XCB)

if (ENABLE_QT)
	target_link_libraries(${PROJECT_NAME} qt-grabber)
endif (ENABLE_QT)

if (ENABLE_DX)
	include_directories(${DIRECTX9_INCLUDE_DIRS})
	target_link_libraries(${PROJECT_NAME} directx-grabber)
endif (ENABLE_DX)

if (ENABLE_CEC)
        target_link_libraries(${PROJECT_NAME} cechandler)
endif (ENABLE_CEC)

if (APPLE)
	set_target_properties( ${PROJECT_NAME} PROPERTIES
		MACOSX_BUNDLE TRUE
		MACOSX_BUNDLE_INFO_PLIST ${CMAKE_SOURCE_DIR}/cmake/osxbundle/Info.plist
		MACOSX_BUNDLE_BUNDLE_NAME "Hyperion"
		MACOSX_BUNDLE_BUNDLE_VERSION ${HYPERION_VERSION}
		MACOSX_BUNDLE_COPYRIGHT "Copyright (c) 2014-2021 Hyperion Project"
		MACOSX_BUNDLE_GUI_IDENTIFIER "com.hyperion-project.${PROJECT_NAME}"
		MACOSX_BUNDLE_ICON_FILE "Hyperion.icns"
		MACOSX_BUNDLE_INFO_STRING "${PROJECT_NAME} Version ${HYPERION_VERSION}"
		MACOSX_BUNDLE_LONG_VERSION_STRING ${HYPERION_VERSION}
		MACOSX_BUNDLE_SHORT_VERSION_STRING ${HYPERION_VERSION}
	)

	install ( TARGETS ${PROJECT_NAME} DESTINATION . COMPONENT "Hyperion")
elseif(NOT WIN32)
	install ( TARGETS ${PROJECT_NAME} DESTINATION "share/hyperion/bin" COMPONENT "Hyperion" )
	install ( DIRECTORY ${CMAKE_SOURCE_DIR}/bin/service DESTINATION "share/hyperion/" COMPONENT "Hyperion" )
	install ( FILES ${CMAKE_SOURCE_DIR}/effects/readme.txt DESTINATION "share/hyperion/effects" COMPONENT "Hyperion" )
	install ( FILES ${CMAKE_SOURCE_DIR}/resources/icons/hyperion-icon-32px.png DESTINATION "share/hyperion/icons" COMPONENT "Hyperion" )

	# Desktop file for Hyperion
	install ( FILES ${CMAKE_SOURCE_DIR}/cmake/desktop/hyperiond_128.png DESTINATION "share/hyperion/desktop" COMPONENT "Hyperion" )
	install ( FILES ${CMAKE_SOURCE_DIR}/cmake/desktop/hyperiond.desktop DESTINATION "share/hyperion/desktop" COMPONENT "Hyperion" )
else()
	install ( TARGETS ${PROJECT_NAME} DESTINATION "bin" COMPONENT "Hyperion" )
	install ( FILES ${CMAKE_SOURCE_DIR}/effects/readme.txt DESTINATION "effects" COMPONENT "Hyperion" )

	#set( CMAKE_INSTALL_UCRT_LIBRARIES TRUE )
	#set( CMAKE_INSTALL_SYSTEM_RUNTIME_LIBS_SKIP TRUE )
	#include( InstallRequiredSystemLibraries )
endif()

if (CMAKE_HOST_UNIX AND NOT APPLE)
	install( CODE "EXECUTE_PROCESS(COMMAND ln -sf \"../share/hyperion/bin/${PROJECT_NAME}\" \"${CMAKE_BINARY_DIR}/symlink_${PROJECT_NAME}\" )" COMPONENT "Hyperion" )
	install( FILES "${CMAKE_BINARY_DIR}/symlink_${PROJECT_NAME}" DESTINATION "bin" RENAME ${PROJECT_NAME} COMPONENT "Hyperion" )
	install( CODE "FILE (REMOVE ${CMAKE_BINARY_DIR}/symlink_${PROJECT_NAME})" COMPONENT "Hyperion" )
endif()

# Deploy Qt DLLs into the binary folder.
# This is necessary for starting the application from within the IDE
if (WIN32)
	get_target_property(QMAKE_EXECUTABLE Qt5::qmake IMPORTED_LOCATION)
	get_filename_component(QT_BIN_DIR "${QT_QMAKE_EXECUTABLE}" DIRECTORY)
	find_program(WINDEPLOYQT_EXECUTABLE windeployqt HINTS "${QT_BIN_DIR}")
	set(WINDEPLOYQT_PARAMS --verbose 0 --no-compiler-runtime --no-opengl-sw --no-system-d3d-compiler)
	add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD COMMAND ${WINDEPLOYQT_EXECUTABLE} ${WINDEPLOYQT_PARAMS} "$<TARGET_FILE:${PROJECT_NAME}>")
endif(WIN32)

if(ENABLE_DEPLOY_DEPENDENCIES)
	# Deploy all dependencies for package creation
	include(${CMAKE_SOURCE_DIR}/cmake/Dependencies.cmake)

	if (APPLE) #macOS
		DeployMacOS(${PROJECT_NAME})
	elseif (NOT WIN32) # Linux
		DeployLinux(${PROJECT_NAME})
	elseif(WIN32) # Windows
		DeployWindows(${PROJECT_NAME})
	endif ()
endif(ENABLE_DEPLOY_DEPENDENCIES)
