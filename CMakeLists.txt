project(piserver)
cmake_minimum_required(VERSION 3.1)

set (CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")

find_package(Gettext)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
find_package(CURL REQUIRED)
find_package(LibArchive REQUIRED)
find_package(PkgConfig)
pkg_check_modules(GTKMM REQUIRED gtkmm-3.0)
link_directories(${GTKMM_LIBRARY_DIRS})
include_directories(${GTKMM_INCLUDE_DIRS})

include(CheckIncludeFile)
check_include_file(ldap.h HAVE_LDAP_H)
find_library(LDAP_LIBRARY ldap)

if(NOT HAVE_LDAP_H OR NOT LDAP_LIBRARY)
    message( FATAL_ERROR "Missing libldap. Try 'sudo apt-get install libldap2-dev'" )
endif()

# for crypt_r()
find_library(CRYPT_LIBRARY crypt)

aux_source_directory(src SRC_LIST)
add_executable(${PROJECT_NAME} ${SRC_LIST} resources.c)
target_link_libraries(${PROJECT_NAME} ${GTKMM_LIBRARIES} ${LDAP_LIBRARY}
    ${CRYPT_LIBRARY} ${CURL_LIBRARY} ${LibArchive_LIBRARIES} Threads::Threads)

# Languages

if (GETTEXT_FOUND)
    file(GLOB LANGUAGES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} po/*.po)
    GETTEXT_CREATE_TRANSLATIONS(po/piserver.pot ALL ${LANGUAGES})
    add_definitions(-DLOCALE_DIR=/usr/share/locale)
endif()

# Resources

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/macros)
include(GlibCompileResourcesSupport)

set(RESOURCE_LIST
    data/firstuse.glade
    data/piserver.glade
    data/addhostdialog.glade
    data/edithostdialog.glade
    data/adduserdialog.glade
    data/edituserdialog.glade
    data/adddistrodialog.glade
    )

compile_gresources(RESOURCE_FILE
                   XML_OUT
                   TYPE EMBED_C
                   RESOURCES ${RESOURCE_LIST})

add_custom_target(resource ALL DEPENDS ${RESOURCE_FILE})

# Installation

install(TARGETS piserver DESTINATION bin)
install(FILES data/piserver.desktop DESTINATION share/applications)
install(DIRECTORY DESTINATION /var/lib/piserver)
install(DIRECTORY DESTINATION /var/lib/piserver/os)
install(DIRECTORY DESTINATION /var/lib/piserver/tftproot)
install(DIRECTORY DESTINATION /var/lib/piserver/scripts)
install(PROGRAMS scripts/postinstall.sh scripts/convert.sh scripts/chroot_image.sh DESTINATION /var/lib/piserver/scripts)
install(FILES scripts/convert_chrooted_cmds DESTINATION /var/lib/piserver/scripts)
install(FILES scripts/50-piserver-dhcpcd-hook DESTINATION /lib/dhcpcd/dhcpcd-hooks)
