# DroneSDK

Shared Protobuf messages and gRPC service definitions for
[DGC](../DGC/README.md) and [DGCServer](../DGCServer/README.md).
The protocol is a development prototype and can change with its consumers.

## Scope and layout

```text
protocol/
├── CMakeLists.txt
├── devices.proto
├── attributes.proto
└── timeseries.proto
```

All definitions use the package `dronegroundcontrol`. The SDK contains schemas
and generated bindings, not application models, a database layer, or an MQTT
client implementation.

`DGCProto` is a static library containing generated message classes (`*.pb.h`,
`*.pb.cc`) and gRPC bindings (`*.grpc.pb.h`, `*.grpc.pb.cc`). It exports its build
include directory and links `gRPC::grpc++` and `protobuf::libprotobuf` publicly.
There is no top-level SDK CMake project or installed SDK package.

The desktop's application models live in `DGC/src/drone.h`; its transport
conversion stays inside `gRPCClient`. Server cache/simulator models and MQTT
support live in `DGCServer/sdk/`. These are separate from the generated messages.

## Dependencies and build

- CMake 3.21 or later.
- A C++ compiler; the application projects use C++23.
- Protobuf compiler and development library, discoverable by CMake's
  `FindProtobuf` module.
- gRPC C++ development files, `gRPCConfig.cmake`, and `grpc_cpp_plugin`.

Build from DroneSDK:

```sh
cmake -S protocol -B build/protocol -DCMAKE_CXX_STANDARD=23
cmake --build build/protocol --parallel
```

The applications normally build the protocol as a subdirectory instead:

```cmake
add_subdirectory(
    "${CMAKE_CURRENT_SOURCE_DIR}/../DroneSDK/protocol"
    "${CMAKE_CURRENT_BINARY_DIR}/dronesdkproto"
)

# Create your_target before linking it.
target_link_libraries(your_target PRIVATE DGCProto)
```

Consumers handling Protobuf/gRPC can include generated headers such as:

```cpp
#include <devices.pb.h>
#include <devices.grpc.pb.h>
```

Edit `.proto` files, not generated files. Rebuild consumers together after
protocol changes. Preserve established field numbers when compatible changes
are required; the current UUID `Device.id` is incompatible with earlier numeric
IDs. Use the current SQL schema for development databases.

## Identity and values

| Message | Fields and representation |
| --- | --- |
| `Client` | Nonzero uint32 `id`, plus `name`. |
| `Device` | Canonical lowercase UUID string `id`, `name`, `type`, and nested `client_id`. New server-generated IDs are UUID v4. |
| `Position` | Decimal coordinate strings `lat` and `long`. |
| `Attribute` | `name` plus oneof bool, string, uint32, or position. |
| `TimeSerie` | `name`, decimal UTC Unix-millisecond `timestamp`, and oneof bool, string, or uint32. |
| `Attributes` | Repeated attributes in the field named `timeseries`. |
| `TimeSeries` | Repeated samples in `timeseries`, plus response metadata `start` and `range`. |

The schemas can represent more attributes and series than the server currently
stores. Supported current attributes are `online` (bool), `state` (uint32), and
`position`. Supported history names are `latitude` and `longitude`, with string
coordinate values. The server cache does not currently represent device type or
client name, although these fields exist in the protocol.

Coordinates must be finite and within latitude `-90..90`, longitude `-180..180`.
Timestamp strings are signed 64-bit UTC Unix milliseconds. Optional query bounds
use signed `int64` fields directly; presence distinguishes omission from epoch
zero. Representation and validation rules are enforced by consumers, not by
Protobuf's string fields.

## RPC reference

