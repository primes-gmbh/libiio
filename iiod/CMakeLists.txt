cmake_minimum_required(VERSION 3.13)
project(iiod C)

include(${CMAKE_SOURCE_DIR}/cmake/Utilities.cmake)
CHECK_PTHREAD_SET_NAME(HAS_PTHREAD_SETNAME_NP)

add_executable(iiod
	iiod.c interpreter.c responder.c rw.c thread-pool.c
)
set_target_properties(iiod PROPERTIES
	C_STANDARD 99
	C_STANDARD_REQUIRED ON
	C_EXTENSIONS OFF
)
target_link_libraries(iiod LINK_PRIVATE iio iio_common_config iiod-responder
	${PTHREAD_LIBRARIES} ${AVAHI_COMMON_LIBRARIES} ${AVAHI_CLIENT_LIBRARIES}
)
target_include_directories(iiod PRIVATE ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_definitions(iiod PRIVATE _GNU_SOURCE)

option(WITH_IIOD_V0_COMPAT "Build support for Libiio v0.x protocol" ON)
if (WITH_IIOD_V0_COMPAT)
	include(FindBISON)
	include(FindFLEX)

	flex_target(lexer
		${CMAKE_CURRENT_SOURCE_DIR}/lexer.l ${CMAKE_CURRENT_BINARY_DIR}/lexer.c)
	bison_target(parser
		${CMAKE_CURRENT_SOURCE_DIR}/parser.y ${CMAKE_CURRENT_BINARY_DIR}/parser.c)
	add_flex_bison_dependency(lexer parser)

	target_sources(iiod PRIVATE ops.c ${BISON_parser_OUTPUTS} ${FLEX_lexer_OUTPUTS})
endif (WITH_IIOD_V0_COMPAT)
if (CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
	# flex sometimes generates code which generate sign comparison errors
	set_source_files_properties(${FLEX_lexer_OUTPUTS} PROPERTIES COMPILE_FLAGS "-Wno-sign-compare")
	set_source_files_properties(${BISON_parser_OUTPUTS} PROPERTIES COMPILE_FLAGS "-Wno-sign-compare")
endif ()

option(WITH_AIO "Build IIOD with async. I/O support" ON)
if (WITH_AIO)
	find_library(LIBAIO_LIBRARIES aio)
	find_path(LIBAIO_INCLUDE_DIR libaio.h)

	if (NOT LIBAIO_LIBRARIES OR NOT LIBAIO_INCLUDE_DIR)
		message(SEND_ERROR "Unable to find libaio dependency.\n"
			"If you want to disable async. I/O support, set WITH_AIO=OFF.")
	endif ()

	option(WITH_IIOD_USBD "Add support for USB through FunctionFS within IIOD" ON)
	if (WITH_IIOD_USBD)
		include(CheckTypeSize)
		set(CMAKE_EXTRA_INCLUDE_FILES linux/usb/functionfs.h)
		check_type_size("struct usb_functionfs_descs_head_v2" FUNCTIONFS_V2)
		set(CMAKE_EXTRA_INCLUDE_FILES)

		if (NOT HAVE_FUNCTIONFS_V2)
			message(SEND_ERROR "Linux kernel headers are too old.\n"
				"If you want to disable USB support, set WITH_IIOD_USBD=OFF.")
		endif()

		target_sources(iiod PRIVATE usbd.c)


		option(WITH_IIOD_USB_DMABUF "Enable DMABUF support on the USB stack" OFF)
		if (WITH_IIOD_USB_DMABUF)
			target_sources(iiod PRIVATE usb-dmabuf.c)
		endif()
	endif()

	target_include_directories(iiod PRIVATE ${LIBAIO_INCLUDE_DIR})
	target_link_libraries(iiod LINK_PRIVATE ${LIBAIO_LIBRARIES})
elseif (WITH_IIOD_USBD)
	message(SEND_ERROR "Sorry, USB support in iiod (WITH_IIOD_USBD)"
		" requires async I/O support (WITH_AIO)")
endif()

option(WITH_IIOD_SERIAL "Add serial (UART) support" ON)
if (WITH_IIOD_SERIAL)
	target_sources(iiod PRIVATE serial.c)
endif()

option(WITH_IIOD_NETWORK "Add network (TCP/IP) support" ON)
if (WITH_IIOD_NETWORK)
	target_sources(iiod PRIVATE network.c)

	if (HAVE_AVAHI)
		target_sources(iiod PRIVATE dns-sd.c)
	endif()
endif()

if (WITH_ZSTD)
	target_include_directories(iiod PRIVATE ${LIBZSTD_INCLUDE_DIR})
	target_link_libraries(iiod LINK_PRIVATE ${LIBZSTD_LIBRARIES})
endif()

include(GNUInstallDirs)
if(NOT SKIP_INSTALL_ALL)
	install(TARGETS iiod RUNTIME DESTINATION ${CMAKE_INSTALL_SBINDIR})
endif()

option(WITH_SYSTEMD "Enable installation of systemd service file for iiod" OFF)
if (WITH_SYSTEMD)
	if(${WITH_HWMON})
		set(CMAKE_SERVICE_PATH "ConditionPathExists=|/sys/bus/iio\nConditionPathExists=|/sys/class/hwmon")
	else()
		set(CMAKE_SERVICE_PATH "ConditionPathExists=/sys/bus/iio")
	endif()
	set(SYSTEMD_UNIT_INSTALL_DIR /lib/systemd/system CACHE PATH
		"default install path for systemd unit files"
	)
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/init/iiod.service.cmakein
		${PROJECT_BINARY_DIR}/init/iiod.service
	)
	install(FILES ${PROJECT_BINARY_DIR}/init/iiod.service
		DESTINATION ${SYSTEMD_UNIT_INSTALL_DIR}
	)
endif()

option(WITH_SYSVINIT "Enable installation of SysVinit script for iiod" OFF)
if (WITH_SYSVINIT)
	set(SYSVINIT_INSTALL_DIR /etc/init.d CACHE PATH
		"default install path for SysVinit scripts"
	)
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/init/iiod.init.cmakein
		${PROJECT_BINARY_DIR}/init/iiod
	)
	install(FILES ${PROJECT_BINARY_DIR}/init/iiod
	        PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ
	        DESTINATION ${SYSVINIT_INSTALL_DIR}
	)
