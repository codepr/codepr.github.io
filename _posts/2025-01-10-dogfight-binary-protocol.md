---
layout: post
title: "Dogfight! - Binary protocol"
description: "Old school multiplayer arcade game development journey"
categories: elixir networking zig game-development low-level
---

A very simple core that should allow for basic interactions is roughly defined,
the prototype is currenctly comprised by:

- A TCP server, with very rudimentary handling of connections
- A game state, representing the main structure of any given match
    - A default spaceship
- An event-based system for game state updates

The next step is to setup a proper transport layer to communicate with
the connecting clients and start to show something. Before moving on
with the first implementation of a client and some basic graphics,
we're going to define how this communication layer will be implemented.

Although various formats can be used, from a text based protocol, JSON or
orther various binary based structure serialization buffers such as
protobuf or thrift, we're going to roll our own micro protocol by hand.
One suggestion could be to use the built-in `:erlang.to_binary_term/2`
and `:erlang.term_to_binary/1`, but I like the idea of writing a small
compact format that I can easily support on the client side without having
to dig deeper into the Erlang bnary format.

As a first draft, it should be something that allows to represent two main
entities

- The game state
- The game events

## The game events

We start from the simplest and smallest of the two, literally two or three
fields stuffed in a byte array, focussing on just two actions which can be
easily represented as

- `move` - `player_id` - `direction`
    - action: `u8`
    - direction: `u8`
    - player_id: `[]u8`

- `shoot` - `player_id`
    - action: `u8`
    - player_id: `[]u8`

### Implementation

Again, a little behaviour can be useful here and doesn't really represent an
excessive burden, if one day we wanted to add multiple protocols, this would
provide a simple interface for that

```elixir
defmodule Dogfight.Game.Codec do
  @moduledoc """
  Generic encoding and decoding behaviour
  """

  alias Dogfight.Game.State
  alias Dogfight.Game.Event

  @callback encode(State.t()) :: binary()
  @callback decode(binary()) :: {:ok, State.t()} | {:error, :codec_error}

  @callback encode_event(Event.t()) :: binary()
  @callback decode_event(binary()) :: {:ok, Event.t()} | {:error, :codec_error}
end
```

Probably not necessary but let's start by explicitly dividing the responsibilities
of the encoder/decoder functions by the target struct that they will handle. Let's
also add a small helper module to deal with binary payloads. Luckily, `Elixir`
allows for very convenient handling of input from the wire, pattern matching really
shines here.

```elixir
defmodule Dogfight.Game.Codecs.Helpers do
  @moduledoc """
  This module provides helper functions for encoding data into binary format.

  The module supports encoding integers of various sizes (half word, word,
  double word, quad word) and binary data.
  """

  @half_word 8
  @word 16
  @double_word 32
  @quad_word 64

  @type word_size :: :half_word | :word | :double_word | :quad_word | :binary
  @type field :: {integer(), word_size()}

  @doc """
  Encodes a list of fields into a binary.

  Each field is a tuple containing an integer and a word size. The word size
  determines the number of bits used to encode the integer.

  ## Parameters

    - fields: A list of tuples where each tuple contains an integer and a word size.

  ## Returns

    - A binary representing the encoded fields.

  ## Examples

      iex> Dogfight.Game.Codecs.Helpers.encode_list([{1, :half_word}, {2, :word}])
      <<1::8, 2::16>>
  """
  @spec encode_list([field(), ...]) :: binary()
  def encode_list(fields) do
    fields
    |> Enum.map(fn {field, word_size} ->
      case word_size do
        :half_word -> encode_half_word(field)
        :word -> encode_word(field)
        :double_word -> encode_double_word(field)
        :quad_word -> encode_quad_word(field)
        :binary -> field
      end
    end)
    |> IO.iodata_to_binary()
  end

  def encode_half_word(data), do: encode_integer(data, @half_word)
  def encode_word(data), do: encode_integer(data, @word)
  def encode_double_word(data), do: encode_integer(data, @double_word)
  def encode_quad_word(data), do: encode_integer(data, @quad_word)

  def encode_integer(true, size), do: encode_integer(1, size)
  def encode_integer(false, size), do: encode_integer(0, size)
  def encode_integer(data, size) when is_integer(data), do: <<data::integer-size(size)>>

  def half_word_size, do: @half_word
  def word_size, do: @word
  def double_word_size, do: @double_word
  def quad_word_size, do: @quad_word
end
```

We're gonna serialize our payloads in network order aka big-endian, the main
function of interest here is `encode_list/1` which allows to pass a list of
tuples in the form of `{field_value, value_size}`, this will be convenient
when implementing the actual serialization of the game state.


## The game state

Let's start simple, an header containing the full length of the packet and
then straight with the payload

- **Header**
    - total_length: `usize`
