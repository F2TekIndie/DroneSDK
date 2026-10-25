cmake_minimum_required(VERSION 3.21)
project(DGCProto LANGUAGES CXX)

find_package(Protobuf REQUIRED MODULE)
find_package(gRPC CONFIG REQUIRED)


add_library(DGCProto STATIC
	timeseries.proto
	attributes.proto
	devices.proto
)

protobuf_generate(
    TARGET DGCProto
    LANGUAGE cpp
)

protobuf_generate(
    TARGET DGCProto
    LANGUAGE grpc
    GENERATE_EXTENSIONS .grpc.pb.h .grpc.pb.cc
    PLUGIN "protoc-gen-grpc=$<TARGET_FILE:gRPC::grpc_cpp_plugin>"
)

# Generated headers are written into this library's build directory.
target_include_directories(DGCProto PUBLIC
    "${CMAKE_CURRENT_BINARY_DIR}"
)

target_link_libraries(DGCProto PUBLIC
    gRPC::grpc++
    protobuf::libprotobuf
)

