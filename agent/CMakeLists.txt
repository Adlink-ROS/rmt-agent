# minimum cmake version
cmake_minimum_required(VERSION 3.10)

# project name, version
project(RMT_agent VERSION 0.9.8)

# Check Neuron Library exist or not (default is ON while Neuron Library exists)
option(USE_NLIB "Build agent with Neuron Library" ON)
set(nlib_path "/opt/adlink/neuron-sdk/neuron-library")
if(EXISTS ${nlib_path} AND USE_NLIB)
  message("Build with Neuron Library")
  add_definitions(-DSUPPORT_NLIB)
  include_directories(${nlib_path}/include)
  link_directories(${nlib_path}/lib)
  set(NLIB mraa)
else()
  message("Build without Neuron Library")
  set(NLIB "")
endif()

# Support ROS or not (default is OFF)
find_program(LSB_RELEASE_EXEC lsb_release)
execute_process(COMMAND ${LSB_RELEASE_EXEC} -cs
  OUTPUT_VARIABLE UBUNTU_VERSION
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(UBUNTU_VERSION STREQUAL "bionic")
  message("OS: Ubuntu 18.04")
  set(ROS_PATH "/opt/ros/dashing")
else()
  message("OS: Ubuntu 20.04")
  set(ROS_PATH "/opt/ros/foxy")
endif()
option(USE_ROS "Build agent with ROS" OFF)
if(USE_ROS)
  message("Build agent with ROS")
  add_definitions(-DSUPPORT_ROS)
  include_directories(${ROS_PATH}/include)
  link_directories(${ROS_PATH}/lib)
  set(ROSLIB rclcpp rcutils)
else()
  message("Build agent without ROS")
  set(ROSLIB "")
endif()

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBNM REQUIRED libnm)
pkg_check_modules(GLIB REQUIRED glib-2.0)
pkg_check_modules(YAMLCPP REQUIRED yaml-cpp>=0.5)

include_directories(${CMAKE_SOURCE_DIR}/agent)
include_directories(${YAMLCPP_INCLUDE_DIRS})
add_executable(rmt-agent agent_main.cpp datainfo.cpp fileinfo.cpp)

target_link_libraries(rmt-agent PRIVATE rmt_agent ${NLIB} ${ROSLIB} ${YAMLCPP_LIBRARIES} ${LIBNM_LIBRARIES} ${GLIB_LIBRARIES})
target_include_directories(rmt-agent PRIVATE ${LIBNM_INCLUDE_DIRS} ${GLIB_INCLUDE_DIRS})
target_compile_options(rmt-agent PRIVATE ${LIBNM_CFLAGS_OTHER} ${GLIB_CFLAGS_OTHER})
# copy multi_agents.py to 'build/agent/'
add_custom_target(agent_link_target ALL COMMAND ${CMAKE_COMMAND} -E 
                  copy "${PROJECT_SOURCE_DIR}/multi_agents.py" "${PROJECT_BINARY_DIR}/multi_agents.py")

set(INSTALL_PATH "/opt/adlink/rmt/")
install(
  FILES
    ${CMAKE_BINARY_DIR}/rmt-agent
  DESTINATION 
    ${INSTALL_PATH}
)
install(
  FILES
    ${CMAKE_SOURCE_DIR}/rmt-agent.service
  DESTINATION 
    ${INSTALL_PATH}
)
install(
  FILES
    ${CMAKE_SOURCE_DIR}/rmt.conf
  DESTINATION
    ${INSTALL_PATH}
)
install(
  DIRECTORY
    ${CMAKE_SOURCE_DIR}/tasks
  DESTINATION
    ${INSTALL_PATH}
)

# Package
set(CPACK_GENERATOR "DEB")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_PACKAGE_NAME "rmt_agent")
set(CPACK_SET_DESTDIR true)
set(CPACK_INSTALL_PREFIX "/")
set(CPACK_OUTPUT_FILE_PREFIX ${CMAKE_SOURCE_DIR}/build)
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "ChenYing Kuo")
set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "RMT Agent Example")
execute_process(COMMAND dpkg --print-architecture OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE CPACK_DEBIAN_PACKAGE_ARCHITECTURE)
set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}")
set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA
    "${CMAKE_CURRENT_SOURCE_DIR}/postinst;${CMAKE_CURRENT_SOURCE_DIR}/postrm")
include(CPack)

# Create RUN packages
add_custom_target(makeself
    COMMAND rm -rf ${PROJECT_BINARY_DIR}/packages
    COMMAND cp -r ${PROJECT_SOURCE_DIR}/../packages ${PROJECT_BINARY_DIR}
    COMMAND cp ${PROJECT_BINARY_DIR}/*.deb ${PROJECT_BINARY_DIR}/packages
    COMMAND makeself ${PROJECT_BINARY_DIR}/packages ${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}.run "RMT Agent" ./setup.sh 
    )
