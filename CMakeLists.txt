# RetroShare decentralized communication platform
#
# Copyright (C) 2021-2022  Gioacchino Mazzurco <gio@retroshare.cc>
# Copyright (C) 2026       David Bears <dbear4q@gmail.com>
#
# SPDX-License-Identifier: CC0-1.0

# 3.1 : configure_package_config_file(INSTALL_PREFIX)
# 3.7 : if(VERSION_GREATER_EQUAL)
# 3.11: add_library() ... target_sources()
cmake_minimum_required(VERSION 3.11...4.4)

project(bitdht)

include(GNUInstallDirs)
set(BD_INSTALL_DATADIR "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}")
set(BD_INSTALL_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}")

add_library(${PROJECT_NAME})
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

set(BD_BOOT_FILE "${PROJECT_SOURCE_DIR}/src/bitdht/bdboot.txt")

if(WIN32)
	target_link_libraries(${PROJECT_NAME} "-lws2_32" "-lwsock32")
	target_compile_definitions(${PROJECT_NAME} PRIVATE UNICODE _UNICODE)

	# Some files trip strict GCC >= 15 diagnostics that are now errors by
	# default; downgrade them to warnings for now.
	if(
		CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR
		CMAKE_CXX_COMPILER_ID STREQUAL "Clang"
	)
		target_compile_options(${PROJECT_NAME} PRIVATE
			-Wno-error=incompatible-pointer-types
			-Wno-error=int-conversion
			-Wno-error=implicit-function-declaration
		)
	endif()
endif()


add_subdirectory(src)


### installation

if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.23)
	set(FILESET_INSTALL
		FILE_SET HEADERS DESTINATION "${BD_INSTALL_INCLUDEDIR}"
	)
else()
	set(FILESET_INSTALL INCLUDES DESTINATION ${BD_INSTALL_INCLUDEDIR})
	get_target_property(${PROJECT_NAME}_SOURCES ${PROJECT_NAME} SOURCES)
	foreach(source ${${PROJECT_NAME}_SOURCES})
		string(TOLOWER ${source} sourceLower)
		if(NOT sourceLower MATCHES "\\.(h|hh|h\\+\\+|hpp)\$")
			continue()
		endif()
		foreach(base_ ${PROJECT_SOURCE_DIR}/src ${PROJECT_BINARY_DIR}/src)
			string(FIND ${source} ${base_}/ basePos)
			if(basePos EQUAL 0)
				set(base ${base_})
				break()
			endif()
		endforeach()
		if(NOT basePos EQUAL 0)
			message(SEND_ERROR "couldn't find base for header ${source}")
			continue()
		endif()

		get_filename_component(dest ${source} DIRECTORY)
		file(RELATIVE_PATH dest ${base} ${dest})
		install(FILES ${source}
			DESTINATION ${BD_INSTALL_INCLUDEDIR}/${dest}
			COMPONENT ${PROJECT_NAME}
		)
	endforeach()
endif()

install(TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}_export
	COMPONENT ${PROJECT_NAME}
  ${FILESET_INSTALL}
)


install(
	FILES "${BD_BOOT_FILE}"
	DESTINATION "${BD_INSTALL_DATADIR}"
	COMPONENT ${PROJECT_NAME}
)

# config package for install tree
set(CONFIG_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
install(EXPORT ${PROJECT_NAME}_export
  DESTINATION "${CONFIG_DIR}"
  FILE ${PROJECT_NAME}-targets.cmake
  NAMESPACE ${PROJECT_NAME}::
	COMPONENT ${PROJECT_NAME}
)
include(CMakePackageConfigHelpers)
set(BD_INSTALL_BOOT_FILE "${BD_INSTALL_DATADIR}/bdboot.txt")
configure_package_config_file(
	mk/cmake/${PROJECT_NAME}-config.cmake.in
  cmake/${PROJECT_NAME}-config_installed.cmake
  INSTALL_DESTINATION "${CONFIG_DIR}"
	PATH_VARS BD_INSTALL_BOOT_FILE
)
install(FILES
	"${CMAKE_CURRENT_BINARY_DIR}/cmake/${PROJECT_NAME}-config_installed.cmake"
  DESTINATION "${CONFIG_DIR}"
  RENAME ${PROJECT_NAME}-config.cmake
	COMPONENT ${PROJECT_NAME}
)

# config package for build tree
export(EXPORT ${PROJECT_NAME}_export
  FILE cmake/${PROJECT_NAME}-targets.cmake
  NAMESPACE ${PROJECT_NAME}::
)
include(CMakePackageConfigHelpers)
set(PACKAGE_BD_INSTALL_BOOT_FILE "${BD_BOOT_FILE}")
configure_package_config_file(
	mk/cmake/${PROJECT_NAME}-config.cmake.in
  cmake/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION cmake
	INSTALL_PREFIX ${PROJECT_BINARY_DIR}
)

# config package for CMAKE_FIND_PACKAGE_REDIRECTS_DIR
if(CMAKE_VERSION VERSION_LESS 3.24)
	set(CMAKE_FIND_PACKAGE_REDIRECTS_DIR ${CMAKE_BINARY_DIR}/pkgRedirects)
endif()
configure_file(
	mk/cmake/${PROJECT_NAME}-config_redirects.cmake.in
  "${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/${PROJECT_NAME}-config.cmake"
	@ONLY
)
