# CMakeLists.txt
#
# $Id$
#
# Wireshark - Network traffic analyzer
# By Gerald Combs <gerald@wireshark.org>
# Copyright 1998 Gerald Combs
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
#

project(Wireshark C CXX)

cmake_minimum_required(VERSION 2.6)
set(CMAKE_BACKWARDS_COMPATIBILITY 2.6)

# Needs to be set after cmake_minimum_required or cmake_policy(VERSION)
if(COMMAND cmake_policy)
	# Backward compatibility for versions < 2.6.3
	cmake_policy(SET CMP0011 OLD)
#	cmake_policy(SET CMP0015 NEW)
endif(COMMAND cmake_policy)

# set(PROJECT_MAJOR_VERSION 1)
# set(PROJECT_MINOR_VERSION 9)
# set(PROJECT_PATCH_VERSION 0)
# set(PROJECT_VERSION_EXTENSION "-rc5")
# If not set, copy over Wireshark version from configure.ac
if(NOT PROJECT_MAJOR_VERSION)
	file(STRINGS
		${CMAKE_SOURCE_DIR}/configure.ac
		PROJECT_MAJOR_VERSION_TMP
		REGEX "^m4_define\\(\\[?version_major\\]?, *\\[?[0-9]+\\]?\\)"
	)
	file(STRINGS
		${CMAKE_SOURCE_DIR}/configure.ac
		PROJECT_MINOR_VERSION_TMP
		REGEX "^m4_define\\(\\[?version_minor\\]?, *\\[?[0-9]+\\]?\\)"
	)
	file(STRINGS
		${CMAKE_SOURCE_DIR}/configure.ac
		PROJECT_PATCH_VERSION_TMP
		REGEX "^m4_define\\(\\[?version_micro\\]?, *\\[?[0-9]+\\]?\\)"
	)
	# XXX pull VERSION_EXTENSION out of configure.ac ?

	string(REGEX REPLACE "m4_define\\(\\[?version_major\\]?, *\\[?([0-9]+)\\]?\\)"
		"\\1"
		PROJECT_MAJOR_VERSION
		${PROJECT_MAJOR_VERSION_TMP}
	)
	string(REGEX REPLACE "m4_define\\(\\[?version_minor\\]?, *\\[?([0-9]+)\\]?\\)"
		"\\1"
		PROJECT_MINOR_VERSION
		${PROJECT_MINOR_VERSION_TMP}
	)
	string(REGEX REPLACE "m4_define\\(\\[?version_micro\\]?, *\\[?([0-9]+)\\]?\\)"
		"\\1"
		PROJECT_PATCH_VERSION
		${PROJECT_PATCH_VERSION_TMP}
	)
endif()


if(PROJECT_VERSION_EXTENSION)
	set(PROJECT_VERSION ${PROJECT_MAJOR_VERSION}.${PROJECT_MINOR_VERSION}.${PROJECT_PATCH_VERSION}${PROJECT_VERSION_EXTENSION})
else()
	set(PROJECT_VERSION ${PROJECT_MAJOR_VERSION}.${PROJECT_MINOR_VERSION}.${PROJECT_PATCH_VERSION})
endif()

message(STATUS "V: ${PROJECT_VERSION}, MaV: ${PROJECT_MAJOR_VERSION}, MiV: ${PROJECT_MINOR_VERSION}, PL: ${PROJECT_PATCH_VERSION}, EV: ${PROJECT_VERSION_EXTENSION}.")

#Where to find local cmake scripts
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/modules)
INCLUDE(UseLemon)
INCLUDE(UseMakeDissectorReg)
INCLUDE(UseMakeTapReg)
INCLUDE(UseAsn2Wrs)

# Under linux the release mode (CMAKE_BUILD_TYPE=release) defines NDEBUG

# Enable with: "make VERBOSE=1"
set(CMAKE_VERBOSE_MAKEFILE OFF)

#Defines CMAKE_INSTALL_BINDIR, CMAKE_INSTALL_DATADIR, etc ...
include(CMakeInstallDirs)

include_directories(
	${CMAKE_BINARY_DIR}
	${CMAKE_SOURCE_DIR}
	${CMAKE_SOURCE_DIR}/epan
	${CMAKE_SOURCE_DIR}/tools/lemon
	${CMAKE_SOURCE_DIR}/wiretap
)

#Where to put libraries
if(NOT LIBRARY_OUTPUT_PATH)
	set(LIBRARY_OUTPUT_PATH ${Wireshark_BINARY_DIR}/lib CACHE INTERNAL
		   "Single output directory for building all libraries.")
endif()


include(CMakeOptions.txt)


set(WIRESHARK_C_FLAGS
	-O2
	-g
	-Wall
	-W
	-Wextra
	-Wendif-labels
	-Wpointer-arith
	-Warray-bounds
	-Wformat-security
	-Wshorten-64-to-32
	-Wvla
	-Waddress
	-Warray-bounds
	-Wattributes
	-Wdiv-by-zero
	-Wignored-qualifiers
	-Wno-overlength-strings
	-Wwrite-strings
	-Wno-long-long
	-Wpragmas
)

set(WIRESHARK_C_ONLY_FLAGS
	# The following are C only, not C++
	-Wc++-compat
	-Wdeclaration-after-statement
	-Wshadow
	-Wno-pointer-sign
	-Wold-style-definition
	-Wstrict-prototypes
	-Wlogical-op
	-Wjump-misses-init
)

