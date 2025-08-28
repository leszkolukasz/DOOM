# Input Handling System

DOOM's input handling system is designed around an event-driven architecture that processes keyboard, mouse, and network inputs efficiently. The system transforms low-level hardware events into game actions through multiple abstraction layers.

## Architecture Overview

### Event-Driven Design
The input system uses an asynchronous event queue to decouple input gathering from game logic processing:

```
Hardware Input → Platform Layer → Event Queue → Game Logic → Actions
```

### Key Components
1. **Platform Interface** (`i_*` files) - Hardware-specific input gathering
2. **Event Processing** (`d_main.c`) - Central event dispatch system  
3. **Input Translation** (`g_game.c`) - Convert events to game commands
4. **Menu/Game Responders** - Process events based on current game state

## Core Input Files

### Platform Input Interface (`i_video.c`, `i_system.c`)
Handles low-level input from X11 system:

```c
// X11 keyboard and mouse event handling
#include <X11/Xlib.h>
#include <X11/keysym.h>

// Key translation table
int xlatekey(XKeyEvent* ev)
{
    switch(XLookupKeysym(ev, 0)) {
        case XK_Left: return KEY_LEFTARROW;
        case XK_Right: return KEY_RIGHTARROW;
        case XK_Up: return KEY_UPARROW;
        case XK_Down: return KEY_DOWNARROW;
        case XK_space: return ' ';
        // ... more key mappings
    }
}
```

### Event Management (`d_main.c`)
Central event processing system:

```c
#define MAXEVENTS 64
event_t events[MAXEVENTS];    // Circular event buffer
int     eventhead;            // Write pointer
int     eventtail;            // Read pointer

// Post event to queue
void D_PostEvent(event_t* ev)
{
    events[eventhead] = *ev;
    eventhead = (++eventhead) & (MAXEVENTS-1);
}

// Process all queued events
void D_ProcessEvents(void)
{
    event_t* ev;
    
    for (; eventtail != eventhead; eventtail = (++eventtail) & (MAXEVENTS-1)) {
        ev = &events[eventtail];
        
        if (M_Responder(ev))      // Try menu system first
            continue;
        G_Responder(ev);          // Then game system
    }
}
```

## Event Types and Structures

### Event Structure (`d_event.h`)
```c
typedef enum {
    ev_keydown,
    ev_keyup,
    ev_mouse,
    ev_joystick
} evtype_t;

typedef struct {
    evtype_t type;
    int      data1;    // Key code or mouse/joystick state
    int      data2;    // Mouse X movement
    int      data3;    // Mouse Y movement
} event_t;
```

### Key Codes (`doomdef.h`)
Standard key definitions:
```c
#define KEY_RIGHTARROW  0xae
#define KEY_LEFTARROW   0xac  
#define KEY_UPARROW     0xad
#define KEY_DOWNARROW   0xaf
#define KEY_ESCAPE      27
#define KEY_ENTER       13
#define KEY_TAB         9
#define KEY_F1          (0x80+0x3b)
#define KEY_F2          (0x80+0x3c)
// ... more function keys
```

## Input Processing Layers

### 1. Hardware Event Capture
X11 event loop captures raw input events:

```c
// Simplified from i_video.c
void I_GetEvent(void)
{
    XEvent X_event;
    event_t event;
    
    while (XPending(X_display) > 0) {
        XNextEvent(X_display, &X_event);
        
        switch (X_event.type) {
            case KeyPress:
                event.type = ev_keydown;
                event.data1 = xlatekey(&X_event.xkey);
                D_PostEvent(&event);
                break;
                
            case KeyRelease:
                event.type = ev_keyup;
                event.data1 = xlatekey(&X_event.xkey);
                D_PostEvent(&event);
                break;
                
            case MotionNotify:
                event.type = ev_mouse;
                event.data2 = X_event.xmotion.x_root;
                event.data3 = X_event.xmotion.y_root;
                D_PostEvent(&event);
                break;
        }
    }
}
```

### 2. Event Responder Chain
Events are processed through a priority chain:

#### Menu Responder (`m_menu.c`)
Handles menu navigation and interaction:
```c
boolean M_Responder(event_t* ev)
{
    if (menuactive) {
        switch(ev->type) {
            case ev_keydown:
                switch(ev->data1) {
                    case KEY_DOWNARROW:
                        M_NextItem();
                        return true;
                    case KEY_UPARROW:
                        M_PrevItem();
                        return true;
                    case KEY_ENTER:
                        M_SelectItem();
                        return true;
                }
                break;
        }
    }
    return false;  // Event not handled
}
```

#### Game Responder (`g_game.c`)
Converts events to game commands:
```c
boolean G_Responder(event_t* ev)
{
    if (gamestate == GS_LEVEL) {
        switch(ev->type) {
            case ev_keydown:
                gamekeydown[ev->data1] = true;
                return true;
            case ev_keyup:
                gamekeydown[ev->data1] = false; 
                return true;
            case ev_mouse:
                // Handle mouse look/movement
                return true;
        }
    }
    return false;
}
```

### 3. Command Building (`g_game.c`)
Raw input events are translated into game commands:

```c
boolean gamekeydown[NUMKEYS];  // Current key states

void G_BuildTiccmd(ticcmd_t* cmd)
{
    memset(cmd, 0, sizeof(*cmd));
    
    // Movement
    if (gamekeydown[key_right] || gamekeydown[KEY_RIGHTARROW])
        cmd->angleturn -= angleturn[tspeed];
    if (gamekeydown[key_left] || gamekeydown[KEY_LEFTARROW])
        cmd->angleturn += angleturn[tspeed];
        
    if (gamekeydown[key_up] || gamekeydown[KEY_UPARROW])
        cmd->forwardmove += forwardmove[speed];
    if (gamekeydown[key_down] || gamekeydown[KEY_DOWNARROW])
        cmd->forwardmove -= forwardmove[speed];
        
    // Strafing
    if (gamekeydown[key_straferight])
        cmd->sidemove += sidemove[speed];
    if (gamekeydown[key_strafeleft])
        cmd->sidemove -= sidemove[speed];
        
    // Actions
    if (gamekeydown[key_fire])
        cmd->buttons |= BT_ATTACK;
    if (gamekeydown[key_use])
        cmd->buttons |= BT_USE;
}
```

## Game Command Structure

### Tic Commands (`d_ticcmd.h`)
All player actions are encoded into standardized command structures:

```c
typedef struct {
    signed char forwardmove;    // -50 to 50
    signed char sidemove;       // -40 to 40  
    short       angleturn;      // Angle delta
    short       consistancy;    // For network sync
    byte        chatchar;       // Chat message character
    byte        buttons;        // Action buttons
} ticcmd_t;

// Button flags
#define BT_ATTACK       1
#define BT_USE          2
#define BT_CHANGE       4    // Weapon change
#define BT_WEAPONMASK   (8+16+32)  // Weapon selection
```

## Mouse Input System

### Mouse Sensitivity and Acceleration
```c
extern int mouseSensitivity;  // User-configurable sensitivity

void I_ReadMouse(void)
{
    int mx, my;
    // Get mouse movement delta
    GetMouseMovement(&mx, &my);
    
    // Apply sensitivity scaling
    mx = (mx * mouseSensitivity) / 10;
    my = (my * mouseSensitivity) / 10;
    
    // Post mouse event
    event_t event;
    event.type = ev_mouse;
    event.data2 = mx;
    event.data3 = my;
    D_PostEvent(&event);
}
```

### Mouse Look Integration
Mouse movement affects both turning and movement:
```c
void G_BuildTiccmd(ticcmd_t* cmd) 
{
    // Mouse turning
    if (mousex) {
        cmd->angleturn -= mousex * 0x8;
        mousex = 0;
    }
    
    // Mouse forward/back movement  
    if (mousey) {
        if (mouseforward)
            cmd->forwardmove += mousey;
        mousey = 0;
    }
}
```

## Keyboard Configuration

### Key Binding System (`m_misc.c`)
Configurable key assignments:
```c
int key_right = KEY_RIGHTARROW;
int key_left = KEY_LEFTARROW;
int key_up = KEY_UPARROW;  
int key_down = KEY_DOWNARROW;
int key_strafeleft = ',';
int key_straferight = '.';
int key_fire = KEY_RCTRL;
int key_use = ' ';
int key_strafe = KEY_RALT;
int key_speed = KEY_RSHIFT;

// Configuration file handling
void M_LoadDefaults(void)
{
    // Load key bindings from config file
    // Format: key_name value
}

void M_SaveDefaults(void)
{
    // Save current key bindings
}
```

## Network Input Synchronization

### Multiplayer Command Sync
In network games, all player commands must be synchronized:

```c
ticcmd_t netcmds[MAXPLAYERS][BACKUPTICS];  // Command history buffer

void D_RunNetTic(void)
{
    // Collect commands from all players
    for (i = 0; i < MAXPLAYERS; i++) {
        if (playeringame[i]) {
            G_BuildTiccmd(&netcmds[i][gametic % BACKUPTICS]);
        }
    }
    
    // Execute synchronized commands
    G_Ticker();
}
```

## Special Input Features

### Cheat Code Detection (`m_cheat.c`)
Monitors key sequences for cheat codes:
```c
typedef struct {
    char sequence[20];
    int  p;              // Current position in sequence
} cheatseq_t;

boolean M_CheatResponder(event_t* ev)
{
    if (ev->type == ev_keydown) {
        for (i = 0; i < numcheats; i++) {
            if (AdvanceCheatSequence(&cheats[i], ev->data1)) {
                // Cheat activated!
                return true;
            }
        }
    }
    return false;
}
```

### Demo Recording/Playback
Input commands are recorded for demo functionality:
```c
void G_RecordDemo(ticcmd_t* cmd)
{
    *demo_p++ = cmd->forwardmove;
    *demo_p++ = cmd->sidemove;  
    *demo_p++ = cmd->angleturn >> 8;
    *demo_p++ = cmd->angleturn & 255;
    *demo_p++ = cmd->buttons;
}
```

## Performance Considerations

### Input Timing
- **Fixed timestep**: Game runs at 35Hz regardless of input rate
- **Input buffering**: Events queued until next game tic
- **Smooth movement**: Multiple keys can be held simultaneously

### Input Lag Minimization
- **Direct hardware access**: Minimal abstraction layers
- **Efficient event processing**: Quick dispatch through responder chain  
- **Predictive input**: Local player movement predicted before network sync

The input system's design allows for responsive controls while maintaining deterministic behavior essential for network play and demo recording. The multi-layered architecture provides flexibility for different input sources while keeping the core game logic simple and focused.