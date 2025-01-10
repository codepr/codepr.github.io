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

TBD

#### References

- [nibble](https://en.wikipedia.org/wiki/Nibble)