set(WIRESHARK_EXTRA_C_FLAGS
	-pedantic
	-Woverflow
	-fstrict-overflow -Wstrict-overflow=4
	-Wunreachable-code
	-Wunsafe-loop-optimizations
	-Wcast-align
	-Wcast-qual
	-Wformat-security
	-Wredundant-decls
	# All the registration functions block these for now.
	-Wmissing-declarations
)

set(WIRESHARK_EXTRA_C_ONLY_FLAGS
	# The following are C only, not C++
	-Wbad-function-cast
	# All the registration functions block these for now.
	-Wmissing-prototypes
)

# With clang some tests don't fail properly during testing but only
# during real compiles

if(NOT CMAKE_C_COMPILER_ID MATCHES "Clang")

	set(WIRESHARK_C_FLAGS ${WIRESHARK_C_FLAGS}
	    -fexcess-precision=fast
	)

	set(WIRESHARK_C_ONLY_FLAGS ${WIRESHARK_C_ONLY_FLAGS}
	)
endif()

if(ENABLE_EXTRA_GCC_CHECKS)   # This overrides -Werror
	set(WIRESHARK_C_FLAGS ${WIRESHARK_C_FLAGS} ${WIRESHARK_EXTRA_C_FLAGS})
	set(WIRESHARK_C_ONLY_FLAGS ${WIRESHARK_C_ONLY_FLAGS} ${WIRESHARK_EXTRA_C_ONLY_FLAGS})
endif()

add_definitions(
	-DG_DISABLE_DEPRECATED
	-DG_DISABLE_SINGLE_INCLUDES
)

# Counterhack to work around some cache magic in CHECK_C_SOURCE_COMPILES
include(CheckCCompilerFlag)

if(NOT DISABLE_WERROR AND NOT ENABLE_EXTRA_GCC_CHECKS)
	check_c_compiler_flag(-Werror=unknown-warning-option WERR_UNKNOWN)
	check_c_compiler_flag(-Werror WERROR)
else()
	unset(WERR_UNKNOWN)
	unset(WERROR)
endif()
# string of additional compile command line flags for check_c_compiler_flag
if(WERR_UNKNOWN)
	set(CMAKE_REQUIRED_FLAGS -Werror=unknown-warning-option )
endif()

set(C 0)
# Sigh: Have to use THIS_FLAG instead of ${F} for some reason
foreach(THIS_FLAG ${WIRESHARK_C_FLAGS})
	set(F WS_C_FLAG_${C})
	set(${F} ${THIS_FLAG})
	set(V WS_C_FLAG_VALID${C})
	message(STATUS "Checking for flag: ${${F}}")
	check_c_compiler_flag(${${F}} ${V})
	if (${${V}})
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${${F}}")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${${F}}")
	endif()
	math(EXPR C "${C} + 1")
endforeach()
# Continue increasing C from previous loop
foreach(THIS_FLAG ${WIRESHARK_C_ONLY_FLAGS})
	set(F WS_C_FLAG_${C})
	set(${F} ${THIS_FLAG})
	set(V WS_C_FLAG_VALID${C})
	message(STATUS "Checking for flag: ${${F}}")
	check_c_compiler_flag(${${F}} ${V})
	if (${${V}})
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${${F}}")
	endif()
	math(EXPR C "${C} + 1")
endforeach()
message(STATUS "C-Flags: ${CMAKE_C_FLAGS}\nCXX-Flags: ${CMAKE_CXX_FLAGS}")

check_c_compiler_flag(-fvisibility=hidden FVHIDDEN)
if((FVHIDDEN))
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
else() # TODO add alternate compiler flags for hiding symbols
	message(WARNING "Hiding shared library symbols is not supported by the compiler."
		" All shared library symbols will be exported.")
endif()

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "Clang")
	set (C_UNUSED "__attribute__((unused))" )
else()
	set (C_UNUSED "" )
endif()

set(WIRESHARK_LD_FLAGS
	-Wl,--as-needed
	# -flto
	# -fwhopr
	# -fwhole-program
)

include(CheckCLinkerFlag)
set(C 0)
# Sigh: Have to use THIS_FLAG instead of ${F} for some reason
foreach(THIS_FLAG ${WIRESHARK_LD_FLAGS})
	set(F WS_LD_FLAG_${C})
	set(${F} ${THIS_FLAG})
	set(V WS_LD_FLAG_VALID${C})
	check_c_linker_flag(${${F}} ${V})
	if (${${V}})
		set(WS_LINK_FLAGS ${WS_LINK_FLAGS} ${${F}})
	endif()
	math(EXPR C "${C} + 1")
endforeach()

if(ENABLE_STATIC)
  set(LINK_MODE_LIB STATIC)
  set(LINK_MODE_MODULE STATIC)
else()
  set(LINK_MODE_LIB SHARED)
  set(LINK_MODE_MODULE MODULE)
endif()

# The packagelist is doing some magic:  If we add XXX to the packagelist, we
# - may optionally set XXX_OPTIONS to pass to the find_package command
# - will call FindXXX.cmake
# - return found libraries in XXX_LIBRARIES
# - return found include in XXX_INCLUDE_DIRS
# - set HAVE_XXX

