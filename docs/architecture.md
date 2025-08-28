# Architecture Overview

DOOM represents a masterpiece of software architecture, implementing a sophisticated 3D game engine on 1990s hardware through innovative design patterns and optimization techniques. This document provides a high-level overview of the system's architecture and core design principles.

## Core Architectural Principles

### 1. Layered Architecture
DOOM follows a clean layered architecture separating concerns:

```
┌─────────────────────────────────────────────────┐
│                Application Layer                │
│              (Game Logic & Rules)               │
├─────────────────────────────────────────────────┤
│               Presentation Layer                │
│         (Rendering, Audio, Input, UI)          │
├─────────────────────────────────────────────────┤
│                 Engine Layer                    │
│        (Physics, AI, Memory, Networking)       │
├─────────────────────────────────────────────────┤
│                Platform Layer                   │
│            (OS Interface, Hardware)             │
└─────────────────────────────────────────────────┘
```

### 2. Component-Based Design
Each major system is designed as an independent component with well-defined interfaces:

- **Rendering System** - Graphics pipeline and visual effects
- **Physics Engine** - Collision detection and movement simulation
- **Audio System** - 3D positional sound and music
- **Input Manager** - Event-driven input processing
- **Memory Manager** - Zone-based allocation system
- **File System** - WAD resource management

### 3. Data-Driven Architecture
Game content is separated from code through data files:

- **WAD Files** - Store all game assets (levels, textures, sounds, music)
- **Lump System** - Named resource management
- **State Machines** - Data-driven object behavior
- **Level Scripts** - Map-based special effects and triggers

## System Architecture Diagram

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   Input System   │    │  Rendering Sys   │    │   Audio System   │
│                  │    │                  │    │                  │
│ • Keyboard/Mouse │    │ • BSP Renderer   │    │ • 3D Positioning │
│ • Event Queue    │    │ • Texture Maps   │    │ • Sound Effects  │
│ • Command Build  │    │ • Sprite System  │    │ • Music Playback │
└────────┬─────────┘    └─────────┬────────┘    └─────────┬────────┘
         │                        │                       │
         ▼                        ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Game Engine Core                          │
│                                                                     │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐│
│  │   Physics Eng   │  │   Memory Manager │  │   Resource Manager  ││
│  │                 │  │                  │  │                     ││
│  │ • Collision     │  │ • Zone Alloc     │  │ • WAD Files         ││
│  │ • Movement      │  │ • Tag System     │  │ • Lump Cache        ││
│  │ • Line of Sight │  │ • Garbage Coll   │  │ • Asset Loading     ││
│  └─────────────────┘  └──────────────────┘  └─────────────────────┘│
│                                                                     │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐│
│  │   Game Logic    │  │    AI System     │  │   Network System    ││
│  │                 │  │                  │  │                     ││
│  │ • Player Actions│  │ • State Machines │  │ • Multiplayer       ││
│  │ • Weapon System │  │ • Pathfinding    │  │ • Demo Record       ││
│  │ • Interactions  │  │ • Combat Logic   │  │ • Synchronization   ││
│  └─────────────────┘  └──────────────────┘  └─────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Platform Abstraction                        │
│                                                                     │
│    ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐    │
│    │   Video     │    │    Audio     │    │    System        │    │
│    │ Interface   │    │  Interface   │    │   Interface      │    │
│    │             │    │              │    │                  │    │
│    │ • X11/Linux │    │ • Sound Srv  │    │ • File I/O       │    │
│    │ • Framebuf  │    │ • Audio Mix  │    │ • Timing         │    │
│    │ • Palette   │    │ • Music      │    │ • Error Handle   │    │
│    └─────────────┘    └──────────────┘    └──────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

## Design Patterns and Techniques

### 1. State Machine Pattern
Extensive use of finite state machines for object behavior:

```c
// Object states drive behavior and animation
typedef struct state_s {
    spritenum_t sprite;         // Visual representation
    int         frame;          // Animation frame
    int         tics;           // Duration
    void        (*action)();    // Behavior function
    statenum_t  nextstate;      // Next state in sequence
} state_t;
```

### 2. Observer Pattern
Event-driven architecture for input and game events:

```c
// Event distribution system
boolean M_Responder(event_t* ev);    // Menu system
boolean G_Responder(event_t* ev);    // Game system
boolean HU_Responder(event_t* ev);   // HUD system
```

### 3. Strategy Pattern
Pluggable algorithms for different rendering and gameplay modes:

```c
// Function pointers for different rendering strategies
void (*colfunc)(void);        // Column drawing function
void (*basecolfunc)(void);    // Base column function
void (*fuzzcolfunc)(void);    // Translucent column function
void (*transcolfunc)(void);   // Colored translucent function
```