- **Payload**
    - status: `u8`
    - power_ups_count: `u16`
    - power_ups
        - pos_x: `i32`
        - pos_y: `i32`
        - kind: `u8`
    - players
        - pos_x: `i32`
        - pos_y: `i32`
        - hp: `i8`
        - alive: `u8`
        - direction: `u8`
        - player_id: `[]u8`
        - bullets
            - pos_x: `i32`
            - pos_y: `i32`
            - active: `u8`
            - direction: `u8`

Pretty small, non-optmized format containing all the information needed to have
basic movements and interaction between entities. For many fields we could
actually use some `nibbles` or encode more aggressively by using each byte
space more efficiently, for example, `status`, `direction` and `kind` are
likely to not ever reprensent more then a bunch of values, `active` and `alive`
are even just 1 bit, so all those informations could be easily packed in 1 or 2
bytes but it's not that pressuring as the full payload is still fairly small

What we expect to receive on the client side, after the deserialization of the
payload, is something resembling the following

```
//
// The structure is simple enough for the time being, just a variable power up
// count followed by the players (spaceships), each one has only a 5 bullets.
//
// GameState
// ---------
//  Status: In Progress
//
//  Power ups
//  ---------
//   Kind: hp_plus_three
//   Position: (407, 209)
//
//  Players
//  -------
//   Player 0 - 5afd1a7c-50c6-4a55-be57-0f02cef8e48e:
//    position: (533, 353)
//    HP: 5 Direction: up Alive: true
//
//    Bullets
//    -------
//     position: (343, 123) Direction: up Active: true
//     position: (241, 123) Direction: up Active: true
//     position: (167, 123) Direction: up Active: true
//     position: (101, 123) Direction: up Active: true
//     position: (0, 0) Direction: idle Active: false
```

### Implementation

The following module will basically expose the `Dogfight.Game.Codec` behaviour
defined above with a couple of helpers to translate atoms to integer back and
forth to simplify the encoding. Those values could probably be sent through the
wire as strings too, avoiding the endianess handling altogether, I just find it
more appropriate and often more compact to encode them as integer values.

