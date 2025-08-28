# Game Logic and Physics Systems

DOOM's game logic and physics systems form the core of the interactive experience. These systems handle everything from player movement and collision detection to AI behavior and weapon mechanics. The design emphasizes deterministic behavior essential for multiplayer synchronization and demo playback.

## Architecture Overview

### Entity-Component System
DOOM uses a sophisticated object system based on **Map Objects (mobjs)**:

```
All Game Entities → mobj_t → Specific Behaviors
    ↓
Players, Enemies, Items, Projectiles, Effects
```

### Core Components
1. **Map Objects** (`p_mobj.c`) - Universal entity system
2. **Movement & Collision** (`p_map.c`) - Physics simulation  
3. **Player Actions** (`p_user.c`) - Player-specific behaviors
4. **AI Systems** (`p_enemy.c`) - Enemy artificial intelligence
5. **Interactions** (`p_inter.c`) - Object interaction handling
6. **Special Effects** (`p_spec.c`) - Environmental systems

## Map Objects (MObjs) - Universal Entity System

### Core Entity Structure
All interactive objects in DOOM inherit from the `mobj_t` structure:

```c
// From p_mobj.h - The universal game object
typedef struct mobj_s
{
    thinker_t       thinker;        // Thinking/updating system
    
    // Position
    fixed_t         x, y, z;        // World coordinates
    struct mobj_s*  snext;          // Sector list linkage
    struct mobj_s*  sprev;
    
    // Movement
    fixed_t         momx, momy, momz;  // Momentum/velocity
    angle_t         angle;          // Facing direction
    
    // Appearance  
    spritenum_t     sprite;         // Sprite index
    int             frame;          // Animation frame
    
    // Game properties
    mobjtype_t      type;           // Object type (player, enemy, etc.)
    mobjinfo_t*     info;           // Static object definition
    int             health;         // Hit points
    int             flags;          // Behavior flags
    
    // State machine
    state_t*        state;          // Current animation state
    int             tics;           // Time until next state
    
    // Relationships
    struct mobj_s*  target;         // AI target or damage source
    struct mobj_s*  tracer;         // Homing missile target
    player_t*       player;         // Player data (if player object)
    
} mobj_t;
```

### Object Types and Behaviors
Objects are categorized by type with specific behaviors:

```c
// From info.h - Object type definitions
typedef enum {
    MT_PLAYER,          // Player character
    MT_POSSESSED,       // Possessed marine (enemy)
    MT_SHOTGUY,         // Shotgun marine
    MT_SERGEANT,        // Chaingun sergeant
    MT_SHADOW,          // Demon
    MT_IMPBALL,         // Imp fireball projectile
    MT_CLIP,            // Ammunition clip
    MT_SHOTGUN,         // Shotgun weapon
    // ... hundreds more types
} mobjtype_t;
```

### State Machine System
All objects use finite state machines for animation and behavior:

```c
// State transition and action execution
boolean P_SetMobjState(mobj_t* mobj, statenum_t state)
{
    state_t* st;
    
    do {
        if (state == S_NULL) {
            P_RemoveMobj(mobj);  // Object destroyed
            return false;
        }
        
        st = &states[state];
        mobj->state = st;
        mobj->tics = st->tics;        // Duration of this state
        mobj->sprite = st->sprite;    // Visual representation
        mobj->frame = st->frame;
        
        // Execute state action (AI, movement, effects)
        if (st->action.acp1)
            st->action.acp1(mobj);
            
        state = st->nextstate;        // Chain to next state
    } while (!mobj->tics);
    
    return true;
}
```

## Physics and Movement System

### Movement Integration (`p_mobj.c`)
Object movement uses simple Euler integration:

```c
void P_MobjThinker(mobj_t* mobj)
{
    // Apply momentum to position
    if (mobj->momx || mobj->momy || (mobj->flags & MF_SKULLFLY)) {
        P_XYMovement(mobj);
    }
    
    // Vertical movement
    if (mobj->momz || mobj->z != mobj->floorz) {
        P_ZMovement(mobj);
    }
    
    // State transitions
    if (mobj->tics != -1) {
        mobj->tics--;
        if (!mobj->tics)
            P_SetMobjState(mobj, mobj->state->nextstate);
    }
}
```

### Collision Detection (`p_map.c`)
Sophisticated collision system handles movement and interaction:

