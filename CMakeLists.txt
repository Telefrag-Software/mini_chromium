cmake_minimum_required(VERSION 3.14)

list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_CURRENT_SOURCE_DIR}/CMake)

project(mini_chromium VERSION 0.0.1 LANGUAGES CXX)

if(WIN32)
	set(PLATFORM "Windows")
elseif(APPLE)
	set(PLATFORM "MacOS")
else()
	set(PLATFORM "Linux")
endif()

string(TOLOWER "${PLATFORM}" PLATFORM_LOWER)
string(TOUPPER "${PLATFORM}" PLATFORM_UPPER)

include(SourceFiles)

set(MINI_CHROMIUM_HEADER_FILES ${MINI_CHROMIUM_SOURCE_FILES})
list(FILTER MINI_CHROMIUM_HEADER_FILES INCLUDE REGEX ".*\\.h$")

add_library(${PROJECT_NAME}
	STATIC
		${MINI_CHROMIUM_SOURCE_FILES}
)

target_include_directories(${PROJECT_NAME}
	PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
		$<INSTALL_INTERFACE:>
)

if(WIN32)
	target_compile_definitions(${PROJECT_NAME}
		PRIVATE
			NOMINMAX
			UNICODE
			WIN32
			WIN32_LEAN_AND_MEAN
			_CRT_SECURE_NO_WARNINGS
			_HAS_EXCEPTIONS=0
			_UNICODE
	)

	target_compile_options(${PROJECT_NAME}
		PRIVATE
			/FS
			/W4
			/Zi
			/bigobj  # Support larger number of sections in obj file.
			/wd4100  # Unreferenced formal parameter.
			/wd4127  # Conditional expression is constant.
			/wd4324  # Structure was padded due to alignment specifier.
			/wd4351  # New behavior: elements of array will be default initialized.
			/wd4577  # 'noexcept' used with no exception handling mode specified.
			/wd4996  # 'X' was declared deprecated.
	)

	target_link_libraries(${PROJECT_NAME}
		PUBLIC
			advapi32
	)
elseif(UNIX)
	set_target_properties(${PROJECT_NAME}
		PROPERTIES
			CXX_STANDARD 17
			C_STANDARD 11
	)

	target_compile_options(${PROJECT_NAME}
		PRIVATE
			-Wno-multichar
			-fPIC
	)

	target_link_libraries(${PROJECT_NAME}
		PUBLIC
			pthread
			dl
	)

	if(NOT APPLE)
		target_compile_definitions(${PROJECT_NAME}
			PRIVATE
				_FILE_OFFSET_BITS=64
		)
	else()
		target_link_libraries(${PROJECT_NAME}
			PUBLIC
				"-framework ApplicationServices"
				"-framework CoreFoundation"
				"-framework Foundation"
				"-framework IOKit"
				"-framework Security"
		)
	endif()
endif()

include(GNUInstallDirs)

set(CONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(VERSION_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}ConfigVersion.cmake")
set(PROJECT_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(NAMESPACE "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
	"${VERSION_CONFIG}" COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
	"cmake/Config.cmake.in"
	"${PROJECT_CONFIG}"
	INSTALL_DESTINATION "${CONFIG_INSTALL_DIR}"
)

install(
	TARGETS ${PROJECT_NAME}
	EXPORT "${TARGETS_EXPORT_NAME}"
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
	INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

foreach(_HEADER_FILE ${MINI_CHROMIUM_HEADER_FILES})
	string(REGEX MATCH ".*[/\\]" _HEADER_DIRECTORY_PATH ${_HEADER_FILE})
	install(FILES ${_HEADER_FILE} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${_HEADER_DIRECTORY_PATH})
endforeach()

install(
	FILES "${PROJECT_CONFIG}" "${VERSION_CONFIG}"
	DESTINATION "${CONFIG_INSTALL_DIR}"
)

install(
	EXPORT "${TARGETS_EXPORT_NAME}"
	NAMESPACE "${NAMESPACE}"
	DESTINATION "${CONFIG_INSTALL_DIR}"
)
