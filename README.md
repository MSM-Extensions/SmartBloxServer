
# SmartBlox

**A lightweight SmartFoxServer 2X (SFS2X) protocol re-implementation for Roblox Lua**

SmartBlox brings the full SFSObject / SFSArray data model from SmartFoxServer 2X to Roblox using a single `RemoteEvent`. It provides type-safe, compact, deeply nested structures, automatic request/response callbacks, built-in user login & variable synchronization, and expanded array support — perfect for inventory, player data, islands, monsters, matchmaking, and any server ↔ client data syncing.

---

## Features

- Full SFSObject and SFSArray API (exact SFS2X naming & behavior)
- **Expanded primitive array types**: `BoolArray`, `ByteArray`, `ShortArray`, `LongArray`, `FloatArray`, `DoubleArray`, `IntArray`, `UtfStringArray`
- Automatic RID-based request/response callbacks (hidden from user)
- Deeply nested objects and arrays (fully supported)
- Built-in **User login system** with `USER_LOGIN` / `USER_DISCONNECT` / `SYNC_USERS`
- User variable system with public/private sync (real-time updates to clients)
- Serializer that packs/unpacks everything for the `RemoteEvent`
- Zero external dependencies (pure Roblox Lua)
- Debug logging built-in
- Server and Client modules cleanly separated
- Production-ready (heavily stress-tested with 100+ requests, 50-level nesting, 2000+ item arrays)

---

## Installation

1. Place the entire **SmartBlox** folder in `ReplicatedStorage`.
2. On the **Server** (in a Script):

```lua
local SmartBloxServer = require(ReplicatedStorage.SmartBlox.SmartBloxServer)
SmartBloxServer.Init() -- optionally pass settings table
```

3. On the **Client** (in a LocalScript):

```lua
local SmartBloxClient = require(ReplicatedStorage.SmartBlox.SmartBloxClient)
SmartBloxClient.Init() -- or SmartBloxClient.Init(true) for extra debug prints
```

`SmartBloxServer.Init()` automatically creates the `SmartBloxEvent` RemoteEvent in ReplicatedStorage.

---

## Quick Start Example

### Server (in a Script)

```lua
local SmartBloxServer = require(ReplicatedStorage.SmartBlox.SmartBloxServer)
local SmartBloxObject = require(ReplicatedStorage.SmartBlox.Types.SmartBloxObject)
local SmartBloxArray = require(ReplicatedStorage.SmartBlox.Types.SmartBloxArray)
local SmartBloxEvent = require(ReplicatedStorage.SmartBlox.Types.Enums.SmartBloxEvent)

SmartBloxServer.Init()

-- Login handler (required for authentication)
SmartBloxServer:addRequestHandler(SmartBloxEvent.USER_LOGIN, function(player: Player, params: SmartBloxObject)
    print(player.Name .. " is attempting login")
    local secretCode = params:getUtfString("secretCode")
    if secretCode ~= "ABCDEF123" then
        warn(player.Name .. " login failed")
        return false -- kicks the player
    end
    return true -- success, user object will be created
end)

-- Normal command handler (receives User object)
SmartBloxServer:addRequestHandler("db_island", function(user, params: SmartBloxObject)
    print(user.Player.Name .. " requested islands")

    local islandsArray = SmartBloxArray.new()
    local islandObj = SmartBloxObject.new()
    islandObj:putInt("island_id", 1)
    islandObj:putUtfString("name", "Starter Island")
    islandsArray:addSFSObject(islandObj)

    local response = SmartBloxObject.new()
    response:putSFSArray("islands", islandsArray)
    return response
end)

-- Optional: handle user disconnect
SmartBloxServer:addRequestHandler(SmartBloxEvent.USER_DISCONNECT, function(user)
    print(user.Player.Name .. " disconnected")
end)
```

### Client (in a LocalScript)

