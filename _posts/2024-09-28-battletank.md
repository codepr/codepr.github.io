---
layout: post
title: "Battletank"
description: ""
categories: c game-development low-level
---

Small project idea to have somme fun with C, a micro multiplayer classi game
with as less dependencies as possible. I've always been curious about the
challenges of game development so I started from one of the simplest ideas. The
latest can be found in the repository
[codepr/battletank.git](https://github.com/codepr/battletank.git).

## The game

The idea I was thinking about is a best effort terminal based implementation of
the classic battle-tank, starting from a single player single tank and extending
it to work as a multiplayer server with sockets to sync the game state across
players.
To begin with, the only dependency is `ncurses`, later on, if I get confident enough,
I will consider something fancier such as `Raylib`.

### Why
To have some fun, small old school programs are fun to mess with.

### Ideas
In no particular order, and not necessarily mandatory:
- Implement a very simple and stripped down game logic ✅
    - 1 player, 1 tank, 1 bullet ✅
    - Handle keyboard input ✅
    - Design a better structured game state, multiple tanks, each tank loaded with bullets
- Multiplayer through TCP sockets
    - Implement a TCP server ✅
    - Implement a TCP client ✅
    - Implement a network protocol (text based or binary) ✅
    - The clients will send their input, server handle the game state and broadcasts it to the connected clients ✅
    - Heartbeat server side, drop inactive clients
    - Small chat? maybe integrate [chatlite](https://github.com/codepr/chatlite.git)
    - Ensure screen size scaling is maintained in sync with players
- Walls
- Life points
- Bullets count
- Recharge bullets on a time basis
- Power ups (faster bullets? larger hit box? Mines?)
- Explore SDL2 or Raylib for some graphic or sprites

## Main challenges

The game is pretty simple, the logic is absolutely elementary, just x, y axis
as direction for both the tank and the bullets. The main challenges are
represented by keeping players in sync and ensure that the battlefield size is
correctly scaled for each one of them (not touched this yet).

For the communication part I chose the simplest and obiquitous solution, a
non-blocking server on `select` (yikes, maybe at least `poll`) and a timeout on
read client, this way the main loops are not blocked indefinitely and the game
state can flow. There are certainly infinitely better ways to do it, but
avoiding threading and excessive engineered solutions was part of the scope.
There is not plan to make anything more than have some fun out of this project,
one simple upgrade would be to migrate the server implementation to `epoll` or
`kqueue`, however it's definitely not a game to be expected to have a
sufficiently high number of players to prove problematic to handle for the good
old `select` call.

Something I find more interesting is delving a little more into graphics, I
literally don't know anything about it but I can see libraries such as `Raylib`
seem to make it simple enough to generate some basic animations and sprites,
the program is sufficiently simple to try and plug it in, hopefully without too
much troubles.

## Implementation

I won't go deep in details, some parts are to be considered boilerplate, such as
serialization, TCP helpers and such. To get into details of those parts, the
repository is [codepr/battletank.git](https://github.com/codepr/battletank.git).

The game can be divided in the main modules:
- The **game state**
    - Tank
    - Bullet
- A **game server**, handles  the game state and serves as the unique authoritative source of truth.
- A **game client**, connects to the server, provides a very crude terminal based
  graphic battlefield and handles input from the player.
- A **protocol** to communicate. Initially I went for a text-based protocol, but
  I'm not very fond of them, so I decided for a binary one eventually, a very
  simple one.

### The game state

The game state is the most simple I could imagine to begin with
- An array of tanks
- Each tank has 1 bullet

Simple and easy to keep in sync, the server is only required to update the coordinates of
each tank and bullet and send them to the clients.

```c
// Possible directions a tank or bullet can move.
typedef enum { IDLE, UP, DOWN, LEFT, RIGHT } Direction;

// Only fire for now, can add something else such as
// DROP_MINE etc.
typedef enum {
    FIRE = 5,
} Action;

// Represents a bullet with its position, direction, and status.
// Can include bullet kinds as a possible update for the future.
typedef struct {
    int x;
    int y;
    Direction direction;
    bool active;
} Bullet;

// Represents a tank with its position, direction, and status.
// Contains a single bullet for simplicity, can be extendend in
// the future to handle multiple bullets, life points, power-ups etc.
typedef struct {
    int x;
    int y;
    Direction direction;
    bool alive;
    Bullet bullet;
} Tank;

typedef struct {
    Tank *players;
    size_t players_count;
    size_t player_index;
} Game_State;

// General game state managing
void game_state_init(Game_State *state);
void game_state_free(Game_State *state);
void game_state_update(Game_State *state);

// Tank management
void game_state_spawn_tank(Game_State *state, size_t index);
void game_state_dismiss_tank(Game_State *state, size_t index);
void game_state_update_tank(Game_State *state, size_t tank_index,
                            unsigned action);
```

#### Tanks and bullets

Although the structures introduced are trivial, some helper functions to manage
tanks and bullets can come handy; when the server starts, the first thing will
be to init a global game state. In hindsight, I could've easily set a fixed
number of players such as 10, I went for a dynamic array on auto-pilot
basically.

```c
void game_state_init(Game_State *state) {
    state->players_count = 2;
    state->players = calloc(2, sizeof(Tank));
    for (size_t i = 0; i < state->players_count; ++i) {
        state->players[i].alive = false;
        state->players[i].bullet.active = false;
    }
}

void game_state_free(Game_State *state) { free(state->players); }

void game_state_spawn_tank(Game_State *state, size_t index) {
    // Extend the players pool if we're at capacity
    if (index > state->players_count) {
        state->players_count *= 2;
        state->players = realloc(state->players, state->players_count);
    }

    if (!state->players[index].alive) {
        state->players[index].alive = true;
        state->players[index].x = RANDOM(15, 25);
        state->players[index].y = RANDOM(15, 25);
        state->players[index].direction = 0;
    }
}

void game_state_dismiss_tank(Game_State *state, size_t index) {
    state->players[index].alive = false;
}

```

And here to follow the remaining functions needed to actually update the state of the game,
mainly manipulation of the X, Y axis for the tank and bullet directions based on actions
coming from each client.

To check collision initially we just check that the coordinates of a given tank
collide with those of a given bullet. Admittedly I didn't focus much on that,
for a first test run I was more interested into seeing actualy tanks moving and
be in sync with each other through the network, but `check_collision` still
provides a good starting point to expand on later.

```c
static void fire_bullet(Tank *tank) {
    if (!tank->bullet.active) {
        tank->bullet.active = true;
        tank->bullet.x = tank->x;
        tank->bullet.y = tank->y;
        tank->bullet.direction = tank->direction;
    }
}

void game_state_update_tank(Game_State *state, size_t tank_index,
                            unsigned action) {
    switch (action) {
        case UP:
            state->players[tank_index].y--;
            state->players[tank_index].direction = UP;
            break;
        case DOWN:
            state->players[tank_index].y++;
            state->players[tank_index].direction = DOWN;
            break;
        case LEFT:
            state->players[tank_index].x--;
            state->players[tank_index].direction = LEFT;
            break;
        case RIGHT:
            state->players[tank_index].x++;
            state->players[tank_index].direction = RIGHT;
            break;
        case FIRE:
            fire_bullet(&state->players[tank_index]);
            break;
        default:
            break;
    }
}

static void update_bullet(Bullet *bullet) {
    if (!bullet->active) return;

    switch (bullet->direction) {
        case UP:
            bullet->y--;
            break;
        case DOWN:
            bullet->y++;
            break;
        case LEFT:
            bullet->x -= 2;
            break;
        case RIGHT:
            bullet->x += 2;
            break;
        default:
            break;
    }

    if (bullet->x < 0 || bullet->x >= COLS || bullet->y < 0 ||
        bullet->y >= LINES) {
        bullet->active = false;
    }
}

static void check_collision(Tank *tank, Bullet *bullet) {
    if (bullet->active && tank->x == bullet->x && tank->y == bullet->y) {
        tank->alive = false;
        bullet->active = false;
    }
}

/**
 * Updates the game state by advancing bullets and checking for collisions
 * between tanks and bullets.
 *
 * - Creates an array of pointers to each player's bullet for easy access during
 * collision checks.
 * - For each player:
 *   - Updates their bullet by calling `update_bullet`.
 *   - Checks for collisions between the player's tank and every other player's
 * bullet using `check_collision`.
 * - Skips collision checks between a player and their own bullet.
 */
void game_state_update(Game_State *state) {
    Bullet *bullets[state->players_count];
    for (size_t i = 0; i < state->players_count; ++i)
        bullets[i] = &state->players[i].bullet;

    for (size_t i = 0; i < state->players_count; ++i) {
        update_bullet(&state->players[i].bullet);
        for (size_t j = 0; j < state->players_count; ++j) {
            if (j == i) continue;  // Skip self collision
            check_collision(&state->players[i], bullets[j]);
        }
    }
}
```

#### The client side, where the player sends commands

**TO BE CONTINUED**