endif()

option(WITH_UPSTART "Enable installation of upstart config file for iiod" OFF)
if (WITH_UPSTART)
	set(UPSTART_CONF_INSTALL_DIR /etc/init CACHE PATH
		"default install path for upstart conf files"
	)
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/init/iiod.conf.cmakein
		${PROJECT_BINARY_DIR}/init/iiod.conf
	)
	install(FILES ${PROJECT_BINARY_DIR}/init/iiod.conf
		DESTINATION ${UPSTART_CONF_INSTALL_DIR}
	)
endif()

toggle_iio_feature("${WITH_IIOD_SERIAL}" iiod-serial)
toggle_iio_feature("${WITH_IIOD_NETWORK}" iiod-network)
toggle_iio_feature("${WITH_IIOD_V0_COMPAT}" iiod-v0-compat)
toggle_iio_feature("${WITH_AIO}" iiod-aio)
toggle_iio_feature("${WITH_IIOD_USBD}" iiod-usb)
toggle_iio_feature("${WITH_SYSTEMD}" iiod-systemd)
toggle_iio_feature("${WITH_SYSVINIT}" iiod-sysvinit)
toggle_iio_feature("${WITH_UPSTART}" iiod-upstart)
toggle_iio_feature("${WITH_IIOD_USB_DMABUF}" iiod-usb-dmabuf)

set(IIOD_FEATURES_ON "${IIO_FEATURES_ON}" PARENT_SCOPE)
set(IIOD_FEATURES_OFF "${IIO_FEATURES_OFF}" PARENT_SCOPE)
