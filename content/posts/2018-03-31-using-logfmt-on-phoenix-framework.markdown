---
layout: post
title: "Using logfmt on Phoenix Framework"
date: 2018-03-31 02:20:16 +0800
comments: true
tags: [logging, phoenix, elixir]
---

![Phoenix logs](/img/phoenix-log.png)

Nowadays application log is not only serve as an assets for tracing application error but also a data for analyse user requests.
[Logfmt](https://brandur.org/logfmt) is a format to show the logs in key/value pairs, which is readable by human or computer.
This article shows you to:

1. Convert default [Phoenix Framework](http://phoenixframework.org/) output into logfmt
2. Add remote ip to the request log

In case you don't want to study a long steps, you can go to <https://github.com/dhoeric/phoenix-logfmt-demo>, check the commits and play around in heroku :)

---

## Convert to Logfmt
- Add logfmt package

Put following into your deps in `./mix.exs` file, then run `mix deps.get` for pulling the dependencies.

{{< highlight elixir >}}
{:logfmt, "~> 3.3"}
{{< / highlight >}}

- Replace with your own logger plug

Save the following as `./lib/[your_app]_web/logger_plug.ex`, remember to update the module name in line 1 as well.

{{< highlight elixir "linenos=table" >}}
defmodule HelloWeb.LoggerPlug do
  @moduledoc """
  Provide standardize logfmt output on request
  """
  require Logger
  alias Plug.Conn
  @behaviour Plug

  def init(opts) do
    Keyword.get(opts, :log, :info)
  end

  def call(conn, level) do
    start = System.monotonic_time()

    conn
    |> Conn.register_before_send(fn conn ->
      Logger.log level, fn ->
        stop = System.monotonic_time()
        diff = System.convert_time_unit(stop - start, :native, :micro_seconds)
        Logfmt.encode [
          level: level,
          time: Poison.encode!(DateTime.utc_now),
          elapsed: formatted_diff(diff),
          method: conn.method,
          path: conn.request_path,
          status: conn.status
        ]
      end

      conn
    end)
  end

  defp formatted_diff(diff) when diff > 1000, do: [diff |> div(1000) |> Integer.to_string(), "ms"] |> Enum.join(" ")
  defp formatted_diff(diff), do: [Integer.to_string(diff), "µs"] |> Enum.join(" ")
end
{{< / highlight >}}

After that using the new logger plug in your `./lib/[your_app]_web/endpoint.ex`

{{< highlight diff >}}
-  plug Plug.Logger
+  plug HelloWeb.LoggerPlug
{{< / highlight >}}

- Overwrite with the default format in production environment (optional)

The default display format is `"$time $metadata[$level] $message\n"` (in `config/config.ex`), where $message is the logfmt content.
To overwrite in production environment, we can update it in `config/prod.ex`

{{< highlight diff >}}
-config :logger, level: :info
+config :logger, :console,
+  level: :info,
+  format: "$metadata$message\n"
{{< / highlight >}}


## Add Remote IP on Request Log
- To show remote ip in logfmt, we need to add a formatted field in `logger_plug.ex`

{{< highlight diff >}}
...
           path: conn.request_path,
-          status: conn.status
+          status: conn.status,
+          remote_ip: formatted_ip(conn.remote_ip)
        ]
      end
...
   defp formatted_diff(diff), do: [Integer.to_string(diff), "µs"] |> Enum.join(" ")
+  defp formatted_ip(ip) do
+    to_string(:inet_parse.ntoa(ip))
+  end
end
{{< / highlight >}}

- Listening to IPv4 only (optional)

To display ip address in a readable format, could switch to listen only IPv4 addresses in `./lib/[your_app]_web/endpoint.ex`.

{{< highlight diff >}}
   def init(_key, config) do
     if config[:load_from_system_env] do
       port = System.get_env("PORT") || raise "expected the PORT environment variable to be set"
-      {:ok, Keyword.put(config, :http, [:inet6, port: port])}
+      {:ok, Keyword.put(config, :http, [:inet, port: port])}
     else
       {:ok, config}
     end
{{< / highlight >}}

- Show IP address behind load balancer (optional)

Sometimes the application is put behind load balancer and we need to show the real client IP address, we can add a custom plug to read `X-Forwarded-For` http request header and modify `conn.remote_ip` field.

In `./lib/[your_app]_web/endpoint.ex`

{{< highlight diff >}}
   plug Plug.RequestId
+  plug PlugForwardedPeer
   plug HelloWeb.LoggerPlug

   plug Plug.Parsers,
{{< / highlight >}}

And save the custom plug in `./lib/plug_forwarded_peer.ex`.

{{< highlight elixir "linenos=table" >}}
defmodule PlugForwardedPeer do
  import Plug.Conn
  def init(_), do: []
  def call(conn,_) do
    case get_req_header(conn,"x-forwarded-for") do
      []->
        case get_req_header(conn,"forwarded") do
          []-> conn
          [header|_]->
            ips = for "for="<>quoted_ip<-String.split(header,~r/\s*,\s*/), ip=clean_ip(quoted_ip), !is_nil(ip), do: ip
            case ips do
              []->conn
              [ip|_]->%{conn|remote_ip: ip}
            end
        end
      [header|_]->
        ips = for quoted_ip<-String.split(header,~r/\s*,\s*/), ip=clean_ip(quoted_ip), !is_nil(ip), do: ip
        case ips do
          []->conn
          [ip|_]->%{conn|remote_ip: ip}
        end
    end
  end

  def clean_ip(maybe_quoted_ip) do
    maybe_ip = maybe_quoted_ip |> String.strip(?") |> String.rstrip(?]) |> String.lstrip(?[)
    case :inet_parse.address('#{maybe_ip}') do
      {:ok,ip}->ip
      _->nil
    end
  end
end
{{< / highlight >}}

## Feeling tired on reading the source code?
I have already create a [demo repo](https://www.github.com/dhoeric/phoenix-logfmt-demo) contains the above changes. You can directly deploy to Heroku and see the logs in Papertail.
