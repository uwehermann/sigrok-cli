##
## This file is part of the sigrok-cli project.
##
## Copyright (C) 2020 Uwe Hermann <uwe@hermann-uwe.de>
##
## This program is free software: you can redistribute it and/or modify
## it under the terms of the GNU General Public License as published by
## the Free Software Foundation, either version 2 of the License, or
## (at your option) any later version.
##
## This program is distributed in the hope that it will be useful,
## but WITHOUT ANY WARRANTY; without even the implied warranty of
## MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
## GNU General Public License for more details.
##
## You should have received a copy of the GNU General Public License
## along with this program.  If not, see <http://www.gnu.org/licenses/>.
##

cmake_minimum_required(VERSION 3.12)

project(sigrok-cli VERSION 1.0.0 LANGUAGES C)

include(GNUInstallDirs)

find_package(Git QUIET)
if(Git_FOUND)
	execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} ERROR_QUIET
		OUTPUT_VARIABLE GITHASH OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND
		${GIT_EXECUTABLE} name-rev --tags --name-only ${GITHASH}
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} ERROR_QUIET
		OUTPUT_VARIABLE GITTAG OUTPUT_STRIP_TRAILING_WHITESPACE)
	if((DEFINED GITTAG) AND ("${GITTAG}" STREQUAL "undefined"))
		string(APPEND PROJECT_VERSION "-git-${GITHASH}")
	endif()
endif()

option(BUILD_SHARED_LIBS "Shared (non-static) build" TRUE)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY VALUE RelWithDebInfo)
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS TRUE)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBGLIB REQUIRED IMPORTED_TARGET "glib-2.0>=2.32.0")

find_package(sigrok 6.0.0 REQUIRED)
find_package(sigrokdecode 6.0.0)
set(LIBSIGROKDECODE_FOUND ${sigrokdecode_FOUND})
message(STATUS "libsigrok: ${sigrok_VERSION}")
message(STATUS "libsigrokdecode: ${sigrokdecode_VERSION}")

add_executable(${PROJECT_NAME} "")

target_sources(${PROJECT_NAME} PRIVATE anykey.c decode.c device.c input.c
	main.c options.c output.c parsers.c session.c show.c sigrok-cli.h)

target_compile_features(${PROJECT_NAME} PRIVATE c_std_99)

target_compile_options(${PROJECT_NAME} PRIVATE
	$<IF:$<C_COMPILER_ID:MSVC>,/W4,-Wall -Wextra>)

target_compile_definitions(${PROJECT_NAME} PRIVATE
	$<IF:$<BOOL:${MINGW}>,__STDC_FORMAT_MACROS -D__STDC_LIMIT_MACROS -D_POSIX -D__printf__=__gnu_printf__,>)

function(pkgconfig_lib p)
	if(${p}_FOUND AND BUILD_SHARED_LIBS)
		target_link_libraries(${PROJECT_NAME} PRIVATE PkgConfig::${p})
	elseif(${p}_FOUND)
		target_include_directories(${PROJECT_NAME} PRIVATE ${${p}_STATIC_INCLUDE_DIRS})
		target_link_libraries(${PROJECT_NAME} PRIVATE ${${p}_STATIC_LDFLAGS})
	endif()
endfunction()

pkgconfig_lib(LIBGLIB)

target_link_libraries(${PROJECT_NAME} PRIVATE sigrok::sigrok
	$<IF:$<BOOL:${sigrokdecode_FOUND}>,sigrok::sigrokdecode,>)

target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(FILES doc/${PROJECT_NAME}.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
install(FILES contrib/org.sigrok.${PROJECT_NAME}.desktop DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)
install(FILES contrib/${PROJECT_NAME}.svg DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps)

message(STATUS "CMake: ${CMAKE_COMMAND} ${CMAKE_VERSION}")
message(STATUS "Building ${PROJECT_NAME} ${PROJECT_VERSION} for ${CMAKE_SYSTEM} ${CMAKE_SYSTEM_PROCESSOR}")

set(CPACK_SOURCE_PACKAGE_FILE_NAME "${PROJECT_NAME}-${PROJECT_VERSION}")
set(CPACK_SOURCE_IGNORE_FILES ${CMAKE_CURRENT_BINARY_DIR} ".git*")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)

configure_file(config.h.in config.h @ONLY)
configure_file(contrib/${PROJECT_NAME}_cross.nsi.in contrib/${PROJECT_NAME}_cross.nsi @ONLY)
