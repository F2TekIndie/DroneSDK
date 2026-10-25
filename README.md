# DroneSDK

**Status: proof-of-concept prototype.** This project is intended for exploration
and testing. Features, APIs, and protocols are incomplete and may change; it is
not production-ready.

Shared Protocol Buffer definitions and C++ library scaffolding for Drone
Ground Control.

## Current status

| CMake target | Contents |
| --- | --- |
| `DGCProto` | Generated Protocol Buffer C++ messages and the protobuf runtime dependency. |
| `DGCSources` | Shared drone, ASIO, and MQTT source placeholders. |

The files in `sources/` are currently empty. The SDK does not yet provide a
drone-control API, network transport, or MQTT implementation.

Consumers are the sibling [desktop application](../DGC/README.md) and
[server/simulator project](../DGCServer/README.md).

## Source layout

```text
protocol/
├── CMakeLists.txt
├── messages.proto
├── devices.proto
├── attributes.proto
└── timeseries.proto

sources/
├── CMakeLists.txt
├── drone.h / drone.cpp
├── asio.h / asio.cpp
└── mqtt.h / mqtt.cpp
```

There is no top-level SDK CMake project. Include or build the two subprojects
separately.

## Dependencies

- CMake 3.20 or later.
- A C++ compiler.
- Protocol Buffers compiler and development library for `DGCProto`.

The desktop and server projects select C++23. The SDK subprojects do not
independently select a language standard. No additional networking or MQTT
libraries are currently linked by `DGCSources`.

## Build separately

From this repository:

```sh
cmake -S protocol -B build/protocol -DCMAKE_CXX_STANDARD=23
cmake --build build/protocol --parallel

cmake -S sources -B build/sources -DCMAKE_CXX_STANDARD=23
cmake --build build/sources --parallel
```

## Use from another CMake project

For a consumer checked out alongside `DroneSDK`:

```cmake
add_subdirectory(
    "${CMAKE_CURRENT_SOURCE_DIR}/../DroneSDK/protocol"
    "${CMAKE_CURRENT_BINARY_DIR}/dronesdkproto"
)

add_subdirectory(
    "${CMAKE_CURRENT_SOURCE_DIR}/../DroneSDK/sources"
    "${CMAKE_CURRENT_BINARY_DIR}/dronesdksrc"
)

# Replace your_target with an executable or library already created by CMake.
target_link_libraries(your_target PRIVATE DGCProto DGCSources)
```

Each target exports its include directory. `DGCProto` also propagates
`protobuf::libprotobuf` to consumers.

```cpp
#include <messages.pb.h>
#include <devices.pb.h>
#include <drone.h>
```

Protocol Buffer headers and implementations are generated in the build
directory. Edit the `.proto` definitions rather than generated files.

The current integration uses `add_subdirectory`; no installed CMake package
or SDK installation rules are provided.

## Protocol overview

All messages use the `dronegroundcontrol` package.

| Definition | Purpose |
| --- | --- |
| `Message` | Request, device, and client IDs with a getter or setter payload. |
| `SetMessage`, `GetMessage` | Operation-specific payloads selected through `oneof`. |
| `Client`, `Device`, `Devices` | Client/device descriptions and a paginated device list. |
| `Attribute`, `Attributes`, `Position` | Named values and latitude/longitude strings. |
| `TimeSerie`, `TimeSeries` | Named, timestamped values and paginated collections. |

Timestamps are strings; their format and timezone are not yet specified.
Numeric attribute and time-series values currently use `uint32`. Transport
framing, validation rules, error responses, and persistence semantics are not
defined by the current implementation.

## Schema changes

The schema is still being developed, with no stored messages reported so far.
Rebuild all consumers together when changing it.

Once persisted data or independently deployed clients rely on the protocol,
preserve existing field numbers and assign unused numbers to new fields.
