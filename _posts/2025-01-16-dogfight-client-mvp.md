---
layout: post
title: "Dogfight! - Client MVP
description: "Old school multiplayer arcade game development journey"
categories: elixir networking c game-development low-level
---

In the previous parts the focus has been on laying some foundations for the
core logic of the game, there is still plenty to do and the code is barely
prototypical, no real test has been done yet but it's good enough to try
and show something on a terminal and a window, what we have so far:

- A TCP server accepting incoming connection and creating player contexts
- A game state to represent entities in a fixed size grid
    - Spaceships
    - Bullets
    - Power-ups (this not yet governed in practice but the concept is there)
- A rudimentary event system to apply changes and actions to the game state
- A binary based protocol to exchange payloads

The next bit is a client side to communicate with the server these information
and see it reflected based on the user input, we're gonna need

- A TCP client to connect to the server component
- An implementation of the binary protocol, this is gonna be from scratch as
  well as we're not using any structured library
- A graphic layer to display the state of the game
- A way to capture the user input and serialized it to be sent to the server

### Brief intro on the language of choice

I will use the original [battletank_client.c]() as a starting point, it's
pretty raw and simple but the logic is roughly the same. This time around
I will implement it in a different language than C.
I love C, it's my main go-to choice for low-level applications and, for how
many flaws and dangers and annoyances it poses, I can't help but find joy
in implementing stuff in it. So it was naturally my first choice here as
well, but after having implemented a basic version of the game already,
I decided to take the chance to give an honest try to some other new kids on
the block.

The choices were mostly driven by two requirements

- No garbage collection
- Simplicity, not too much frills, in the style of C

There are a growing number of so-called C-killer languages being shipped
consistently in the recent years, none of them really worthy of the title yet,
but some feel at least like a good start with some pretty nice design and
features, the shortlist was

- Zig
- Odin
- Hare

`Hare` fell immediately as a second contender, I don't know much of the language
itself but for some reason I wasn't impressed by its showcase, one cool point
is the QBF, felt nice to see a new language not relying on LLVM as a backend
but producing (and releasing for adoption on other compilers too) something
around 70% of LLVM with 90% of bloat less, pretty impressive feat.

The final choice has really been a toss of coin between `Odin` and `Zig`, both
provide a nice set of features and ports to `Raylib` (which was already the main
choice for the GUI layer). `Zig` seemed a little more mature and is showing some
impressive feats recently with some great products shipped in it (`TigerBeetle`,
`Bun` and more recently `Ghostty` are all great quality and high performance
software) plus it was slightly higher on my TO-TRY list.

### Starting the client

For the client side, we will start from where we left, probably not the best
choice but it somehow felt natural to begin the development of it by the main
connecting point, the protocol.