### 4. Factory Pattern
Object creation through centralized spawn functions:

```c
// Universal object creation
mobj_t* P_SpawnMobj(fixed_t x, fixed_t y, fixed_t z, mobjtype_t type);
mobj_t* P_SpawnMissile(mobj_t* source, mobj_t* dest, mobjtype_t type);
mobj_t* P_SpawnPuff(fixed_t x, fixed_t y, fixed_t z);
```

## Performance Architecture

### 1. Fixed-Point Mathematics
All calculations use integer-based fixed-point arithmetic:

```c
typedef int fixed_t;              // 16.16 fixed point
#define FRACBITS    16
#define FRACUNIT    (1<<FRACBITS) // 65536

// Fast fixed-point operations
fixed_t FixedMul(fixed_t a, fixed_t b);
fixed_t FixedDiv(fixed_t a, fixed_t b);
```

### 2. Lookup Tables
Pre-computed tables for expensive calculations:

```c
// Trigonometric tables
fixed_t finesine[5*FINEANGLES/4];     // Sine values
fixed_t finetangent[FINEANGLES/2];    // Tangent values

// Screen mapping tables
int     ylookup[SCREENHEIGHT];        // Screen row offsets
int     columnofs[SCREENWIDTH];       // Screen column offsets
```

### 3. Memory Pool Management
Custom zone allocation minimizes fragmentation:

```c
// Tag-based memory management
#define PU_STATIC   1    // Never freed
#define PU_LEVEL    50   // Freed on level change
#define PU_CACHE    101  // Freed when memory needed
```

### 4. Optimized Data Structures
Cache-friendly data layout and access patterns:

```c
// Hot data placed first in structures
typedef struct mobj_s {
    // Most frequently accessed data first
    fixed_t x, y, z;           // Position (cache-friendly)
    fixed_t momx, momy, momz;  // Momentum
    
    // Less frequently accessed data later
    int health;
    int flags;
    // ...
} mobj_t;
```

## Scalability and Modularity

### 1. Subsystem Independence
Each major system can be understood and modified independently:

- **Clear interfaces** between subsystems
- **Minimal coupling** reduces inter-dependencies  
- **Modular replacement** allows system upgrades

### 2. Data/Code Separation
Content creation separated from programming:

- **Designers** create levels without programming
- **Artists** create textures and sprites independently
- **Sound designers** work with audio tools

### 3. Platform Abstraction
Hardware-specific code isolated in interface layer:

```c
// Platform-independent interface
void I_StartSound(int id, int vol, int sep, int pitch, int priority);
void I_UpdateSound(void);
void I_FinishUpdate(void);

// Platform-specific implementation in i_*.c files
```

## Innovation and Technical Achievements

### 1. BSP Rendering Engine
Revolutionary spatial data structure for 3D rendering:

- **Automatic occlusion** - Hidden surfaces eliminated naturally
- **Optimal ordering** - Back-to-front rendering without Z-buffer
- **Efficient culling** - Only visible geometry processed

### 2. Network Architecture
Peer-to-peer networking with deterministic simulation:

- **Lockstep execution** - All players stay synchronized
- **Command distribution** - Input commands shared, not game state
- **Demo recording** - Perfect playback through command streams

### 3. Resource Management
Sophisticated WAD file system:

- **Virtual file system** - Resources accessed by name
- **Mod support** - Additional WADs override base content
- **Efficient loading** - Resources loaded on demand

### 4. Scalable Audio
3D positional audio system:

- **Distance attenuation** - Realistic sound falloff
- **Stereo positioning** - Directional audio cues
- **Priority management** - Important sounds never missed

## Legacy and Influence

DOOM's architecture established patterns still used today:

### Modern Game Engines
- **Entity-Component Systems** evolved from DOOM's mobj system
- **Scene Graphs** build upon BSP tree concepts
- **Resource Managers** follow WAD file principles
- **State Machines** remain standard for AI and animation

### Technical Innovations
- **Software Rendering** techniques influenced early 3D acceleration
- **Level Editors** democratized content creation
- **Modding Support** created entire communities
- **Network Gaming** established multiplayer paradigms

### Software Engineering
- **Clean Architecture** demonstrates separation of concerns
- **Performance Focus** shows optimization without premature complexity
- **Cross-Platform Design** enables wide hardware support
- **Open Source Release** allows continued study and improvement

The DOOM engine represents a perfect balance of innovation, performance, and maintainability. Its architectural principles remain relevant for modern software development, demonstrating that good design transcends technological limitations and stands the test of time.