# ScreenCapture Plugin Architecture

## Overview

The ScreenCapture plugin is a WPEFramework-based Thunder plugin that provides screen capture functionality for RDK (Reference Design Kit) systems. It enables applications and services to capture screenshots of the current display output and upload them to remote servers or retrieve them for analysis and debugging purposes.

## System Architecture

### Component Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    WPEFramework/Thunder                    │
│                      (Plugin Host)                         │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                ScreenCapture Plugin                        │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   ScreenCapture │    │ ScreenCaptureImplementation    │ │
│  │   (Main Plugin) │◄──►│      (Core Logic)             │ │
│  └─────────────────┘    └─────────────────┬───────────────┘ │
└──────────────────────────────────────────┼─────────────────┘
                                           │
┌──────────────────────────────────────────▼─────────────────┐
│              Platform Abstraction Layer                   │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │  DRM/KMS Stack  │    │   Platform-Specific Capture    │ │
│  │   (Linux)       │    │       (Optional)               │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. ScreenCapture Plugin (Main Interface)
- **File**: `ScreenCapture.cpp`, `ScreenCapture.h`
- **Role**: Primary plugin interface that integrates with WPEFramework
- **Responsibilities**:
  - Plugin lifecycle management (Initialize/Deinitialize)
  - JSON-RPC interface exposure
  - Event notification handling
  - Service registration and discovery

#### 2. ScreenCaptureImplementation (Core Logic)
- **File**: `ScreenCaptureImplementation.cpp`, `ScreenCaptureImplementation.h`
- **Role**: Contains the business logic for screen capture operations
- **Responsibilities**:
  - Screen capture process coordination
  - Image processing and formatting
  - Upload mechanism handling
  - Configuration management
  - Asynchronous job management through ScreenShotJob

#### 3. Platform Abstraction Layer
- **DRM/KMS Implementation**: `Implementation/drm/`
  - `kms.cpp`, `kms.h`: Kernel Mode Setting interface
  - `drmsc.cpp`, `drmsc.h`: DRM-based screen capture implementation
- **Alternative**: Platform-specific capture via `screencaptureplatform.h`

## Data Flow

### Screen Capture Process

1. **Request Initiation**: Client sends JSON-RPC request via `uploadScreenCapture` method
2. **Job Scheduling**: Request is queued as a ScreenShotJob for asynchronous processing
3. **Display Access**: Platform layer accesses frame buffer through DRM/KMS or platform APIs
4. **Image Capture**: Raw frame buffer data is captured and converted to PNG format
5. **Upload Process**: Captured image is uploaded to specified URL via cURL
6. **Event Notification**: Upload completion status is sent back via event system

### Interface Integration

The plugin implements several WPEFramework interfaces:
- `PluginHost::IPlugin`: Core plugin lifecycle
- `PluginHost::IDispatcher`: JSON-RPC message dispatching
- `Exchange::IScreenCapture`: Screen capture specific operations
- `Exchange::IConfiguration`: Runtime configuration support

## Dependencies

### External Libraries
- **libdrm**: Direct Rendering Manager for low-level graphics access
- **libpng**: PNG image format encoding
- **libcurl**: HTTP/HTTPS upload functionality

### RDK Framework Integration
- **WPEFramework/Thunder**: Core plugin framework
- **IARMBus** (Optional): Inter-process communication
- **RFC (Runtime Feature Control)**: Configuration management

### Helper Utilities
- **UtilsLogging.h**: Standardized logging mechanisms
- **UtilsJsonRpc.h**: JSON-RPC utility functions
- **UtilsgetRFCConfig.h**: RFC configuration access

## Technical Implementation Details

### Threading Model
The plugin uses an asynchronous job-based architecture:
- Main thread handles JSON-RPC requests and plugin lifecycle
- Worker threads (via ScreenShotJob) handle actual capture and upload operations
- Thread-safe notification system for event propagation

### Memory Management
- RAII principles for resource management
- Smart pointer usage for automatic cleanup
- Efficient buffer management for large frame data

### Error Handling
- Comprehensive error checking at each layer
- Graceful degradation when platform features are unavailable
- Detailed error reporting through event system

### Configuration
- Runtime configuration through JSON configuration files
- RFC-based feature toggling support
- Platform-specific capability detection

## Platform Support

### Primary Platform: Linux with DRM/KMS
- Uses Direct Rendering Manager for frame buffer access
- Kernel Mode Setting for display information
- Supports hardware-accelerated capture where available

### Alternative Platforms
- Extensible design allows platform-specific implementations
- Interface-based design enables easy platform adaptation
- Conditional compilation for different capture mechanisms