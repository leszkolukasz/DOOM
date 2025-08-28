# Rendering System

DOOM's rendering system is one of the most innovative and influential graphics engines in gaming history. It uses a sophisticated combination of 2.5D techniques, BSP trees, and optimized rasterization to achieve real-time 3D-like graphics on 1990s hardware.

## Architecture Overview

The rendering system is built around these core concepts:

### 2.5D Rendering
- **Height-based 3D**: Walls have variable heights, but floors and ceilings are flat planes
- **No room-over-room**: Sectors cannot overlap vertically (originally)
- **Fixed vertical view**: Player cannot look up/down (camera always horizontal)
- **Perspective projection**: Walls use true perspective, floors/ceilings use affine mapping

### Binary Space Partitioning (BSP)
- **Scene organization**: Level geometry organized in BSP tree for efficient rendering
- **Back-to-front rendering**: BSP traversal ensures proper depth ordering
- **Frustum culling**: Only visible portions of the tree are processed
- **Automatic occlusion**: BSP structure naturally handles hidden surface removal

## Core Rendering Files

### Main Rendering Controller (`r_main.c`)
The central hub that coordinates the entire rendering process:

```c
void R_RenderPlayerView (player_t* player)
{
    R_SetupFrame (player);      // Set up view parameters
    R_ClearPlanes ();           // Clear floor/ceiling spans  
    R_ClearSprites ();          // Clear sprite list
    R_RenderBSPNode (numnodes-1); // Render BSP tree
    R_DrawPlanes ();            // Draw floors and ceilings
    R_DrawMasked ();            // Draw sprites and transparent walls
}
```

**Key Functions:**
- `R_InitTextureMapping()` - Initialize perspective transformation tables
- `R_SetupFrame()` - Configure view position, angle, and clipping
- `R_ScaleFromGlobalAngle()` - Calculate texture scaling for walls

### Low-Level Drawing (`r_draw.c`)
Optimized pixel-level rendering functions:

#### Column Rendering
```c
void R_DrawColumn (void)
{
    // Vertical texture mapping for walls
    dest = ylookup[dc_yl] + columnofs[dc_x];
    fracstep = dc_iscale; 
    frac = dc_texturemid + (dc_yl-centery)*fracstep;
    
    do {
        *dest = dc_colormap[dc_source[(frac>>FRACBITS)&127]];
        dest += SCREENWIDTH; 
        frac += fracstep;
    } while (count--);
}
```

#### Span Rendering  
```c
void R_DrawSpan (void)
{
    // Horizontal texture mapping for floors/ceilings
    // Steps through texture coordinates u,v
}
```

**Key Features:**
- **Texture mapping**: Affine mapping for spans, perspective for columns
- **Lighting**: Per-column light levels with distance attenuation
- **Color remapping**: Palette-based color translation for different light levels

### Wall Rendering (`r_segs.c`)
Handles wall segment rendering with texture mapping:

- **Texture alignment**: Proper texture coordinate calculation
- **Upper/lower texture handling**: For walls with height differences
- **Masked textures**: Windows, gratings with transparency
- **Sky rendering**: Special case for outdoor areas

### Floor/Ceiling Rendering (`r_plane.c`)
Renders horizontal surfaces using span-based algorithms:

```c
void R_MapPlane (int y, int x1, int x2)
{
    // Calculate texture coordinates for horizontal span
    distance = planeheight / yslope[y];
    ds_xstep = FixedMul (basexscale, distance);
    ds_ystep = FixedMul (baseyscale, distance);
    
    // Render the span
    R_DrawSpan ();
}
```

**Features:**
- **Flat textures**: 64x64 texture tiles repeated across surfaces  
- **Distance-based scaling**: Texture detail decreases with distance
- **Sky rendering**: Special infinite-distance sky texture

### Sprite Rendering (`r_things.c`)
Handles all sprite-based objects (enemies, items, decorations):

- **Billboarding**: Sprites always face the camera
- **Depth sorting**: Sprites rendered back-to-front for proper transparency
- **Clipping**: Sprites clipped by walls and other geometry
- **Animation**: Frame-based sprite animation system

### BSP Traversal (`r_bsp.c`)
Manages the Binary Space Partitioning tree traversal:

```c
void R_RenderBSPNode (int bspnum)
{
    if (bspnum & NF_SUBSECTOR) {
        R_Subsector (bspnum & (~NF_SUBSECTOR));
        return;
    }
    
    // Determine which side of BSP line camera is on
    side = R_PointOnSide (viewx, viewy, &bsp->line);
    
    // Render front side first, then back side
    R_RenderBSPNode (bsp->children[side]);
    R_RenderBSPNode (bsp->children[side^1]); 
}
```

## Graphics Libraries and Dependencies

### X11 Graphics System
DOOM uses X11 for Linux graphics output:

```c
// From i_video.c
#include <X11/Xlib.h>
#include <X11/Xutil.h>
#include <X11/keysym.h>
#include <X11/extensions/XShm.h>
```

**Key Features:**
- **Shared memory**: XShm extension for fast pixel buffer transfers
- **Palette management**: 8-bit color palette manipulation
- **Double buffering**: Smooth animation through buffer swapping
- **Input handling**: Keyboard and mouse event processing

### Video Buffer Management (`v_video.c`)
Manages multiple screen buffers for different purposes:

```c
byte* screens[5];  // Multiple screen buffers
int   dirtybox[4]; // Track dirty regions for optimization
byte  gammatable[5][256]; // Gamma correction lookup tables
```

**Screen Buffers:**
- **Screen 0**: Primary display buffer updated by `I_Update`
- **Screen 1**: Secondary buffer for effects and transitions  
- **Screen 2-4**: Additional buffers for menu overlays and special effects

## Mathematical Foundations

### Fixed-Point Arithmetic (`m_fixed.h`)
All rendering calculations use 32-bit fixed-point math:

```c
#define FRACBITS    16
#define FRACUNIT    (1<<FRACBITS)  // 65536
typedef int fixed_t;               // 16.16 fixed point

fixed_t FixedMul(fixed_t a, fixed_t b);  // Multiply with overflow protection
fixed_t FixedDiv(fixed_t a, fixed_t b);  // Divide with precision
```

**Advantages:**
- **Performance**: Much faster than floating-point on 1990s processors
- **Determinism**: Consistent results across different platforms
- **Precision**: Sufficient accuracy for game mathematics

### Trigonometric Tables (`tables.c`)
Pre-computed sine and tangent tables for performance:

```c
fixed_t finetangent[FINEANGLES/2];  // Tangent lookup table
fixed_t finesine[5*FINEANGLES/4];   // Sine lookup table

#define FINEANGLES  8192   // High precision angle resolution
#define ANGLETOFINESHIFT  19
```

## Rendering Pipeline

### 1. Setup Phase
- Initialize view parameters (position, angle, FOV)
- Clear rendering buffers and structures
- Set up clipping boundaries

### 2. BSP Traversal  
- Traverse BSP tree from back to front
- Cull invisible subsectors
- Render visible wall segments
- Build sprite and plane lists

### 3. Wall Rendering
- Draw wall columns with texture mapping
- Apply lighting and color translation
- Handle special wall types (transparent, animated)

### 4. Floor/Ceiling Rendering
- Render horizontal spans between wall segments
- Apply texture mapping with proper perspective correction
- Handle sky rendering for outdoor areas

### 5. Sprite Rendering
- Sort sprites by depth
- Draw sprites with proper clipping
- Handle transparency and special effects

### 6. Post-Processing
- Apply screen effects (palette shifts, flashing)
- Handle screen wipes and transitions
- Update display buffer

## Performance Optimizations

### Assembly Language Optimizations
Critical rendering loops written in hand-optimized assembly:

```assembly
// From README.asm - R_DrawColumn inner loop
_R_DrawColumn:
    movl    esi,[_dc_source]
    movl    ebx,[_dc_iscale]
    
patch1:
    movl    ebx,0x12345678  // Self-modifying code for speed
```

### Lookup Tables
Extensive use of precomputed tables:
- **ylookup[]**: Screen row offsets for fast pixel addressing  
- **columnofs[]**: Screen column offsets
- **viewangletox[]**: Angle to screen x coordinate mapping
- **xtoviewangle[]**: Screen x to view angle mapping

### Dirty Rectangle Tracking
Only update changed portions of the screen:
```c
extern int dirtybox[4];  // left, right, top, bottom
void V_MarkRect(int x, int y, int width, int height);
```

This rendering system achieves remarkable performance for its time through clever algorithms, mathematical optimizations, and careful attention to hardware limitations. The combination of BSP trees, fixed-point math, and optimized assembly code allows DOOM to render complex 3D environments at playable frame rates on 1990s PC hardware.