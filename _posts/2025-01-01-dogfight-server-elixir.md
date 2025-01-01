---
layout: post
title: "Dogfight! - Elixir server"
description: "Old school multiplayer arcade game development journey"
categories: elixir networking c game-development low-level
---

Starting with the server side of the game, Elixir is pretty neat and allows for
various ways to implement a distributed TCP server. Going for the most simple
and straight forward approach, leaving refinements and polish for later. After
more than 3 years of using Elixir full time, I believe it's really one of the
cleanest backend languages around, extremely productive and convenient (pattern
matching in Elixir slaps), there are two points that make it stand-out and
interesting to try for this little project:

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
    - A game server, this will be the entity representing the main source of
      truth for connected clients, it's basically an instantiated game. For
      the first iteration we will have a single running game server; subsequently,
      it will extended to be spawned on-demand as part of a hosting game logic
      from players requesting it
    - Each connecting player, once the acceptor complete the handshake, it
      will demand the managing of the client to its own process
- The main structures exchanged between server and clients will be
    - A game state, will contain all the players, initially capped to a maximum
      to keep things simple, and each player will have a fixed amount of projectiles
      to shoot
    - Actions, all of which will be directional in nature, such as
        - `MOVE UP`
        - `MOVE DOWN`
        - `MOVE LEFT`
        - `MOVE RIGHT`
        - `SHOOT` (this will follow the direction of the projectile, which in turns
           will coincide with the player direction)
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
  `GenServer`, the `Dogfight.Player`. The `Dogfight.Game.Server` governs the
  main logic of an instantiated game and broadcasts updates to each registered
  player.
  """
  require Logger

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
         player_id <- Dogfight.IndexAgent.next_id(),
         player_spec <- player_spec(client_socket, player_id),
         {:ok, pid} <- Horde.DynamicSupervisor.start_child(ClusterServiceSupervisor, player_spec) do
      Logger.info("Player #{player_id} connected")
      Dogfight.Game.Server.register_player(pid, player_id)
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
- if the accept call succeed, the first thing we do is to generate a basic ID (using a monotonic integer
  for the time being, this to keep compatibility with the original implementation of `battletank`). This is
  achieved with a very simple [Agent](https://hexdocs.pm/elixir/agents.html) (a builtin abstraction on top
  of the `GenServer` for handy state management).

  Something similar is more than enough to start

  ```elixir
    defmodule Dogfight.IndexAgent do
      @moduledoc """
      Simple agent to generate monotonic increasing indexes for connecting players,
      they will be used as player ids as a first extremely basic iteration, later we
      may prefer to adopt some proper UUID.
      """
      use Agent

      def start_link(_initial_value) do
        Agent.start_link(fn -> 0 end, name: __MODULE__)
      end

      def next_id do
        Agent.get_and_update(__MODULE__, fn index ->
          {index, index + 1}
        end)
      end
    end
    ```
- After the ID generation, we're gonna use `Horde` [dynamic supervisor](https://hexdocs.pm/elixir/DynamicSupervisor.html)
  to spawn processes that are automatically cluster-aware, the node where they will be spawned is completely
  abstracted away and doesn't matter. (For a simpler solution we could even manually start the `GenSever` with a
  call to `start_link/1`, it's really that simple).
- The `Dogfight.Game.Server.register_player/2` call basically add the `pid` of the newly instantiated process
  to the existing game server (as explained above, there will be a single global game server for the first version). This
  assumes that the game server is already running here, started at application level already.
- Finally, we transfer the ownership of the connected socket to the newly spawned process through `:gen_tcp.controlling_process/2`

## The player process

After the accept has been successfully performed, the `player_spec/2` call
provides all that's required to the `Horde.DynamicSupervisor` to spawn a
`Dogfight.Player` process somewhere in the cluster. Conceptually it's nothing
more then a connection state, and thus it's pretty trivial:

```elixir
defmodule Dogfight.Player do
  @moduledoc """
  This module represents a player in the Dogfight game. It handles the player's
  connection, actions, and communication with the game server.
  """

  require Logger
  use GenServer

  alias Dogfight.Game.State, as: GameState
  alias Dogfight.Game.Action, as: GameAction

  def start_link(player_id, socket) do
    GenServer.start_link(__MODULE__, {player_id, socket})
  end

  def init({player_id, socket}) do
    Logger.info("Player connected, registering to game server")
    {:ok, %{player_id: player_id, socket: socket}}
  end

  def handle_info({:tcp, _socket, data}, state) do
    action = GameAction.decode!(data)
    Dogfight.Game.Server.apply_action(self(), action, state.player_id)
    {:noreply, state}
  end

  def handle_info({:tcp_closed, _socket}, state) do
    Logger.info("Player disconnected")
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
    :gen_tcp.send(socket, GameState.encode(game_state))
  end
end
```
Most of the handlers are the typical ones for a barebone TCP connection, the only
real addition is the `send_game_update/2` call which forward a message to the
connected client, in this case the binary encoded game state as the payload.

## The game server

This is the main process that govern the state of a game, it is in essence a match
which can be hosted by one of the player to invite others to join. At the moment
this logic is yet to be implemented and it's essentially skipped, there is a single
game server started as part of the application startup.

The main responsibities of the `GenServer` is to provide a single source of truth to
all the connected clients:

- Each connecting player, after having received an identifier, are registered to the
  game server (this happens in the acceptor TCP server via the `register_player/2` call)
- Once a player is registered, a ship is spawned in a random position for them via the
  `GameState.spawn_ship/2` for the given ID.
- Broadcast periodic updates of the game state to all connected clients to keep updating
  the GUI of each of them (currently set at 50 ms by `@tick_rate_ms`, but it's not really
  important now)

```elixir
defmodule Dogfight.Game.Server do
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
    {:ok, %{players: [], game_state: game_state}}
  end

  def register_player(pid, player_id) do
    GenServer.call(__MODULE__, {:register_player, pid})
    GenServer.cast(__MODULE__, {:spawn_new_ship, player_id})
  end

  def apply_action(pid, action, player_index) do
    GenServer.cast(__MODULE__, {:apply_action, pid, action, player_index})
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
  def handle_call({:register_player, pid}, from, state) do
    new_state =
      Map.update!(state, :players, fn players -> [{:player_id, %{pid: pid}} | players] end)

    {:reply, from, new_state}
  end

  @impl true
  def handle_cast({:spawn_new_ship, player_id}, state) do
    game_state =
      case GameState.spawn_ship(state.game_state, player_id) do
        {:ok, game_state} ->
          broadcast_game_state(%{state | game_state: game_state})
          game_state

        {:error, error} ->
          Logger.error("Failed spawining ship, reason: #{inspect(error)}")
          state.game_state
      end

    {:noreply, %{state | game_state: game_state}}
  end

  @impl true
  def handle_cast({:apply_action, _pid, action, player_index}, state) do
    game_state = GameState.apply_action(state.game_state, action, player_index)
    updated_state = %{state | game_state: game_state}
    broadcast_game_state(updated_state)
    {:noreply, updated_state}
  end

  defp broadcast_game_state(state) do
    Enum.each(state.players, fn {_pid, player} ->
      send(player.pid, state.game_state)
    end)
  end
end
```

The `init/1` call is pretty cheap, no need to add an `handle_continue/2` as it
won't really block. To be noted that the registration of a new player, performs
a call and a cast. That because we want to make sure the state is updated
before spawning the ship for the new player. Also there is no check yet for
alreay existing ships, disconnections and reconnections etc. It's pretty simple
and straight-forward.

## The game state

TBD

#### References

- [BEAM](https://www.erlang.org/blog/a-brief-beam-primer/)
- [GenServer](https://hexdocs.pm/elixir/GenServer.html)
- [horde](https://hexdocs.pm/horde/getting_started.html)
- [libcluster](https://hexdocs.pm/libcluster/readme.html)
