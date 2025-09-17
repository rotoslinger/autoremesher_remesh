# AutoRemesher Command-Line Interface Implementation Plan

## Executive Summary

AutoRemesher is an excellent candidate for command-line interface implementation. The core remeshing functionality is well-architected and can be cleanly separated from the GUI components to create a standalone CLI tool while preserving all existing functionality.

## Repository Analysis

### Current Architecture

```
AutoRemesher/
├── src/
│   ├── AutoRemesher/           # Core remeshing library (KEEP)
│   │   ├── autoremesher.cpp/h  # Main remeshing class
│   │   ├── isotropicremesher.cpp/h
│   │   ├── vdbremesher.cpp/h
│   │   ├── parameterizer.cpp/h
│   │   ├── quadextractor.cpp/h
│   │   └── meshseparator.cpp/h
│   ├── main.cpp                # GUI entry point (REPLACE)
│   ├── mainwindow.cpp/h        # GUI components (REMOVE)
│   ├── graphics*.cpp/h         # Rendering (REMOVE)
│   ├── pbrshader*.cpp/h        # Shaders (REMOVE)
│   ├── quadmeshgenerator.cpp/h # Qt wrapper (REFACTOR)
│   └── tiny_obj_loader.h       # I/O utility (KEEP)
├── thirdparty/                 # Dependencies (KEEP)
└── autoremesher.pro           # Build system (REFACTOR)
```

### Core Dependencies Analysis

**Essential Dependencies (KEEP):**
- **Geogram**: Core mesh processing algorithms
- **libigl**: Geometry processing utilities
- **OpenVDB**: Volumetric remeshing
- **CGAL**: Computational geometry
- **TBB**: Parallel processing
- **Eigen**: Linear algebra
- **zlib**: Compression
- **blosc**: Fast compression

**GUI Dependencies (REMOVE/OPTIONAL):**
- **Qt5 Widgets**: GUI components
- **Qt5 OpenGL**: 3D rendering
- **Qt5 Network**: Update checking
- **QtAwesome**: Icons
- **QtWaitingSpinner**: UI components

## Build System Analysis

### Current Build Configuration (`autoremesher.pro`)

The current build system uses Qt's qmake with extensive third-party library integration:

```pro
QT += core widgets opengl network  # GUI dependencies
CONFIG += c++14
INCLUDEPATH += thirdparty/libigl/include
INCLUDEPATH += thirdparty/eigen
# ... extensive third-party library configuration
```

### Required Build System Changes

1. **Create Core Library Build**
   ```pro
   # autoremesher_core.pro
   QT += core  # Only core Qt, no widgets/opengl
   CONFIG += c++14
   CONFIG += staticlib  # Build as static library
   TARGET = autoremesher_core
   # Include only core AutoRemesher sources
   ```

2. **Create CLI Build**
   ```pro
   # autoremesher_cli.pro
   QT += core
   CONFIG += c++14
   TARGET = autoremesher_cli
   INCLUDEPATH += ../autoremesher_core
   LIBS += -L../autoremesher_core -lautoremesher_core
   # Include CLI-specific sources
   ```

## Implementation Strategy

### Phase 1: Core Library Extraction

#### 1.1 Create Core Library Project
```bash
# Create new directory structure
mkdir autoremesher_core
mkdir autoremesher_cli
```

#### 1.2 Extract Core Components
**Files to Move to `autoremesher_core/`:**
```
src/AutoRemesher/          # Entire directory
src/tiny_obj_loader.h      # I/O utility
thirdparty/                # All third-party libraries
```

**Core Library Interface (`autoremesher_core.h`):**
```cpp
namespace AutoRemesher {
    class CoreRemesher {
    public:
        // Constructor
        CoreRemesher(const std::vector<Vector3>& vertices,
                    const std::vector<std::vector<size_t>>& triangles);
        
        // Configuration
        void setTargetTriangleCount(size_t count);
        void setScaling(double scaling);
        void setModelType(ModelType type);
        
        // Processing
        bool remesh();
        
        // Results
        const std::vector<Vector3>& getRemeshedVertices() const;
        const std::vector<std::vector<size_t>>& getRemeshedQuads() const;
        
        // Progress reporting
        void setProgressCallback(std::function<void(float)> callback);
    };
    
    // I/O utilities
    bool loadObjFile(const std::string& filename,
                    std::vector<Vector3>& vertices,
                    std::vector<std::vector<size_t>>& triangles);
    
    bool saveObjFile(const std::string& filename,
                    const std::vector<Vector3>& vertices,
                    const std::vector<std::vector<size_t>>& faces);
}
```

#### 1.3 Refactor Qt Dependencies
**Current Issue:** Core classes use Qt types and signals
```cpp
// Current: QuadMeshGenerator inherits from QObject
class QuadMeshGenerator: public QObject {
    Q_OBJECT
signals:
    void reportProgress(float progress);
    void finished();
};
```

