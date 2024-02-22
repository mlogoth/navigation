cmake_minimum_required(VERSION 2.8.3)
project(map_server)

if(NOT "/usr/lib/x86_64-linux-gnu/cmake/sdl2_image/sdl2_image-config.cmake")
    message("-- Configuring SDL2_image")
    message("Try to install sdl2...")
    execute_process(COMMAND sudo apt-get install -y sdl2-*)
    if(NOT "/usr/lib/x86_64-linux-gnu/cmake/sdl2_image/") 
        message("/usr/lib/x86_64-linux-gnu/cmake/sdl2_image not found.")
        execute_process(COMMAND sudo mkdir /usr/lib/x86_64-linux-gnu/cmake/sdl2_image)
    endif (NOT "/usr/lib/x86_64-linux-gnu/cmake/sdl2_image/")
    message("Copy cmake file to /usr/lib/x86_64-linux-gnu/cmake/sdl2_image")
    execute_process(COMMAND sudo cp ${CMAKE_CURRENT_LIST_DIR}/cmake/sdl2_image-config.cmake /usr/lib/x86_64-linux-gnu/cmake/sdl2_image/.)
endif(NOT "/usr/lib/x86_64-linux-gnu/cmake/sdl2_image/sdl2_image-config.cmake")

find_package(catkin REQUIRED
        COMPONENTS
            roscpp
            nav_msgs
            tf2
        )
find_package(Bullet REQUIRED)
find_package(SDL2 REQUIRED)
find_package(SDL2_image REQUIRED)
find_package(Boost REQUIRED COMPONENTS filesystem)

find_package(PkgConfig REQUIRED)
pkg_check_modules(YAMLCPP yaml-cpp REQUIRED)
if(YAMLCPP_VERSION VERSION_GREATER "0.5.0")
    add_definitions(-DHAVE_YAMLCPP_GT_0_5_0)
endif()
link_directories(${YAMLCPP_LIBRARY_DIRS})

catkin_package(
    INCLUDE_DIRS
        include
    LIBRARIES
        map_server_image_loader
    CATKIN_DEPENDS
        roscpp
        nav_msgs
        tf2
)

include_directories(
    include
    ${BULLET_INCLUDE_DIRS}
    ${catkin_INCLUDE_DIRS}
    ${SDL2_INCLUDE_DIR}
    ${SDL2_IMAGE_INCLUDE_DIRS}
    ${YAMLCPP_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
)

add_library(map_server_image_loader src/image_loader.cpp)
add_dependencies(map_server_image_loader ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(map_server_image_loader
    ${BULLET_LIBRARIES}
    ${catkin_LIBRARIES}
    ${SDL2_LIBRARY}
    ${SDL2_IMAGE_LIBRARIES}
)

add_executable(map_server src/main.cpp)
add_dependencies(map_server ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(map_server
    map_server_image_loader
    ${YAMLCPP_LIBRARIES}
    ${catkin_LIBRARIES}
)

add_executable(map_server-map_saver src/map_saver.cpp)
add_dependencies(map_server-map_saver ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
set_target_properties(map_server-map_saver PROPERTIES OUTPUT_NAME map_saver)
target_link_libraries(map_server-map_saver
    ${catkin_LIBRARIES}
)

# copy test data to same place as tests are run
function(copy_test_data)
    cmake_parse_arguments(PROJECT "" "" "FILES" ${ARGN})
    foreach(datafile ${PROJECT_FILES})
        file(COPY ${datafile} DESTINATION ${PROJECT_BINARY_DIR}/test)
    endforeach()
endfunction()

## Tests
if(CATKIN_ENABLE_TESTING)
  copy_test_data( FILES
      test/testmap.bmp
      test/testmap.png )
  catkin_add_gtest(${PROJECT_NAME}_utest test/utest.cpp test/test_constants.cpp)
  target_link_libraries(${PROJECT_NAME}_utest
    map_server_image_loader
    ${SDL2_LIBRARY}
    ${SDL2_IMAGE_LIBRARIES}
  )

  add_executable(rtest test/rtest.cpp test/test_constants.cpp)
  add_dependencies(rtest ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
  add_dependencies(tests rtest)
  target_link_libraries( rtest
      ${GTEST_LIBRARIES}
      ${catkin_LIBRARIES}
  )

  # This has to be done after we've already built targets, or catkin variables get borked
  find_package(rostest REQUIRED)
  add_rostest(test/rtest.xml)
endif()

## Install executables and/or libraries
install(TARGETS map_server-map_saver map_server
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})

install(TARGETS map_server_image_loader
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION})

## Install project namespaced headers
install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  FILES_MATCHING PATTERN "*.h"
  PATTERN ".svn" EXCLUDE)

## Install excutable python script
install(
    PROGRAMS
      scripts/crop_map
    DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})
