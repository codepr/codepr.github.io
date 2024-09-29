---
layout: post
title: "Battletank!"
description: "Dumb terminal-based implementation of the old classic Battletank"
categories: c game-development low-level terminal ncurses
---

Small project idea to have somme fun with C, a micro multiplayer classic game
with as less dependencies as possible. I've always been curious about the
challenges of game development so I started from one of the simplest ideas. The
latest update of the code can be found in the repository
[codepr/battletank.git](https://github.com/codepr/battletank.git).

## The game

It's about  a best effort terminal based implementation of the classic
battle-tank, starting from a single player single tank and extending it to work
as a multiplayer server with sockets to sync the game state across players.
To begin with, the only dependency is `ncurses`, later on, if I get confident enough,
I will consider something fancier such as [`Raylib`](https://www.raylib.com/index.html).

### Why

To have some fun, small old school programs are fun to mess with. In addition,
although I worked for a brief stint for an AAA gaming company, I was mainly
developing on the backend side of the product, pre-game lobbies, chat rooms and
game server deployments; but didn't really know much about the inner logic of
the game itself. This was a good opportunity to try and learn something about
game development starting from the basics.

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

Short and sweet, to keep in sync, the server is only required to update the
coordinates of each tank and bullet and send them to the clients. This
structure is probably where additional improvements mentioned in the intro
paragraph could live, power ups, walls, bullets and their kinds, mines etc.

<hr>
**game_state.h**
<hr>

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
<hr>

#### Tanks and bullets

Although the structures introduced are trivial, some helper functions to manage
tanks and bullets can come handy; when the server starts, the first thing will
be to init a global game state. When a new player connects, a tank will be
spawned in the battlefield, I opted for a random position spawning in a small
set of coordinates. In hindsight, I could've easily set a fixed number of
players such as 10, I went for a dynamic array on auto-pilot basically. To be
noted that as of now I'm not really correctly freeing the allocated tanks
(these are the only structure that is heap allocated) as it's not really
necessary, the memory will be released at shutdown of the program anyway and
the number of expected players is not that big. That said, it's definitely best
practice to handle the case correctly, I may address that at a later stage.

<hr>
**game_state.c**
<hr>
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
<hr>

And here to follow the remaining functions needed to actually update the state of the game,
mainly manipulation of the X, Y axis for the tank and bullet directions based on actions
coming from each client.

To check collision initially I just check that the coordinates of a given tank
collide with those of a given bullet. Admittedly I didn't focus much on
that (after all there isn't even a score logic yet), for a first test run I
was more interested into seeing actualy tanks moving and be in sync with
each other through the network, but `check_collision` still provides a good
starting point to expand on later.

<hr>
**game_state.c**
<hr>

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
 *   collision checks.
 * - For each player:
 *   - Updates their bullet by calling `update_bullet`.
 *   - Checks for collisions between the player's tank and every other player's
 *     bullet using `check_collision`.
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
<hr>

#### The client side

The client is the main entry point for each player, once started it connects to
the battletank server and provides a very crude terminal based graphic battlefield
and handles input from the player:

 - upon conenction,it  syncs with the server on the game state, receiving
   an index that uniquely identifies the player tank in the game state
 - the server continually broadcasts the game state to keep the clients in
   sync
 - clients will send actions to the servers such as movements or bullet fire
 - the server will update the general game state and let it be broadcast in
   the following cycle

##### Out of scope (for now)

The points above provide a very rudimentary interface to just see something work,
there are many improvements and limitations to be overcomed in the pure technical
aspect that are not yet handled, some of these in no particular order:

- screen size scaling: each client can have a different screen size, this makes it
  tricky to ensure a consistent experience between all the participants, in the
  current state of things, a lot of glitches are likely to happen due to this fact.
- clients disconnections and reconnections, reusing exising tanks if already
  instantiated
- heartbeat logic to ensure clients aliveness

These are all interesting challenges (well, probaly the heartbeat and proper
client tracking are less exciting, but the screen scaling is indeed
interesting) and some of these limitations may be address in an hypothetical
`battletank v0.0.2` depending on ispiration.

Moving on with the code, the first part of the client side requires some helpers to
handle the UI, as agreed, this is not gonna be a graphical game (yet?) so `ncurses`
provides very handy and neat functions to draw something basic on terminal. I don't
know much about the library itself but by the look of the APIs and their behaviour,
from my understanding of the docs it provides some nice wrappers around manipulation
of escape sequences for VT100 terminals and compatibles, similarly operating in raw
mode allowing for a fine-grained control over the keyboard input and such.

<hr>
**battletank_client.c**
<hr>

```c
static void init_screen(void) {
    // Start curses mode
    initscr();
    cbreak();
    // Don't echo keypresses to the screen
    noecho();
    // Enable keypad mode
    keypad(stdscr, TRUE);
    nodelay(stdscr, TRUE);
    // Hide the cursor
    curs_set(FALSE);
}

static void render_tank(const Tank *const tank) {
    if (tank->alive) {
        // Draw the tank at its current position
        mvaddch(tank->y, tank->x, 'T');
    }
}

static void render_bullet(const Bullet *const bullet) {
    if (bullet->active) {
        // Draw the bullet at its current position
        mvaddch(bullet->y, bullet->x, 'o');
    }
}

static void render_game(const Game_State *state) {
    clear();
    for (size_t i = 0; i < state->players_count; ++i) {
        render_tank(&state->players[i]);
        render_bullet(&state->players[i].bullet);
    }
    refresh();
}

static unsigned handle_input(void) {
    unsigned action = IDLE;
    int ch = getch();
    switch (ch) {
        case KEY_UP:
            action = UP;
            break;
        case KEY_DOWN:
            action = DOWN;
            break;
        case KEY_LEFT:
            action = LEFT;
            break;
        case KEY_RIGHT:
            action = RIGHT;
            break;
        case ' ':
            action = FIRE;
            break;
    }
    return action;
}
```
<hr>

In the last function `handle_input` the `unsigned action` returned will
be the main command we send to the server side (pretty simple huh? ample
margin to enrich this semantic).

Next in line comes the networking helpers, required to manage the communication
with the server side, connection, send and receive:

<hr>
**battletank_client.c**
<hr>

```c
static int socket_connect(const char *host, int port) {
    struct sockaddr_in serveraddr;
    struct hostent *server;
    struct timeval tv = {0, 10000};

    // socket: create the socket
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd < 0) goto err;

    setsockopt(sfd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(struct timeval));
    setsockopt(sfd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(struct timeval));

    // gethostbyname: get the server's DNS entry
    server = gethostbyname(host);
    if (server == NULL) goto err;

    // build the server's address
    bzero((char *)&serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    bcopy((char *)server->h_addr, (char *)&serveraddr.sin_addr.s_addr,
          server->h_length);
    serveraddr.sin_port = htons(port);

    // connect: create a connection with the server
    if (connect(sfd, (const struct sockaddr *)&serveraddr, sizeof(serveraddr)) <
        0)
        goto err;

    return sfd;

err:

    perror("socket(2) opening socket failed");
    return -1;
}

static int client_connect(const char *host, int port) {
    return socket_connect(host, port);
}

static int client_send_data(int sockfd, const char *data, size_t datasize) {
    ssize_t n = network_send(sockfd, data, datasize);
    if (n < 0) {
        perror("write() error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    return n;
}

static int client_recv_data(int sockfd, char *data) {
    ssize_t n = network_recv(sockfd, data);
    if (n < 0) {
        perror("read() error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    return n;
}

```
<hr>

All simple boilerplate code mostly, to handle a fairly traditional TCP
connection, the only bit that's interesting here is represented by the
lines

```c
setsockopt(sfd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(struct timeval));
setsockopt(sfd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(struct timeval));
```

These two lines ensure that the `read` system call times out after a certain
period, seeminly simulating a non-blocking socket behaviour (not really, but
the part that's interesting for us). This is the very first solution and the
simplest that came to mind but it allows to run a recv loop without blocking
indefinitely, as the server will constatly push updates, the client wants to be
as up-to-date as possible to keep rendering an accurate and consitent game
state.

This part happens in the `game_loop` function, a very slim and stripped down
client-side engine logic to render and gather inputs from the client to the
server:

<hr>
**battletank_client.c**
<hr>

```c
// Main game loop, capture input from the player and communicate with the game
// server
static void game_loop(void) {
    int sockfd = client_connect("127.0.0.1", 6699);
    if (sockfd < 0) exit(EXIT_FAILURE);
    Game_State state;
    game_state_init(&state);
    char buf[BUFSIZE];
    // Sync the game state for the first time
    int n = client_recv_data(sockfd, buf);
    protocol_deserialize_game_state(buf, &state);
    unsigned action = IDLE;

    while (1) {
        action = handle_input();
        if (action != IDLE) {
            memset(buf, 0x00, sizeof(buf));
            n = protocol_serialize_action(action, buf);
            client_send_data(sockfd, buf, n);
        }
        n = client_recv_data(sockfd, buf);
        protocol_deserialize_game_state(buf, &state);
        render_game(&state);
    }
}

int main(void) {
    init_screen();
    game_loop();
    endwin();
    return 0;
}
```
<hr>

The main function is as light as it gets, just initing the `ncurses` screen to
easily calculate `COLS` and `LINES` the straight to the game loop, with the
flow being:

- Connection to the server
- Sync of the game state, including other possibly already connected players
- Non blocking wait for input, if made, send it to the server to update the
  game state for everyone connected
- Receive data from the server, i.e. the game state, non blocking.

#### The server

The server side handles the game state and serves as the unique authoritative
source of truth.

- clients sync at their first connection and their tank is spawned in the
  battlefield, the server will send a unique identifier to the clients (an
  int index for the time being, that represents the tank assigned to the
  player in the game state)

As with the client, a bunch of communication helpers to handle the TCP
connections. The server will be a TCP non-blocking server (man `select` /
`poll` / `epoll` / `kqueue`), relying on `select` call to handle I/O events.
Select is not the most efficient mechanism for I/O multiplexing, it's in fact
quite dated, the first approach to the problem, it's a little quirky and among
other things it requires to linearly scan over all the monitored descriptors
each time an event is detected, it's also an user-space call, which adds a
minor over-head in context switching and it's limited to 1024 file descriptor
in total but:

- It's obiquitous, basically every *nix system provides the call
- It's very simple to use and provides everything required for a PoC
- It's more than enough for the use case, even with tenth of players
  it would handle the load very well, `poll` and `epoll` are really
  designe towards other scales, in the order of 10K of connected sockets.

<hr>
**battletank_server.c**
<hr>

```c
// We don't expect big payloads
#define BUFSIZE 1024
#define BACKLOG 128
#define TIMEOUT 70000

// Generic global game state
static Game_State game_state = {0};

/* Set non-blocking socket */
static int set_nonblocking(int fd) {
    int flags, result;
    flags = fcntl(fd, F_GETFL, 0);

    if (flags == -1) goto err;

    result = fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    if (result == -1) goto err;

    return 0;

err:

    fprintf(stderr, "set_nonblocking: %s\n", strerror(errno));
    return -1;
}

static int server_listen(const char *host, int port, int backlog) {
    int listen_fd = -1;
    const struct addrinfo hints = {.ai_family = AF_UNSPEC,
                                   .ai_socktype = SOCK_STREAM,
                                   .ai_flags = AI_PASSIVE};
    struct addrinfo *result, *rp;
    char port_str[6];

    snprintf(port_str, 6, "%i", port);

    if (getaddrinfo(host, port_str, &hints, &result) != 0) goto err;

    /* Create a listening socket */
    for (rp = result; rp != NULL; rp = rp->ai_next) {
        listen_fd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
        if (listen_fd < 0) continue;

        /* set SO_REUSEADDR so the socket will be reusable after process kill */
        if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &(int){1},
                       sizeof(int)) < 0)
            goto err;

        /* Bind it to the addr:port opened on the network interface */
        if (bind(listen_fd, rp->ai_addr, rp->ai_addrlen) == 0)
            break;  // Succesful bind
        close(listen_fd);
    }

    freeaddrinfo(result);
    if (rp == NULL) goto err;

    /*
     * Let's make the socket non-blocking (strongly advised to use the
     * eventloop)
     */
    (void)set_nonblocking(listen_fd);

    /* Finally let's make it listen */
    if (listen(listen_fd, backlog) != 0) goto err;

    return listen_fd;
err:
    return -1;
}

static int server_accept(int server_fd) {
    int fd;
    struct sockaddr_in addr;
    socklen_t addrlen = sizeof(addr);

    /* Let's accept on listening socket */
    fd = accept(server_fd, (struct sockaddr *)&addr, &addrlen);
    if (fd <= 0) goto exit;

    (void)set_nonblocking(fd);
    return fd;
exit:
    if (errno != EWOULDBLOCK && errno != EAGAIN) perror("accept");
    return -1;
}

static int broadcast(int *client_fds, const char *buf, size_t count) {
    int written = 0;
    for (int i = 0; i < FD_SETSIZE; i++) {
        if (client_fds[i] >= 0) {
            // TODO check for errors writing
            written += network_send(client_fds[i], buf, count);
        }
    }

    return written;
}
```
<hr>

Again, the main just initialise the `ncurses` screen (this is the reason why
the PoC will assume that the players will play from their own full size
terminal, as currently there is no scaling mechanism in place to ensure
consistency) and run the main `select` loop waiting for connections. Clients
are tracked in the simplest way possible by using an array and each new
connected client will be assigned its index in the main array as the index for
his tank in the game state.

<hr>
**battletank_server.c**
<hr>

```c
static void server_loop(int server_fd) {
    fd_set readfds;
    int client_fds[FD_SETSIZE];
    int maxfd = server_fd;
    int i = 0;
    char buf[BUFSIZE];
    struct timeval tv = {0, TIMEOUT};

    // Initialize client_fds array
    for (i = 0; i < FD_SETSIZE; i++) {
        client_fds[i] = -1;
    }

    while (1) {
        FD_ZERO(&readfds);
        FD_SET(server_fd, &readfds);

        for (i = 0; i < FD_SETSIZE; i++) {
            if (client_fds[i] >= 0) {
                FD_SET(client_fds[i], &readfds);
                if (client_fds[i] > maxfd) {
                    maxfd = client_fds[i];
                }
            }
        }
        memset(buf, 0x00, sizeof(buf));
        int num_events = select(maxfd + 1, &readfds, NULL, NULL, &tv);

        if (num_events == -1) {
            perror("select() error");
            exit(EXIT_FAILURE);
        }

        if (FD_ISSET(server_fd, &readfds)) {
            // New connection request
            int client_fd = server_accept(server_fd);
            if (client_fd < 0) {
                perror("accept() error");
                continue;
            }

            for (i = 0; i < FD_SETSIZE; i++) {
                if (client_fds[i] < 0) {
                    client_fds[i] = client_fd;
                    game_state.player_index = i;
                    break;
                }
            }

            if (i == FD_SETSIZE) {
                fprintf(stderr, "Too many clients\n");
                close(client_fd);
                continue;
            }

            printw("[info] New player connected\n");
            printw("[info] Syncing game state\n");
            printw("[info] Player assigned [%ld] tank\n",
                   game_state.player_index);

            // Spawn a tank in a random position for the new connected
            // player
            game_state_spawn_tank(&game_state, game_state.player_index);

            // Send the game state
            ssize_t bytes = protocol_serialize_game_state(&game_state, buf);
            bytes = network_send(client_fd, buf, bytes);
            if (bytes < 0) {
                perror("network_send() error");
                continue;
            }
            printw("[info] Game state sync completed (%d bytes)\n", bytes);
        }

        for (i = 0; i < FD_SETSIZE; i++) {
            int fd = client_fds[i];
            if (fd >= 0 && FD_ISSET(fd, &readfds)) {
                ssize_t count = network_recv(fd, buf);
                if (count <= 0) {
                    close(fd);
                    game_state_dismiss_tank(&game_state, i);
                    client_fds[i] = -1;
                    printw("[info] Player [%d] disconnected\n", i);
                } else {
                    unsigned action = 0;
                    protocol_deserialize_action(buf, &action);
                    printw(
                        "[info] Received an action %s from player [%d] (%ld "
                        "bytes)\n",
                        str_action(action), i, count);
                    game_state_update_tank(&game_state, i, action);
                    printw("[info] Updating game state completed\n");
                }
            }
        }
        // Main update loop here
        game_state_update(&game_state);
        size_t bytes = protocol_serialize_game_state(&game_state, buf);
        broadcast(client_fds, buf, bytes);
        // We're using ncurses for convienince to initialise ROWS and LINES
        // without going raw mode in the terminal, this requires a refresh to
        // print the logs
        refresh();
    }
}

int main(void) {
    srand(time(NULL));
    // Use ncurses as its handy to calculate the screen size
    initscr();
    scrollok(stdscr, TRUE);

    printw("[info] Starting server %d %d\n", COLS, LINES);
    game_state_init(&game_state);

    int server_fd = server_listen("127.0.0.1", 6699, BACKLOG);
    if (server_fd < 0) exit(EXIT_FAILURE);

    server_loop(server_fd);

    return 0;
}
```
<hr>

That's all folks, an extremely small and simple battletank should allow
multiple players to join and shoot single bullets. No collisions nor scores or
life points yet, but it's a starting point, in roughly 600 LOC:

```bash
battletank (main) $ ls *.[c,h] | xargx cloc
       8 text files.
       8 unique files.
       0 files ignored.

github.com/AlDanial/cloc v 2.02  T=0.02 s (521.2 files/s, 61767.2 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
C                                5            134            172            563
C/C++ Header                     3             17             10             52
-------------------------------------------------------------------------------
SUM:                             8            151            182            615
-------------------------------------------------------------------------------
```

#### References

- [ncurses](https://invisible-island.net/ncurses/)
- [Raylib](https://www.raylib.com/index.html)
- [SDL2](https://www.libsdl.org/)
- [setsockopt](https://linux.die.net/man/2/setsockopt)
- [select](https://man7.org/linux/man-pages/man2/select.2.html)
- [Select is fundamentally broken](https://idea.popcount.org/2017-01-06-select-is-fundamentally-broken/)
- [beej.us - serialization techniques](https://beej.us/guide/bgnet/html/split/slightly-advanced-techniques.html#serialization)