#The minimum package list
set(PACKAGELIST GLIB2 GMODULE2 GTHREAD2 M LEX YACC Perl SH PythonInterp)
set(GLIB2_FIND_REQUIRED)
set(GLIB2_MIN_VERSION 2.14.0)
set(GTHREAD2_REQUIRED)
set(PCAP_REQUIRED)
set(M_REQUIRED)
set(PythonInterp_FIND_VERSION 2)
set(Python_ADDITIONAL_VERSIONS 3)

set(PACKAGELIST ${PACKAGELIST} HtmlViewer)

if(ENABLE_PCAP)
	set(PACKAGELIST ${PACKAGELIST} PCAP)
endif()

# Build the GTK-GUI?
if(BUILD_wireshark)
	if(ENABLE_GTK3)
		set(PACKAGELIST ${PACKAGELIST} GTK3)
	else()
		set(PACKAGELIST ${PACKAGELIST} GTK2)
		set(GTK2_OPTIONS COMPONENTS gtk)
		set(GTK2_FIND_VERSION 2.12)
		set(GTK2_DEBUG false)
	endif()
endif()

# Build the Qt GUI?
if(BUILD_qtshark)
	if(ENABLE_QT5)
		set(PACKAGELIST ${PACKAGELIST} Qt5Widgets Qt5PrintSupport Qt5LinguistTools)
		set(QT_VERSION 5)
	else()
		set(PACKAGELIST ${PACKAGELIST} Qt4)
		# set(Qt4_OPTIONS 4.7.1 REQUIRED QtCore QtGui)
		set(QT_VERSION 4)
	endif()
endif()

# SMI SNMP
if(ENABLE_SMI)
	set(PACKAGELIST ${PACKAGELIST} SMI)
endif()

# GNU crypto
if(ENABLE_GCRYPT)
	set(PACKAGELIST ${PACKAGELIST} GCRYPT)
endif()

# GNU SSL/TLS support
if(ENABLE_GNUTLS)
	set(PACKAGELIST ${PACKAGELIST} GNUTLS)
endif()

# Kerberos
if(ENABLE_KERBEROS)
	set(PACKAGELIST ${PACKAGELIST} KERBEROS)
endif()

# Portable audio
if(ENABLE_PORTAUDIO)
	set(PACKAGELIST ${PACKAGELIST} PORTAUDIO)
endif()


# Prefer c-ares over adns
if(ENABLE_CARES)        # C Asynchronouse resolver
	set(PACKAGELIST ${PACKAGELIST} CARES)
elseif(ENABLE_ADNS)     # Gnu asynchronous DNS
	set(PACKAGELIST ${PACKAGELIST} ADNS)
endif()

# Zlib compression
if(ENABLE_ZLIB)
	set(PACKAGELIST ${PACKAGELIST} ZLIB)
endif()

# Lua 5.1 dissectors
if(ENABLE_LUA)
	set(PACKAGELIST ${PACKAGELIST} LUA)
endif()

# GeoIP address resolving
if(ENABLE_GEOIP)
	set(PACKAGELIST ${PACKAGELIST} GEOIP)
endif()

if(ENABLE_NETLINK)
	set(PACKAGELIST ${PACKAGELIST} NL)
endif()

# Capabilities
if(ENABLE_CAP)
	set(PACKAGELIST ${PACKAGELIST} CAP)
endif()

if(ENABLE_PYTHON)
	set(PACKAGELIST ${PACKAGELIST} PythonLibs)
endif()

set(PACKAGELIST ${PACKAGELIST} YAPP)

set(PACKAGELIST ${PACKAGELIST} POD)


set(PROGLIST text2pcap mergecap capinfos editcap reordercap dumpcap)

#Sort the package list
list(SORT PACKAGELIST)
message(STATUS "Packagelist: ${PACKAGELIST}")
#Let's loop the package list
foreach(PACKAGE ${PACKAGELIST})
	if(${PACKAGE} STREQUAL "Qt4")
		set(PACKAGE_VAR "QT")
	elseif(${PACKAGE} STREQUAL "PythonInterp")
		set(PACKAGE_VAR "PYTHON")
	elseif(${PACKAGE} STREQUAL "PythonLibs")
		set(PACKAGE_VAR "PYTHON")
	else()
		set(PACKAGE_VAR ${PACKAGE})
	endif()
	if(${PACKAGE}_OPTIONS)
		find_package(${PACKAGE} ${${PACKAGE}_OPTIONS})
	elseif(${PACKAGE}_REQUIRED)
		find_package(${PACKAGE} REQUIRED)
	else()
		find_package(${PACKAGE})
	endif()
	message(${PACKAGE_VAR}_FOUND)
	if (${PACKAGE_VAR}_FOUND)
		set(HAVE_LIB${PACKAGE_VAR} 1)
		include_directories(${${PACKAGE_VAR}_INCLUDE_DIRS})
		message(STATUS "${PACKAGE} includes: ${${PACKAGE_VAR}_INCLUDE_DIRS}")
		message(STATUS "${PACKAGE} libs: ${${PACKAGE_VAR}_LIBRARIES}")
		if (${PACKAGE}_DEFINITIONS)
			message(STATUS "${PACKAGE} definitions: ${${PACKAGE_VAR}_DEFINITIONS}")
		endif()
		if (${PACKAGE_VAR}_EXECUTABLE)
			message(STATUS "${PACKAGE} executable: ${${PACKAGE_VAR}_EXECUTABLE}")
		endif()
	endif()
