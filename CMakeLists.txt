cmake_policy(SET CMP0048 NEW)
project(
  ${SKBUILD_PROJECT_NAME}
  VERSION ${SKBUILD_PROJECT_VERSION}
  LANGUAGES CXX)

cmake_minimum_required(VERSION 3.15...3.27)

option(BUILD_PYTHON_BINDING "Build Python Binding" off)
option(BUILD_SDK_EXAMPLES "Build example programs" off)

set(CMAKE_CXX_STANDARD 17)

execute_process(
    COMMAND uname -m
    OUTPUT_VARIABLE MACHINE_TYPE
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

string(FIND "${MACHINE_TYPE}" "x86_64" IS_X86_64)


add_library(booster_sdk STATIC IMPORTED GLOBAL)
target_include_directories(booster_sdk INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
set_target_properties(booster_sdk PROPERTIES
    CMAKE_POSITION_INDEPENDENT_CODE ON)

target_compile_features(booster_sdk INTERFACE cxx_std_17)

if(NOT IS_X86_64 EQUAL -1)
    message(STATUS "This machine's cpu architecture is x86_64")
    set_target_properties(booster_sdk PROPERTIES
        IMPORTED_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/lib/x86_64/libbooster_robotics_sdk.a
    )
else()
    message(STATUS "This machine's cpu architecture is arm")
    set_target_properties(booster_sdk PROPERTIES
        IMPORTED_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/lib/aarch64/libbooster_robotics_sdk.a
    )
endif()

target_link_libraries(booster_sdk INTERFACE fastrtps fastcdr libfoonathan_memory-0.7.3.a)


if(BUILD_SDK_EXAMPLES)
    add_executable(b1_loco_example_client example/high_level/b1_loco_example_client.cpp)
    add_executable(b1_arm_sdk_example_client example/high_level/b1_arm_sdk_example.cpp)
    add_executable(b1_7dof_arm_sdk_example_client example/high_level/b1_7dof_arm_sdk_example.cpp)
    add_executable(b1_low_level_publisher example/low_level/low_level_publisher.cpp)
    add_executable(b1_low_level_subscriber example/low_level/low_level_subscriber.cpp)
    add_executable(low_level_hand_data_subscriber example/low_level/low_level_hand_data_subscriber.cpp)
    add_executable(b1_low_sdk_example example/low_level/b1_low_sdk_example.cpp)
    add_executable(b1_7dof_arm_low_sdk_example example/low_level/b1_7dof_arm_low_sdk_example.cpp)
    add_executable(odometer_example example/low_level/odometer_example.cpp)

    target_link_libraries(b1_loco_example_client PUBLIC booster_sdk)
    target_link_libraries(b1_arm_sdk_example_client PUBLIC booster_sdk)
    target_link_libraries(b1_7dof_arm_sdk_example_client PUBLIC booster_sdk)
    target_link_libraries(b1_low_level_publisher PUBLIC booster_sdk)
    target_link_libraries(b1_low_level_subscriber PUBLIC booster_sdk)
    target_link_libraries(low_level_hand_data_subscriber PUBLIC booster_sdk)
    target_link_libraries(b1_low_sdk_example PUBLIC booster_sdk)
    target_link_libraries(b1_7dof_arm_low_sdk_example PUBLIC booster_sdk)
    target_link_libraries(odometer_example PUBLIC booster_sdk)
endif()


if(BUILD_PYTHON_BINDING)
    find_package(Python3 REQUIRED COMPONENTS Interpreter Development.Module)
    message(STATUS "Python3 executable: ${Python3_EXECUTABLE}")

    execute_process(
        COMMAND ${Python3_EXECUTABLE} -c "import sysconfig; print(sysconfig.get_paths()['purelib'])"
        OUTPUT_VARIABLE PYTHON_SITE_PACKAGES
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )

    include_directories(${Python3_INCLUDE_DIRS})
    find_package(pybind11 CONFIG REQUIRED)

    python3_add_library(booster_robotics_sdk_python MODULE python/binding.cpp WITH_SOABI)

    target_compile_definitions(booster_robotics_sdk_python PRIVATE VERSION_INFO=${PROJECT_VERSION})

    find_program(PYBIND11_STUBGEN_EXECUTABLE pybind11-stubgen)
    if(NOT PYBIND11_STUBGEN_EXECUTABLE)
        message(FATAL_ERROR "pybind11-stubgen not found")
    endif()

    add_custom_command(
        TARGET booster_robotics_sdk_python
        POST_BUILD
        COMMAND PYTHONPATH=${CMAKE_SOURCE_DIR}/build:/${PYTHONPATH} pybind11-stubgen -o ${CMAKE_SOURCE_DIR}/build booster_robotics_sdk_python
    )

    install(TARGETS booster_robotics_sdk_python LIBRARY DESTINATION ${PYTHON_SITE_PACKAGES})
    install(FILES ${CMAKE_SOURCE_DIR}/build/booster_robotics_sdk_python.pyi DESTINATION ${PYTHON_SITE_PACKAGES})    
endif()

