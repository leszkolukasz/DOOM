# Build System and Compilation

DOOM uses a straightforward build system designed for simplicity and portability across Unix-like systems. The build process compiles the game from C source code using GCC and links against X11 libraries for graphics support.

## Build System Overview

### Build Requirements
- **Compiler**: GCC (GNU Compiler Collection)
- **Platform**: Linux/Unix with X11 Window System
- **Libraries**: X11 core libraries and extensions
- **Make**: GNU Make for build automation

### Build Configuration (`Makefile`)
The main Makefile defines the complete build process:

```makefile
################################################################
#
# DOOM Linux Makefile
#
################################################################

CC = gcc  # Compiler (gcc or g++)

# Compiler flags
CFLAGS = -g -Wall -DNORMALUNIX -DLINUX # -DUSEASM 
LDFLAGS = -L/usr/X11R6/lib
LIBS = -lXext -lX11 -lnsl -lm

# Build directory
O = linux

# Object file dependencies
OBJS = \
    $(O)/doomdef.o      \
    $(O)/doomstat.o     \
    $(O)/dstrings.o     \
    $(O)/i_system.o     \
    $(O)/i_sound.o      \
    $(O)/i_video.o      \
    $(O)/i_net.o        \
    $(O)/tables.o       \
    # ... (complete object list)
    $(O)/sounds.o

all: $(O)/linuxxdoom

clean:
    rm -f *.o *~ *.flc
    rm -f linux/*

$(O)/linuxxdoom: $(OBJS) $(O)/i_main.o
    $(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) $(O)/i_main.o \
    -o $(O)/linuxxdoom $(LIBS)

$(O)/%.o: %.c
    $(CC) $(CFLAGS) -c $< -o $@
```

## Compilation Flags and Configuration

### Preprocessor Definitions
The build system uses several preprocessor flags to configure compilation:

```c
// Platform identification
#define NORMALUNIX      // Standard Unix/Linux build
#define LINUX           // Linux-specific features

// Optional features (commented out by default)
// #define USEASM       // Enable assembly optimizations
```

### Compiler Flags Explained
```bash
-g                      # Include debugging information
-Wall                   # Enable all common warnings
-DNORMALUNIX           # Define NORMALUNIX macro
-DLINUX                # Define LINUX macro
```

### Linker Configuration
```bash
-L/usr/X11R6/lib       # X11 library directory
-lXext                 # X11 extensions library
-lX11                  # Core X11 library  
-lnsl                  # Network services library
-lm                    # Math library
```

## Source File Organization

### Build Dependencies
The build system automatically handles dependencies between source files:

```makefile
# Core engine objects
CORE_OBJS = \
    $(O)/doomdef.o      # Global definitions
    $(O)/doomstat.o     # Global state variables
    $(O)/tables.o       # Mathematical lookup tables

# System interface layer
SYSTEM_OBJS = \
    $(O)/i_main.o       # Program entry point
    $(O)/i_system.o     # System services
    $(O)/i_video.o      # Graphics interface
    $(O)/i_sound.o      # Audio interface
    $(O)/i_net.o        # Network interface

# Rendering system
RENDER_OBJS = \
    $(O)/r_main.o       # Rendering controller
    $(O)/r_draw.o       # Low-level drawing
    $(O)/r_plane.o      # Floor/ceiling rendering
    $(O)/r_segs.o       # Wall rendering
    $(O)/r_things.o     # Sprite rendering
    $(O)/r_bsp.o        # BSP tree traversal
    $(O)/r_data.o       # Texture management
    $(O)/r_sky.o        # Sky rendering

# Game logic and physics
GAME_OBJS = \
    $(O)/g_game.o       # Main game controller
    $(O)/p_mobj.o       # Map objects
    $(O)/p_map.o        # Collision detection
    $(O)/p_setup.o      # Level loading
    $(O)/p_spec.o       # Special effects
    # ... more physics objects

# Complete object list
OBJS = $(CORE_OBJS) $(SYSTEM_OBJS) $(RENDER_OBJS) $(GAME_OBJS) # ... etc
```

## Platform-Specific Configuration

### Linux/X11 Graphics Support
The build system configures X11 graphics support:

```c
// From i_video.c - X11 includes
#include <X11/Xlib.h>
#include <X11/Xutil.h>
#include <X11/keysym.h>
#include <X11/extensions/XShm.h>

#ifdef LINUX
// Linux-specific X11 extensions
int XShmGetEventBase(Display* dpy);
#endif
```

### Network Support Configuration
```c
// Network library configuration
#include <sys/socket.h>
#include <netinet/in.h>

// Linux-specific networking
#ifdef LINUX
#include <sys/types.h>
#include <unistd.h>
#endif
```

## Build Process Details

### Compilation Steps
1. **Preprocessing**: Expand macros and includes
2. **Compilation**: Generate object files (.o) from C source (.c)
3. **Linking**: Combine object files into executable

