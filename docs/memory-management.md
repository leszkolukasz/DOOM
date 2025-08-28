# Memory Management System

DOOM implements a sophisticated custom memory management system called "Zone Memory" that provides efficient allocation, automatic cleanup, and memory debugging capabilities. This system was crucial for managing limited memory resources on 1990s hardware while maintaining high performance.

## Zone Memory Architecture

### Design Philosophy
The zone memory system addresses several key challenges:
- **Memory fragmentation** - Minimize fragmentation through intelligent allocation
- **Automatic cleanup** - Group-based memory freeing for different game states  
- **Debugging support** - Built-in heap validation and memory leak detection
- **Performance** - Fast allocation/deallocation optimized for game objects

### Core Components (`z_zone.c`, `z_zone.h`)
```c
// Memory block header structure  
typedef struct memblock_s {
    int         size;       // Block size including header
    void**      user;       // Pointer to user's pointer (NULL if free)
    int         tag;        // Memory tag for group operations
    int         id;         // Magic number for validation (0x1d4a11)
    struct memblock_s* next;   // Next block in linked list
    struct memblock_s* prev;   // Previous block in linked list
} memblock_t;
```

## Memory Tags and Lifecycle Management

### Tag-Based Memory Organization
Memory is organized by purpose using hierarchical tags:

```c
// Static memory - never freed automatically
#define PU_STATIC       1       // Permanent for entire execution
#define PU_SOUND        2       // Static while playing sound
#define PU_MUSIC        3       // Static while playing music
#define PU_DAVE         4       // Programmer-specific static memory

// Level-specific memory - freed when leaving level
#define PU_LEVEL        50      // Static until level exit
#define PU_LEVSPEC      51      // Special level thinkers

// Purgeable memory - can be freed when needed
#define PU_PURGELEVEL   100     // Purgeable level data
#define PU_CACHE        101     // Cache memory (textures, sounds)
```

### Memory Allocation with Tags
```c
// Allocate memory with specific tag and user pointer
void* Z_Malloc(int size, int tag, void* user)
{
    memblock_t* block;
    
    // Align size to 8-byte boundary for performance
    size = (size + 7) & ~7;
    size += sizeof(memblock_t);  // Add header size
    
    // Try to find suitable free block
    block = Z_FindFreeBlock(size);
    if (!block) {
        // Purge cache memory to make room
        Z_FreeTags(PU_PURGELEVEL, PU_CACHE);
        block = Z_FindFreeBlock(size);
        if (!block)
            I_Error("Z_Malloc: failed on allocation of %i bytes", size);
    }
    
    // Initialize block
    block->size = size;
    block->user = (void**)user;
    block->tag = tag;
    block->id = ZONEID;  // Magic number for validation
    
    // Set user pointer to point to data area
    if (user)
        *(void**)user = (byte*)block + sizeof(memblock_t);
        
    return (byte*)block + sizeof(memblock_t);
}
```

### Automatic Memory Cleanup
```c
// Free all memory blocks with tags in specified range
void Z_FreeTags(int lowtag, int hightag)
{
    memblock_t* block = mainzone->blocklist.next;
    
    while (block != &mainzone->blocklist) {
        memblock_t* next = block->next;
        
        if (block->tag >= lowtag && block->tag <= hightag) {
            if (block->user)
                *(block->user) = NULL;  // Clear user pointer
            Z_Free((byte*)block + sizeof(memblock_t));
        }
        block = next;
    }
}
```

## Advanced Memory Features

### Smart Pointer System
Zone memory implements automatic pointer management:

```c
// Change memory tag (with validation)
#define Z_ChangeTag(p,t) \
{ \
    if (((memblock_t*)((byte*)(p) - sizeof(memblock_t)))->id != ZONEID) \
        I_Error("Z_CT at "__FILE__":%i", __LINE__); \
    Z_ChangeTag2(p,t); \
}

void Z_ChangeTag2(void* ptr, int tag)
{
    memblock_t* block = (memblock_t*)((byte*)ptr - sizeof(memblock_t));
    
    if (block->id != ZONEID)
        I_Error("Z_ChangeTag: block without a ZONEID");
        
    block->tag = tag;
}
```

