---
layout: post
title: "Exploring nerves on an RPI4"
description: "Trying out nerves and phoenix liveview on a Raspberry PI 4"
categories: elixir iot
---

Small dump of a tiny presentation I made for a rather small audience on [Elixir](https://elixir-lang.org/) [Nerves](https://nerves-project.org/) and [Phoenix liveview](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html). Can be followed as a mini tutorial step by step to get an IoT system running a livebook with a small application backed by Phoenix.

### Requirements

#### Hardware

- Raspberry PI 4 Model The B
- Micro SD

#### Software dependencies

- Elixir + mix [install](https://elixir-lang.org/install.html#macos) (alternatively [asdf](https://github.com/asdf-vm/asdf))
- Docker

### Touched arguments

- Little history, development
- Docs "Getting started" and the actual procedure to burn a firmware
- Burn firmware into RPI4 Model B
- Connect to livebook on http://nerves.local
  - Walkthrough some examples
- Connect through ssh by `ssh nerves.local`
  - `h Toolshed`
  - `weather/0` :)
- Try MQTT communication
  - `Mix.install` is not yet supported
  - `mix archive.install hex nerves_bootstrap`
- Something better: Phoenix LiveView on the board
  - Poncho Project
  - Small `GenServer` to publish data periodically
  - ssh into the board, `log_attach`
- Nerves HUB
- Peridio

### Quickstart

Check [nerves](https://nerves-project.org/), the getting started link points directly to
the simplest example: [running livebook on embedded systems](https://github.com/livebook-dev/nerves_livebook/blob/main/README.md)

- `fwup` as the quickest setup.
- exploring some pre-loaded samples
- check the same commands through an `ssh` session: `ssh nerves.local` pw: `nerves`
  - `h Toolshed`
  - `log_attach` to attach to the Elixir `Logger` (useful to debug running apps)
  - `weather` :)
- test communication with a simple snippet

#### Testing MQTT communication, setup:

Run an MQTT broker on the host machine, let's use `mosquitto` (if an `Address not available error` happens, use the following conf)

{% highlight ini %}
persistence false
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
listener 1883
allow_anonymous true
{% endhighlight %}

Finally run a `mosquitto` container mounting the local config


{% highlight bash %}
docker run -it -p 1883:1883 -v $PWD/mosquitto.conf:/mosquitto/config/mosquitto.conf eclipse-mosquitto
{% endhighlight %}

Create a new livebook and on the setup paste this dependency

{% highlight elixir %}
Mix.install([{:emqtt, github: "emqx/emqtt", tag: "1.4.4", system_env: [{"BUILD_WITHOUT_QUIC", "1"}]}])
{% endhighlight %}

Publish random temperature values in a subsequent `code` block

{% highlight elixir %}
client_id = "weather_sensor"
report_topic = "reports/#{client_id}/temperature"
host = '192.168.10.12' # This should point to the local machine address, where our MQTT broker is listening
port = 1883

emqtt_opts = %{
  host: host,
  port: port,
  clean_start: false,
  name: :emqtt
}

{:ok, pid} = :emqtt.start_link(emqtt_opts)
{:ok, _} = :emqtt.connect(pid)

temperature = 10.0 + 2.0 * :rand.normal()
message = %{"timestamp" => System.system_time(:millisecond), "temperature" => temperature}
payload = Jason.encode!(message)
:emqtt.publish(pid, report_topic, payload)
:emqtt.stop(pid)
{% endhighlight %}

Oh no, must rebuild the livebook :( one current limitation of livebook in `nerves` is that
`Mix.install` is not supported yet. Livebook can be customized easily on the host machine
(your PC).

Flash `nerves_livebook` into device (`rpi4`)

{% highlight bash %}
git clone https://github.com/livebook-dev/nerves_livebook.git
cd nerves_livebook
{% endhighlight %}

Add dependency to the `mix.exs`

<hr>
**mix.exs**
<hr>
{% highlight elixir %}
defp deps do
    [
      ...,
      {:emqtt, github: "emqx/emqtt", tag: "1.4.4", system_env: [{"BUILD_WITHOUT_QUIC", "1"}]}
      ...
    ]
end
{% endhighlight %}

Build firmware and push it to the device

{% highlight bash %}

# Set the MIX_TARGET to the desired platform (rpi0, bbb, rpi3, etc.)
export MIX_TARGET=rpi4
mix deps.get
mix firmware

# Option 1: Insert a MicroSD card
mix burn

# Option 2: Upload to an existing Nerves Livebook device
mix firmware.gen.script
./upload.sh livebook@nerves.local
{% endhighlight %}

Now the previous snippet should work correctly and we should see the message published onto our MQTT broker.

### Daring something more

Let's expand the previous snippet: a `GenServer` perpetually generating temperature values and also
accepting command from the broker

{% highlight elixir %}

defmodule WeatherSensor do
  @moduledoc false
  use GenServer

  @client_id "weather_sensor"
  @report_topic "reports/#{@client_id}/temperature"
  @host '192.168.10.12' # This should point to the local machine address, where our MQTT broker is listening
  @port 1883
  @interval_ms 2000

  def start_link([]) do
    GenServer.start_link(__MODULE__, [])
  end

  def init([]) do
    {:ok, pid} = :emqtt.start_link(emqtt_opts())

    state = %{
      interval: @interval_ms,
      timer: nil,
      report_topic: @report_topic,
      pid: pid
    }

    {:ok, set_timer(state), {:continue, :start_emqtt}}
  end

  defp emqtt_opts,
    do: %{
      host: @host,
      port: @port,
      clean_start: false,
      name: :emqtt
    }

  def handle_continue(:start_emqtt, %{pid: pid} = state) do
    {:ok, _} = :emqtt.connect(pid)

    {:ok, _, _} = :emqtt.subscribe(pid, {"commands/#{@client_id}/set_interval", 1})
    {:noreply, state}
  end

  def handle_info(:tick, %{report_topic: topic, pid: pid} = state) do
    report_temperature(pid, topic)
    {:noreply, set_timer(state)}
  end

  def handle_info({:publish, publish}, state) do
    handle_publish(parse_topic(publish), publish, state)
  end

  defp handle_publish(["commands", _, "set_interval"], %{payload: payload}, state) do
    {:noreply, set_timer(%{state | interval: String.to_integer(payload)})}
  end

  defp handle_publish(_, _, state) do
    {:noreply, state}
  end

  defp parse_topic(%{topic: topic}) do
    String.split(topic, "/", trim: true)
  end

  defp set_timer(state) do
    if state.timer do
      Process.cancel_timer(state.timer)
    end

    timer = Process.send_after(self(), :tick, state.interval)
    %{state | timer: timer}
  end

  defp report_temperature(pid, topic) do
    temperature = 10.0 + 2.0 * :rand.normal()
    message = %{"timestamp" => System.system_time(:millisecond), "temperature" => temperature}
    payload = Jason.encode!(message)
    :emqtt.publish(pid, topic, payload)
  end
end
{% endhighlight %}

On a subsequent block, we can run the `GenServer`

{% highlight elixir %}
{:ok, pid} = WeatherSensor.start_link([])
{% endhighlight %}

On the host we should see data coming every ~2s, we can confirm it by peeking into the
`mosquitto` logs or by subscribing to the topic on `localhost`

{% highlight bash %}
mosquitto_sub -t reports/weather_sensor/temperature -h 127.0.0.1
{% endhighlight %}

### Phoenix liveview

Let's build a simple Phoenix app to subscribe to the topic and see real-time data coming through `LiveView`.
Let's not forget `--no-ecto` we don't need DBs here, `--no-mailer`, `--no-gettext`, `--no-dashboard`, and `--live`.
This should cut out a fair amount of boilerplate.

{% highlight bash %}
mix phx.new temp_dashboard --no-ecto --no-mailer --no-gettext --no-dashboard --live
{% endhighlight %}

First thing, let's add the required dependencies to the `mix.exs`

<hr>
**temp_dashboard/mix.exs**
<hr>
{% highlight elixir %}
  defp deps do
    [
      ...,
      {:emqtt, github: "emqx/emqtt", tag: "1.4.4", system_env: [{"BUILD_WITHOUT_QUIC", "1"}]},
      {:contex, github: "mindok/contex"}, # We will need this for SVG charts
      ...
    ]
  end
{% endhighlight %}

Update the dependencies with the newly added bit

{% highlight bash %}
mix deps.get
{% endhighlight %}

Let's add few lines to the main configuration and the development one:

<hr>
**temp_dashboard/config/config.exs**
<hr>
{% highlight elixir %}
config :temp_dashboard, :emqtt,
  host: '192.168.10.12',  # Again, the MQTT broker address here
  port: 1883

config :temp_dashboard, :sensor_id, "weather_sensor"

# Period for chart
config :temp_dashboard, :timespan, 60
{% endhighlight %}

Same on `temp_dashboard/config/dev.exs`.

Let's now generate a `LiveView` controller

{% highlight elixir %}
mix phx.gen.live Measurements Temperature temperatures  --no-schema --no-context
{% endhighlight %}

We can remove some of them for our single-page app

{% highlight bash %}
rm lib/temp_dashboard_web/live/temperature_live/form_component.*
rm lib/temp_dashboard_web/live/temperature_live/show.*
rm lib/temp_dashboard_web/live/live_helpers.ex
{% endhighlight %}

Finally, we want to add some business logic to our app, fire up the editor on `temp_dashboard/lib/temp_dashboard_web/live/temperature_live/index.ex`
and replace the content with the following snippet.

<hr>
**temp_dashboard/lib/temp_dashboard_web/live/temperature_live/index.ex**
<hr>
{% highlight elixir %}
defmodule TempDashboardWeb.TemperatureLive.Index do
  use TempDashboardWeb, :live_view

  require Logger

  @impl true
  def mount(_params, _session, socket) do
    reports = []
    emqtt_opts = Application.get_env(:temp_dashboard, :emqtt)
    Logger.info(emqtt_opts)
    {:ok, pid} = :emqtt.start_link(emqtt_opts)
    {:ok, _} = :emqtt.connect(pid)
    # Listen reports
    {:ok, _, _} = :emqtt.subscribe(pid, "reports/#")

    {:ok,
     assign(socket,
       reports: reports,
       pid: pid,
       plot: nil,
       interval: nil
     )}
  end

  @impl true
  def handle_params(_params, _url, socket) do
    {:noreply, socket}
  end

  @impl true
  def handle_event("set-interval", %{"interval" => interval_s}, socket) do
    case Integer.parse(interval_s) do
      {interval, ""} ->
        id = Application.get_env(:temp_dashboard, :sensor_id)
        # Send command to device
        topic = "commands/#{id}/set_interval"

        :ok =
          :emqtt.publish(
            socket.assigns[:pid],
            topic,
            interval_s,
            retain: true
          )

        {:noreply, assign(socket, interval: interval)}

      _ ->
        {:noreply, socket}
    end
  end

  def handle_event(name, data, socket) do
    Logger.info("handle_event: #{inspect([name, data])}")
    {:noreply, socket}
  end

  @impl true
  def handle_info({:publish, packet}, socket) do
    Logger.info("handle_event: #{inspect(packet)}")
    handle_publish(parse_topic(packet), packet, socket)
  end

  defp handle_publish(["reports", id, "temperature"], %{payload: payload}, socket) do
    if id == Application.get_env(:temp_dashboard, :sensor_id) do
      report = Jason.decode!(payload)
      {reports, plot} = update_reports(report, socket)
      {:noreply, assign(socket, reports: reports, plot: plot)}
    else
      {:noreply, socket}
    end
  end

  defp update_reports(%{"timestamp" => ts, "temperature" => val}, socket) do
    new_report = {DateTime.from_unix!(ts, :millisecond), val}
    now = DateTime.utc_now()

    deadline =
      DateTime.add(
        DateTime.utc_now(),
        -2 * Application.get_env(:temp_dashboard, :timespan),
        :second
      )

    reports =
      [new_report | socket.assigns[:reports]]
      |> Enum.filter(fn {dt, _} -> DateTime.compare(dt, deadline) == :gt end)
      |> Enum.sort()

    {reports, plot(reports, deadline, now)}
  end

  defp parse_topic(%{topic: topic}) do
    String.split(topic, "/", trim: true)
  end

  defp plot(reports, deadline, now) do
    x_scale =
      Contex.TimeScale.new()
      |> Contex.TimeScale.domain(deadline, now)
      |> Contex.TimeScale.interval_count(10)

    y_scale =
      Contex.ContinuousLinearScale.new()
      |> Contex.ContinuousLinearScale.domain(0, 30)

    options = [
      smoothed: false,
      custom_x_scale: x_scale,
      custom_y_scale: y_scale,
      custom_x_formatter: &x_formatter/1,
      axis_label_rotation: 45
    ]

    reports
    |> Enum.map(fn {dt, val} -> [dt, val] end)
    |> Contex.Dataset.new()
    |> Contex.Plot.new(Contex.LinePlot, 600, 250, options)
    |> Contex.Plot.to_svg()
  end

  defp x_formatter(datetime), do: Calendar.strftime(datetime, "%H:%M:%S")
end
{% endhighlight %}

We also need a template view for our page, let's replace the `index.html.heex`

<hr>
**temp_dashboard/lib/temp_dashboard_web/live/temperature_live/index.html.heex**
<hr>
{% highlight html %}
<!-- temp_dashboard/lib/temp_dashboard_web/live/temperature_live/index.html.heex -->
<div>
  <%= if @plot do %>
    <%= @plot %>
  <% end %>
</div>

<div>
  <form phx-submit="set-interval">
    <label for="interval">Interval</label>
    <input type="text" name="interval" value={@interval}/>
    <input type="submit" value="Set interval"/>
  </form>
</div>
{% endhighlight %}

Last step, adding a handler to route our requests to index, removing the existing `/` `scope`

<hr>
**lib/temp_dasboard_web/router.ex**
<hr>
{% highlight elixir %}
  scope "/", TempDashboardWeb do
    pipe_through :browser

    live "/", TemperatureLive.Index
  end
{% endhighlight %}

Get the dependencies and run the application, we should see it connecting to the broker

{% highlight bash %}
mix phx.server
{% endhighlight %}

Let's open the brower to `localhost:4000`.

### Phoenix LiveView on RPI4

A "poncho project" is similar to an umbrella project except that it's actually
multiple separate-but-related Elixir apps that use path dependencies instead of
`in_umbrella` dependencies

#### References

- [Raspberry PI 4B - https://www.raspberrypi.com/products/raspberry-pi-4-model-b/](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
- [MQTT - https://mqtt.org/](https://mqtt.org/)
- [Docker - https://www.docker.com/](https://www.docker.com/)
- [Elixir - https://elixir-lang.org/](https://elixir-lang.org/)
- [Mix - https://hexdocs.pm/elixir/introduction-to-mix.html](https://hexdocs.pm/elixir/introduction-to-mix.html)
- [GenServer - https://hexdocs.pm/elixir/GenServer.html](https://hexdocs.pm/elixir/GenServer.html)
- [Livebook - https://livebook.dev/](https://livebook.dev/)
- [Nerves - https://nerves-project.org/](https://nerves-project.org/)
- [Phoenix liveview - https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html)