endforeach()

#packaging
include(CPackConfig.txt)

if(HAVE_LIBPYTHON)
	set(HAVE_PYTHON 1)
	set(PYTHON_DIR "${CMAKE_INSTALL_PREFIX}/lib/wireshark/python/${CPACK_PACKAGE_VERSION}")
endif()
if(HAVE_LIBLUA)
	set(HAVE_LUA_H 1)
	set(HAVE_LUA 1)
endif()
if(HAVE_LIBKERBEROS)
	set(HAVE_KERBEROS 1)
	# HAVE_HEIMDAL_KERBEROS
	set(HAVE_MIT_KERBEROS 1)
	set(HAVE_KEYTYPE_ARCFOUR_56 1)
endif()
if(HAVE_LIBGEOIP)
	set(HAVE_GEOIP 1)
endif()
if(HAVE_LIBCARES)
	set(HAVE_C_ARES 1)
endif()
if(HAVE_LIBADNS)
	set(HAVE_GNU_ADNS 1)
endif()
if(ENABLE_AIRPCAP)
	set(HAVE_AIRPCAP 1)
endif()
if(HAVE_LIBNL AND ENABLE_AIRPCAP)
	message(ERROR "Airpcap and Libnl support are mutually exclusive")
endif()
# No matter which version of GTK is present
if(GTK2_FOUND OR GTK3_FOUND)
	set(GTK_FOUND ON)
endif()
# That's the name autofoo uses
if(HAVE_LIBZLIB)
	set(HAVE_LIBZ 1)
endif()
if (Qt5Widgets_FOUND)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${Qt5Widgets_EXECUTABLE_COMPILE_FLAGS}")
	if (Qt5_POSITION_INDEPENDENT_CODE)
		set(CMAKE_POSITION_INDEPENDENT_CODE ON)
	endif()
	set (QT_FOUND ON)
	set (QT_LIBRARIES ${Qt5Widgets_LIBRARIES} ${Qt5PrintSupport_LIBRARIES})
# If Qt4: QT_LIBRARIES and QT_INCLUDES are not set above. They require extra magic
elseif(QT_FOUND)
	include(${QT_USE_FILE})
	include_directories(${QT_INCLUDE_DIR})
	message(STATUS "QT includes: ${QT_INCLUDE_DIR}")
	message(STATUS "QT libs: ${QT_LIBRARIES}")
endif()
include(ConfigureChecks.cmake)

#Big or little endian ?
include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)

set(DATAFILE_DIR "${CMAKE_INSTALL_PREFIX}/share/${CPACK_PACKAGE_NAME}")

if(ENABLE_PLUGINS)
	set(HAVE_PLUGINS 1)
	set(PLUGIN_DIR="${DATAFILE_DIR}/plugins/${CPACK_PACKAGE_VERSION}")
	set(PLUGIN_SRC_DIRS
		plugins/asn1
		plugins/docsis
		plugins/ethercat
		plugins/gryphon
		plugins/irda
		plugins/m2m
		plugins/mate
		plugins/opcua
		plugins/profinet
		plugins/stats_tree
		plugins/unistim
		plugins/wimax
		plugins/wimaxasncp
		plugins/wimaxmacphy
	)
# It seems this stuff doesn't build with autofoo either...
#	if(YAPP_FOUND)
#		set(PLUGIN_SRC_DIRS
#			${PLUGIN_SRC_DIRS}
#			plugins/tpg
#		)
#	endif()
else()
	set(PLUGIN_SRC_DIRS )
endif()

foreach(PLUGIN_DIR ${PLUGIN_SRC_DIRS})
	add_subdirectory( ${PLUGIN_DIR} )
endforeach()

add_subdirectory( asn1 EXCLUDE_FROM_ALL )
add_subdirectory( codecs )
add_subdirectory( epan )
add_subdirectory( tools/lemon )
add_subdirectory( ui )
add_subdirectory( wiretap )
add_subdirectory( wsutil )

add_custom_target(dumpabi DEPENDS dumpabi-libwireshark dumpabi-libwiretap dumpabi-libwsutil color.h)

if(ENABLE_ECHLD)
	add_subdirectory( echld )
endif()

if(BUILD_wireshark AND GTK_FOUND)
	add_subdirectory( ui/gtk )
endif()

if(BUILD_qtshark AND QT_FOUND)
	add_subdirectory( ui/qt )
endif()


# Basedir where to install guides
set(DOC_DIR "$ENV{docdir}" CACHE FILEPATH "Installation directory for ug and dg pdfs.")
message(STATUS "docdir: ${DOC_DIR}")

if(ENABLE_GUIDES)
	add_subdirectory( docbook )
endif()

if(ENABLE_PCAP_NG_DEFAULT)
	set(PCAP_NG_DEFAULT 1)
endif()

# Large file support (e.g. make off_t 64 bit if supported)
include(gmxTestLargeFiles)
gmx_test_large_files(GMX_LARGEFILES)

#Platform specific
#if(UNIX)
#endif()

if(APPLE)
	#
	# We assume that APPLE means OS X so that we have the OS X
	# frameworks.
	#
	set(HAVE_OS_X_FRAMEWORKS 1)
	FIND_LIBRARY (APPLE_APPLICATION_SERVICES_LIBRARY ApplicationServices)
	FIND_LIBRARY (APPLE_CORE_FOUNDATION_LIBRARY CoreFoundation)
	FIND_LIBRARY (APPLE_SYSTEM_CONFIGURATION_LIBRARY SystemConfiguration)