### Build Example
```bash
# Create build directory
mkdir -p linux

# Compile source files to object files
gcc -g -Wall -DNORMALUNIX -DLINUX -c doomdef.c -o linux/doomdef.o
gcc -g -Wall -DNORMALUNIX -DLINUX -c i_main.c -o linux/i_main.o
# ... compile all source files

# Link final executable
gcc -g -Wall -L/usr/X11R6/lib linux/*.o -o linux/linuxxdoom \
    -lXext -lX11 -lnsl -lm
```

### Build Output
The build process creates:
- **Object files**: `linux/*.o` - Compiled but not linked code
- **Executable**: `linux/linuxxdoom` - Final game executable

## Advanced Build Features

### Assembly Optimization Support
The build system supports optional assembly optimizations:

```makefile
# Uncomment to enable assembly optimizations
# CFLAGS += -DUSEASM

# Assembly source files (when enabled)
ASM_OBJS = \
    $(O)/tmap.o         # Texture mapping assembly
    $(O)/math.o         # Fixed-point math assembly
```

### Debug vs Release Builds
Different build configurations for development vs distribution:

```makefile
# Debug build (default)
CFLAGS_DEBUG = -g -Wall -DDEBUG -DNORMALUNIX -DLINUX

# Release build (optimized)
CFLAGS_RELEASE = -O2 -DNDEBUG -DNORMALUNIX -DLINUX

# Select build type
CFLAGS = $(CFLAGS_DEBUG)  # or $(CFLAGS_RELEASE)
```

### Build Variants
Different executable configurations:

```makefile
# Standard game build
$(O)/linuxxdoom: $(OBJS) $(O)/i_main.o
    $(CC) $(CFLAGS) $(LDFLAGS) $^ -o $@ $(LIBS)

# Development/debugging build
$(O)/linuxxdoom-debug: $(OBJS) $(O)/i_main.o
    $(CC) $(CFLAGS) -DDEBUG $(LDFLAGS) $^ -o $@ $(LIBS)

# Dedicated server build (hypothetical)
$(O)/linuxxdoom-server: $(SERVER_OBJS)
    $(CC) $(CFLAGS) -DDEDICATED $(LDFLAGS) $^ -o $@ $(SERVER_LIBS)
```

## Dependency Management

### Header Dependencies
The build system relies on implicit header dependencies:

```c
// Common header inclusions create build dependencies
#include "doomdef.h"        // Core definitions
#include "doomstat.h"       // Global state
#include "r_local.h"        // Rendering internals
#include "p_local.h"        // Physics internals
#include "i_system.h"       // System interface
```

### Automatic Dependency Generation
Advanced dependency tracking (not used by default):

```makefile
# Generate dependency files
%.d: %.c
    @$(CC) -M $(CFLAGS) $< > $@.tmp; \
    sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.tmp > $@; \
    rm -f $@.tmp

# Include generated dependencies
-include $(OBJS:.o=.d)
```

## Build Optimization

### Parallel Compilation
The build system supports parallel compilation:

```bash
# Build with multiple processors
make -j4              # Use 4 parallel jobs
make -j$(nproc)       # Use all available processors
```

### Incremental Builds
Only recompile changed files:

```bash
make                  # Only rebuild changed files
make clean && make    # Full rebuild from scratch
```

### Build Performance Tips
```bash
# Fast incremental builds
make -j$(nproc)

# Clean specific targets
rm linux/r_*.o && make  # Rebuild only rendering system

# Build specific targets
make linux/linuxxdoom   # Build only the executable
```

## Cross-Platform Considerations

### Portability Features
The build system supports adaptation to different Unix systems:

```c
// Platform detection
#ifdef LINUX
    // Linux-specific code
#elif defined(__FreeBSD__)
    // FreeBSD-specific code  
#elif defined(__sun)
    // Solaris-specific code
#else
    // Generic Unix code
#endif
```

### Library Path Configuration
Different systems may require different library paths:

```makefile
# Linux (standard)
LDFLAGS = -L/usr/X11R6/lib

# FreeBSD
# LDFLAGS = -L/usr/local/lib

# Solaris
# LDFLAGS = -L/usr/openwin/lib
```

### Compiler Alternatives
Support for different compilers:

```makefile
# GCC (default)
CC = gcc

# Alternative compilers
# CC = clang        # Modern LLVM compiler
# CC = icc          # Intel compiler
# CC = cc           # System default compiler
```

## Installation and Distribution

### Installation Process
```bash
# Build the game
make clean && make

# Install to system directories
sudo cp linux/linuxxdoom /usr/local/bin/
sudo chmod +x /usr/local/bin/linuxxdoom

# Create desktop entry
cat > ~/.local/share/applications/doom.desktop << EOF
[Desktop Entry]
Name=DOOM
Exec=linuxxdoom
Type=Application
Categories=Game;
EOF
```

### Distribution Packaging
The build system supports packaging for different distributions:

```bash
# Create binary distribution
tar -czf doom-linux-$(date +%Y%m%d).tar.gz linux/linuxxdoom README.TXT

# Debian packaging (hypothetical)
dpkg-buildpackage -b -uc

# RPM packaging (hypothetical)  
rpmbuild -bb doom.spec
```

The DOOM build system exemplifies the Unix philosophy of simplicity and modularity. Its straightforward Makefile approach makes the build process transparent and easily customizable while maintaining compatibility across different Unix-like systems.