```elixir
defmodule Dogfight.Game.Codecs.BinaryCodec do
  @moduledoc """
  Binary implementation of the game econding and decoding logic

  Handles

  - Game state
  - Game event
  """

  @behaviour Dogfight.Game.Codec

  alias Dogfight.Game.Codecs.Helpers
  alias Dogfight.Game.State
  alias Dogfight.Game.Vec2

  @power_up_size Helpers.double_word_size() * 2 + Helpers.half_word_size()
  @bullet_size Helpers.double_word_size() * 2 + Helpers.half_word_size() * 2
  # (32 bytes and 4 hyphens as per UUID)
  @player_id_size Helpers.double_word_size() + 4
  @spaceship_size @player_id_size + 5 * @bullet_size +
                    3 * Helpers.double_word_size() + 2 * Helpers.half_word_size()

  @impl true
  def encode_event(event) do
    case event do
      {:move, {player_id, direction}} ->
        Helpers.encode_list([
          {event_to_int(:move), :half_word},
          {direction_to_int(direction), :half_word},
          {player_id, :binary}
        ])

      {player_action, player_id}
      when player_action in [:player_connection, :player_disconnection, :shoot] ->
        Helpers.encode_list([
          {event_to_int(player_action), :half_word},
          {player_id, :binary}
        ])
    end
  end

  @impl true
  def decode_event(binary) do
    <<action::big-integer-size(8), rest::binary>> = binary

    action = int_to_event(action)

    case action do
      :move ->
        <<direction::big-integer-size(8), player_id::binary-size(@player_id_size)>> =
          rest

        {:ok, {action, {player_id, int_to_direction(direction)}}}

      player_action when player_action in [:player_connection, :player_disconnection, :shoot] ->
        <<player_id::binary-size(@player_id_size)>> = rest

        {:ok, {action, player_id}}
    end
  rescue
    _e -> {:error, :codec_error}
  end

  # For now we can just handle the basic events from the
  # client, which are only move and shoot really
  defp event_to_int(action) do
    case action do
      :player_connection -> 0
      :player_disconnection -> 1
      :move -> 2
      :shoot -> 3
    end
  end

  defp int_to_event(intval) do
    case intval do
      0 -> :player_connection
      1 -> :player_disconnection
      2 -> :move
      3 -> :shoot
    end
  end

  @doc "Encode a `Game.State` struct into a raw binary payload"
  @impl true
  def encode(game_state) do
    binary_ships =
      game_state.players
      |> Enum.map(fn {player_id, spaceship} -> encode_spaceship(player_id, spaceship) end)
      |> IO.iodata_to_binary()

    binary_power_ups =
      game_state.power_ups |> Enum.map(&encode_power_up/1) |> IO.iodata_to_binary()

    binary_ships_byte_size = byte_size(binary_ships)
    binary_power_ups_byte_size = byte_size(binary_power_ups)

    total_length =
      binary_ships_byte_size +
        binary_power_ups_byte_size +
        Helpers.double_word_size() +
        Helpers.half_word_size() +
        Helpers.word_size()

    Helpers.encode_list([
      {total_length, :double_word},
      {status_to_int(game_state.status), :half_word},
      {binary_power_ups_byte_size, :word},
      {binary_power_ups, :binary},
      {binary_ships, :binary}
    ])
  end

  defp encode_spaceship(player_id, spaceship) do
    direction = direction_to_int(spaceship.direction)

    binary_bullets =
      spaceship.bullets
      |> Enum.map(&encode_bullet/1)
      |> IO.iodata_to_binary()

    %{x: x, y: y} = spaceship.position

    Helpers.encode_list([
      {x, :double_word},
      {y, :double_word},
      {spaceship.hp, :half_word},
      {if(spaceship.alive?, do: 1, else: 0), :half_word},
      {direction, :half_word},
      {player_id, :binary},
      {binary_bullets, :binary}
    ])
  end

  defp encode_power_up(power_up) do
    kind = power_up_to_int(power_up.kind)
    %{position: %{x: x, y: y}} = power_up

    Helpers.encode_list([
      {x, :double_word},
      {y, :double_word},
      {kind, :half_word}
    ])
  end

  defp encode_bullet(bullet) do
    direction = direction_to_int(bullet.direction)

    %{x: x, y: y} = bullet.position

    Helpers.encode_list([
      {x, :double_word},
      {y, :double_word},
      {if(bullet.active?, do: 1, else: 0), :half_word},
      {direction, :half_word}
    ])
  end

  @doc "Decode a raw binary payload into a `Game.State` struct"
  @impl true
  def decode(binary) do
    <<_total_length::big-integer-size(32), status::big-integer-size(8),
      power_ups_len::big-integer-size(16),
      rest::binary>> =
      binary

    <<power_ups_bin::binary-size(power_ups_len), player_records::binary>> = rest

    power_ups =
      power_ups_bin
      |> chunk_bits(@power_up_size)
      |> Enum.map(&decode_power_up!/1)

    players =
      player_records
      |> chunk_bits(@spaceship_size)
      |> Enum.map(&decode_spaceship!/1)
      |> Map.new()

    {:ok,
     %State{
       players: players,
       power_ups: power_ups,
       status: int_to_status(status)
     }}
  rescue
    # TODO add custom errors
    _e -> {:error, :codec_error}
  end

  defp decode_power_up!(
         <<power_up_x::big-integer-size(32), power_up_y::big-integer-size(32),
           power_up_kind::big-integer-size(8)>>
       ) do
    %{position: %Vec2{x: power_up_x, y: power_up_y}, kind: int_to_power_up(power_up_kind)}
  end

  defp decode_spaceship!(
         <<x::big-integer-size(32), y::big-integer-size(32), hp::big-integer-size(8),
           alive::big-integer-size(8), direction::big-integer-size(8),
           player_id::binary-size(@player_id_size), bullets::binary>>
       ) do
    {player_id,
     %{
       position: %Vec2{x: x, y: y},
       hp: hp,
       direction: int_to_direction(direction),
       alive?: alive == 1,
       bullets:
         bullets
         |> chunk_bits(@bullet_size)
         |> Enum.map(&decode_bullet!/1)
     }}
  end

  defp decode_bullet!(
         <<x::big-integer-size(32), y::big-integer-size(32), active::big-integer-size(8),
           direction::big-integer-size(8)>>
       ) do
    %{
      position: %{x: x, y: y},
      active?: if(active == 0, do: false, else: true),
      direction: int_to_direction(direction)
    }
  end

  defp chunk_bits(binary, n) do
    for <<chunk::binary-size(n) <- binary>>, do: <<chunk::binary-size(n)>>
  end

  defp power_up_to_int(nil), do: 0
  defp power_up_to_int(:hp_plus_one), do: 1
  defp power_up_to_int(:hp_plus_three), do: 2
  defp power_up_to_int(:ammo_plus_one), do: 3

  defp int_to_power_up(0), do: nil
  defp int_to_power_up(1), do: :hp_plus_one
  defp int_to_power_up(2), do: :hp_plus_three
  defp int_to_power_up(3), do: :ammo_plus_one

  defp status_to_int(nil), do: 0
  defp status_to_int(:in_progress), do: 1
  defp status_to_int(:closed), do: 2

  defp int_to_status(0), do: nil
  defp int_to_status(1), do: :in_progress
  defp int_to_status(2), do: :closed

  def direction_to_int(:idle), do: 0
  def direction_to_int(:up), do: 1
  def direction_to_int(:down), do: 2
  def direction_to_int(:left), do: 3
  def direction_to_int(:right), do: 4

  def int_to_direction(0), do: :idle
  def int_to_direction(1), do: :up
  def int_to_direction(2), do: :down
  def int_to_direction(3), do: :left
  def int_to_direction(4), do: :right
end

```

#### References

- [nibble](https://en.wikipedia.org/wiki/Nibble)
- [endianness](https://en.wikipedia.org/wiki/Endianness)