```c
// Movement with collision detection
boolean P_TryMove(mobj_t* thing, fixed_t x, fixed_t y)
{
    // Set up collision context
    tmthing = thing;
    tmflags = thing->flags;
    tmx = x;
    tmy = y;
    
    // Calculate bounding box
    tmbbox[BOXTOP] = y + tmthing->radius;
    tmbbox[BOXBOTTOM] = y - tmthing->radius;
    tmbbox[BOXRIGHT] = x + tmthing->radius;
    tmbbox[BOXLEFT] = x - tmthing->radius;
    
    // Check against world geometry
    if (!P_CheckPosition(thing, x, y))
        return false;
        
    // Move succeeded
    P_UnsetThingPosition(thing);
    thing->floorz = tmfloorz;
    thing->ceilingz = tmceilingz;
    thing->x = x;
    thing->y = y;
    P_SetThingPosition(thing);
    
    return true;
}
```

### Line-of-Sight System
Determines visibility between objects:

```c
// Check if target is visible (p_sight.c)
boolean P_CheckSight(mobj_t* t1, mobj_t* t2)
{
    // Use BSP tree for efficient line-of-sight testing
    sightzstart = t1->z + t1->height - (t1->height >> 2);
    topslope = (t2->z + t2->height) - sightzstart;
    bottomslope = t2->z - sightzstart;
    
    return P_CrossBSPNode(numnodes - 1);
}
```

## Player Mechanics (`p_user.c`)

### Player Thinking
Player objects have specialized behavior:

```c
void P_PlayerThink(player_t* player)
{
    ticcmd_t* cmd = &player->cmd;
    
    // Handle death state
    if (player->playerstate == PST_DEAD) {
        P_DeathThink(player);
        return;
    }
    
    // Process movement commands
    if (cmd->forwardmove)
        P_Thrust(player->mo, player->mo->angle, cmd->forwardmove * 2048);
    if (cmd->sidemove)
        P_Thrust(player->mo, player->mo->angle - ANG90, cmd->sidemove * 2048);
        
    // Handle turning
    if (cmd->angleturn) {
        player->mo->angle += cmd->angleturn << 16;
    }
    
    // Weapon and action handling
    P_CalcHeight(player);      // View bobbing
    P_MoveWeapon(player);      // Weapon animation
    P_PlayerInSpecialSector(player);  // Sector effects
}
```

### Movement Physics
Player movement includes realistic physics:

- **Friction**: Ground friction slows movement
- **Momentum**: Smooth acceleration and deceleration  
- **Air control**: Limited control while falling
- **Step climbing**: Automatic step-up for small height differences
- **Sector effects**: Damage floors, moving platforms

```c
void P_XYMovement(mobj_t* mo)
{
    // Apply friction
    if (mo->flags & MF_SKULLFLY) {
        // Special case for charging demons
    } else if (mo->z > mo->floorz) {
        // Air friction
        mo->momx = FixedMul(mo->momx, FRICTION_FLY);
        mo->momy = FixedMul(mo->momy, FRICTION_FLY);
    } else if (mo->flags & MF_CORPSE) {
        // Dead body friction  
        mo->momx = FixedMul(mo->momx, FRICTION_LOW);
        mo->momy = FixedMul(mo->momy, FRICTION_LOW);
    } else {
        // Normal ground friction
        mo->momx = FixedMul(mo->momx, FRICTION_NORMAL);
        mo->momy = FixedMul(mo->momy, FRICTION_NORMAL);
    }
}
```

## Artificial Intelligence (`p_enemy.c`)

### AI State System
Enemies use state machines for intelligent behavior:

```c
// Basic AI thinking pattern
void A_Look(mobj_t* actor)
{
    // Look for players to target
    target = P_LookForTargets(actor);
    if (target) {
        actor->target = target;
        P_SetMobjState(actor, actor->info->seestate);
    }
}

void A_Chase(mobj_t* actor) 
{
    // Move toward target
    if (!actor->target || actor->target->health <= 0) {
        P_SetMobjState(actor, actor->info->spawnstate);
        return;
    }
    
    // Check for attack opportunity  
    if (P_CheckMeleeRange(actor)) {
        P_SetMobjState(actor, actor->info->meleestate);
        return;
    }
    
    if (P_CheckMissileRange(actor)) {
        P_SetMobjState(actor, actor->info->missilestate);
        return;
    }
    
    // Move toward target
    P_NewChaseDir(actor);
}
```

### Pathfinding
Simple but effective pathfinding using sector connectivity:

```c
void P_NewChaseDir(mobj_t* actor)
{
    fixed_t deltax = actor->target->x - actor->x;
    fixed_t deltay = actor->target->y - actor->y;
    
    // Calculate desired direction
    dirtype_t d[3];
    if (deltax > 10*FRACUNIT)      d[1] = DI_EAST;
    else if (deltax < -10*FRACUNIT) d[1] = DI_WEST;
    else                           d[1] = DI_NODIR;
    
    // Try movement in preferred directions
    if (P_TryWalk(actor))
        return;
        
    // If blocked, try alternative directions
    // Smart pathfinding around obstacles
}
```

## Weapon Systems (`p_pspr.c`)

### Weapon State Machine
Weapons use their own state system:

```c
void P_SetPsprite(player_t* player, int position, statenum_t stnum)
{
    pspdef_t* psp = &player->psprites[position];
    
    do {
        if (!stnum) {
            psp->state = NULL;
            break;
        }
        
        state = &states[stnum];
        psp->state = state;
        psp->tics = state->tics;
        
        // Execute weapon action
        if (state->action.acp2)
            state->action.acp2(player, psp);
            
        stnum = state->nextstate;
    } while (!psp->tics);
}
```

### Projectile Physics
Projectiles follow realistic ballistic paths:

```c
mobj_t* P_SpawnMissile(mobj_t* source, mobj_t* dest, mobjtype_t type)
{
    mobj_t* th = P_SpawnMobj(source->x, source->y, source->z + 32*FRACUNIT, type);
    
    // Calculate trajectory
    fixed_t dist = P_AproxDistance(dest->x - source->x, dest->y - source->y);
    fixed_t speed = th->info->speed;
    
    th->momx = FixedMul(speed, FixedDiv(dest->x - source->x, dist));
    th->momy = FixedMul(speed, FixedDiv(dest->y - source->y, dist));
    
    // Add vertical component for arc
    if (dest->z > source->z)
        th->momz = FixedDiv(dest->z - source->z, dist/speed);
        
    return th;
}
```

## Interactive Systems (`p_inter.c`)

### Damage System
Sophisticated damage calculation with armor, random factors:

```c
void P_DamageMobj(mobj_t* target, mobj_t* inflictor, mobj_t* source, int damage)
{
    player_t* player = target->player;
    
    // Invulnerability check
    if (target->flags & MF_INVULNERABLE && damage < 10000)
        return;
        
    // Player armor absorption
    if (player && player->armorpoints) {
        saved = damage / (player->armortype == 1 ? 3 : 2);
        if (player->armorpoints <= saved) {
            saved = player->armorpoints;
            player->armortype = 0;
        }
        player->armorpoints -= saved;
        damage -= saved;
    }
    
    // Apply damage
    target->health -= damage;
    
    if (target->health <= 0) {
        P_KillMobj(source, target);
        return;
    }
    
    // Enter pain state  
    if (target->info->painstate && (P_Random() < target->info->painchance))
        P_SetMobjState(target, target->info->painstate);
}
```

### Item Interaction
Pickup system with inventory management:

```c
boolean P_GiveAmmo(player_t* player, ammotype_t ammo, int num)
{
    int oldammo;
    
    if (ammo == am_noammo)
        return false;
        
    if (player->ammo[ammo] == player->maxammo[ammo])
        return false;  // Already full
        
    if (num)
        num *= ClipAmmo[ammo];
    else
        num = ClipAmmo[ammo] / 2;  // Default amount
        
    oldammo = player->ammo[ammo];
    player->ammo[ammo] += num;
    
    if (player->ammo[ammo] > player->maxammo[ammo])
        player->ammo[ammo] = player->maxammo[ammo];
        
    return (oldammo != player->ammo[ammo]);
}
```

## Performance and Determinism

### Fixed-Point Arithmetic
All calculations use deterministic fixed-point math:

```c
// Consistent across all platforms
fixed_t FixedMul(fixed_t a, fixed_t b)
{
    return ((long long)a * (long long)b) >> FRACBITS;
}
```

### Thinker System
Efficient update system for all active objects:

```c
void P_RunThinkers(void)
{
    for (currentthinker = thinkercap.next; 
         currentthinker != &thinkercap; 
         currentthinker = currentthinker->next) {
        if (currentthinker->function.acp1)
            currentthinker->function.acp1(currentthinker);
    }
}
```

This game logic system achieves remarkable complexity while maintaining the deterministic behavior essential for multiplayer synchronization. The modular design with state machines, universal entity system, and sophisticated physics creates an engaging and technically robust gaming experience.