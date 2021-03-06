project(telepathy-ofono)
cmake_minimum_required(VERSION 3.1)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules)
set(TELEPATHY_OFONO telepathy-ofono)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Find includes in corresponding build directories
set(CMAKE_INCLUDE_CURRENT_DIR ON)

# Standard install paths
include(GNUInstallDirs)

# Check for include files
include(CheckIncludeFileCXX)

include(EnableCoverageReport)
#####################################################################
# Enable code coverage calculation with gcov/gcovr/lcov
# Usage:
#  * Switch build type to coverage (use ccmake or cmake-gui)
#  * Invoke make, make test, make coverage
#  * Find html report in subdir coveragereport
#  * Find xml report feasible for jenkins in coverage.xml
#####################################################################
IF(CMAKE_BUILD_TYPE MATCHES [cC][oO][vV][eE][rR][aA][gG][eE])
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ftest-coverage -fprofile-arcs" )
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ftest-coverage -fprofile-arcs" )
  SET(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -coverage" )
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -coverage" )
  ENABLE_COVERAGE_REPORT(TARGETS ${TELEPATHY_OFONO})
ENDIF(CMAKE_BUILD_TYPE MATCHES [cC][oO][vV][eE][rR][aA][gG][eE])

find_package(LibPhoneNumber REQUIRED)
find_package(Qt5 REQUIRED COMPONENTS Core DBus Network)
add_definitions(-DQT_NO_KEYWORDS)

find_package(PkgConfig REQUIRED)
pkg_check_modules(TP_QT5 REQUIRED TelepathyQt5)
pkg_check_modules(SQLITE3 REQUIRED sqlite3)
pkg_check_modules(MC_PLUGINS REQUIRED mission-control-plugins)
pkg_check_modules(OFONO_QT REQUIRED ofono-qt)

find_program(DBUS_RUNNER dbus-test-runner)

set(CMAKE_AUTOMOC ON)

include_directories(${TP_QT5_INCLUDE_DIRS})
include_directories(${SQLITE3_INCLUDE_DIRS})
include_directories(${LibPhoneNumber_INCLUDE_DIRS})
include_directories(/usr/include/telepathy-qt5/)
include_directories(${OFONO_QT_INCLUDE_DIRS})

find_library(TELEPATHY_QT5_LIBRARIES telepathy-qt5)
find_library(TELEPATHY_QT5_SERVICE_LIBRARIES telepathy-qt5-service)

qt5_add_resources(telepathyfono_RES sqlitetelepathyofono.qrc countrycodes.qrc)

# update the .qrc file automatically when there are new schema files
file(GLOB QRC_RESOURCE_FILES ${CMAKE_CURRENT_SOURCE_DIR}/schema/*.sql ${CMAKE_CURRENT_SOURCE_DIR}/schema/*.info)
set(QRC_FILE ${CMAKE_CURRENT_SOURCE_DIR}/sqlitetelepathyofono.qrc)
add_custom_command(
    OUTPUT ${QRC_FILE}
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/update_qrc.sh ${QRC_FILE}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${QRC_RESOURCE_FILES}
    )
add_custom_target(qrc_update DEPENDS ${QRC_FILE} schema_update)

set(TELEPATHY_OFONO_SRC 
   main.cpp
   protocol.cpp
   connection.cpp
   ofonotextchannel.cpp
   ofonocallchannel.cpp
   ofonoconferencecallchannel.cpp
   emergencymodeiface.cpp
   voicemailiface.cpp
   mmsdmanager.cpp
   mmsdservice.cpp
   mmsdmessage.cpp
   mmsgroupcache.cpp
   pendingmessagesmanager.cpp
   phoneutils.cpp
   sqlitedatabase.cpp
   ussdiface.cpp
   ${telepathyfono_RES})

add_executable(${TELEPATHY_OFONO} ${TELEPATHY_OFONO_SRC})
qt5_use_modules(${TELEPATHY_OFONO} Core DBus Sql)
add_dependencies(${TELEPATHY_OFONO} schema_update qrc_update)

enable_testing()

target_link_libraries(${TELEPATHY_OFONO} Qt5::Core Qt5::DBus -ltelepathy-qt5 ${TELEPATHY_QT5_SERVICE_LIBRARIES}  Qt5::Network ${OFONO_QT_LIBRARIES} ${SQLITE3_LIBRARIES} ${LibPhoneNumber_LIBRARIES})
install(TARGETS ${TELEPATHY_OFONO} DESTINATION ${CMAKE_INSTALL_LIBEXECDIR})

configure_file(ofono.service.in org.freedesktop.Telepathy.ConnectionManager.ofono.service)
install (FILES ofono.manager DESTINATION share/telepathy/managers)
install (FILES ${CMAKE_CURRENT_BINARY_DIR}/org.freedesktop.Telepathy.ConnectionManager.ofono.service DESTINATION share/dbus-1/services)

add_subdirectory(schema)
add_subdirectory(tests)
add_subdirectory(mc-plugin)
