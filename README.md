# Vcpkg.Grpc.Tools.Cpp 

[![build](https://github.com/MartinKuschnik/Vcpkg.Grpc.Tools.Cpp/workflows/NuGet/badge.svg)](https://github.com/MartinKuschnik/Vcpkg.Grpc.Tools.Cpp/actions) 
[![NuGet Status](http://img.shields.io/nuget/v/Vcpkg.Grpc.Tools.Cpp.svg?style=flat)](https://www.nuget.org/packages/Vcpkg.Grpc.Tools.Cpp/)

**gRPC and Protocol Buffer compiler integration for native C++ projects using vcpkg.**

This NuGet package provides seamless integration of Protocol Buffer (`.proto`) compilation and gRPC code generation into Visual Studio C++ projects that use vcpkg for dependency management.

---

## ?? Features

- ? **Automatic Compilation**: `.proto` files are automatically compiled during build
- ? **Incremental Builds**: Only changed `.proto` files are recompiled (saves build time!)
- ? **Per-File Configuration**: Configure compilation settings individually for each `.proto` file
- ? **Visual Studio Integration**: Full property page support in Visual Studio
- ? **vcpkg Integration**: Works seamlessly with vcpkg's manifest and classic mode
- ? **Smart Import Handling**: Flexible import path configuration per file
- ? **Zero Configuration**: Works out-of-the-box with sensible defaults

---

## ?? Prerequisites

- **Visual Studio** (2015 or later) with C++ development tools
- **Windows** operating system
- **vcpkg** installed and integrated with Visual Studio
- **protobuf** and **grpc** packages installed via vcpkg

---

## ?? Quick Start

### 1. Install Required vcpkg Packages

#### Option A: Using vcpkg Manifest Mode (Recommended)

Add to your `vcpkg.json`:
```json
{
  "dependencies": [
    "grpc",
    "protobuf"
  ]
}
```

#### Option B: Using vcpkg Classic Mode

```bash
vcpkg install grpc:x64-windows
vcpkg install protobuf:x64-windows
```

### 2. Install the NuGet Package

Add the NuGet package to your C++ project:

```powershell
Install-Package Vcpkg.Grpc.Tools.Cpp
```

Or via Package Manager UI in Visual Studio.

### 3. Add .proto Files to Your Project

1. Add your `.proto` files to your project
2. Right-click on a `.proto` file ? **Properties**
3. Set **Item Type** to **Protobuf Compile**

Example `.proto` file structure:
```
YourProject/
??? protos/
?   ??? messages.proto
?   ??? service.proto
??? YourProject.vcxproj
```

### 4. Build Your Project

When you build your project, the following files are automatically generated:

For each `.proto` file (e.g., `messages.proto`):
- `messages.pb.h` / `messages.pb.cc` - Protocol Buffer message definitions
- `messages.grpc.pb.h` / `messages.grpc.pb.cc` - gRPC service stubs

Generated files are placed in:
- `$(IntDir)\protobuf\` - for `.pb.h` and `.pb.cc` files
- `$(IntDir)\grpc\` - for `.grpc.pb.h` and `.grpc.pb.cc` files

---

## ?? Configuration

### Per-File Properties

Right-click on any `.proto` file in Solution Explorer ? **Properties** to configure:

| Property | Description | Default |
|----------|-------------|---------|
| **Compile Protobuf** | Whether to compile this file or only use it for imports | `true` |
| **Proto Root Directory** | Root directory for proto imports | Auto-detected |
| **Additional Import Directories** | Additional paths for proto imports (semicolon-separated) | Empty |

### Example Configuration in .vcxproj

```xml
<ItemGroup>
  <ProtobufCompile Include="protos\messages.proto">
    <ProtoCompile>true</ProtoCompile>
    <ProtoRoot>.</ProtoRoot>
  </ProtobufCompile>
  
  <!-- This file is only for imports, not compiled -->
  <ProtobufCompile Include="protos\common.proto">
    <ProtoCompile>false</ProtoCompile>
  </ProtobufCompile>
  
  <!-- Custom import paths -->
  <ProtobufCompile Include="protos\service.proto">
    <ProtoCompile>true</ProtoCompile>
    <AdditionalImportDirs>external\protos;third_party\protos</AdditionalImportDirs>
  </ProtobufCompile>
</ItemGroup>
```

### Proto Root Directory Logic

The package automatically determines the proto root for imports:

- **Files inside project directory**: `ProtoRoot` = `.` (project directory)
- **Files outside project directory**: `ProtoRoot` = file's directory
- **Explicit configuration**: Use the property to override

---

## ?? How It Works

### Build Process

1. **Validation**: Checks if vcpkg and required tools are available
2. **File Selection**: Determines which `.proto` files need compilation
3. **Up-to-Date Check**: Skips files that haven't changed (incremental build)
4. **Compilation**: Runs `protoc.exe` with `grpc_cpp_plugin.exe` for outdated files
5. **Integration**: Adds generated `.cc` files to the compilation

### Generated File Locations

```
$(IntDir) (e.g., x64\Debug\)
??? protobuf\
?   ??? messages.pb.h
?   ??? messages.pb.cc
??? grpc\
    ??? messages.grpc.pb.h
    ??? messages.grpc.pb.cc
```

Include paths are automatically configured, so you can simply:
```cpp
#include "messages.pb.h"
#include "messages.grpc.pb.h"
```

---

## ?? Example Usage

### Simple Message Definition

**protos/person.proto:**
```protobuf
syntax = "proto3";

package example;

message Person {
  string name = 1;
  int32 age = 2;
  string email = 3;
}
```

**main.cpp:**
```cpp
#include "person.pb.h"
#include <iostream>

int main() {
    example::Person person;
    person.set_name("John Doe");
    person.set_age(30);
    person.set_email("john@example.com");
    
    std::cout << "Name: " << person.name() << std::endl;
    std::cout << "Age: " << person.age() << std::endl;
    
    return 0;
}
```

### gRPC Service Definition

**protos/greeter.proto:**
```protobuf
syntax = "proto3";

package greeter;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

**server.cpp:**
```cpp
#include "greeter.grpc.pb.h"
#include <grpcpp/grpcpp.h>

class GreeterServiceImpl final : public greeter::Greeter::Service {
    grpc::Status SayHello(grpc::ServerContext* context,
                         const greeter::HelloRequest* request,
                         greeter::HelloReply* reply) override {
        reply->set_message("Hello, " + request->name());
        return grpc::Status::OK;
    }
};

int main() {
    std::string server_address("0.0.0.0:50051");
    GreeterServiceImpl service;

    grpc::ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    
    std::unique_ptr<grpc::Server> server(builder.BuildAndStart());
    std::cout << "Server listening on " << server_address << std::endl;
    server->Wait();
    
    return 0;
}
```

---

## ?? Troubleshooting

### Error: "Vcpkg not enabled"

**Solution**: Enable vcpkg integration in Visual Studio:
```bash
vcpkg integrate install
```

### Error: "protobuf not installed" or "gRPC not found"

**Solution**: Install the required packages via vcpkg:
```bash
vcpkg install grpc:x64-windows protobuf:x64-windows
```

Or add them to your `vcpkg.json` if using manifest mode.

### Generated Files Not Found

**Solution**: 
1. Check that the build succeeded without errors
2. Look in `$(IntDir)\protobuf\` and `$(IntDir)\grpc\` (e.g., `x64\Debug\protobuf\`)
3. Rebuild the project to regenerate files

### Proto File Changes Not Detected

This package uses **CustomBuild** items with MSBuild's incremental build tracking. Changes should be detected automatically. If not:
1. Clean the project (Build ? Clean Solution)
2. Rebuild (Build ? Rebuild Solution)

### Include Errors in Generated Files

If you see errors like `cannot open include file: 'google/protobuf/...'`:

**Solution**: Make sure the protobuf include directories are in your project's additional include directories. This is usually handled automatically by vcpkg integration.

---

## ?? Incremental Build Support

**Key Feature**: Only changed `.proto` files are recompiled!

The package uses MSBuild's `CustomBuild` item type with input/output tracking:
- ? Compares timestamps of `.proto` files and generated outputs
- ? Skips compilation if files are up-to-date
- ? Significantly reduces build times in large projects
- ? Detects changes to `.proto` files automatically

---

## ?? What's Included

This NuGet package contains:
- `Vcpkg.Grpc.Tools.CPP.targets` - MSBuild integration
- `Vcpkg.Grpc.Tools.CPP.proto.xml` - Visual Studio property page definitions

**Note**: The actual `protoc.exe` and `grpc_cpp_plugin.exe` compilers come from your vcpkg installation.

---

## ?? Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## ?? License

This project is provided as-is without any specific license restrictions. You are free to use, modify, and distribute this software as you see fit, but please do not claim it as your own work.

**Copyright © 2025 Martin Kuschnik**

---

## ?? Version History

For detailed version history and changelog, see the [Releases](https://github.com/MartinKuschnik/Vcpkg.Grpc.Tools.Cpp/releases) page.

---

## ?? Related Links

- [Protocol Buffers Documentation](https://protobuf.dev/)
- [gRPC C++ Documentation](https://grpc.io/docs/languages/cpp/)
- [vcpkg Documentation](https://vcpkg.io/)
- [NuGet Package](https://www.nuget.org/packages/Vcpkg.Grpc.Tools.Cpp/)
- [GitHub Repository](https://github.com/MartinKuschnik/Vcpkg.Grpc.Tools.Cpp)

---

## ?? Support

For issues and feature requests, please use the [GitHub Issues](https://github.com/MartinKuschnik/Vcpkg.Grpc.Tools.Cpp/issues) page.

---

## ?? Why This Package?

While vcpkg provides the protobuf and gRPC libraries, integrating proto file compilation into your build process can be tedious. This package:

- Eliminates manual `protoc` command execution
- Provides Visual Studio UI integration for configuration
- Handles incremental builds automatically
- Works seamlessly with vcpkg's dependency management
- Reduces boilerplate in your project files

Just install the package, add your `.proto` files, and build! ??