```lua
local SmartBloxClient = require(ReplicatedStorage.SmartBlox.SmartBloxClient)
local SmartBloxObject = require(ReplicatedStorage.SmartBlox.Types.SmartBloxObject)

SmartBloxClient.Init()

-- Login first (required)
local loginParams = SmartBloxObject.new()
loginParams:putUtfString("secretCode", "ABCDEF123")
SmartBloxClient:login(loginParams)

-- Send request with callback
SmartBloxClient:send("db_island", SmartBloxObject.new(), function(response: SmartBloxObject)
    local islands = response:getSFSArray("islands")
    local first = islands:getSFSObject(1)
    print(first:getInt("island_id")) -- 1
    print(first:getUtfString("name")) -- Starter Island
end)
```

---

## Additional Type Examples

### Using Array types

```lua
local arr = SmartBloxArray.new()

arr:addBool(true)
arr:addByte(127)
arr:addShort(32000)
arr:addInt(2147483647)
arr:addLong(9007199254740991)
arr:addFloat(3.14159)
arr:addDouble(3.141592653589793)
arr:addUtfString("hello")
arr:addText("long text string")

-- Primitive arrays
arr:addBoolArray({true, false, true})
arr:addByteArray({1, 2, 3})
arr:addIntArray({10, 20, 30})
arr:addUtfStringArray({"a", "b", "c"})

-- Nested
local nestedObj = SmartBloxObject.new()
nestedObj:putInt("score", 999)
arr:addSFSObject(nestedObj)
arr:addSFSArray(SmartBloxArray.new():addInt(42))

print(arr:size()) -- 12+
```

### Reading mixed data

```lua
local obj = SmartBloxObject.new()
obj:putSFSArray("inventory", arr)

local inv = obj:getSFSArray("inventory")
print(inv:getBool(1))
print(inv:getIntArray(9)) -- table of numbers
print(inv:getSFSObject(11):getInt("score"))
```

### User variables (server)

```lua
-- Inside any handler that receives `user`
user:setVariable("score", 150, true)     -- public → all clients get update
user:setVariable("health", 100, false)   -- private → only this player

local score = user:getVariable("score")
local all = user:getAllVariables()
local public = user:getPublicVariables()
```

### Listening to variable updates (client)

```lua
SmartBloxClient:addRequestHandler(SmartBloxEvent.USER_VARIABLES_UPDATE, function(response)
    local userId = response:getInt("userId")
    local key = response:getUtfString("key")
    local value = response:getUtfString("value") -- or getInt, getBool, etc.
    local isPublic = response:getBool("public")
    print(`User {userId} updated {key} = {value} (public: {isPublic})`)
end)
```

---

## API Reference

### SmartBloxClient (Client only)

#### `SmartBloxClient.Init(debugEnabled: boolean?)`
Initializes the client and connects to `SmartBloxEvent`.

#### `SmartBloxClient:login(loginParams: SmartBloxObject?)`
Performs login. After success, automatically requests `SYNC_USERS`.

#### `SmartBloxClient:send(command: string, sfsObj: SmartBloxObject, callback: ((response: SmartBloxObject) -> ())?)`
Sends a request. Callback is called with response when server replies.

#### `SmartBloxClient:addRequestHandler(command: string, handler: (response: SmartBloxObject) -> ())`
Registers handler for unsolicited server messages (e.g. `USER_VARIABLES_UPDATE`).

#### `SmartBloxClient:isConnected(): boolean`
Returns whether the client is ready.

---

### SmartBloxServer (Server only)

#### `SmartBloxServer.Init(settings: {UseBuffers: boolean}? )`
Initializes server and creates `SmartBloxEvent`.

#### `SmartBloxServer:addRequestHandler(command: string, handler: (user: User, params: SmartBloxObject) -> SmartBloxObject?)`
Registers handler. For normal commands, `user` is the `User` object.  
**Special commands** (`USER_LOGIN`, `USER_DISCONNECT`) use slightly different signatures (see Quick Start).

#### `SmartBloxServer:send(command: string, sfsObj: SmartBloxObject, target: Player)`
Sends message to a specific client (internal use).

#### `SmartBloxServer:getClientFromPlayer(player: Player): User?`
Returns the `User` wrapper for a player.

#### `SmartBloxServer:getClientFromUserId(userId: number): User?`
Finds user by UserId.

---

### User (Server only – `require(ReplicatedStorage.SmartBlox.Types.Server.User)`)