**Solution:** Create Qt-agnostic wrapper
```cpp
// New: Core class without Qt dependencies
class CoreQuadMeshGenerator {
public:
    void setProgressCallback(std::function<void(float)> callback);
    void setFinishedCallback(std::function<void()> callback);
    void process();
    // ... rest of functionality
};

// Optional: Qt wrapper for GUI compatibility
class QuadMeshGenerator: public QObject {
    Q_OBJECT
private:
    std::unique_ptr<CoreQuadMeshGenerator> m_core;
};
```

### Phase 2: CLI Implementation

#### 2.1 Create CLI Entry Point
**File: `autoremesher_cli/src/main.cpp`**
```cpp
#include <QCoreApplication>
#include <QCommandLineParser>
#include <QDebug>
#include "autoremesher_core.h"

int main(int argc, char *argv[]) {
    QCoreApplication app(argc, argv);
    
    QCommandLineParser parser;
    parser.setApplicationDescription("AutoRemesher Command Line Interface");
    parser.addHelpOption();
    parser.addVersionOption();
    
    // Define command line options
    parser.addPositionalArgument("input", "Input OBJ file");
    parser.addPositionalArgument("output", "Output OBJ file");
    
    QCommandLineOption densityOption("density", "Mesh density (0.0-1.0)", "density", "0.5");
    QCommandLineOption scalingOption("scaling", "Edge scaling factor", "scaling", "2.0");
    QCommandLineOption modelTypeOption("type", "Model type (organic|hardsurface)", "type", "organic");
    QCommandLineOption verboseOption("verbose", "Verbose output");
    
    parser.addOption(densityOption);
    parser.addOption(scalingOption);
    parser.addOption(modelTypeOption);
    parser.addOption(verboseOption);
    
    parser.process(app);
    
    // Process arguments and run remeshing
    // ... implementation details
}
```

#### 2.2 CLI Features Implementation
```cpp
class AutoRemesherCLI {
public:
    struct Options {
        std::string inputFile;
        std::string outputFile;
        float density = 0.5f;
        float scaling = 2.0f;
        AutoRemesher::ModelType modelType = AutoRemesher::ModelType::Organic;
        bool verbose = false;
    };
    
    int run(const Options& options);
    
private:
    void printProgress(float progress);
    void printStatistics(const AutoRemesher::CoreRemesher& remesher);
};
```

### Phase 3: File Format Extensions

#### 3.1 Add STL Support
```cpp
// File: autoremesher_core/src/stl_loader.h
namespace AutoRemesher {
    bool loadStlFile(const std::string& filename,
                    std::vector<Vector3>& vertices,
                    std::vector<std::vector<size_t>>& triangles);
    
    bool saveStlFile(const std::string& filename,
                    const std::vector<Vector3>& vertices,
                    const std::vector<std::vector<size_t>>& faces);
}
```

#### 3.2 Add PLY Support
```cpp
// File: autoremesher_core/src/ply_loader.h
namespace AutoRemesher {
    bool loadPlyFile(const std::string& filename,
                    std::vector<Vector3>& vertices,
                    std::vector<std::vector<size_t>>& triangles);
    
    bool savePlyFile(const std::string& filename,
                    const std::vector<Vector3>& vertices,
                    const std::vector<std::vector<size_t>>& faces);
}
```

## Safe Removal Analysis

### GUI Components (SAFE TO REMOVE)
```
src/mainwindow.cpp/h           # Main GUI window
src/graphics*.cpp/h            # 3D rendering widgets
src/pbrshader*.cpp/h           # Shader programs
src/rendermeshgenerator.cpp/h  # GUI-specific mesh generation
src/spinnableawesomebutton.cpp/h # UI buttons
src/floatnumberwidget.cpp/h    # UI controls
src/aboutwidget.cpp/h          # About dialog
src/updatescheck*.cpp/h        # Update checking
src/theme.cpp/h                # GUI theming
src/logbrowser*.cpp/h          # Debug dialog
src/ddsfile.cpp/h              # Texture loading
```

### Qt GUI Dependencies (SAFE TO REMOVE)
```pro
# From autoremesher.pro
QT -= widgets opengl network
# Remove these lines:
SOURCES += src/mainwindow.cpp
SOURCES += src/graphics*.cpp
SOURCES += src/pbrshader*.cpp
# ... all GUI-related sources
```

### Third-Party GUI Libraries (SAFE TO REMOVE)
```
thirdparty/QtAwesome/          # Icon fonts
thirdparty/QtWaitingSpinner/   # Loading spinners
```

