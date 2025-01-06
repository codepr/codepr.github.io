---
layout: post
title: "Dogfight! - Elixir server"
description: "Old school multiplayer arcade game development journey"
categories: elixir networking c game-development low-level
---

Starting with the server side of the game (code can be found in the [GitHub
repository](https://github.com/codepr/dogfight/tree/main/server)), Elixir is
pretty neat and allows for various ways to implement a distributed TCP server.
Going for the most simple and straight forward approach, leaving refinements
and polish for later. After more than 3 years of using Elixir full time, I
believe it's really one of the cleanest backend languages around, extremely
productive and convenient (pattern matching in Elixir slaps), there are two
points that make it stand-out and interesting to try for this little project:

1. **Concurrency and Scalability**: Built on the Erlang VM (BEAM), which is
   known for its ability to handle a large number of concurrent processes with
   low latency, a real outstanding piece of engineering created by the major
   telecom experts on earth. This makes Elixir highly suitable for applications
   that require high concurrency and scalability, such as real-time systems and
   web applications.

2. **Fault Tolerance**: Elixir inherits Erlang's "let it crash" philosophy,
   which encourages developers to write systems that can recover from failures
   automatically, this results in highly resilient applications that can continue
   to operate even when parts of the system fail. Each process operates in
   isolation and supervision trees allow for automatically recovery; potentially,
   it's possible to ship code really fast without the need of being too defensive
   and tune the system based on the evolution of the use-cases.

The main idea for the application is to write a highly concurrent and distributed
TCP based system to handle players, for the first iteration:

- A TCP entry point that acts as an acceptor process, its only purpose will be
  to accept new connecting clients and spawn process inside the cluster to handle
  the lifetime of the connection (i.e. the game)
- `GenServer` is the main abstraction (kinda low level but easy to use for a first
  run) that will be used to represent
    - A game server, or better, an event handler, this will be the entity representing
      the main source of truth for connected clients, it's basically an instantiated
      game. For the first iteration we will have a single running game server;
      subsequently, it will extended to be spawned on-demand as part of a hosting game
      logic from players requesting it
    - Each connecting player, once the acceptor complete the handshake, it
      will demand the managing of the client to its own process
- The main structures exchanged between server and clients will be
    - A game state, will contain all the players, initially capped to a maximum
      to keep things simple, and each player will have a fixed amount of projectiles
      to shoot
    - Events, which may be directional in nature, such as
        - `MOVE UP`
        - `MOVE DOWN`
        - `MOVE LEFT`
        - `MOVE RIGHT`
        - `SHOOT` (this will follow the direction of the projectile, which in turns
           will coincide with the player direction)
        - `SPAWN POWER UP`
      And so on
- Each player process and game process will be spawned inside a cluster on multiple
  nodes, with location transparency and message passing this will be a breeze

## A very basic TCP server acceptor

We're gonna use `:gen_tcp` from the builtin `Erlang` library, starting the
listen loop in a dedicated `Task` is enough for the time being, management of
newly connected clients will be demanded to their own proper `GenServer`.

Let's start with something really bare-bone, basically a slight variation of
the [official docs](https://hexdocs.pm/elixir/task-and-gen-tcp.html) example is
all we need to get started

```elixir
defmodule Dogfight.Server do
  @moduledoc """

  Main entry point of the server, accepts connecting clients and defer their
  ownership to the game server. Each connected client is handled by a
  `GenServer`, the `Dogfight.Player`. The `Dogfight.Game.EventHandler` governs
  the main logic of an instantiated game and broadcasts updates to each
  registered player.

  """
  require Logger

  alias Dogfight.Game.Event, as: GameEvent
  alias Dogfight.ClusterServiceSupervisor

  def listen(port) do
    # The options below mean:
    #
    # 1. `:binary` - receives data as binaries (instead of lists)
    # 2. `packet: :raw` - receives data as it comes (stream of binary over the wire)
    # 3. `active: true` - non blocking on `:gen_tcp.recv/2`, doesn't wait for data to be available
    # 4. `reuseaddr: true` - allows us to reuse the address if the listener crashes
    #
    {:ok, socket} =
      :gen_tcp.listen(port, [:binary, packet: :raw, active: true, reuseaddr: true])

    Logger.info("Accepting connections on port #{port}")
    accept_loop(socket)
  end

  defp accept_loop(socket) do
    with {:ok, client_socket} <- :gen_tcp.accept(socket),
         player_id <- UUID.uuid4(),
         player_spec <- player_spec(client_socket, player_id),
         {:ok, pid} <- Horde.DynamicSupervisor.start_child(ClusterServiceSupervisor, player_spec) do
      Dogfight.Game.EventHandler.apply_event(pid, GameEvent.player_connection(player_id))
      :gen_tcp.controlling_process(client_socket, pid)
    else
      error -> Logger.error("Failed to accept connection, reason: #{inspect(error)}")
    end

    accept_loop(socket)
  end

  defp player_spec(socket, player_id) do
    %{
      id: Dogfight.Player,
      start: {Dogfight.Player, :start_link, [player_id, socket]},
      type: :worker,
      restart: :transient
    }
  end
end
```

The main interesting points are
- `accept_loop/1` is a recursive call, once a new client connect it dispatches it to a process and
  call itself again
- if the accept call succeed, the first thing we do is to generate an UUID. This part is currently
  very brittle, no protocol defined to establish an handshake yet, but later on what we expect to do
  is a sort of authentication, with a token session state to handle reconnections and security concerns,
  not a priority at the moment.
- After the ID generation, we're gonna use `Horde` [dynamic supervisor](https://hexdocs.pm/elixir/DynamicSupervisor.html)
  to spawn processes that are automatically cluster-aware, the node where they will be spawned is completely
  abstracted away and doesn't matter. (For a simpler solution we could even manually start the `GenSever` with a
  call to `start_link/1`, it's really that simple).
- The `Dogfight.Game.EventHandler.register_player/2` call basically add the `pid` of the newly instantiated process
  to the existing game server (as explained above, there will be a single global game server for the first version). This
  assumes that the game server is already running here, started at application level already.
- Finally, we transfer the ownership of the connected socket to the newly spawned process through `:gen_tcp.controlling_process/2`

## The player process

After the accept has been successfully performed, the `player_spec/2` call
provides all that's required for the `Horde.DynamicSupervisor` to spawn a
`Dogfight.Player` process somewhere in the cluster. Conceptually it's nothing
more than a connection state, and thus it's pretty trivial. The registration of
each PID to the game server can be a little cumbersome later on, and it may be
a good opportunity to explore some pub-sub based solution such as the one shipped
by [Phoenix](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html); for the time
being it's ok to keep things simple.

```elixir
defmodule Dogfight.Player do
  @moduledoc """
  This module represents a player in the Dogfight game. It handles the player's
  connection, actions, and communication with the game server.
  """

  require Logger
  use GenServer

  alias Dogfight.Game.Codecs.BinaryCodec
  alias Dogfight.Game.Event, as: GameEvent

  def start_link(player_id, socket) do
    GenServer.start_link(__MODULE__, {player_id, socket})
  end

  def init({player_id, socket}) do
    Logger.info("Player #{player_id} connected, registering to game server")
    {:ok, %{player_id: player_id, socket: socket}}
  end

  def handle_info({:tcp, _socket, data}, state) do
    with {:ok, event} <- BinaryCodec.decode_event(data) do
      Dogfight.Game.EventHandler.apply_event(self(), event)
    else
      {:ok, :codec_error} -> Logger.error("Decode failed, unknown event")
    end

    {:noreply, state}
  end

  def handle_info({:tcp_closed, _socket}, state) do
    Logger.info("Player #{state.player_id} disconnected")

    Dogfight.Game.EventHandler.apply_event(
      self(),
      GameEvent.player_disconnection(state.player_id)
    )

    {:noreply, state}
  end

  def handle_info({:tcp_error, _socket, reason}, state) do
    Logger.error("Player transmission error #{inspect(reason)}")
    {:noreply, state}
  end

  def handle_info({:update, game_state}, state) do
    send_game_update(state.socket, game_state)
    {:noreply, state}
  end

  defp send_game_update(socket, game_state) do
    :gen_tcp.send(socket, BinaryCodec.encode(game_state))
  end
end
```

Most of the handlers are the typical ones for a bare bone TCP connection, the
only real addition is the `send_game_update/2` call which forward a message to
the connected client, in this case the binary encoded game state as the
payload.

The main TCP handler, is responsible for receiving packets from the client, in
this first iteration, it's main responsibility will be to just decode the
payload into an event to be applied to the game state; currently there is no
definition of this component, it will be the main topic of the next section.

## A little event handling system

As explained above, the game handling will be designed as a sort of event
queue, where each event will influence the state of the game; naturally, the
first brick required is the definition of what an event is, in this game,
events could include things like

- player connections and/or drop
- player movements
- shooting
- power-up generation and placement
- power-up collection

First thing, we're gonna define a very simple set of events that we can extend
later

```elixir

defmodule Dogfight.Game.Event do
  @moduledoc """
  This structure represents any game event to be applied to a game state to
  transition to a new state
  """

  alias Dogfight.Game.State

  @type t ::
          {:player_connection, State.player_id()}
          | {:player_disconnection, State.player_id()}
          | {:move, {State.player_id(), State.direction()}}
          | {:shoot, State.player_id()}
          | {:pickup_power_up, State.player_id()}
          | {:spawn_power_up, State.power_up_kind()}
          | {:start_game}
          | {:end_game}

  def player_connection(player_id), do: {:player_connection, player_id}
  def player_disconnection(player_id), do: {:player_disconnection, player_id}
  def move(player_id, direction), do: {:move, {player_id, direction}}
  def shoot(player_id), do: {:shoot, player_id}
end
```

### The game event handler

Now that we have a small set of events, we want to process them. The event
handler is the main process that govern the state of a game, it is in essence a
match which can be hosted by one of the player to invite others to join. At the
moment this logic is yet to be implemented and it's essentially skipped, there
is a single game event handler started as part of the application startup.

At the core, it's basically an event queue, events received by connected clients as
well as time based generated for the natural progression of the game are handled here,
applying them to the associated `Game.State` will provide new updated versions of the
game.

The main responsibilities of the `GenServer` is to provide a single source of truth to
all the connected clients:

- Each connecting player, after having received an identifier, are registered to the
  game server (this happens in the acceptor TCP server via the `register_player/2` call)
- Once a player is registered, a ship is spawned in a random position for them via the
  `GameState.add_player/2` for the given ID.
- Broadcast periodic updates of the game state to all connected clients to keep updating
  the GUI of each of them (currently set at 50 ms by `@tick_rate_ms`, but it's not really
  important now)

```elixir
defmodule Dogfight.Game.EventHandler do
  @moduledoc """
  Represents a game server for the Dogfight game. It handles the game state and
  player actions. It is intended as the main source of thruth for each instantiated game,
  broadcasting the game state to each connected player every `@tick` milliseconds.
  """
  require Logger
  use GenServer

  alias Dogfight.Game.State, as: GameState

  @tick_rate_ms 50

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  @impl true
  def init(_) do
    game_state = GameState.new()
    schedule_tick()
    {:ok, %{players: %{}, game_state: game_state}}
  end

  def register_player(pid, player_id) do
    GenServer.call(__MODULE__, {:register_player, pid, player_id})
    GenServer.cast(__MODULE__, {:add_player, player_id})
  end

  def apply_event(pid, event) do
    case event do
      {:player_connection, player_id} ->
        GenServer.call(__MODULE__, {:register_player, pid, player_id})
        GenServer.cast(__MODULE__, {:add_player, player_id})

      {:player_disconnection, player_id} ->
        GenServer.call(__MODULE__, {:unregister_player, player_id})
        GenServer.cast(__MODULE__, {:drop_player, player_id})

      event ->
        GenServer.cast(__MODULE__, {:apply_event, pid, event})
    end
  end

  defp schedule_tick() do
    Process.send_after(self(), :tick, @tick_rate_ms)
  end

  @impl true
  def handle_info(:tick, state) do
    new_game_state = GameState.update(state.game_state)
    updated_state = %{state | game_state: new_game_state}
    broadcast_game_state(updated_state)
    schedule_tick()
    {:noreply, updated_state}
  end

  @impl true
  def handle_call({:register_player, pid, player_id}, from, state) do
    new_state =
      Map.update!(state, :players, fn players ->
        Map.put(players, player_id, pid)
      end)

    {:reply, from, new_state}
  end

  @impl true
  def handle_call({:unregister_player, player_id}, from, state) do
    new_state =
      Map.update!(state, :players, fn players ->
        Map.delete(players, player_id)
      end)

    {:reply, from, new_state}
  end

  @impl true
  def handle_cast({:add_player, player_id}, state) do
    game_state =
      case GameState.add_player(state.game_state, player_id) do
        {:ok, game_state} ->
          broadcast_game_state(%{state | game_state: game_state})
          game_state

        {:error, error} ->
          Logger.error("Failed spawining spaceship, reason: #{inspect(error)}")
          state.game_state
      end

    {:noreply, %{state | game_state: game_state}}
  end

  @impl true
  def handle_cast({:drop_player, player_id}, state) do
    {:noreply, %{state | game_state: GameState.drop_player(state.game_state, player_id)}}
  end

  @impl true
  def handle_cast({:apply_event, _pid, event}, state) do
    game_state = GameState.apply_event(state.game_state, event)
    updated_state = %{state | game_state: game_state}
    broadcast_game_state(updated_state)
    {:noreply, updated_state}
  end

  defp broadcast_game_state(state) do
    Enum.each(Map.values(state.players), fn pid ->
      send(pid, {:update, state.game_state})
    end)
  end
end
```

The `init/1` call is pretty cheap, no need to add an `handle_continue/2` as it
won't really block. To be noted that the registration of a new player, performs
a call and a cast. That because we want to make sure the state is updated
before spawning the ship for the new player. Also there is no check yet for
already existing ships, disconnections and reconnection etc. For all the
unexpected cases we just log error or crash, it will be for a second pass to
actually clean up and harden the code. It's pretty simple and straight-forward.

## The game state

The game state represents the main entities present in a game, it's the main
payload of information exchanged between clients and server to keep all the
players in sync.

It is indeed pretty simple and definitely not optimized at the moment, there's
a number of things that can be improved and done differently, some of the
fields are a carry over from the original C implementation which I'm gonna use
as the main test-bed, but the aim is to then update the state for a better v2
representation. For the sake of simplicity, the first version will carry the
following entities:

- A set of players active in the game, for simplicity it will be a simple map
  `player_id -> spaceship`
  - Each player's spaceship is represented by
      - **position:** a vector 2D x and y representing the position in the screen
      - **hp:** just health points, initially set at 5
      - **direction:** self-explanatory, this can be (not exactly precise but work for a prototype)
          - `:idle | :up | :down | :left | :right`
      - **alive?:** an alive boolean flag
      - **bullets:** set to a maximum of 5, composed by
          - **position:** a vector 2D x and y representing the position in the screen
          - **direction:** again, a direction, this will be aligned with the player direction
          - **active?:** an active boolean flag
- **power_ups:** these will be spawned randomly on a time-basis, made of
    - **position:** a vector 2D x and y representing the position in the screen
    - **kind:** an atom representing the type of effect it will have on the player that
      captures it, initially can be
        - `:hp_plus_one | :hp_plus_three | :ammo_plus_one | nil`

At the moment we're not gonna handle scaling and screen size for each player, it is
initially assumed that each player will play in a small 800x600 window.

### The main entity, spaceship

Before we create the `Game.State` module, it may be a good idea to define a
simple behaviour for the spaceship: generally, I tend to be against
abstractions and go for the most concrete implementation first, but I have the
feeling the spaceship will be one of the things that will be easily generated
with different characteristics. It's a small interface anyway, in Elixir the
coupling is not as prohibitive as other languages anyway.

```elixir

defmodule Dogfight.Game.Spaceship do
  @moduledoc """
  Spaceship behaviour
  """

  alias Dogfight.Game.State

  @type t :: any()

  @callback move(t(), State.direction()) :: t()
  @callback shoot(t()) :: t()
  @callback update_bullets(t()) :: t()
end
```

The main traits we're interested in any given spaceship are

- The move logic, think about slow heavily-armed spaceship or quick smaller fighters etc.
- The shoot logic, again, heavy hitters, barrage rockets or multi-directional machine guns.
- The update bullets logic, fast bullets, lasers, or slow big boys etc.

A first simple implementation, something like a no-frills no-fancy default
spaceship, the `DefaultSpaceship` follows

```elixir
defmodule Dogfight.Game.DefaultSpaceship do
  @moduledoc """
  DefaultSpaceship module, represents the main entity of each player, a default
  spaceship without any particular trait, with a base speed of 3 units and base HP
  of 5.
  """

  defmodule Bullet do
    @moduledoc """
    Bullet inner module, represents a bullet of the spaceship entity
    """

    @type t :: %__MODULE__{
            position: Vec2.t(),
            direction: State.direction(),
            active?: boolean()
          }

    defstruct [:position, :direction, :active?]

    alias Dogfight.Game.State
    alias Dogfight.Game.Vec2

    @bullet_base_speed 6

    def new do
      %__MODULE__{
        position: %Vec2{x: 0, y: 0},
        direction: State.idle(),
        active?: false
      }
    end

    def update(%{active?: false} = bullet), do: bullet

    def update(bullet) do
      position = bullet.position

      case bullet.direction do
        :up ->
          %{bullet | position: Vec2.add_y(position, -@bullet_base_speed)}

        :down ->
          %{bullet | position: Vec2.add_y(position, @bullet_base_speed)}

        :left ->
          %{bullet | position: Vec2.add_x(position, -@bullet_base_speed)}

        :right ->
          %{bullet | position: Vec2.add_x(position, @bullet_base_speed)}

        _ ->
          bullet
      end
    end
  end

  @behaviour Dogfight.Game.Spaceship

  alias Dogfight.Game.State
  alias Dogfight.Game.Bullet
  alias Dogfight.Game.Vec2

  @type t :: %__MODULE__{
          position: Vec2.t(),
          hp: non_neg_integer(),
          direction: State.direction(),
          alive?: boolean(),
          bullets: [Bullet.t(), ...]
        }

  defstruct [:position, :hp, :direction, :alive?, :bullets]

  @base_hp 5
  @base_bullet_count 5
  @base_spaceship_speed 3

  def spawn(width, height) do
    %__MODULE__{
      position: Vec2.random(width, height),
      direction: State.idle(),
      hp: @base_hp,
      alive?: true,
      bullets: Stream.repeatedly(&__MODULE__.Bullet.new/0) |> Enum.take(@base_bullet_count)
    }
  end

  @impl true
  def move(spaceship, direction) do
    position = spaceship.position

    updated_position =
      case direction do
        :up -> Vec2.add_y(position, -@base_spaceship_speed)
        :down -> Vec2.add_y(position, @base_spaceship_speed)
        :left -> Vec2.add_x(position, -@base_spaceship_speed)
        :right -> Vec2.add_x(position, @base_spaceship_speed)
      end

    %{spaceship | direction: direction, position: updated_position}
  end

  @impl true
  def shoot(spaceship) do
    bullets =
      spaceship.bullets
      |> Enum.map_reduce(false, fn
        bullet, false when bullet.active? == false ->
          {
            %{
             bullet
             | active?: true,
               direction: spaceship.direction,
               position: spaceship.position
           }, true}

        bullet, updated ->
          {bullet, updated}
      end)
      |> elem(0)

    %{
      spaceship
      | bullets: bullets
    }
  end

  @impl true
  def update_bullets(%{alive?: false} = spaceship), do: spaceship

  @impl true
  def update_bullets(spaceship) do
    %{spaceship | bullets: Enum.map(spaceship.bullets, &__MODULE__.Bullet.update/1)}
  end
end
```

As shown, it's a rather basic implementation that does very little, but
everything we expect from a spaceship nonetheless:

- Base HP of 5 units, each bullet will deal 1 unit of damage to begin
- Base bullet count of 5, power ups may provide additional ammos, bullets are meant to
  automatically replenish over time
- A base movement speed of 3 units, this will be largely dependent on the client FPS
- An internal representation of the most elementary bullet with a base speed double of
  the ship

As a final side-note, there is a `Vec2` utility module called there, we will omit
its implementation as it's pretty straight forward and very short, it does exactly
what it looks like, vector 2D capabilities such as

- spawn of a random vector
- sum of 2 vectors
- sum of a constant on both coordinates

We finally define the main entity, this will carry the main piece of information
to keep all the client in sync:

```elixir
defmodule Dogfight.Game.State do
  @moduledoc """
  Game state management, Ships and bullet logics, including collision.
  Represents the source of truth for each connecting player, and its updated
  based on the input of each one of the connected active players
  """

  alias Dogfight.Game.Event
  alias Dogfight.Game.Spaceship
  alias Dogfight.Game.DefaultSpaceship
  alias Dogfight.Game.Vec2

  @type player_id :: String.t()

  @type power_up_kind :: :hp_plus_one | :hp_plus_three | :ammo_plus_one | nil

  @type power_up :: %{
          position: Vec2.t(),
          kind: power_up_kind()
        }

  @type direction :: :idle | :up | :down | :left | :right

  @typep status :: :in_progress | :closed | nil

  @type t :: %__MODULE__{
          players: %{player_id() => Spaceship.t()},
          power_ups: [power_up()],
          status: status()
        }

  defstruct [:players, :power_ups, :status]

  @screen_width 800
  @screen_height 600

  def new do
    %__MODULE__{
      power_ups: [],
      status: :closed,
      players: %{}
    }
  end

  @spec add_player(t(), player_id()) :: {:ok, t()} | {:error, :dismissed_ship}
  def add_player(game_state, player_id) do
    case Map.get(game_state.players, player_id) do
      nil ->
        players =
          Map.put(
            game_state.players,
            player_id,
            DefaultSpaceship.spawn(@screen_width, @screen_height)
          )

        {:ok, %{game_state | players: players}}

      %{alive?: true} ->
        {:ok, game_state}

      _spaceship ->
        {:error, :dismissed_ship}
    end
  end

  @spec drop_player(t(), player_id()) :: t()
  def drop_player(game_state, player_id) do
    Map.delete(game_state, player_id)
  end

  @spec update(t()) :: t()
  def update(game_state) do
    %{
      game_state
      | players:
          Map.new(game_state.players, fn {player_id, spaceship} ->
            {player_id, DefaultSpaceship.update_bullets(spaceship)}
          end)
    }
  end

  defp fetch_spaceship(players_map, player_id) do
    case Map.fetch(players_map, player_id) do
      :error -> {:error, :dismissed_ship}
      %{alive?: false} -> {:error, :dismissed_ship}
      {:ok, _spaceship} = ok -> ok
    end
  end

  @spec apply_event(t(), Event.t()) :: {:ok, t()} | {:error, :dismissed_ship}
  def apply_event(game_state, {:move, {player_id, direction}}) do
    with {:ok, spaceship} <- fetch_spaceship(game_state.players, player_id) do
      {:ok,
       %{
         game_state
         | players:
             Map.put(game_state.players, player_id, DefaultSpaceship.move(spaceship, direction))
       }}
    end
  end

  def apply_event(game_state, {:shoot, player_id}) do
    with {:ok, spaceship} <- fetch_spaceship(game_state.players, player_id) do
      {:ok,
       %{
         game_state
         | players: Map.put(game_state.players, player_id, DefaultSpaceship.shoot(spaceship))
       }}
    end
  end

  def apply_event(game_state, {:spawn_power_up, power_up_kind}) do
    power_up = %{position: Vec2.random(@screen_width, @screen_height), kind: power_up_kind}

    {:ok,
     %{
       game_state
       | power_ups: [power_up | game_state.power_ups]
     }}
  end

  def apply_event(_game_state, _event), do: raise("Not implemented")

  def idle, do: :idle
  def move_up, do: :up
  def move_down, do: :down
  def move_left, do: :left
  def move_right, do: :right
  def shoot, do: :shoot
end
```

A small set of functions to manage a basic game state:

- `new/0` just creates an empty, or base state
- `add_player/2` called by the `EventHandler`, it's triggered on connect when a new
  player connects
- `apply_event/2` called by the `EventHandler` when any event coming from players
  or scheduled on a time-basis
- `update/1` this is a kind of engine for the state, it's called on a time-basis
  by the `EventHandler` and it allows the state to naturally evolve based on
  it's current configuration; for example, if there are 3 bullets active across
  2 players that fired, this function will update their position according to their
  speed (and possibly acceleration/deceleration later on, if we feel like adding that
  logic too)

We will need to update the main entry point of the program, the `Application` to
correctly start a supervision tree including the `Server` and the `EventHandler`.

```elixir

defmodule Dogfight.Application do
  @moduledoc """
  Documentation for `Dogfight`.
  Server component for the Dogfight battle arena game, developed to have
  some fun with game dev and soft-real time distributed systems.
  """

  use Application

  def start(_type, _args) do
    port = String.to_integer(System.get_env("DOGFIGHT_TCP_PORT") || "6699")

    children = [
      {Cluster.Supervisor, [topologies(), [name: Dogfight.ClusterSupervisor]]},
      {
        Horde.Registry,
        name: Dogfight.ClusterRegistry, keys: :unique, members: :auto
      },
      {
        Horde.DynamicSupervisor,
        name: Dogfight.ClusterServiceSupervisor, strategy: :one_for_one, members: :auto
      },
      Dogfight.Game.EventHandler,
      Supervisor.child_spec({Task, fn -> Dogfight.Server.listen(port) end}, restart: :permanent)
    ]

    opts = [strategy: :one_for_one, name: Dogfight.Supervisor]
    Supervisor.start_link(children, opts)
  end

  # Can also read this from conf files, but to keep it simple just hardcode it for now.
  # It is also possible to use different strategies for autodiscovery.
  # Following strategy works best for docker setup we using for this app.
  defp topologies do
    [
      game_state_nodes: [
        strategy: Cluster.Strategy.Epmd,
        config: [
          hosts: [:"app@n1.dev", :"app@n2.dev", :"app@n3.dev"]
        ]
      ]
    ]
  end
end
```
Code can be found in the [GitHub repository](https://github.com/codepr/dogfight/tree/main/server).

#### References

- [BEAM](https://www.erlang.org/blog/a-brief-beam-primer/)
- [GenServer](https://hexdocs.pm/elixir/GenServer.html)
- [horde](https://hexdocs.pm/horde/getting_started.html)
- [libcluster](https://hexdocs.pm/libcluster/readme.html)
