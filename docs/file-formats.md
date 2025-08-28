# File Formats and Resource Management

DOOM revolutionized game asset management through its WAD (Where's All the Data) file system, which provides efficient resource storage, loading, and modding capabilities. This document details the file formats and resource management systems used throughout the engine.

## WAD File System

### WAD File Structure
WAD files are the primary container format for all DOOM assets:

```c
// WAD file header (12 bytes)
typedef struct {
    char identification[4];     // "IWAD" or "PWAD"
    int  numlumps;             // Number of resources
    int  infotableofs;         // Offset to directory table
} wadinfo_t;

// Resource directory entry (16 bytes)
typedef struct {
    int  filepos;              // File offset to resource data
    int  size;                 // Size in bytes
    char name[8];              // Resource name (uppercase, null-padded)
} filelump_t;
```

### WAD File Types

#### IWAD Files (Internal WAD)
Main game data files containing essential resources:
- `DOOM1.WAD` - DOOM shareware episode 1
- `DOOM.WAD` - DOOM registered episodes 1-3  
- `DOOM2.WAD` - DOOM II commercial release
- `PLUTONIA.WAD` - The Plutonia Experiment
- `TNT.WAD` - TNT: Evilution

#### PWAD Files (Patch WAD)
Modification files that override or add to IWAD resources:
- Custom levels and episodes
- New textures and sprites
- Replacement sounds and music
- Gameplay modifications

### Resource Naming Conventions

DOOM uses specific naming patterns for different resource types:

```c
// Level data (maps)
"E1M1"          // Episode 1, Map 1 (DOOM)
"MAP01"         // Map 01 (DOOM II)

// Textures
"WALL01_1"      // Wall texture
"FLOOR1_1"      // Floor texture  
"CEIL1_1"       // Ceiling texture

// Sprites (8-directional)
"TROO"          // Imp sprite base name
"TROOA0"        // Imp facing angle 0
"TROOB1"        // Imp facing angle 1, frame B

// Sounds
"DSPISTOL"      // Pistol sound effect
"DSSHOTGN"      // Shotgun sound effect

// Music
"D_E1M1"        // Episode 1 Map 1 music
"D_RUNNIN"      // "Running From Evil" track
```

## Level Data Format

### Map Structure
Each level consists of multiple lumps defining geometry and objects:

```c
// Level lump sequence (11 lumps per level)
"ExMy" or "MAPxx"    // Level marker
"THINGS"             // Object placement data
"LINEDEFS"           // Line definitions
"SIDEDEFS"           // Line side definitions  
"VERTEXES"           // Vertex coordinates
"SEGS"               // BSP segments
"SSECTORS"           // BSP subsectors
"NODES"              // BSP tree nodes
"SECTORS"            // Sector definitions
"REJECT"             // Line-of-sight table
"BLOCKMAP"           // Collision grid
```

### Map Data Structures

#### Vertex Data
```c
// VERTEXES lump - 2D coordinate points
typedef struct {
    short x;                   // X coordinate (map units)
    short y;                   // Y coordinate (map units)
} mapvertex_t;
```

#### Line Definitions  
```c
// LINEDEFS lump - Wall and boundary lines
typedef struct {
    short v1;                  // Start vertex index
    short v2;                  // End vertex index
    short flags;               // Line properties
    short special;             // Special effect type
    short tag;                 // Sector tag for triggers
    short sidenum[2];          // Side definition indices [right, left]
} maplinedef_t;

// Line flags
#define ML_BLOCKING        1   // Solid wall
#define ML_BLOCKMONSTERS   2   // Blocks monsters only
#define ML_TWOSIDED        4   // Two-sided line
#define ML_DONTPEGTOP      8   // Don't peg texture to top
#define ML_DONTPEGBOTTOM   16  // Don't peg texture to bottom
#define ML_SECRET          32  // Secret door/area
#define ML_SOUNDBLOCK      64  // Blocks sound
#define ML_DONTDRAW        128 // Invisible on automap
#define ML_MAPPED          256 // Already seen on automap
```

#### Side Definitions
```c
// SIDEDEFS lump - Line side texture information
typedef struct {
    short textureoffset;       // Horizontal texture offset
    short rowoffset;           // Vertical texture offset
    char  toptexture[8];       // Upper texture name
    char  bottomtexture[8];    // Lower texture name
    char  midtexture[8];       // Middle texture name
    short sector;              // Adjacent sector index
} mapsidedef_t;
```

#### Sector Definitions
```c
// SECTORS lump - Floor and ceiling areas
typedef struct {
    short floorheight;         // Floor height
    short ceilingheight;       // Ceiling height
    char  floorpic[8];         // Floor texture name
    char  ceilingpic[8];       // Ceiling texture name
    short lightlevel;          // Light level (0-255)
    short special;             // Special effect type
    short tag;                 // Tag for triggers
} mapsector_t;

// Sector special effects
#define LIGHT_PHASED       1   // Phased lighting
#define DAMAGE_HELLSLIME   5   // Hellslime damage
#define DAMAGE_NUKAGE      7   // Nukage damage
#define LIGHT_OSCILLATE    8   // Oscillating light
#define SECRET_AREA        9   // Secret area
#define DAMAGE_END         11  // Instant death
#define LIGHT_SYNC_SLOW    12  // Synchronized slow strobe
#define LIGHT_SYNC_FAST    13  // Synchronized fast strobe
#define DAMAGE_SUPER       16  // Super damage
```

#### Object Placement
```c
// THINGS lump - Monster, item, and player spawn points
typedef struct {
    short x;                   // X coordinate
    short y;                   // Y coordinate
    short angle;               // Facing angle (degrees)
    short type;                // Object type
    short options;             // Spawn options
} mapthing_t;

// Spawn options
#define MTF_EASY           1   // Spawn on easy skill
#define MTF_NORMAL         2   // Spawn on normal skill
#define MTF_HARD           4   // Spawn on hard skill
#define MTF_AMBUSH         8   // Deaf monster (ambush)
#define MTF_NOTSINGLE      16  // Not in single player
#define MTF_NOTDM          32  // Not in deathmatch
#define MTF_NOTCOOP        64  // Not in cooperative
```

### BSP Data Structures

#### BSP Nodes
```c
// NODES lump - Binary space partitioning tree
typedef struct {
    short x;                   // Partition line x coordinate
    short y;                   // Partition line y coordinate  
    short dx;                  // Partition line x delta
    short dy;                  // Partition line y delta
    short bbox[2][4];          // Bounding boxes [right/left][top/bottom/left/right]
    short children[2];         // Child node indices [right/left]
} mapnode_t;
```

#### BSP Subsectors
```c
// SSECTORS lump - BSP leaf nodes
typedef struct {
    short numsegs;             // Number of segments
    short firstseg;            // First segment index
} mapsubsector_t;
```

#### BSP Segments
```c
// SEGS lump - Line segments for rendering
typedef struct {
    short v1;                  // Start vertex
    short v2;                  // End vertex
    short angle;               // Angle (binary angle units)
    short linedef;             // Parent linedef index
    short side;                // Side of linedef (0=right, 1=left)
    short offset;              // Distance from linedef start
} mapseg_t;
```

## Texture and Graphics Formats

### Texture System

#### Texture Definitions
```c
// Texture composition structure
typedef struct {
    char    name[8];           // Texture name
    int     masked;            // Transparency flag
    short   width;             // Texture width
    short   height;            // Texture height
    int     columndirectory;   // Unused (legacy)
    short   patchcount;        // Number of patches
    texpatch_t patches[1];     // Patch array
} texture_t;

// Texture patch placement
typedef struct {
    short   originx;           // X offset in texture
    short   originy;           // Y offset in texture  
    short   patch;             // Patch index
    short   stepdir;           // Unused
    short   colormap;          // Unused
} texpatch_t;
```

#### Patch Format (Sprites and Textures)
```c
// Patch header (doom picture format)
typedef struct {
    short   width;             // Image width
    short   height;            // Image height
    short   leftoffset;        // Sprite center X offset
    short   topoffset;         // Sprite center Y offset
    int     columnofs[1];      // Column data offsets
} patch_t;

// Column data structure (run-length encoded)
// Each column contains posts of non-transparent pixels
typedef struct {
    byte    topdelta;          // Y position (255 = end of column)
    byte    length;            // Number of pixels in post
    byte    unused;            // Padding
    // Followed by 'length' pixel bytes
    // Then another unused padding byte
} post_t;
```

### Palette and Color System
```c
// Palette data (768 bytes)
typedef struct {
    byte rgb[256][3];          // 256 colors × RGB components
} palette_t;

// Colormap for lighting (256×32 bytes)  
typedef struct {
    byte colormap[32][256];    // 32 light levels × 256 colors
} colormap_t;
```

## Sound Format

### Sound Effect Format
```c
// Sound lump header
typedef struct {
    short   format;            // Format identifier (3)
    short   rate;              // Sample rate (typically 11025 Hz)
    int     length;            // Number of samples
    // Followed by 8-bit unsigned PCM samples
} dmx_sound_t;
```

### Music Format
DOOM uses MUS format for music, which is converted to MIDI:

```c
// MUS file header
typedef struct {
    char    id[4];             // "MUS\x1A"
    short   scorelength;       // Length of score data
    short   scorestart;        // Offset to score data
    short   primarychannels;   // Number of primary channels
    short   secondarychannels; // Number of secondary channels
    short   instrumentcount;   // Number of instruments
    short   dummy;             // Unused
} mus_header_t;
```

## Resource Loading and Management

### WAD Loading System
```c
// WAD initialization
void W_InitMultipleFiles(char** filenames)
{
    // Load all specified WAD files
    for (i = 0; filenames[i]; i++) {
        W_AddFile(filenames[i]);
    }
    
    // Build resource lookup tables
    W_GenerateHashTable();
    
    // Initialize lump cache
    lumpcache = Z_Malloc(numlumps * sizeof(void*), PU_STATIC, NULL);
    memset(lumpcache, 0, numlumps * sizeof(void*));
}
```

### Resource Lookup
```c
// Fast resource name lookup using hash table
int W_GetNumForName(char* name)
{
    char    uname[9];
    int     hash;
    
    // Convert to uppercase
    strupr(strcpy(uname, name));
    
    // Calculate hash
    hash = W_HashName(uname);
    
    // Search hash chain
    for (i = hash_table[hash]; i != -1; i = hash_next[i]) {
        if (!strcmp(lumpinfo[i].name, uname))
            return i;
    }
    
    I_Error("W_GetNumForName: %s not found!", name);
}
```

### Caching System
```c
// Resource caching with automatic memory management
void* W_CacheLumpNum(int lump, int tag)
{
    if (!lumpcache[lump]) {
        // Load and cache resource
        lumpcache[lump] = Z_Malloc(W_LumpLength(lump), tag, &lumpcache[lump]);
        W_ReadLump(lump, lumpcache[lump]);
    } else {
        // Update memory tag for cache management
        Z_ChangeTag(lumpcache[lump], tag);
    }
    
    return lumpcache[lump];
}
```

## Modding and Extensibility

### PWAD Override System
PWAD files can override any resource from IWAD files:

```c
// Resource search order: last loaded WAD has priority
void W_AddFile(char* filename)
{
    // Open WAD file
    handle = open(filename, O_RDONLY | O_BINARY);
    
    // Read header and directory
    read(handle, &header, sizeof(header));
    
    // Add lumps to global directory (overriding existing names)
    for (i = 0; i < header.numlumps; i++) {
        // Later lumps override earlier ones with same name
        W_AddLump(&fileinfo[i]);
    }
}
```

### Level Replacement
Custom levels completely replace original maps:

```c
// Level loading checks all loaded WADs
void P_SetupLevel(int episode, int map, int playermask, skill_t skill)
{
    char lumpname[9];
    
    if (gamemode == commercial) {
        sprintf(lumpname, "MAP%02d", map);
    } else {
        sprintf(lumpname, "E%dM%d", episode, map);
    }
    
    // Find level data (PWADs override IWADs)
    lumpnum = W_GetNumForName(lumpname);
    
    // Load all level lumps
    P_LoadVertexes(lumpnum + ML_VERTEXES);
    P_LoadSegs(lumpnum + ML_SEGS);
    // ... load all level data
}
```

### Asset Replacement
Any game asset can be replaced through WAD files:

- **Textures**: Replace wall, floor, ceiling graphics
- **Sprites**: Replace monster, weapon, item graphics  
- **Sounds**: Replace sound effects and music
- **Graphics**: Replace menu graphics and fonts

The WAD system's elegance lies in its simplicity - a straightforward file format that enables extensive modding capabilities while maintaining excellent performance. This design has influenced resource management systems in countless games and continues to be used in modern game engines.