### Core Components (MUST KEEP)
```
src/AutoRemesher/              # Entire core library
src/tiny_obj_loader.h          # OBJ file I/O
thirdparty/geogram/            # Core mesh processing
thirdparty/libigl/             # Geometry processing
thirdparty/openvdb/            # Volumetric remeshing
thirdparty/cgal/               # Computational geometry
thirdparty/eigen/              # Linear algebra
thirdparty/tbb/                # Parallel processing
thirdparty/zlib/               # Compression
thirdparty/blosc/              # Fast compression
```

## Build System Implementation

### Core Library Build (`autoremesher_core.pro`)
```pro
QT += core
CONFIG += c++14
CONFIG += staticlib
TARGET = autoremesher_core

# Core AutoRemesher sources
SOURCES += src/AutoRemesher/autoremesher.cpp
SOURCES += src/AutoRemesher/isotropicremesher.cpp
SOURCES += src/AutoRemesher/vdbremesher.cpp
SOURCES += src/AutoRemesher/parameterizer.cpp
SOURCES += src/AutoRemesher/quadextractor.cpp
SOURCES += src/AutoRemesher/meshseparator.cpp
SOURCES += src/AutoRemesher/relativeheight.cpp
SOURCES += src/AutoRemesher/positionkey.cpp

HEADERS += src/AutoRemesher/autoremesher.h
HEADERS += src/AutoRemesher/isotropicremesher.h
HEADERS += src/AutoRemesher/vdbremesher.h
HEADERS += src/AutoRemesher/parameterizer.h
HEADERS += src/AutoRemesher/quadextractor.h
HEADERS += src/AutoRemesher/meshseparator.h
HEADERS += src/AutoRemesher/relativeheight.h
HEADERS += src/AutoRemesher/positionkey.h
HEADERS += src/AutoRemesher/vector3.h
HEADERS += src/AutoRemesher/vector2.h
HEADERS += src/AutoRemesher/radians.h
HEADERS += src/AutoRemesher/double.h
HEADERS += src/tiny_obj_loader.h

# Third-party libraries (same as current)
INCLUDEPATH += thirdparty/libigl/include
INCLUDEPATH += thirdparty/eigen
# ... rest of third-party configuration
```

### CLI Build (`autoremesher_cli.pro`)
```pro
QT += core
CONFIG += c++14
TARGET = autoremesher_cli

# CLI sources
SOURCES += src/main.cpp
SOURCES += src/cli_processor.cpp

HEADERS += src/cli_processor.h

# Link against core library
INCLUDEPATH += ../autoremesher_core/src
LIBS += -L../autoremesher_core -lautoremesher_core

# Third-party libraries (inherit from core)
INCLUDEPATH += ../autoremesher_core/thirdparty/libigl/include
INCLUDEPATH += ../autoremesher_core/thirdparty/eigen
# ... rest of third-party configuration
```

## Implementation Timeline

### Week 1-2: Core Library Extraction
- [ ] Create `autoremesher_core` project structure
- [ ] Move core AutoRemesher classes
- [ ] Remove Qt dependencies from core classes
- [ ] Create core library interface
- [ ] Build and test core library

### Week 3-4: CLI Implementation
- [ ] Create CLI entry point
- [ ] Implement command-line argument parsing
- [ ] Create CLI processor class
- [ ] Add progress reporting to console
- [ ] Basic OBJ file processing

### Week 5-6: File Format Extensions
- [ ] Add STL file support
- [ ] Add PLY file support
- [ ] Add format auto-detection
- [ ] Test with various file formats

### Week 7-8: Polish and Testing
- [ ] Add comprehensive error handling
- [ ] Add batch processing capabilities
- [ ] Performance optimization
- [ ] Documentation and examples
- [ ] Cross-platform testing

## Risk Mitigation

### Preserving Core Functionality
1. **No Algorithm Changes**: Keep all remeshing algorithms unchanged
2. **Interface Compatibility**: Maintain same input/output data structures
3. **Testing**: Extensive testing against original GUI version
4. **Version Control**: Keep original code in separate branch

### Dependency Management
1. **Static Linking**: Build core as static library to avoid runtime dependencies
2. **Minimal Qt**: Use only `QCoreApplication` for CLI
3. **Third-Party Isolation**: Keep all third-party libraries in core

### Build System Safety
1. **Separate Projects**: Core and CLI as independent build targets
2. **Incremental Migration**: Test each component as it's extracted
3. **Fallback**: Maintain ability to build original GUI version

## Conclusion

This implementation plan provides a clean separation between the core AutoRemesher functionality and the command-line interface, ensuring that:

1. **Core functionality remains unchanged**
2. **CLI is built as a separate, lightweight application**
3. **Original GUI version can still be built**
4. **Third-party dependencies are properly managed**
5. **Build system is simplified and maintainable**

The approach minimizes risk while maximizing the benefits of having both GUI and CLI versions of AutoRemesher.