```zig
const std = @import("std");
const net = std.net;
const testing = std.testing;

const max_players: usize = 5;
const max_bullets: usize = 5;
const player_size: usize = @sizeOf(Player) + 36;
const power_up_size: usize = @sizeOf(PowerUp);

const Vector2D = struct {
    x: i32,
    y: i32,

    pub fn print(self: Vector2D, writer: anytype) !void {
        try writer.print("({d}, {d})", .{ self.x, self.y });
    }
};

pub const Direction = enum {
    idle,
    up,
    down,
    left,
    right,

    pub fn print(self: Direction, writer: anytype) !void {
        const dir_names = [_][]const u8{
            "idle", "up", "down", "left", "right",
        };
        try writer.print("{s}", .{dir_names[@intFromEnum(self)]});
    }
};

pub const Bullet = struct {
    position: Vector2D,
    direction: Direction,
    active: bool,

    pub fn print(self: Bullet, writer: anytype) !void {
        try writer.print("    position: ", .{});
        try self.position.print(writer);
        try writer.print(" Direction: ", .{});
        try self.direction.print(writer);
        try writer.print(" Active: {s}\n", .{if (self.active) "true" else "false"});
    }
};

pub const Player = struct {
    position: Vector2D,
    hp: i32,
    direction: Direction,
    alive: bool,
    bullets: [max_bullets]Bullet,

    pub fn print(self: Player, writer: anytype) !void {
        try writer.print("   position: ", .{});
        try self.position.print(writer);
        try writer.print("   HP: {d}\n", .{self.hp});
        try writer.print("   Direction: ", .{});
        try self.direction.print(writer);
        try writer.print(" Alive: {s}\n", .{if (self.alive) "true" else "false"});

        try writer.print("   Bullets:\n", .{});
        for (self.bullets) |bullet| {
            try bullet.print(writer);
        }
    }
};

const PowerUpKind = enum {
    none,
    hp_plus_onene,
    hp_plus_threeee,
    ammo_plus_onene,

    pub fn print(self: PowerUpKind, writer: anytype) !void {
        const kind_names = [_][]const u8{
            "none", "hp_plus_one", "hp_plus_three", "ammo_plus_one",
        };
        try writer.print("{s}", .{kind_names[@intFromEnum(self)]});
    }
};

pub const GameStatus = enum {
    in_progresss,
    closed,

    pub fn print(self: GameStatus, writer: anytype) !void {
        const status_names = [_][]const u8{ "In Progress", "Closed" };
        try writer.print("{s}", .{status_names[@intFromEnum(self)]});
    }
};

pub const PowerUp = struct {
    position: Vector2D,
    kind: PowerUpKind,

    pub fn print(self: PowerUp, writer: anytype) !void {
        try writer.print("position: ", .{});
        try self.position.print(writer);
        try writer.print(" Kind: ", .{});
        try self.kind.print(writer);
        try writer.print("\n", .{});
    }
};

pub const GameState = struct {
    players: std.StringHashMap(Player),
    status: GameStatus,
    power_ups: []PowerUp,

    pub fn print(self: GameState, writer: anytype) !void {
        try writer.print("GameState:\n", .{});
        try writer.print(" Power ups:\n", .{});
        for (self.power_ups) |power_up| {
            try power_up.print(writer);
        }

        try writer.print(" Players:\n", .{});
        var iterator = self.players.iterator();
        var player_index: usize = 0;
        while (iterator.next()) |entry| {
            try writer.print("  Player {d} - {s}:\n", .{ player_index, entry.key_ptr.* });
            try entry.value_ptr.print(writer);
            player_index += 1;
        }
    }
};

pub fn decode(buffer: []const u8, allocator: std.mem.Allocator) !GameState {
    var buffered_stream = std.io.fixedBufferStream(buffer);
    var reader = buffered_stream.reader();

    const total_length = try reader.readInt(i32, .big);
    const game_status = try reader.readByte();

    const power_ups = try decode_power_ups(reader, allocator);

    const usize_total_length: usize = @intCast(total_length);

    // Rough calculation of the players count based on the bytes already
    // read and the size expected for each player
    const players_count = (usize_total_length - (power_ups.len * power_up_size) - @sizeOf(u8) - @sizeOf(i32) - @sizeOf(i16)) / player_size;

    var players = std.StringHashMap(Player).init(allocator);
    for (0..players_count) |_| {
        var player = Player{
            .position = Vector2D{
                .x = try reader.readInt(i32, .big),
                .y = try reader.readInt(i32, .big),
            },
            .hp = try reader.readInt(u8, .big),
            .alive = try reader.readInt(u8, .big) != 0, // Deserialize bool as u8
            .direction = @enumFromInt(try reader.readInt(u8, .big)),
            .bullets = undefined,
        };

        var binary_player_id: [36]u8 = undefined;
        _ = try reader.readAll(&binary_player_id);

        const player_id = try std.fmt.allocPrint(allocator, "{s}", .{binary_player_id[0..]});

        var bullets: [max_bullets]Bullet = undefined;
        for (&bullets) |*bullet| {
            bullet.position = Vector2D{
                .x = try reader.readInt(i32, .big),
                .y = try reader.readInt(i32, .big),
            };
            bullet.active = try reader.readInt(u8, .big) != 0; // Deserialize bool as u8
            bullet.direction = @enumFromInt(try reader.readInt(u8, .big));
        }
        player.bullets = bullets; // Assign bullets array

        try players.put(player_id, player);
    }

    return GameState{ .players = players, .power_ups = power_ups, .status = @enumFromInt(game_status) };
}

fn decode_power_ups(reader: anytype, allocator: std.mem.Allocator) ![]PowerUp {
    const raw_size = try reader.readInt(i16, .big);
    const size = @divExact(raw_size, ((@sizeOf(i32) * 2) + @sizeOf(u8)));

    if (size < 0) {
        return error.InvalidSize; // Return an error for negative size
    }

    const usize_size: usize = @intCast(size);

    const power_ups = try allocator.alloc(PowerUp, usize_size);
    for (power_ups) |*power_up| {
        power_up.position = Vector2D{ .x = try reader.readInt(i32, .big), .y = try reader.readInt(i32, .big) };
        power_up.kind = @enumFromInt(try reader.readByte());
    }

    return power_ups;
}

```
Basically the `Zig` representation of the defined structures in the previous
part, enriched with a convenient debugging utility in the form of a simple
print function for each entity.

