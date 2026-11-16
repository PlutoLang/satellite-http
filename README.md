# Satellite HTTP

A HTTP server library for Pluto, a fork of Lua.

## Example: Echo Server

```lua
;(require"http".request"//use.agnostic.pm"|>load)()

git "https://github.com/PlutoLang/satellite-http"
    from "core.pluto" to "lib/satellite/http/core.pluto"
```
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
