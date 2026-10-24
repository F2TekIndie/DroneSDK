cmake_minimum_required(VERSION 3.16)
project(DGCSources LANGUAGES CXX)

add_library(DGCSources STATIC
        drone.cpp drone.h
)


# Generated headers are written into this library's build directory.
target_include_directories(DGCSources PUBLIC
    "${CMAKE_CURRENT_BINARY_DIR}"
)

target_link_libraries(DGCSources PUBLIC
)

