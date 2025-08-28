# Sound System

DOOM's sound system provides immersive 3D positional audio, dynamic music playback, and efficient sound effect management. The system is designed to handle multiple simultaneous sound sources while maintaining performance on limited 1990s hardware.

## Architecture Overview

### Sound System Components
1. **Sound Interface** (`s_sound.c`) - High-level sound management
2. **Platform Audio** (`i_sound.c`) - Low-level audio hardware interface
3. **Sound Server** (`sndserv/`) - External sound mixing process
4. **Sound Definitions** (`sounds.c`) - Sound effect database
5. **Music System** - Background music playback and management

### Audio Pipeline
```
Game Events → Sound Manager → Platform Audio → Hardware
     ↓              ↓              ↓           ↓
  S_StartSound → I_StartSound → Sound Server → Audio Driver
```

## Core Sound Management (`s_sound.c`)

### Sound Channel System
DOOM manages multiple simultaneous sound channels:

```c
#define NUMSFX          64      // Maximum sound effects
#define NUMCHANNELS     8       // Simultaneous sound channels

// Sound channel structure
typedef struct {
    sfxinfo_t*  sfxinfo;        // Sound effect info
    int         origin_id;      // Source object ID
    int         volume;         // Channel volume (0-127)
    int         separation;     // Stereo separation (-1 to 1)
    int         priority;       // Channel priority
    int         singularity;    // Uniqueness flag
} channel_t;

channel_t channels[NUMCHANNELS];
```

### 3D Positional Audio
Sound positioning based on player location and orientation:

```c
void S_StartSound(void* origin_p, int sfx_id)
{
    mobj_t*     origin = (mobj_t*)origin_p;
    int         rc, sep, vol, priority;
    sfxinfo_t*  sfx;
    
    // Calculate 3D audio parameters
    if (origin && origin != players[consoleplayer].mo) {
        rc = S_getChannel(origin, sfx);
        if (rc < 0) return;  // No free channels
        
        // Calculate distance attenuation
        vol = S_SoundAttenuation(origin, sfx_id);
        if (vol < 1) return;  // Too far away
        
        // Calculate stereo separation
        sep = S_SoundSeparation(origin);
    } else {
        // UI sounds or player sounds - no positioning
        rc = S_getChannel(NULL, sfx);
        vol = snd_SfxVolume;
        sep = NORM_SEP;
    }
    
    // Start the sound
    channels[rc].sfxinfo = sfx;
    channels[rc].origin_id = origin ? origin->id : 0;
    I_StartSound(sfx_id, rc, vol, sep, channels[rc].priority);
}
```

### Distance-Based Attenuation
Sound volume decreases with distance using realistic falloff:

```c
int S_SoundAttenuation(mobj_t* source, int sfx_id)
{
    fixed_t approx_dist;
    fixed_t adx, ady;
    angle_t angle;
    
    // Calculate distance to listener
    adx = abs(players[consoleplayer].mo->x - source->x);
    ady = abs(players[consoleplayer].mo->y - source->y);
    approx_dist = adx + ady - ((adx < ady ? adx : ady) >> 1);
    
    if (gamemap != 8 && approx_dist > S_CLOSE_DIST) {
        // Distance attenuation
        approx_dist = approx_dist >> FRACBITS;
        
        if (approx_dist > S_CLOSE_DIST >> FRACBITS) {
            // Quadratic falloff for distant sounds
            approx_dist = (approx_dist - S_CLOSE_DIST >> FRACBITS) / ATTENUATOR;
            approx_dist = approx_dist * approx_dist;
            
            if (approx_dist > S_CLIPPING_DIST >> FRACBITS) {
                return 0;  // Too far to hear
            }
            
            // Calculate final volume
            return (15 * ((S_CLIPPING_DIST >> FRACBITS) - approx_dist)) / S_CLIPPING_DIST >> FRACBITS);
        }
    }
    
    return snd_SfxVolume;  // Full volume for close sounds
}
```