#### `User.new(player: Player, privilege: number, variables: {[string]: any}?)`
Creates a new user (automatically called after successful login).

#### `User:setVariable(key: string, value: any, isPublic: boolean?)`
Sets a variable. If `isPublic`, it is broadcast to all clients.

#### `User:getVariable(key: string): any`
Returns a single variable.

#### `User:getAllVariables(isSync: boolean?): {[string]: any}`
Returns a copy of all variables.

#### `User:getPublicVariables(): {[string]: any}`
Returns only public variables.

---

### SmartBloxObject

#### Constructor
```lua
SmartBloxObject.new(rawData: table?) -> SmartBloxObject
```

#### Putters (all chainable)
- `putNull(key: string, value: nil)`
- `putBool(key: string, value: boolean)`
- `putByte(key: string, value: number)`
- `putShort(key: string, value: number)`
- `putInt(key: string, value: number)`
- `putLong(key: string, value: number)`
- `putFloat(key: string, value: number)`
- `putDouble(key: string, value: number)`
- `putUtfString(key: string, value: string)`
- `putText(key: string, value: string)`
- `putIntArray(key: string, value: {number})`
- `putBoolArray(key: string, value: {boolean})`
- `putByteArray(key: string, value: {number})`
- `putShortArray(key: string, value: {number})`
- `putLongArray(key: string, value: {number})`
- `putFloatArray(key: string, value: {number})`
- `putDoubleArray(key: string, value: {number})`
- `putUtfStringArray(key: string, value: {string})`
- `putSFSArray(key: string, value: SmartBloxArray)`
- `putSFSObject(key: string, value: SmartBloxObject)`
- `putClass(key: string, value: any)` *(throws error – not supported)*

#### Getters (return `nil` if missing/wrong type)
All matching getters: `getNull`, `getBool`, `getByte`, ..., `getUtfString`, `getText`, `getIntArray`, `getBoolArray`, `getByteArray`, ..., `getUtfStringArray`, `getSFSArray`, `getSFSObject`.

#### Utility
- `containsKey(key: string): boolean`
- `removeElement(key: string)`
- `getKeys(): {string}`
- `size(): number`
- `getRawData(): table`

---

### SmartBloxArray

#### Constructor
```lua
SmartBloxArray.new(rawListData: table?) -> SmartBloxArray
```

#### Adders (all chainable)
- `addNull()`
- `addBool(value: boolean)`
- `addByte(value: number)`
- `addShort(value: number)`
- `addInt(value: number)`
- `addLong(value: number)`
- `addFloat(value: number)`
- `addDouble(value: number)`
- `addUtfString(value: string)`
- `addText(value: string)`
- `addIntArray(value: {number})`
- `addBoolArray(value: {boolean})`
- `addByteArray(value: {number})`
- `addShortArray(value: {number})`
- `addLongArray(value: {number})`
- `addFloatArray(value: {number})`
- `addDoubleArray(value: {number})`
- `addUtfStringArray(value: {string})`
- `addSFSObject(value: SmartBloxObject)`
- `addSFSArray(value: SmartBloxArray)`

#### Getters (return `nil` if missing/wrong type)
All matching getters: `getBool`, `getByte`, ..., `getText`, `getIntArray`, `getBoolArray`, ..., `getUtfStringArray`, `getSFSObject`, `getSFSArray`, plus generic `get(index: number): any`.

#### Utility
- `size(): number`
- `removeElementAt(index: number)`
- `getRawData(): table`
- `contains(value: any): boolean`

---

## Important Notes

- All `put*` / `add*` methods include runtime type assertions (errors in Studio on mismatch).
- Nested SFSObjects and SFSArrays are automatically wrapped/unwrapped by the Serializer.
- `get*` methods are type-safe and return `nil` for missing or wrong-type data.
- The system uses the exact same type codes as SFS2X.
- Login is **mandatory** — players are kicked if `USER_LOGIN` handler returns `false`.
- Public user variables are automatically synced to all clients in real time.
- `SYNC_USERS` is sent automatically after successful login.

**Made for Roblox** — Inspired by SmartFoxServer 2X.  
Happy building!
