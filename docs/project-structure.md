# Project Structure

The DOOM source code is organized into several logical subsystems, each handling a specific aspect of the game engine. This document provides a comprehensive breakdown of the codebase organization.

## Directory Structure

```
DOOM/
├── linuxdoom-1.10/          # Main source code directory
├── sersrc/                  # Serial communication code
├── sndserv/                 # Sound server code  
├── ipx/                     # IPX networking code
├── docs/                    # Documentation (this directory)
├── README.TXT               # Original readme from John Carmack
└── LICENSE.TXT              # GPL license
```

## Source Code Organization (linuxdoom-1.10/)

The main source code contains **124 source files** organized by functionality:

### System Interface Layer (`i_*` files)
Platform-specific code for Linux/X11 implementation:

- `i_main.c` - Program entry point and initialization
- `i_video.c` - X11 graphics interface and video management
- `i_sound.c` - Sound system interface
- `i_system.c` - System services (file I/O, timing, error handling)
- `i_net.c` - Network interface for multiplayer

### Rendering System (`r_*` files)
Core graphics rendering pipeline:

- `r_main.c` - Main rendering controller and view setup
- `r_draw.c` - Low-level drawing functions (columns, spans)
- `r_plane.c` - Floor and ceiling rendering
- `r_segs.c` - Wall segment rendering
- `r_things.c` - Sprite rendering (enemies, items, weapons)
- `r_bsp.c` - BSP tree traversal for rendering
- `r_data.c` - Texture and sprite data management
- `r_sky.c` - Sky rendering
- `r_defs.h` - Rendering data structures
- `r_local.h` - Rendering system private definitions
- `r_state.h` - Rendering state management

### Game Logic (`g_*` files)
Core game mechanics and control:

- `g_game.c` - Main game loop, save/load, demo playback

### Physics and World Simulation (`p_*` files)
Game world physics and entity management:

- `p_mobj.c` - Map objects (mobjs) - entities like players, enemies, items
- `p_map.c` - Line-of-sight, collision detection, movement
- `p_setup.c` - Level loading and BSP tree construction
- `p_spec.c` - Special effects (doors, lifts, lights)
- `p_enemy.c` - AI behavior for enemies
- `p_user.c` - Player movement and actions
- `p_inter.c` - Object interactions (picking up items, damage)
- `p_pspr.c` - Player sprite management (weapon rendering)
- `p_doors.c` - Door mechanics
- `p_floor.c` - Floor movement (lifts, stairs)
- `p_ceilng.c` - Ceiling movement
- `p_plats.c` - Platform mechanics
- `p_lights.c` - Dynamic lighting effects
- `p_switch.c` - Switch activation
- `p_telept.c` - Teleporter mechanics
- `p_tick.c` - Game simulation ticker
- `p_sight.c` - Line-of-sight calculations
- `p_maputl.c` - Map utility functions
- `p_saveg.c` - Save game functionality

### Menu and User Interface (`m_*`, `hu_*`, `st_*` files)
User interface and menu systems:

- `m_menu.c` - Main menu system
- `m_misc.c` - Configuration and settings
- `hu_stuff.c` - Heads-up display
- `hu_lib.c` - HUD library functions
- `st_stuff.c` - Status bar
- `st_lib.c` - Status bar library functions

### Video and Graphics (`v_*`, `f_*` files)
Video output and visual effects:

- `v_video.c` - Video buffer management and drawing functions
- `f_finale.c` - End-game finale screens
- `f_wipe.c` - Screen transition effects

### Sound System (`s_*` files)
Audio processing:

- `s_sound.c` - Sound effect management
- `sounds.c` - Sound definitions and data

### Data Management (`w_*`, `d_*` files)
File and data handling:

- `w_wad.c` - WAD file loading and management
- `d_main.c` - Main program initialization and event handling
- `d_net.c` - Network game management
- `d_items.c` - Item definitions
- `doomdata.h` - Core data structures
- `doomdef.h` - Global definitions and constants
- `doomstat.c` - Global game state variables
- `dstrings.c` - String definitions

### Memory Management (`z_*` files)
Custom memory allocation system:

- `z_zone.c` - Zone memory allocation system

### Utility and Support (`tables.c`, `info.c`, etc.)
Mathematical tables and game data:

- `tables.c` - Trigonometric and other mathematical lookup tables
- `info.c` - Game object definitions (weapons, enemies, etc.)
- `m_fixed.c` - Fixed-point arithmetic functions
- `m_random.c` - Random number generation
- `m_bbox.c` - Bounding box calculations
- `m_argv.c` - Command-line argument parsing
- `m_swap.c` - Byte order swapping
- `m_cheat.c` - Cheat code handling

### Automap (`am_*` files)
In-game map display:

- `am_map.c` - Automap functionality

### Intermission and Statistics (`wi_*` files)
Level completion screens:

- `wi_stuff.c` - Intermission screen management

## File Naming Conventions

The codebase follows a consistent naming convention:

- **Prefix indicates subsystem**: `r_` (rendering), `p_` (physics), `i_` (interface), etc.
- **Header files**: `.h` extension, contain structure definitions and function declarations
- **Implementation files**: `.c` extension, contain actual code implementation
- **Global definitions**: Files like `doomdef.h`, `doomstat.h` contain game-wide constants and variables

## Key Data Structures

- **`mobj_t`** - Map objects (players, enemies, items, projectiles)
- **`sector_t`** - Map sectors with floor/ceiling heights and properties  
- **`line_t`** - Map lines (walls, boundaries)
- **`vertex_t`** - Map vertices (points in 2D space)
- **`player_t`** - Player state and properties
- **`fixed_t`** - Fixed-point numbers (16.16 format)

## Build System

- **`Makefile`** - Simple makefile for GCC compilation
- **Target**: `linux/linuxxdoom` executable
- **Dependencies**: X11 libraries (`-lXext -lX11 -lnsl -lm`)
- **Compilation flags**: `-g -Wall -DNORMALUNIX -DLINUX`

This modular architecture allows for clean separation of concerns, making the codebase maintainable despite its complexity. Each subsystem can be understood and modified independently while maintaining well-defined interfaces between components.