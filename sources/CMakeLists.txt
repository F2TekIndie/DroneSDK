cmake_minimum_required(VERSION 3.20)
project(DGCSources LANGUAGES CXX)

add_library(DGCSources STATIC
        drone.cpp drone.h
        asio.cpp asio.h
        mqtt.cpp mqtt.h
)


target_include_directories(DGCSources PUBLIC
    "${CMAKE_CURRENT_SOURCE_DIR}"
)

target_link_libraries(DGCSources PUBLIC
)