endif()

if(WIN32)
	set(WS_MSVC_NORETURN "__declspec(noreturn)")

	# Disable deprecation
	if(MSVC80 OR MSVC90)
		add_definitions(-D_CRT_SECURE_NO_DEPRECATE -D_CRT_NONSTDC_NO_DEPRECATE)
	endif()
else()
	set(WS_MSVC_NORETURN " ")
endif()

configure_file(${CMAKE_SOURCE_DIR}/cmakeconfig.h.in ${CMAKE_BINARY_DIR}/config.h)

set( configure_input "Built with CMake ${CMAKE_VERSION}" )
set( VERSION ${PROJECT_VERSION} )

configure_file(
	${CMAKE_SOURCE_DIR}/packaging/macosx/Info.plist.in
	${CMAKE_SOURCE_DIR}/packaging/macosx/Info.plist
	@ONLY)

include(FeatureSummary)
#SET_FEATURE_INFO(NAME DESCRIPTION [URL [COMMENT] ]
#FEATURE_SUMMARY(WHAT ALL) Requires removal of our local copy of FeatureSummary
PRINT_ENABLED_FEATURES()
PRINT_DISABLED_FEATURES()

link_directories(
	${CMAKE_BINARY_DIR}/ui
	${CMAKE_BINARY_DIR}/ui/gtk
	${CMAKE_BINARY_DIR}/ui/qt
	${CMAKE_BINARY_DIR}/codecs
	${CMAKE_BINARY_DIR}/epan
	${CMAKE_BINARY_DIR}/wiretap
	${CMAKE_BINARY_DIR}/wsutil
)

ADD_CUSTOM_TARGET(
	svnversion ALL
	COMMAND ${PERL}
		${CMAKE_CURRENT_SOURCE_DIR}/make-version.pl
		${CMAKE_CURRENT_SOURCE_DIR}
	DEPENDS
		${CMAKE_CURRENT_SOURCE_DIR}/make-version.pl
)

ADD_CUSTOM_COMMAND(
	OUTPUT	svnversion.h
	COMMAND ${PERL}
		${CMAKE_CURRENT_SOURCE_DIR}/make-version.pl
		${CMAKE_CURRENT_SOURCE_DIR}
	DEPENDS
		${CMAKE_CURRENT_SOURCE_DIR}/make-version.pl
)

if(UNIX)
	set(PLATFORM_SRC
		capture-pcap-util-unix.c
	)
endif()

if(WIN32)
	set(PLATFORM_SRC
		capture-wpcap.c capture_wpcap_packet.c capture_win_ifnames.c
	)
endif()

set(WTAP_PLUGIN_SOURCES
	epan/plugins.c
	epan/filesystem.c
)

set(SHARK_COMMON_CAPTURE_SRC
	capture_ifinfo.c
	capture_ui_utils.c
)

set(WIRESHARK_COMMON_SRC
	${PLATFORM_SRC}
	capture-pcap-util.c
	cfile.c
	cfutils.c
	clopts_common.c
	frame_tvbuff.c
	svnversion.h
	sync_pipe_write.c
	version_info.c
)

set(TSHARK_TAP_SRC
	ui/cli/tap-afpstat.c
	ui/cli/tap-ansi_astat.c
	ui/cli/tap-bootpstat.c
	ui/cli/tap-camelcounter.c
	ui/cli/tap-camelsrt.c
	ui/cli/tap-comparestat.c
	ui/cli/tap-dcerpcstat.c
	ui/cli/tap-diameter-avp.c
	ui/cli/tap-expert.c
	ui/cli/tap-follow.c
	ui/cli/tap-funnel.c
	ui/cli/tap-gsm_astat.c
	ui/cli/tap-h225counter.c
	ui/cli/tap-h225rassrt.c
	ui/cli/tap-hosts.c
	ui/cli/tap-httpstat.c
	ui/cli/tap-icmpstat.c
	ui/cli/tap-icmpv6stat.c
	ui/cli/tap-iostat.c
	ui/cli/tap-iousers.c
	ui/cli/tap-macltestat.c
	ui/cli/tap-mgcpstat.c
	ui/cli/tap-megacostat.c
	ui/cli/tap-protocolinfo.c
	ui/cli/tap-protohierstat.c
	ui/cli/tap-radiusstat.c
	ui/cli/tap-rlcltestat.c
	ui/cli/tap-rpcstat.c
	ui/cli/tap-rpcprogs.c
	ui/cli/tap-rtp.c
	ui/cli/tap-rtspstat.c
	ui/cli/tap-scsistat.c
	ui/cli/tap-sctpchunkstat.c
	ui/cli/tap-sipstat.c
	ui/cli/tap-smbsids.c
	ui/cli/tap-smbstat.c
	ui/cli/tap-stats_tree.c
	ui/cli/tap-sv.c
	ui/cli/tap-wspstat.c
)

set(INSTALL_DIRS
	diameter
	dtds
	help
	radius
	tpncp
	wimaxasncp
)

