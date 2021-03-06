project(HardInfo)
cmake_minimum_required(VERSION 2.6)
cmake_policy(VERSION 2.6)

set(HARDINFO_VERSION "0.6-alpha")

SET( CMAKE_MODULE_PATH  "${CMAKE_CURRENT_SOURCE_DIR}/cmake" )
include(GNUInstallDirs)

if(${CMAKE_BUILD_TYPE} MATCHES [Dd]ebug)
  set(HARDINFO_DEBUG 1)
endif()

if(${CMAKE_HOST_SYSTEM_NAME} MATCHES "Linux")
	set(HARDINFO_OS "linux")
else()
	message(FATAL_ERROR "Unsupported operating system: ${CMAKE_HOST_SYSTEM_NAME}")
endif()

if(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "i[3-6]86")
	set(HARDINFO_ARCH "x86")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "x86_64")
	set(HARDINFO_ARCH "x86_64")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "ppc{32,64}")
	set(HARDINFO_ARCH "ppc")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "mips")
	set(HARDINFO_ARCH "mips")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "parisc.*")
	set(HARDINFO_ARCH "parisc")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "sparc{32,64}")
	set(HARDINFO_ARCH "sparc")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "arm.*")
	set(HARDINFO_ARCH "arm")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "ia64")
	set(HARDINFO_ARCH "ia64")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "alpha")
	set(HARDINFO_ARCH "alpha")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "s390.*")
	set(HARDINFO_ARCH "s390")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "m68k")
	set(HARDINFO_ARCH "m68k")
elseif(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "sh{3,4}")
	set(HARDINFO_ARCH "sh")
else()
	message(FATAL_ERROR "Unsupported architecture: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()

message(STATUS "Building HardInfo for architecture: ${HARDINFO_OS}-${HARDINFO_ARCH}")

add_definitions("-std=gnu89")

add_subdirectory(po)

include(FindPkgConfig)
pkg_check_modules(GTK REQUIRED gtk+-3.0>=2.10 glib-2.0>=2.10 gthread-2.0>=2.10 gmodule-export-2.0>=2.10)
pkg_check_modules(LIBSOUP libsoup-2.4>=2.24)

include_directories(
	${CMAKE_SOURCE_DIR}
	${CMAKE_SOURCE_DIR}/includes
	${CMAKE_SOURCE_DIR}/includes/${HARDINFO_ARCH}
	${CMAKE_BINARY_DIR}
	${GTK_INCLUDE_DIRS}
	${LIBSOUP_INCLUDE_DIRS}
)
link_directories(
	${GTK_LIBRARY_DIRS}
	${LIBSOUP_LIBRARY_DIRS}
)

set(HARDINFO_MODULES
	computer
	devices
	benchmark
	network
)
set(HARDINFO_RESOURCES
	"benchmark.conf"
	"benchmark.data"
)

set(MODULE_computer_SOURCES
	modules/computer.c
	modules/computer/alsa.c
	modules/computer/boots.c
	modules/computer/display.c
	modules/computer/environment.c
	modules/computer/filesystem.c
	modules/computer/languages.c
	modules/computer/loadavg.c
	modules/computer/memory.c
	modules/computer/modules.c
	modules/computer/os.c
	modules/computer/uptime.c
	modules/computer/users.c
	modules/computer/groups.c
)
set(MODULE_devices_SOURCES
	modules/devices.c
	modules/devices/${HARDINFO_ARCH}/processor.c
	modules/devices/battery.c
	modules/devices/devmemory.c
	modules/devices/dmi.c
	modules/devices/inputdevices.c
	modules/devices/pci.c
	modules/devices/printers.c
	modules/devices/resources.c
	modules/devices/sensors.c
	modules/devices/spd-decode.c
	modules/devices/storage.c
	modules/devices/usb.c
)
set(MODULE_network_SOURCES
	modules/network.c
	modules/network/net.c
	modules/network/nfs.c
	modules/network/samba.c
)
set(MODULE_benchmark_SOURCES
	modules/benchmark.c
	modules/benchmark/blowfish.c
	modules/benchmark/cryptohash.c
	modules/benchmark/drawing.c
	modules/benchmark/fbench.c
	modules/benchmark/fftbench.c
	modules/benchmark/fft.c
	modules/benchmark/fib.c
	modules/benchmark/guibench.c
	modules/benchmark/md5.c
	modules/benchmark/nqueens.c
	modules/benchmark/raytrace.c
	modules/benchmark/sha1.c
)

set_source_files_properties(
	modules/benchmark/blowfish.c
	modules/benchmark/fftbench.c
	modules/benchmark/md5.c
	modules/benchmark/sha1.c
	PROPERTIES
	COMPILE_FLAGS "-O0"
)

foreach (_module ${HARDINFO_MODULES})
	add_library(${_module} MODULE ${MODULE_${_module}_SOURCES})
	set_target_properties(${_module} PROPERTIES PREFIX "")
endforeach()

add_library(hardinfo-shell STATIC
        shell/callbacks.c
        shell/iconcache.c
        shell/loadgraph.c
        shell/menu.c
        shell/report.c
        shell/shell.c
        shell/stock.c
	shell/syncmanager.c
)
add_executable(Hardinfo
	hardinfo/binreloc.c
	hardinfo/expr.c
        hardinfo/hardinfo.c
	hardinfo/socket.c
	hardinfo/util.c
	hardinfo/vendor.c
)
target_link_libraries(Hardinfo
	${GTK_LIBRARIES}
	${LIBSOUP_LIBRARIES}
	hardinfo-shell
	m
)

configure_file(config.h.cmake ${CMAKE_BINARY_DIR}/config.h @ONLY)
configure_file(hardinfo.desktop.cmake ${CMAKE_BINARY_DIR}/hardinfo.desktop @ONLY)

install(TARGETS Hardinfo ${HARDINFO_MODULES}
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/hardinfo/modules
	PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
)
install(FILES ${CMAKE_BINARY_DIR}/hardinfo.desktop
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(FILES ${HARDINFO_RESOURCES}
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/hardinfo
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(DIRECTORY pixmaps
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/hardinfo
	PATTERN "*.{png,svg}"
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(DIRECTORY doc
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/hardinfo
	PATTERN "*.{hlp,png}"
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
