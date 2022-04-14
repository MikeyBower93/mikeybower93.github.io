# Fun with Flags (Elixir LiveView)

# Introduction
Although I could list multiple things that I love about you using Elixir and Phoenix for web development, one thing that really blows me away is how as developers we are being provided with new ways to innovate and solve web development tasks, that can often be difficult to do using regular web development techniques. Whats more, these new tools for innovation (especially in the case of this posts topic) are built on top of robust technology that has existed for a long time now. In this post I will touch on how LiveView and GenServers allow us to do this. 

# A Simple Example
Recently I developed a simple project using Elixir and Phoenix, and wanted to make use of LiveView to see how it can make developing interactive applications a lot easier. To that end, I developed a small website that shows the capital cities of the world, and shows the time in those cities which is updated in real time. I also wanted to make the capital cities searchable as there can be a lot to trawl through.

Although this example is very crude, as you likely wouldn't do real time updates of times at a fast rate using LiveView, its a good demonstration of what can be done. Additionally, the time can jump occasionally, this is mainly due to the way the time is getting updated in the backend, it is nonetheless good for demonstration purposes. 

## GenServers
Initially when I started, I updated the time in the LiveView processes, however I soon realised that this didn't make a great deal of sense, as the time is absolute for anyone on the website so why have every connection updating the time when one place can do it centrally. 

Luckily GenServers give us the ability to have a robust process continuously running at a fast rate, whilst giving clients the ability to request the central state of that GenServer process, what is so great about this is not only how robust GenServers are, but how simple they are to write, for example:
```elixir
defmodule LiveClocks.TimeZones do
  use GenServer

  @init_time_zones [
    %{
      country: "Afghanistan",
      flag_url: "https://restcountries.eu/data/afg.svg",
      location: "Kabul",
      time_zone: "Asia/Kabul"
    },
    ...
  ]

  @update_period_in_milliseconds 500

  def start_link(args) do
    GenServer.start_link(__MODULE__, args, name: __MODULE__)
  end

  @impl true
  def init(state) do
    time_zones = append_current_time(@init_time_zones)

    schedule_work()

    {:ok, Map.put(state, :time_zones, time_zones)}
  end

  @impl true
  def handle_info(:work, state) do
    time_zones = append_current_time(state.time_zones)

    schedule_work()

    {:noreply, Map.put(state, :time_zones, time_zones)}
  end

  def fetch_time_zones() do
    GenServer.call(__MODULE__, :fetch_times)
  end

  @impl
  def handle_call(:fetch_times, _from, state) do
    {:reply, state.time_zones, state}
  end

  defp schedule_work() do
    Process.send_after(self(), :work, @update_period_in_milliseconds)
  end

  defp append_current_time(time_zones) do
    time_zones
    |> Enum.map(fn time_zone ->
        Map.put(time_zone, :date, get_date_time_for_timezone(time_zone.time_zone))
      end)
  end

  defp get_date_time_for_timezone(timezone) do
    {:ok, now} = DateTime.now(timezone)

    {:ok, formatted_date} = Calendar.Strftime.strftime(now, "%d/%m/%y %H:%M:%S")

    formatted_date
  end
end
```
This is a job that runs every half second, that simply goes through a list of capitals, and updates their time. It also provides a public API `fetch_time_zones()` which calls the GenServer to get the current state of the time zones so LiveView processes can get the current values. Here we can see how GenServers are giving me the ability to have code running in the background frequently, which provides a function for clients to interact with the GenServer, with the great recovery mechanisms baked into OTP for very small amounts of code.