### Stereo Positioning
Calculate stereo separation based on listener orientation:

```c
int S_SoundSeparation(mobj_t* source)
{
    fixed_t adx, ady;
    angle_t angle;
    
    // Vector from listener to source
    adx = source->x - players[consoleplayer].mo->x;
    ady = source->y - players[consoleplayer].mo->y;
    
    // Calculate angle relative to listener facing direction
    angle = R_PointToAngle2(players[consoleplayer].mo->x,
                           players[consoleplayer].mo->y,
                           source->x, source->y);
    
    // Subtract listener angle to get relative angle
    angle = angle - players[consoleplayer].mo->angle;
    
    // Convert angle to stereo separation
    if (angle > ANG180) {
        // Sound is to the left
        angle = 0xffffffff - angle;
        return NORM_SEP + (FixedMul(S_STEREO_SWING, angle >> ANGLETOFINESHIFT) >> FRACBITS);
    } else {
        // Sound is to the right
        return NORM_SEP - (FixedMul(S_STEREO_SWING, angle >> ANGLETOFINESHIFT) >> FRACBITS);
    }
}
```

## Sound Effect Database (`sounds.c`)

### Sound Effect Definitions
Comprehensive database of all game sound effects:

```c
// Sound effect information structure
typedef struct sfxinfo_struct {
    char*   name;           // Sound name (for WAD lookup)
    int     singularity;    // Exclusivity flag
    int     priority;       // Playback priority
    int     link;           // Link to other sound
    int     pitch;          // Pitch adjustment
    int     volume;         // Default volume
    void*   data;           // Sound data (cached)
    int     usefulness;     // Usage tracking
    int     lumpnum;        // WAD lump number
} sfxinfo_t;

// Complete sound effects database
sfxinfo_t S_sfx[] = {
    // Weapon sounds
    {"none",    false, 0,     0, -1, -1, NULL, 0, -1},
    {"pistol",  false, 64,    0, -1, -1, NULL, 0, -1},
    {"shotgn",  false, 64,    0, -1, -1, NULL, 0, -1},
    {"sgcock",  false, 64,    0, -1, -1, NULL, 0, -1},
    {"dshtgn",  false, 64,    0, -1, -1, NULL, 0, -1},
    {"dbopn",   false, 64,    0, -1, -1, NULL, 0, -1},
    {"dbcls",   false, 64,    0, -1, -1, NULL, 0, -1},
    {"dbload",  false, 64,    0, -1, -1, NULL, 0, -1},
    {"plasma",  false, 64,    0, -1, -1, NULL, 0, -1},
    {"bfg",     false, 64,    0, -1, -1, NULL, 0, -1},
    
    // Monster sounds  
    {"posit1",  true,  98,    0, -1, -1, NULL, 0, -1},
    {"posit2",  true,  98,    0, -1, -1, NULL, 0, -1},
    {"posit3",  true,  98,    0, -1, -1, NULL, 0, -1},
    {"bgsit1",  true,  98,    0, -1, -1, NULL, 0, -1},
    {"bgsit2",  true,  98,    0, -1, -1, NULL, 0, -1},
    
    // Environmental sounds
    {"doors",   false, 63,    0, -1, -1, NULL, 0, -1},
    {"doorcls", false, 63,    0, -1, -1, NULL, 0, -1},
    {"pstop",   false, 60,    0, -1, -1, NULL, 0, -1},
    {"swtchn",  false, 78,    0, -1, -1, NULL, 0, -1},
    {"swtchx",  false, 78,    0, -1, -1, NULL, 0, -1}
};
```

### Sound Priority System
Manages channel allocation based on sound importance:

```c
int S_getChannel(void* origin, sfxinfo_t* sfxinfo)
{
    int         cnum;
    channel_t*  c;
    
    // Check for replacement of identical sound
    for (cnum = 0; cnum < numChannels; cnum++) {
        c = &channels[cnum];
        
        if (c->sfxinfo && c->sfxinfo->singularity == sfxinfo->singularity
            && origin && c->origin_id == ((mobj_t*)origin)->id) {
            S_StopChannel(cnum);  // Replace existing sound
            break;
        }
    }
    
    // Find free channel
    for (cnum = 0; cnum < numChannels; cnum++) {
        if (!channels[cnum].sfxinfo)
            break;
    }
    
    // No free channels - find lowest priority to replace
    if (cnum == numChannels) {
        int lowest_priority = sfxinfo->priority;
        int lowest_channel = -1;
        
        for (cnum = 0; cnum < numChannels; cnum++) {
            if (channels[cnum].priority <= lowest_priority) {
                lowest_priority = channels[cnum].priority;
                lowest_channel = cnum;
            }
        }
        
        if (lowest_channel == -1) {
            return -1;  // Can't allocate channel
        }
        
        S_StopChannel(lowest_channel);
        cnum = lowest_channel;
    }
    
    return cnum;
}
```

## Music System

### Background Music Management
Dynamic music system that responds to game state:

```c
// Music information structure
typedef struct musicinfo_struct {
    char*   name;           // Music name (for WAD lookup)
    int     lumpnum;        // WAD lump number
    void*   data;           // Music data (cached)
    int     handle;         // Playback handle
} musicinfo_t;

// Music database for different levels/situations
musicinfo_t S_music[] = {
    {"e1m1", 0, NULL, 0},    // Episode 1 Map 1
    {"e1m2", 0, NULL, 0},    // Episode 1 Map 2
    {"e1m3", 0, NULL, 0},    // Episode 1 Map 3
    // ... more music tracks
    {"inter", 0, NULL, 0},   // Intermission music
    {"intro", 0, NULL, 0},   // Introduction music
    {"bunny", 0, NULL, 0},   // Ending music
};

int mus_playing = -1;        // Currently playing music
int nextcleanup = 15;        // Music cleanup timer
```

### Dynamic Music Switching
Music changes based on game events and locations:

```c
void S_ChangeMusic(int musicnum, int looping)
{
    musicinfo_t* music;
    
    if (musicnum <= mus_None || musicnum >= NUMMUSIC) {
        I_Error("Bad music number %d", musicnum);
        return;
    }
    
    music = &S_music[musicnum];
    
    if (mus_playing == music)
        return;  // Already playing
    
    // Stop current music
    S_StopMusic();
    
    // Cache music data
    if (!music->data) {
        char namebuf[9];
        sprintf(namebuf, "d_%s", music->name);
        music->data = W_CacheLumpName(namebuf, PU_MUSIC);
        music->lumpnum = W_GetNumForName(namebuf);
    }
    
    // Start new music
    music->handle = I_RegisterSong(music->data, W_LumpLength(music->lumpnum));
    I_PlaySong(music->handle, looping);
    
    mus_playing = music;
}
```

## Platform Audio Interface (`i_sound.c`)

### Linux Audio Implementation
Platform-specific audio implementation using external sound server:

```c
// Communication with sound server process
static int audio_fd = -1;        // Audio server pipe

int I_StartSound(int id, int channel, int vol, int sep, int pitch, int priority)
{
    char command[256];
    
    if (audio_fd < 0)
        return 0;  // Sound system not available
    
    // Send sound command to external server
    sprintf(command, "play %d %d %d %d %d %d\n", 
            id, channel, vol, sep, pitch, priority);
    
    if (write(audio_fd, command, strlen(command)) < 0) {
        // Sound server communication failed
        return 0;
    }
    
    return 1;  // Success
}

void I_StopSound(int channel, int id)
{
    char command[64];
    
    if (audio_fd < 0)
        return;
        
    sprintf(command, "stop %d\n", channel);
    write(audio_fd, command, strlen(command));
}
```