| Service | RPC | Request | Reply |
| --- | --- | --- | --- |
| `DeviceFunctions` | `CreateDevice` | `Client` | `Device` |
| | `DeletDevice` | `Device` | `DeviceStatusReply` |
| | `ListDevices` | `Client` | `Devices` |
| `AttributeFunctions` | `UpdateAttribute` | `AttributeRequest` | `AttributeStatusReply` |
| | `GetAttribute` | `AttributeQuery` | `Attribute` |
| | `GetAttributes` | `Device` | `Attributes` |
| | `UpdatePosition` | `PositionRequest` | `AttributeStatusReply` |
| | `GetPosition` | `Device` | `Position` |
| `TimeseriesFunctions` | `AddTimeseries` | `TimeseriesRequest` | `TimeseriesStatusReply` |
| | `GetTimeseries` | `TimeseriesQuery` | `TimeSerie` |
| | `GetTimeseriesRange` | `TimeseriesRangeQuery` | `TimeSeries` |

`DeletDevice` is intentionally documented with the spelling in the schema.

`AttributeQuery` contains `device` and `name`. `TimeseriesQuery` contains
`device`, `name`, and optional `timestamp_ms`; omit it for the latest sample.
`TimeseriesRangeQuery` contains `device`, `name`, and optional inclusive
`start_timestamp_ms` and `end_timestamp_ms`. Omitted bounds are unbounded; empty
range name selects both coordinate series. Results are ordered by timestamp,
latitude before longitude at equal timestamps.

Current collection replies use `start=0` and `range=returned entry count`.
These are response fields, not request pagination controls. Broad history reads
are unpaginated and can exceed a client's receive-message limit.

Create returns a new UUID and default name. Metadata, attribute, and history
updates require an existing cached UUID and matching nonzero client ID. Reads
and deletion check client identity when supplied. Unknown drones or exact samples
return `NOT_FOUND`; malformed requests return `INVALID_ARGUMENT`. A range with
no matching samples returns an empty successful collection.

Server handlers interact only with the cache. Successful mutations acknowledge
cache acceptance, not an SQL commit. The DB worker handles synchronization and
imports SQL-created identities independently of incoming requests.

## Write envelopes and MQTT payloads

`AttributeRequest` carries `device` and either legacy `attrib` (field 2) or a
nonempty `attributes` batch (field 3). `TimeseriesRequest` similarly carries
`device` and either `data` (field 2) or a nonempty `timeseries` batch (field 3).
Do not populate both forms. There is no generic `messages.proto` envelope.

The server validates each batch before changing cached state. History writes
must contain both latitude and longitude for each timestamp in the same message.
A lone coordinate is rejected; there is no cross-message pairing buffer. Drone
UUID plus timestamp identifies a position pair, so replay updates that pair.

| MQTT publish topic | Payload | Simulator use |
| --- | --- | --- |
| `/device/<uuid>` | `Device` | Existing identity and name metadata. |
| `/device/<uuid>/attributes` | `AttributeRequest` | One batch containing online and state. |
| `/device/<uuid>/timeseries` | `TimeseriesRequest` | One batch containing latitude and longitude with a shared timestamp. |
| `/status` | Undefined | Server subscribes, but does not decode/store a payload. |

The server subscribes to `/device/+`, `/device/+/attributes`,
`/device/+/timeseries`, and `/status`. The topic structure selects the message
type; payload content remains authoritative for drone/client identity. The topic
UUID is not compared with the payload UUID or used for cache lookup.

Payloads use binary `SerializeToString`/`ParseFromString`. Register drones using
`CreateDevice` or the server's `scripts/create_drones.py` utility before sending
telemetry. SQL-created drones become usable after the worker imports them.

The simulator and subscriptions use MQTT QoS 2. Publications are retained per
drone and topic category. Timeseries retention keeps only the latest batch, not
complete history. QoS and retained delivery do not acknowledge SQL persistence. See the
[server README](../DGCServer/README.md) for startup and broker behavior.

## Verification

The SDK has no standalone test target. Both consumer projects generate these
bindings during their builds. Their integration tests exercise requests,
conversion, validation, cache behavior, and persistence:

```sh
ctest --test-dir ../DGC/build/local --output-on-failure
ctest --test-dir ../DGCServer/build/local --output-on-failure
```

Build each consumer with `BUILD_TESTING=ON` first; see its README for dependencies
and configuration.
