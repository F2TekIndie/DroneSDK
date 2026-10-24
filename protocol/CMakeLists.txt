cmake_minimum_required(VERSION 3.16)
project(DGCProto LANGUAGES CXX)

find_package(Protobuf REQUIRED MODULE)

add_library(DGCProto STATIC
	messages.proto
	timeseries.proto
	attributes.proto
	devices.proto
)

protobuf_generate(
    TARGET DGCProto
    LANGUAGE cpp
)

# Generated headers are written into this library's build directory.
target_include_directories(DGCProto PUBLIC
    "${CMAKE_CURRENT_BINARY_DIR}"
)

target_link_libraries(DGCProto PUBLIC
    protobuf::libprotobuf
)