## LiveView
The code that interacts with the browser using LiveView is similarly simple and effective. Which looks like this:
```elixir
defmodule LiveClocksWeb.LiveView.Clocks do
  use Phoenix.LiveView
  use Phoenix.HTML

  alias LiveClocks.TimeZones

  @update_period_in_milliseconds 100

  def render(assigns) do
    ~L"""
    <form phx-change="search" class="location-search">
      <%= text_input :location_search,
        :query,
        autofocus: true,
        placeholder: "Search a location...",
        "phx-debounce": "300",
        value: @location_search %>
    </form>
    <div class="timezone-container">
      <%= for location_time <- @location_times do %>
        <div class="timezone-card">
          <img src="<%= location_time.flag_url %>"
               class="country-flag">
          <h3 class="country-name"><b><%= location_time.country %></b></h3>
          <h3><%= location_time.location %></h3>
          <span>
            <%= location_time.date %>
          </p>
        </div>
      <% end %>
    </div>
    """
  end

  def mount(params, _session, socket) do
    if connected?(socket), do: Process.send_after(self(), :update, @update_period_in_milliseconds)

    location_search = get_search_from_params(params)

    location_times =
      TimeZones.fetch_time_zones()
      |> filter_time_zones(location_search)

    socket =
      socket
      |> assign(:location_times, location_times)
      |> assign(:location_search, location_search)

    {:ok, socket}
  end

  def handle_info(:update, socket) do
    Process.send_after(self(), :update, @update_period_in_milliseconds)

    location_search = get_search_from_assigns(socket.assigns)

    location_times =
      TimeZones.fetch_time_zones()
      |> filter_time_zones(location_search)

    {:noreply, assign(socket, :location_times, location_times)}
  end

  def handle_params(
    %{"location_search" => location_search},
    _uri,
    socket
  ) when is_binary(location_search)
  do
    {:noreply, assign(socket, :location_search, location_search)}
  end
  def handle_params(_param, _url, socket), do: {:noreply, socket}

  def handle_event("search", %{"location_search" => %{"query" => query}}, socket) when is_binary(query) do
    {:noreply, push_patch(socket, to: "/?location_search=#{query}")}
  end
  def handle_event(_, _, socket), do: {:noreply, socket}

  defp get_search_from_params(%{"location_search" => location_search}), do: location_search
  defp get_search_from_params(_), do: ""

  defp get_search_from_assigns(%{location_search: location_search}), do: location_search
  defp get_search_from_assigns(_), do: ""

  defp filter_time_zones(time_zones, location_search) do
    search_field =
      location_search
      |> String.downcase()
      |> String.trim()

    Enum.filter(time_zones,
      fn time_zone ->
        String.contains?(String.downcase(time_zone.location), search_field) or
        String.contains?(String.downcase(time_zone.country), search_field)
      end)
  end
end
```
A lot of the code here is mainly around general web development stuff, such as the HTML, filtering the results and handling query parameters. I won't go into the depth of each function as its not the purpose of this blog, additionally the purpose of this blog is not to explain how to develop with LiveView or how it works, great resources already exist for this such as the offical documentation [https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html), however I will highlight some of the key areas to demostrate how easy it is to get some of these features working.

- Much like the GenServer, when we get a socket connection to the browser we tell the process to update every half a second. 
- When the the background job periodically updates, it asks the GenServer for the current time zones, this can be seen in the `handle_info(:update, socket)` function. Whats great about this is how efficent LiveView is as when it goes to re render the changes, it only sends new changes, and the packets it sends are very very small. This is why in this example, every time the second changes its able seamlessly update over 200 cities. This is great because this means we now have real time updates going to the browser frequently with very little amount of code, which using other means would have taken a lot of more work.
- When the user inputs a value into the search field we receive an event `handle_event` where we can parse out the query, at that point we put it in the URL (this means if a user reloads the page we will have kept the filtered view), this in turn triggers another event `handle_params` where we can grab the search and put it in the LiveView state, then when the next iteration of the update occurs it will filter the list. As we can see this is a responsive search field which once again to do using other Web Development techniques would have required a lot more effort through some JavaScript and an API endpoint in the project. 

### Conclusions 

As I stated in the introduction, this is an example that you aren't very likely to do, however it is an example that demostrates the ability to write code that has background workers running, realtime updates to a browser, with interactive features such as a search textbox with very little amounts of code. 

I love how the full Elixir stack (Phoenix, Erlang, OTP) is allowing us to develop great features as developers with less code and effort, whilst being able to trust the technology it runs on given the mature OTP platform. 

If you would like see the full code base for this the GitHub repository is: [https://github.com/MikeyBower93/live_clocks](https://github.com/MikeyBower93/live_clocks)

Happy Coding!