set(INSTALL_FILES
	${CMAKE_BINARY_DIR}/AUTHORS-SHORT
	COPYING
	${CMAKE_BINARY_DIR}/capinfos.html
	cfilters
	colorfilters
	dfilters
	${CMAKE_BINARY_DIR}/dftest.html
	${CMAKE_BINARY_DIR}/dumpcap.html
	${CMAKE_BINARY_DIR}/editcap.html
	${CMAKE_BINARY_DIR}/asn2deb.html
	${CMAKE_BINARY_DIR}/idl2deb.html
	${CMAKE_BINARY_DIR}/idl2wrs.html
	ipmap.html
	manuf
	${CMAKE_BINARY_DIR}/mergecap.html
	pdml2html.xsl
	${CMAKE_BINARY_DIR}/randpkt.html
	${CMAKE_BINARY_DIR}/rawshark.html
	${CMAKE_BINARY_DIR}/reordercap.html
	services
	smi_modules
	${CMAKE_BINARY_DIR}/text2pcap.html
	${CMAKE_BINARY_DIR}/tshark.html
	${CMAKE_BINARY_DIR}/wireshark-filter.html
	${CMAKE_BINARY_DIR}/wireshark.html
	docbook/ws.css
)

set(LIBEPAN_LIBS
#		@NSL_LIBS@	# -lnsl
#		@SOCKET_LIBS@	# -lsocket
#		@SSL_LIBS@	# -lcrypto
		epan
#		$(plugin_ldadd)	# in case of static
		${PCAP_LIBRARIES}
		${CARES_LIBRARIES}
		${ADNS_LIBRARIES}
		${KERBEROS_LIBRARIES}
		${LUA_LIBRARIES}
		${PYTHON_LIBRARIES}
		${GEOIP_LIBRARIES}
		${GCRYPT_LIBRARIES}
		${GNUTLS_LIBRARIES}
		${SMI_LIBRARIES}
		${ZLIB_LIBRARIES}
		${M_LIBRARIES}
)

if( (BUILD_wireshark AND GTK_FOUND) OR (BUILD_qtshark AND QT_FOUND) )
	set(wireshark_FILES
		airpcap_loader.c
		capture.c
		capture_info.c
		capture_opts.c
		capture_sync.c
		color_filters.c
		file.c
		fileset.c
		filters.c
		iface_monitor.c
		proto_hier_stats.c
		summary.c
		ws80211_utils.c
		${SHARK_COMMON_CAPTURE_SRC}
		${WIRESHARK_COMMON_SRC}
	)
endif()