The gamestate is only one half of the puzzle, we must also define how to
interact with it, specifically how to send events, this part is crucial for the
client side.

```zig
const std = @import("std");
const gs = @import("gamestate.zig");

pub const PlayerId = [36]u8;

const move_u8: u8 = 2;
const shoot_u8: u8 = 3;

pub const Event = union(enum) {
    move: MoveEvent,
    shoot: PlayerId,

    pub const MoveEvent = struct {
        player_id: PlayerId,
        direction: gs.Direction,
    };
};

pub fn encode(event: Event, allocator: std.mem.Allocator) ![]u8 {
    return switch (event) {
        .move => |move_event| {
            const buffer = try allocator.alloc(u8, @sizeOf(u8) + @sizeOf(Event.MoveEvent));
            errdefer allocator.free(buffer); // Ensure buffer is freed on error

            var buffered_stream = std.io.fixedBufferStream(buffer);
            var writer = buffered_stream.writer();
            try writer.writeInt(u8, move_u8, .big);
            try writer.writeInt(u8, @intFromEnum(move_event.direction), .big);
            try writer.writeAll(&move_event.player_id);

            return buffer;
        },
        .shoot => |player_id| {
            const buffer = try allocator.alloc(u8, @sizeOf(u8) + @sizeOf(PlayerId));
            errdefer allocator.free(buffer); // Ensure buffer is freed on error

            var buffered_stream = std.io.fixedBufferStream(buffer);
            var writer = buffered_stream.writer();
            try writer.writeInt(u8, shoot_u8, .big);
            try writer.writeAll(&player_id);

            return buffer;
        },
    };
}
```

### TCP connection

Going fast here, the next step required is the connection piece to actually
talk with the server side, starting from some helper functions. Zig has a pretty
rich and well defined network layer so the resulting module is very straight
forward and self-contained.

What we need to support is a request-reply paradigm essentially

- A `receiveUpdate` function to read the serialized gamestate from the wire coming
  from the server
- A `sendUpdate` to interact and modify the received gamestate and update it
  on the server side

```zig
const std = @import("std");
const net = std.net;
const gs = @import("gamestate.zig");

const bufsize: usize = 1024;

pub fn connect(host: []const u8, port: comptime_int) !net.Stream {
    const peer = try net.Address.parseIp4(host, port);
    const stream = try net.tcpConnectToAddress(peer);
    return stream;
}

// TODO Placeholder, this will carry some additional logic going forward
pub fn handshake(stream: *const net.Stream, buffer: *[36]u8) !void {
    _ = try stream.reader().read(buffer);
}

pub fn receiveUpdate(stream: *const net.Stream, allocator: std.mem.Allocator) !gs.GameState {
    var buffer: [bufsize]u8 = undefined;

    _ = try stream.reader().read(&buffer);

    return gs.decode(&buffer, allocator);
}

pub fn sendUpdate(stream: *const net.Stream, buffer: []const u8) !void {
    try stream.writer().writeAll(buffer);
}
```

Besides the typical TCP stream features to connect, there is an `handshake`
function defined which will later on handle the handshake between the server
and clients, thinking about authentication, tokens and reconnection logic to
resume a game etc. For now it carries the simplest placeholder of reading what
comes first thing from the server, i.e. the generated player ID.

### Reading the first game state

TBD
