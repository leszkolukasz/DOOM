# DOOM Source Code Documentation

Welcome to the DOOM source code documentation. This directory contains comprehensive documentation for the classic DOOM source code released by John Carmack and id Software.

## Documentation Structure

- [**Architecture Overview**](architecture.md) - High-level system architecture and design principles
- [**Project Structure**](project-structure.md) - Detailed breakdown of source code organization
- [**Rendering System**](rendering.md) - Graphics rendering pipeline and algorithms
- [**Input Handling**](input-handling.md) - Input processing and event management
- [**Game Logic**](game-logic.md) - Core game mechanics and physics simulation
- [**Memory Management**](memory-management.md) - Zone allocation and memory handling
- [**Sound System**](sound-system.md) - Audio processing and sound effects
- [**File Format**](file-formats.md) - WAD files and resource management
- [**Build System**](build-system.md) - Compilation and platform considerations

## Quick Start

The DOOM source code is written in C and originally designed for Linux systems using X11 for graphics. The codebase consists of approximately 124 source files organized into logical subsystems.

### Key Concepts

- **Fixed-Point Mathematics**: DOOM uses fixed-point arithmetic for performance
- **BSP Trees**: Binary Space Partitioning for efficient rendering
- **Zone Memory**: Custom memory management system
- **WAD Files**: Game data stored in "Where's All the Data" format
- **Column Rendering**: Wall textures rendered as vertical columns

### Building

```bash
cd linuxdoom-1.10
make
```

This will create the `linux/linuxxdoom` executable.

## Historical Context

This source code was released by John Carmack on December 23, 1997, under the GPL license. It represents one of the most influential game engines in history, pioneering many techniques still used in modern 3D graphics.

> "The basic rendering concept -- horizontal and vertical lines of constant Z with fixed light shading per band was dead-on" - John Carmack

## License

Released under the GNU General Public License 2.0.
Copyright (c) ZeniMax Media Inc.