if(BUILD_wireshark AND GTK_FOUND)
	set(wireshark_LIBS
		gtkui
		ui
		${GTK2_LIBRARIES}
		${GTK3_LIBRARIES}
		${GTHREAD2_LIBRARIES}
		codecs
		${PORTAUDIO_LIBRARIES}
		${LIBEPAN_LIBS}
		${APPLE_APPLICATION_SERVICES_LIBRARY}
		${APPLE_CORE_SERVICES_LIBRARY}
		${APPLE_SYSTEM_CONFIGURATION_LIBRARY}
		${NL_LIBRARIES}
	)
	# qtshark and wireshark share wireshark_FILES

	add_executable(wireshark ${wireshark_FILES})
	add_dependencies(wireshark svnversion)
	set_target_properties(wireshark PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(wireshark ${wireshark_LIBS})
	install(TARGETS wireshark RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_qtshark AND QT_FOUND)
	set(qtshark_LIBS
		qtui
		ui
		${QT_LIBRARIES}
		${GTHREAD2_LIBRARIES}
		codecs
		${PORTAUDIO_LIBRARIES}
		${LIBEPAN_LIBS}
		${APPLE_APPLICATION_SERVICES_LIBRARY}
		${APPLE_CORE_FOUNDATION_LIBRARY}
		${APPLE_SYSTEM_CONFIGURATION_LIBRARY}
		${NL_LIBRARIES}
	)
	# qtshark and wireshark share wireshark_FILES

	add_executable(qtshark ${wireshark_FILES})
	add_dependencies(qtshark svnversion)
	set_target_properties(qtshark PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(qtshark ${qtshark_LIBS})
	install(TARGETS qtshark RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

register_tap_files(tshark-tap-register.c
	${TSHARK_TAP_SRC}
)

if(BUILD_tshark)
	set(tshark_LIBS
		ui
		${LIBEPAN_LIBS}
		${APPLE_CORE_FOUNDATION_LIBRARY}
		${APPLE_SYSTEM_CONFIGURATION_LIBRARY}
	)
	set(tshark_FILES
		capture_opts.c
		capture_sync.c
		tshark-tap-register.c
		tshark.c
		${TSHARK_TAP_SRC}
		${SHARK_COMMON_CAPTURE_SRC}
	        ${WIRESHARK_COMMON_SRC}
	)
	add_executable(tshark ${tshark_FILES})
	add_dependencies(tshark svnversion)
	set_target_properties(tshark PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(tshark ${tshark_LIBS})
	install(TARGETS tshark RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_rawshark AND PCAP_FOUND)
	set(rawshark_LIBS
		${LIBEPAN_LIBS}
		${APPLE_CORE_FOUNDATION_LIBRARY}
		${APPLE_SYSTEM_CONFIGURATION_LIBRARY}
	)
	set(rawshark_FILES
	        ${WIRESHARK_COMMON_SRC}
		rawshark.c
		ui/util.c
	)
	add_executable(rawshark ${rawshark_FILES})
	add_dependencies(rawshark svnversion)
	set_target_properties(rawshark PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(rawshark ${rawshark_LIBS})
	install(TARGETS rawshark RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_dftest)
	set(dftest_LIBS
		${LIBEPAN_LIBS}
	)
	set(dftest_FILES
		dftest.c
		ui/util.c
	)
	add_executable(dftest ${dftest_FILES})
	set_target_properties(dftest PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(dftest ${dftest_LIBS})
	install(TARGETS dftest RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_randpkt)
	set(randpkt_LIBS
		wiretap
		${M_LIBRARIES}
		${PCAP_LIBRARIES}
#		@SOCKET_LIBS@
#		@NSL_LIBS@
		${CARES_LIBRARIES}
		${ADNS_LIBRARIES}
		${ZLIB_LIBRARIES}
	)
	set(randpkt_FILES
		randpkt.c
	)
	add_executable(randpkt ${randpkt_FILES})
	set_target_properties(randpkt PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(randpkt ${randpkt_LIBS})
	install(TARGETS randpkt RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_text2pcap)
	set(text2pcap_LIBS
		wsutil
		${M_LIBRARIES}
		${ZLIB_LIBRARIES}
	)
	set(text2pcap_CLEAN_FILES
		text2pcap.c
		pcapio.c
	)
	set(text2pcap_FILES
		${text2pcap_CLEAN_FILES}
	)
	add_lex_files(text2pcap_FILES
		text2pcap-scanner.l
	)
	add_executable(text2pcap ${text2pcap_FILES})
	add_dependencies(text2pcap svnversion)
	set_target_properties(text2pcap PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(text2pcap ${text2pcap_LIBS})
	install(TARGETS text2pcap RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_mergecap)
	set(mergecap_LIBS
		wiretap
		${ZLIB_LIBRARIES}
	)
	set(mergecap_FILES
		mergecap.c
		svnversion.h
		${WTAP_PLUGIN_SOURCES}
	)
	add_executable(mergecap ${mergecap_FILES})
	add_dependencies(mergecap svnversion)
	set_target_properties(mergecap PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(mergecap ${mergecap_LIBS})
	install(TARGETS mergecap RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_reordercap)
	set(reordercap_LIBS
		wiretap
		${ZLIB_LIBRARIES}
	)
	set(reordercap_FILES
		reordercap.c
		svnversion.h
		${WTAP_PLUGIN_SOURCES}
	)
	add_executable(reordercap ${reordercap_FILES})
	add_dependencies(reordercap svnversion)
	set_target_properties(reordercap PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(reordercap ${reordercap_LIBS})
	install(TARGETS reordercap RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_capinfos)
	set(capinfos_LIBS
		wiretap
		wsutil
		${ZLIB_LIBRARIES}
		${GCRYPT_LIBRARIES}
	)
	set(capinfos_FILES
		capinfos.c
		${WTAP_PLUGIN_SOURCES}
	)
	add_executable(capinfos ${capinfos_FILES})
	add_dependencies(capinfos svnversion)
	set_target_properties(capinfos PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(capinfos ${capinfos_LIBS})
	install(TARGETS capinfos RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_editcap)
	set(editcap_LIBS
		wiretap
		${ZLIB_LIBRARIES}
	)
	set(editcap_FILES
		editcap.c
		${WTAP_PLUGIN_SOURCES}
	)
	add_executable(editcap ${editcap_FILES})
	add_dependencies(editcap svnversion)
	set_target_properties(editcap PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(editcap ${editcap_LIBS})
	install(TARGETS editcap RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(BUILD_dumpcap AND PCAP_FOUND)
	set(dumpcap_LIBS
#		@INET_NTOP_LO@
		wsutil
		${PCAP_LIBRARIES}
		${CAP_LIBRARIES}
#		@SOCKET_LIBS@
#		@NSL_LIBS@
		${GLIB2_LIBRARIES}
		${GTHREAD2_LIBRARIES}
		${ZLIB_LIBRARIES}
		${APPLE_CORE_FOUNDATION_LIBRARY}
		${APPLE_SYSTEM_CONFIGURATION_LIBRARY}
		${NL_LIBRARIES}
	)
	set(dumpcap_FILES
		svnversion.h
		capture_opts.c
		capture-pcap-util.c
		capture_stop_conditions.c
		cfutils.c
		clopts_common.c
		conditions.c
		dumpcap.c
		pcapio.c
		ringbuffer.c
		sync_pipe_write.c
		version_info.c
		ws80211_utils.c
		${PLATFORM_SRC}
	)
	add_executable(dumpcap ${dumpcap_FILES})
	add_dependencies(dumpcap svnversion)
	set_target_properties(dumpcap PROPERTIES LINK_FLAGS "${WS_LINK_FLAGS}")
	target_link_libraries(dumpcap ${dumpcap_LIBS})
	install(TARGETS dumpcap RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

ADD_CUSTOM_COMMAND(
	OUTPUT	${CMAKE_BINARY_DIR}/AUTHORS-SHORT
	COMMAND ${PERL}
		${CMAKE_SOURCE_DIR}/doc/perlnoutf.pl
		${CMAKE_SOURCE_DIR}/doc/make-authors-short.pl
		< ${CMAKE_SOURCE_DIR}/AUTHORS
		> ${CMAKE_BINARY_DIR}/AUTHORS-SHORT
	DEPENDS
		${CMAKE_SOURCE_DIR}/doc/perlnoutf.pl
		${CMAKE_SOURCE_DIR}/doc/make-authors-short.pl
		${CMAKE_SOURCE_DIR}/AUTHORS
)

ADD_CUSTOM_COMMAND(
	OUTPUT	${CMAKE_BINARY_DIR}/AUTHORS-SHORT-FORMAT
		${CMAKE_BINARY_DIR}/wireshark.pod
	COMMAND ${PERL}
		${CMAKE_SOURCE_DIR}/doc/perlnoutf.pl
		${CMAKE_SOURCE_DIR}/doc/make-authors-format.pl
		< ${CMAKE_BINARY_DIR}/AUTHORS-SHORT
		> ${CMAKE_BINARY_DIR}/AUTHORS-SHORT-FORMAT
	COMMAND cat
		${CMAKE_SOURCE_DIR}/doc/wireshark.pod.template
		${CMAKE_BINARY_DIR}/AUTHORS-SHORT-FORMAT
		> ${CMAKE_BINARY_DIR}/wireshark.pod
	DEPENDS
		${CMAKE_SOURCE_DIR}/doc/perlnoutf.pl
		${CMAKE_SOURCE_DIR}/doc/make-authors-format.pl
		${CMAKE_BINARY_DIR}/AUTHORS-SHORT
		${CMAKE_SOURCE_DIR}/doc/wireshark.pod.template
)

pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/capinfos 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/dftest 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/dumpcap 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/editcap 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/asn2deb 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/idl2deb 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/idl2wrs 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/mergecap 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/randpkt 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/rawshark 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/reordercap 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/text2pcap 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/tshark 1 )
pod2manhtml( ${CMAKE_BINARY_DIR}/wireshark 1 )
pod2manhtml( ${CMAKE_SOURCE_DIR}/doc/wireshark-filter 4 )

add_custom_target(
	auxiliary ALL
	DEPENDS
		AUTHORS-SHORT
		capinfos.html
		dftest.html
		dumpcap.html
		editcap.html
		asn2deb.html
		idl2deb.html
		idl2wrs.html
		mergecap.html
		randpkt.html
		rawshark.html
		reordercap.html
		text2pcap.html
		tshark.html
		wireshark.html
		wireshark-filter.html
)

set(MAN1_FILES
	${CMAKE_BINARY_DIR}/capinfos.1
	${CMAKE_BINARY_DIR}/dftest.1
	${CMAKE_BINARY_DIR}/dumpcap.1
	${CMAKE_BINARY_DIR}/editcap.1
	${CMAKE_BINARY_DIR}/idl2wrs.1
	${CMAKE_BINARY_DIR}/mergecap.1
	${CMAKE_BINARY_DIR}/randpkt.1
	${CMAKE_BINARY_DIR}/rawshark.1
	${CMAKE_BINARY_DIR}/reordercap.1
	${CMAKE_BINARY_DIR}/text2pcap.1
	${CMAKE_BINARY_DIR}/tshark.1
	${CMAKE_BINARY_DIR}/wireshark.1
)

set(MAN4_FILES
	${CMAKE_BINARY_DIR}/wireshark-filter.4
)

set(CLEAN_FILES
	${wireshark_FILES}
	${tshark_FILES}
	${rawshark_FILES}
	${dftest_FILES}
	${randpkt_FILES}
	${text2pcap_CLEAN_FILES}
	${mergecap_FILES}
	${capinfos_FILES}
	${editcap_FILES}
	${dumpcap_FILES}
)

if (WERROR)
	set_source_files_properties(
		${CLEAN_FILES}
		PROPERTIES
		COMPILE_FLAGS -Werror
	)
endif()

install(
	FILES
		${INSTALL_FILES}
	DESTINATION
		${CMAKE_INSTALL_DATADIR}/${CPACK_PACKAGE_NAME}
)

install(
	FILES
		${MAN1_FILES}
	DESTINATION
		${CMAKE_INSTALL_MANDIR}/man1
)

install(
	FILES
		${MAN4_FILES}
	DESTINATION
		${CMAKE_INSTALL_MANDIR}/man4
)

install(
	DIRECTORY
		${INSTALL_DIRS}
	DIRECTORY_PERMISSIONS
		OWNER_EXECUTE OWNER_WRITE OWNER_READ
		GROUP_EXECUTE GROUP_READ
		WORLD_EXECUTE WORLD_READ
	DESTINATION
		${CMAKE_INSTALL_DATADIR}/${CPACK_PACKAGE_NAME}
	PATTERN ".svn" EXCLUDE
	PATTERN "Makefile.*" EXCLUDE
	PATTERN "faq.py" EXCLUDE
)


if(DOC_DIR)
	message(STATUS "Docdir install: ${DOC_DIR}")
	INSTALL(
		DIRECTORY
			${CMAKE_BINARY_DIR}/docbook/
		DIRECTORY_PERMISSIONS
			OWNER_EXECUTE OWNER_WRITE OWNER_READ
			GROUP_EXECUTE GROUP_READ
			WORLD_EXECUTE WORLD_READ
		DESTINATION
			${DOC_DIR}/guides
		FILES_MATCHING
		PATTERN "*.pdf"
	)
endif()