### Memory Debugging and Validation
```c
// Comprehensive heap validation
void Z_CheckHeap(void)
{
    memblock_t* block;
    
    for (block = mainzone->blocklist.next; 
         block != &mainzone->blocklist;
         block = block->next) {
        
        // Validate magic number
        if (block->id != ZONEID)
            I_Error("Z_CheckHeap: block doesn't have ZONEID");
            
        // Validate linked list integrity  
        if (block->next->prev != block)
            I_Error("Z_CheckHeap: next block doesn't have proper back link");
            
        // Validate user pointer consistency
        if (block->user) {
            if (*(block->user) != (byte*)block + sizeof(memblock_t))
                I_Error("Z_CheckHeap: user pointer corrupted");
        }
    }
}
```

### Memory Statistics and Monitoring
```c
int Z_FreeMemory(void)
{
    memblock_t* block;
    int free = 0;
    
    for (block = mainzone->blocklist.next;
         block != &mainzone->blocklist;
         block = block->next) {
        if (!block->user || block->tag >= PU_PURGELEVEL)
            free += block->size;
    }
    return free;
}

void Z_DumpHeap(int lowtag, int hightag)
{
    memblock_t* block;
    
    printf("zone size: %i  location: %p\n", mainzone->size, mainzone);
    printf("tag range: %i to %i\n", lowtag, hightag);
    
    for (block = mainzone->blocklist.next;;block = block->next) {
        if (block->tag >= lowtag && block->tag <= hightag)
            printf("block:%p    size:%7i    user:%p    tag:%3i\n",
                   block, block->size, block->user, block->tag);
    }
}
```

## Memory Usage Patterns

### Level Loading Memory Management
```c
// Typical level loading sequence
void P_SetupLevel(int episode, int map, int playermask, skill_t skill)
{
    // Free all level-specific memory
    Z_FreeTags(PU_LEVEL, PU_LEVSPEC);
    
    // Load new level geometry (tagged PU_LEVEL)
    P_LoadVertexes(lumpnum);      // Vertex data
    P_LoadSegs(lumpnum);          // Line segments  
    P_LoadSubsectors(lumpnum);    // BSP leaf nodes
    P_LoadNodes(lumpnum);         // BSP tree nodes
    P_LoadSectors(lumpnum);       // Sector definitions
    
    // Initialize dynamic objects (tagged PU_LEVSPEC) 
    P_SpawnThings(playermask);    // Spawn map objects
    P_SpawnSpecials();            // Special effects
    
    Z_CheckHeap();  // Validate memory integrity
}
```

### Texture and Resource Caching
```c
// Cache system for frequently accessed data
patch_t* W_CacheLumpNum(int lump, int tag)
{
    lumpcache[lump] = Z_Malloc(W_LumpLength(lump), tag, &lumpcache[lump]);
    W_ReadLump(lump, lumpcache[lump]);
    return lumpcache[lump];
}

// Automatic cache purging when memory is low
void* Z_Malloc(int size, int tag, void* user)
{
    // ... allocation code ...
    
    if (!block) {
        // Free cache memory to make room
        Z_FreeTags(PU_CACHE, PU_CACHE);
        block = Z_FindFreeBlock(size);
        
        if (!block) {
            // Desperate - free purgeable level data
            Z_FreeTags(PU_PURGELEVEL, PU_PURGELEVEL);  
            block = Z_FindFreeBlock(size);
        }
    }
}
```

