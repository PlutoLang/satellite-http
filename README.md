# Satellite HTTP

A HTTP server library for Pluto, a fork of Lua.

## Table of Contents

- [Installation](#installation)
- [Examples](#examples)
  - [Echo Server](#echo-server)
  - [Express-like App](#express-like-app)
  - [Chat Server](#chat-server)
- [API Reference](#api-reference)
  - [Modules](#modules)
  - [*HttpServer*](#httpserver)
  - [*HttpRouter*](#httprouter)
  - [*HttpRequest*](#httprequest)
  - [*HttpResponse*](#httpresponse)
  - [*HttpResponseTable*](#httpresponsetable)

## Installation

Download this library via your **deps.pluto**:

```lua
;(require"http".request"//use.agnostic.pm"|>load)()

git "https://github.com/PlutoLang/satellite-http"
    from "core.pluto" to "lib/satellite/http/core.pluto"
    from "router.pluto" to "lib/satellite/http/router.pluto"
```

## Examples

### Echo Server

```lua
package.path ..= package.config[3].."lib/?.pluto":replace("/", package.config[1])

local { scheduler } = require "*"
local { HttpServer } = require "satellite.http.core"

local sched = new scheduler()
local hs = new HttpServer(function(request)
    print(request.socket:getpeer().." - request")
    return {
        status = "200 :)",
        body = dumpvar(request),
    }
end)
hs:bind(sched, 80)
print("Listening on http://localhost")
hs.onaccept = |s| -> print(s:getpeer().." - connected")
hs.onclose = |s| -> print(s:getpeer().." - disconnected")
sched:run()
```

### Express-like App

```lua
package.path ..= package.config[3].."lib/?.pluto":replace("/", package.config[1])

local { HttpRouter } = require "satellite.http.router"

local app = new HttpRouter()

app:get("/", |_req, res| -> do
    res:html[[
        <p>Welcome to this epic Satellite HTTP server, we've got:</p>
        <ul>
            <li><a href="/echo">echoes</a></li>
            <li><a href="/scream">screaming</a></li>
            <li><a href="/delay">waiting</a></li>
        </ul>
    ]]
end)

app:get("/echo", |req, res| -> do
    res:text(dumpvar(req))
end)

app:get("/scream", |_req, res| -> do
    res:setHeader("Content-Encoding", "gzip")
    res:text(string.fromhex("1F8B08005287FA6A0200ECC181000000008020D6FD2516A90200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000E0F6E040000000004090BFF5002B54000000000000000013EEFB35F32C0F0100"))
end)

app:get("/delay", |_req, res| -> do
    res:status("200 :)")
    res:setText()
    res:setHeader("X-Content-Type-Options", "nosniff")
    res:send("3...\r\n")
    coroutine.sleep(1000)
    res:send("2...\r\n")
    coroutine.sleep(1000)
    res:send("1...\r\n")
    coroutine.sleep(1000)
    return "Boom!"
end)

app:listen(80, |hs| -> do
    print("Listening on http://localhost")
    hs.onaccept = |s| -> print(s:getpeer().." - connected")
    hs.onclose = |s| -> print(s:getpeer().." - disconnected")
end)
```

### Chat Server

```lua
;(require"http".request"//use.agnostic.pm"|>load)()

git "https://github.com/PlutoLang/satellite-http"
    from "core.pluto" to "lib/satellite/http/core.pluto"
    from "router.pluto" to "lib/satellite/http/router.pluto"

out "lib"
    git "https://github.com/PlutoLang/pluto-websocket"
```
```lua
package.path ..= package.config[3].."lib/?.pluto":replace("/", package.config[1])

local { HttpRouter } = require "satellite.http.router"
local { wsIsUpgradeRequest } = require "websocket"

local app = new HttpRouter()

app:get("/", |_req, res| -> do
    res:html[[
        <style>
            body {
                font-family: sans-serif;
                max-width: 1000px;
                margin: 2rem auto;
            }
            form {
                display: flex;
                gap: 0.5rem;
                margin-bottom: 1rem;
            }
            textarea {
                width: 100%;
                height: 400px;
            }
        </style>
        <form onsubmit="sendMessage(event)">
            <input id="name" type="text" placeholder="Name" style="width: 20%;" required />
            <input id="text" type="text" placeholder="Message" style="flex: 1;" required />
            <input type="submit" value="Send" />
        </form>
        <textarea readonly></textarea>
        <script>
            const ws = new WebSocket("/ws");
            ws.onmessage = (event) => {
                document.querySelector("textarea").textContent = event.data + "\n" + document.querySelector("textarea").textContent;
            };
            function sendMessage(event) {
                event.preventDefault();
                if (ws.readyState == WebSocket.OPEN) {
                    ws.send(`${document.getElementById("name").value.trim()}: ${document.getElementById("text").value.trim()}`);
                    document.getElementById("text").value = "";
                }
            }
        </script>
    ]]
end)

local chat_clients = setmetatable({}, { __mode = "k" })
app:get("/ws", |req, _res| -> do
    if key := wsIsUpgradeRequest(req.rawdata) then
        req.socket:wsUpgrade(key)
        chat_clients[req.socket] = true
        pcall(function()
            while data := req.socket:wsRecv() do
                for s in chat_clients do
                    s:wsSend(data)
                end
            end
        end)
        chat_clients[req.socket] = nil
    else
        return { status = 400 }
    end
end)

app:listen(80, |hs| -> do
    print("Listening on http://localhost")
    hs.onaccept = |s| -> print(s:getpeer().." - connected")
    hs.onclose = |s| -> print(s:getpeer().." - disconnected")
end)
```

## API Reference

### Modules
- `satellite.http.core` exports { [*HttpServer*](#httpserver) }
- `satellite.http.router` exports { [*HttpRouter*](#httprouter) }

### *HttpServer*
This class has the following members:
- `__construct(onrequest: function(request: HttpRequest, response: HttpResponse): ?HttpResponseTable|string)` — constructor, sets the *onrequest* field
- `onrequest: function(request: HttpRequest, response: HttpResponse): ?HttpResponseTable|string` — request callback, can return a response as "plain old data" (string or [*HttpResponseTable*](#httpresponsetable)) or use [the *HttpResponse* API](#httpresponse)
- `onaccept: ?function(socket)` — called client connection is accepted
- `onclose: ?function(socket)` — called when a client connection is closed
- `onerror: ?function(err: string)` — called when `onrequest` throws an error, defaults to *|err| -> print(err)*
- `bind: function(sched: scheduler, port: int)` — starts listening on the given port, using the given [scheduler](https://pluto-lang.org/docs/Runtime%20Environment/Scheduler)
- `accept: function(socket)` — serves a client connection, handling requests until the client disconnects (used internally by *bind*)

### *HttpRouter*
This class has the following members:
- `get: function(path: string, handler: function(request: HttpRequest, response: HttpResponse): ?HttpResponseTable|string): HttpRouter` — registers a GET request handler for the given path
- `post: function(path: string, handler: function(request: HttpRequest, response: HttpResponse): ?HttpResponseTable|string): HttpRouter` — registers a POST request handler for the given path
- `listen: function(port: int, callback: ?function(server: HttpServer))` — creates an [*HttpServer*](#httpserver) that dispatches requests through the router; this call does not return, tho it may error
- `handleRequest: function(request: HttpRequest, response: HttpResponse): ?HttpResponseTable|string` — dispatches a request to the matching handler and returns its result or returns an appropriate error response (used internally by *listen*)

### *HttpRequest*
This named type is a table with the following members:
- `method: string`
- `path: string`
- `headers: table<string, string>`
- `body: ?string`
- `socket: socket` — the underlying client connection
- `rawdata: string` — the raw data received for the request, as read from the socket before the body was completed

### *HttpResponse*
This class has the following members:
- `headers_sent: bool`
- `status: function(value: string|int): HttpResponse`
- `hasHeader: function(name: string): bool`
- `getHeader: function(name: string): string|nil`
- `setHeader: function(name: string, value: string): HttpResponse`
- `setText: function(): HttpResponse` — sets the **"Content-Type"** header to **"text/plain; charset=utf-8"**
- `setHtml: function(): HttpResponse` — sets the **"Content-Type"** header to **"text/html; charset=utf-8"**
- `send: function(data: string): HttpResponse` — sends a response chunk
- `end: function(data: ?string): HttpResponse` — finishing the response, optionally sending the given body data (or final response chunk, if *send* was called before)
- `text: function(data: string): HttpResponse` — sets the **"Content-Type"** header to **"text/plain; charset=utf-8"** and sends the given data, finishing the response
- `html: function(data: string): HttpResponse` — sets the **"Content-Type"** header to **"text/html; charset=utf-8"** and sends the given data, finishing the response

### *HttpResponseTable*
This named type is a table with the following members:
- `status: ?string|int`
- `headers: ?table<string, string>`
- `body: ?string`
