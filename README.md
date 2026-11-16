# Satellite HTTP

A HTTP server library for Pluto, a fork of Lua.

## Installation

Download this library via your **deps.pluto**:

```lua
;(require"http".request"//use.agnostic.pm"|>load)()

git "https://github.com/PlutoLang/satellite-http"
    from "core.pluto" to "lib/satellite/http/core.pluto"
    from "router.pluto" to "lib/satellite/http/router.pluto"
```

## Examples

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

app:listen(80, || -> print("Listening on http://localhost"))
```

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
hs.onaccept = function(s)
    print(s:getpeer().." - connected")
end
hs.onclose = function(s)
    print(s:getpeer().." - disconnected")
end
hs:bind(sched, 80)
print("Listening on http://localhost")
sched:run()
```