### Sound and Music Memory Management
```c
// Sound effect loading with proper tagging
void S_LoadSound(sfxinfo_t* sfx)
{
    char namebuf[9];
    byte* data;
    
    sprintf(namebuf, "ds%s", sfx->name);
    
    // Cache sound with appropriate tag
    data = W_CacheLumpName(namebuf, PU_SOUND);
    
    // Convert sound format and store
    sfx->data = (void*)data;
    sfx->lumpnum = W_GetNumForName(namebuf);
}

// Music memory is handled separately
void S_ChangeMusic(int musicnum, int looping)
{
    // Free previous music
    if (mus_playing && mus_playing->data)
        Z_ChangeTag(mus_playing->data, PU_CACHE);
        
    // Load new music  
    music = &S_music[musicnum];
    music->data = W_CacheLumpNum(music->lumpnum, PU_MUSIC);
}
```

## File I/O and WAD Management (`w_wad.c`)

### WAD File Structure
DOOM uses WAD (Where's All the Data) files for resource storage:

```c
// WAD file header
typedef struct {
    char identification[4];     // "IWAD" or "PWAD"  
    int  numlumps;             // Number of resources
    int  infotableofs;         // Offset to directory
} wadinfo_t;

// Resource directory entry
typedef struct {
    int  filepos;              // File position
    int  size;                 // Resource size  
    char name[8];              // Resource name (8 chars max)
} filelump_t;
```

### Efficient Resource Loading
```c
// Initialize WAD file system
void W_InitMultipleFiles(char** filenames)
{
    // Open all WAD files
    for (i = 0; filenames[i]; i++) {
        AddFile(filenames[i]);
    }
    
    // Build resource hash table for fast lookup
    W_GenerateHashTable();
    
    // Allocate lump cache array
    lumpcache = Z_Malloc(numlumps * sizeof(void*), PU_STATIC, NULL);
    memset(lumpcache, 0, numlumps * sizeof(void*));
}

// Fast resource name lookup  
int W_GetNumForName(char* name)
{
    int hash = W_HashName(name);
    
    // Search hash chain
    for (i = hash_table[hash]; i != -1; i = hash_next[i]) {
        if (!strncasecmp(lumpinfo[i].name, name, 8))
            return i;
    }
    
    I_Error("W_GetNumForName: %s not found!", name);
    return -1;
}
```

### Memory-Mapped File Access
```c
// Read resource data with caching
void W_ReadLump(int lump, void* dest)
{
    lumpinfo_t* l = lumpinfo + lump;
    
    if (l->handle == -1) {
        // External file - read from disk
        int handle = open(l->name, O_RDONLY | O_BINARY);
        read(handle, dest, l->size);
        close(handle);
    } else {
        // WAD file - seek and read
        lseek(l->handle, l->position, SEEK_SET);
        read(l->handle, dest, l->size);
    }
}
```

## Performance Optimizations

### Memory Pool Allocation
```c
// Pre-allocate memory pools for common objects
#define MAXMOBJS        1024
mobj_t      mobjpool[MAXMOBJS];     // Pre-allocated mobj pool
thinker_t   thinkerlist;            // Active thinker list

// Fast object allocation from pool
mobj_t* P_SpawnMobj(fixed_t x, fixed_t y, fixed_t z, mobjtype_t type)
{
    mobj_t* mobj = Z_Malloc(sizeof(*mobj), PU_LEVEL, NULL);
    
    // Initialize from static data
    *mobj = default_mobj;
    mobj->type = type;
    mobj->info = &mobjinfo[type];
    
    return mobj;
}
```

### Cache-Friendly Data Layout
```c
// Structure padding for cache alignment
typedef struct {
    // Hot data first (frequently accessed)
    fixed_t x, y, z;              // Position (12 bytes)
    fixed_t momx, momy, momz;      // Momentum (12 bytes)
    
    // Warm data  
    int health;                    // Health points
    int flags;                     // Behavior flags
    
    // Cold data last (rarely accessed)
    char* name;                    // Debug name
    int spawntime;                 // Creation time
} mobj_t __attribute__((aligned(32)));  // Cache line alignment
```

The zone memory system provides a robust foundation for DOOM's memory management needs. Its tag-based organization, automatic cleanup, and debugging features make it both powerful and reliable, while the performance optimizations ensure minimal impact on the game's real-time requirements.