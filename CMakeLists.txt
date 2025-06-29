cmake_minimum_required (VERSION 2.6)
project(rovio)
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/DetectSIMD.cmake)
set(ROVIO_NMAXFEATURE 25 CACHE STRING "Number of features for ROVIO")
set(ROVIO_NCAM 1 CACHE STRING "Number of enabled cameras")
set(ROVIO_NLEVELS 4 CACHE STRING "Number of image leavels for the features")
set(ROVIO_PATCHSIZE 6 CACHE STRING "Size of patch (edge length in pixel)")
set(ROVIO_NPOSE 0 CACHE STRING "Additional estimated poses for external pose measurements")
add_definitions(-DROVIO_NMAXFEATURE=${ROVIO_NMAXFEATURE})
add_definitions(-DROVIO_NCAM=${ROVIO_NCAM})
add_definitions(-DROVIO_NLEVELS=${ROVIO_NLEVELS})
add_definitions(-DROVIO_PATCHSIZE=${ROVIO_PATCHSIZE})
add_definitions(-DROVIO_NPOSE=${ROVIO_NPOSE})
add_definitions(-std=c++11)

add_subdirectory(lightweight_filtering)

##################### Find, include, and compile library #####################
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
if(0)
	find_package(OpenMP REQUIRED)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

find_package(Eigen REQUIRED)
include_directories(${EIGEN_INCLUDE_DIR})

if(MAKE_SCENE)
	message(STATUS "Building ROVIO with openGL Scene Visualization")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DMAKE_SCENE=1")
	find_package(GLEW REQUIRED)
	include_directories(${GLEW_INCLUDE_DIRS})

	find_package(GLUT REQUIRED)
	include_directories(${GLUT_INCLUDE_DIRS})
	link_directories(${GLUT_LIBRARY_DIRS})
	add_definitions(${GLUT_DEFINITIONS})

	find_package(OpenGL REQUIRED)
	include_directories(${OpenGL_INCLUDE_DIRS})
	link_directories(${OpenGL_LIBRARY_DIRS})
	add_definitions(${OpenGL_DEFINITIONS})
endif()

find_package(PkgConfig)

find_package(glog_catkin REQUIRED)

find_package(opencv3_catkin QUIET)
if(opencv3_catkin_FOUND)
  message(STATUS "Found opencv3_catkin!")
  set(OPENCV_PACKAGE "opencv3_catkin")
else()
  message(STATUS "Use the systems opencv installation!")
  set(OPENCV_PACKAGE "")
endif()

find_package(yaml_cpp_catkin QUIET)
if(${yaml_cpp_catkin_FOUND})
  message(STATUS "Found yaml_cpp_catkin, using instead of system library.")
  set(YamlCpp_LIBRARIES ${yaml_cpp_catkin_LIBRARIES})
  set(YamlCpp_INCLUDE_DIRS ${yaml_cpp_catkin_INCLUDE_DIRS})
else()
  message(STATUS "No yaml_cpp_catkin, using yaml-cpp system library instead.")
  pkg_check_modules(YamlCpp REQUIRED yaml-cpp>=0.5)
endif()


##################### Install ROS stuff #####################
find_package(catkin REQUIRED COMPONENTS
  ${OPENCV_PACKAGE}
  cv_bridge
  geometry_msgs
  kindr
  lightweight_filtering
  message_generation
  nav_msgs
  rosbag
  roscpp
  roslib
  sensor_msgs
  std_msgs
  tf
)

add_service_files(
  DIRECTORY srv
  FILES SrvResetToPose.srv
)

generate_messages(DEPENDENCIES
  std_msgs
  geometry_msgs
)

catkin_package(
	INCLUDE_DIRS include ${catkin_INCLUDE_DIRS}
    LIBRARIES ${PROJECT_NAME}
	CATKIN_DEPENDS
  ${OPENCV_PACKAGE}
  cv_bridge
  geometry_msgs
  kindr
  lightweight_filtering
  message_runtime
  nav_msgs
  rosbag
  roscpp
  roslib
  sensor_msgs
  std_msgs
  tf
  yaml_cpp_catkin
)

include_directories(include ${catkin_INCLUDE_DIRS} ${YamlCpp_INCLUDE_DIRS} ${glog_catkin_INCLUDE_DIRS})

set(ROVIO_SRCS
  src/Camera.cpp
  src/CameraCalibration.cpp
  src/FeatureCoordinates.cpp
  src/FeatureDistance.cpp
  src/RovioInterfaces.cpp
)

# Optionally also build the SCENE files. 
if(MAKE_SCENE)
	set(ROVIO_SRCS ${ROVIO_SRCS} src/Scene.cpp)  
endif()

add_library(${PROJECT_NAME} ${ROVIO_SRCS})
target_link_libraries(${PROJECT_NAME} ${catkin_LIBRARIES} ${YamlCpp_LIBRARIES} ${OpenMP_EXE_LINKER_FLAGS} ${OPENGL_LIBRARIES} ${GLUT_LIBRARY} ${GLEW_LIBRARY} ${glog_catkin_LIBRARIES})
add_dependencies(${PROJECT_NAME} ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS} rovio_generate_messages_cpp)

add_executable(rovio_node src/rovio_node.cpp)
target_link_libraries(rovio_node ${PROJECT_NAME})

add_executable(rovio_rosbag_loader src/rovio_rosbag_loader.cpp)
target_link_libraries(rovio_rosbag_loader ${PROJECT_NAME})
add_dependencies(rovio_rosbag_loader ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

add_executable(feature_tracker_node src/feature_tracker_node.cpp)
target_link_libraries(feature_tracker_node ${PROJECT_NAME})
add_dependencies(feature_tracker_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

if(CATKIN_ENABLE_TESTING)
	message(STATUS "Building GTests!")
	option(BUILD_GTEST "build gtest" ON)
  set(GMOCK_LIB /usr/local/lib/libgmock.a)
  set(GMOCK_MAIN_LIB /usr/local/lib/libgmock_main.a)
  message(WARNING ${gtest_main})
	#add_subdirectory(gtest gtest)
	enable_testing()
	include_directories(${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR})
	add_executable(test_patch src/test_patch.cpp src/Camera.cpp src/FeatureCoordinates.cpp src/FeatureDistance.cpp)
	target_link_libraries(test_patch gtest_main gtest ${GMOCK_LIB} ${GMOCK_MAIN_LIB} pthread ${PROJECT_NAME} ${catkin_LIBRARIES} ${YamlCpp_LIBRARIES} ${glog_catkin_LIBRARIES} )
	add_test(test_patch test_patch)

	add_executable(test_mlp src/test_mlp.cpp src/Camera.cpp src/FeatureCoordinates.cpp src/FeatureDistance.cpp)
	target_link_libraries(test_mlp gtest_main gtest ${GMOCK_LIB} ${GMOCK_MAIN_LIB} pthread ${PROJECT_NAME} ${catkin_LIBRARIES} ${YamlCpp_LIBRARIES} ${glog_catkin_LIBRARIES})
	add_test(test_mlp test_mlp)

	add_executable(test_llu src/test_localization_landmark_update.cpp)
  target_link_libraries(test_llu gtest_main gtest ${GMOCK_LIB} ${GMOCK_MAIN_LIB} pthread ${PROJECT_NAME} ${catkin_LIBRARIES} ${YamlCpp_LIBRARIES} ${glog_catkin_LIBRARIES})
  add_test(test_llu test_llu)
endif()
               