### Sound Server Architecture
External process handles audio mixing to avoid blocking main game:

```c
// Sound server initialization
void I_InitSound(void)
{
    int child_pid;
    int sound_pipe[2];
    
    // Create communication pipe
    if (pipe(sound_pipe) < 0) {
        fprintf(stderr, "Could not create sound pipe\n");
        return;
    }
    
    // Fork sound server process
    child_pid = fork();
    if (child_pid == 0) {
        // Child process - become sound server
        close(sound_pipe[1]);  // Close write end
        sound_server_main(sound_pipe[0]);
        exit(0);
    } else if (child_pid > 0) {
        // Parent process - keep write end
        close(sound_pipe[0]);  // Close read end
        audio_fd = sound_pipe[1];
    } else {
        fprintf(stderr, "Could not fork sound server\n");
    }
}
```

## Advanced Audio Features

### Sound Caching and Optimization
Intelligent caching system for frequently used sounds:

```c
void S_CacheSounds(void)
{
    sfxinfo_t* sfx;
    char namebuf[9];
    int i;
    
    // Preload essential sounds
    for (i = 1; i < NUMSFX; i++) {
        sfx = &S_sfx[i];
        
        if (sfx->priority >= 64) {  // High priority sounds
            sprintf(namebuf, "ds%s", sfx->name);
            sfx->data = W_CacheLumpName(namebuf, PU_SOUND);
            sfx->lumpnum = W_GetNumForName(namebuf);
        }
    }
}

void S_PurgeSounds(void)
{
    int i;
    
    // Free cached sound data
    for (i = 1; i < NUMSFX; i++) {
        if (S_sfx[i].data && S_sfx[i].priority < 64) {
            Z_ChangeTag(S_sfx[i].data, PU_CACHE);
            S_sfx[i].data = NULL;
        }
    }
}
```

### Sound Effect Variations
System supports pitch and volume variations for variety:

```c
void S_StartSoundAtVolume(void* origin, int sound_id, int volume)
{
    int rc, sep, pitch;
    
    // Add slight random pitch variation for realism
    pitch = NORM_PITCH + (P_Random() - P_Random()) * 32;
    
    // Clamp pitch to reasonable range
    if (pitch < 0) pitch = 0;
    if (pitch > 255) pitch = 255;
    
    rc = S_getChannel(origin, &S_sfx[sound_id]);
    if (rc < 0) return;
    
    // Calculate stereo separation
    if (origin)
        sep = S_SoundSeparation((mobj_t*)origin);
    else
        sep = NORM_SEP;
    
    // Start sound with variations
    I_StartSound(sound_id, rc, volume, sep, pitch, S_sfx[sound_id].priority);
}
```

### Environmental Audio Effects
Basic environmental audio processing:

```c
void S_UpdateSounds(mobj_t* listener)
{
    int audible;
    int cnum;
    channel_t* c;
    
    // Update all active channels
    for (cnum = 0; cnum < numChannels; cnum++) {
        c = &channels[cnum];
        
        if (!c->sfxinfo)
            continue;
            
        if (c->origin_id) {
            mobj_t* origin = P_FindMobjById(c->origin_id);
            
            if (!origin) {
                // Origin object destroyed - stop sound
                S_StopChannel(cnum);
                continue;
            }
            
            // Update 3D positioning
            audible = S_SoundAttenuation(origin, c->sfxinfo - S_sfx);
            if (audible) {
                I_UpdateSoundParams(cnum, audible, S_SoundSeparation(origin));
            } else {
                S_StopChannel(cnum);  // Too far away
            }
        }
    }
}
```

The DOOM sound system demonstrates sophisticated audio programming techniques while maintaining compatibility with limited hardware. The 3D positioning, priority management, and efficient caching create an immersive audio experience that significantly enhanced the game's atmosphere and gameplay.