# SmartBlox

**A lightweight SmartFoxServer 2X (SFS2X) protocol re-implementation for Roblox Lua**

#### This read-me was created with the help of generative AI. The code was not.

SmartBlox brings the familiar `SFSObject` / `SFSArray` data model from SmartFoxServer 2X to Roblox, using only a single `RemoteEvent`.  
It gives you type-safe, compact, nested data structures and a clean request/response pattern with callbacks — perfect for games that need server → client data syncing (inventory, player data, islands, monsters, etc.).

---

## Features

- Full `SFSObject` and `SFSArray` API (matching SFS2X naming)
- Automatic RID-based request/response callbacks
- Nested objects and arrays supported
- Serializer that handles packing/unpacking for the RemoteEvent
- Zero external dependencies (pure Roblox Lua)
- Debug logging built-in
- Server and Client modules clearly separated

---

## Installation

1. Put the entire `SmartBlox` folder in `ReplicatedStorage`.
2. On the **Server** (in a Script):

```lua
local SmartBloxServer = require(ReplicatedStorage.SmartBlox.SmartBloxServer)
SmartBloxServer.Init()
```

3. On the **Client** (in a LocalScript):

```lua
local SmartBloxClient = require(ReplicatedStorage.SmartBlox.SmartBloxClient)
SmartBloxClient.Init() -- or SmartBloxClient.Init(true) for debug
```

`SmartBloxServer.Init()` automatically creates the `SmartBloxEvent` RemoteEvent in ReplicatedStorage.

---

## Quick Start Example

**Server** (`db_island` handler):

```lua
SmartBloxServer:addRequestHandler("db_island", function(player, params)
    local islandsArray = SmartBloxArray.new()

    local islandObj = SmartBloxObject.new()
    islandObj:putInt("island_id", 1)
    islandsArray:addSFSObject(islandObj)

    local response = SmartBloxObject.new()
    response:putSFSArray("islands", islandsArray)

    return response
end)
```

**Client**:

```lua
SmartBloxClient:send("db_island", SmartBloxObject.new(), function(response)
    local islands = response:getSFSArray("islands")
    local firstIsland = islands:get(1)
    print(firstIsland:getInt("island_id")) -- 1
end)
```

---

## API Reference

### SmartBloxClient (Client only)

#### `SmartBloxClient.Init(debugEnabled: boolean?)`
Initializes the client and connects to the RemoteEvent.  
Call this once at the start of your client scripts.

#### `SmartBloxClient:send(command: string, sfsObj: SmartBloxObject, callback: ((response: SmartBloxObject) -> ())?)`
Sends a request to the server.  
If a callback is provided, it will be called with the response object when the server replies.

#### `SmartBloxClient:addRequestHandler(command: string, handler: (response: SmartBloxObject) -> ())`
(Advanced) Registers a handler for unsolicited messages from the server.

#### `SmartBloxClient:isConnected(): boolean`
Returns whether the client is ready.

---

### SmartBloxServer (Server only)

#### `SmartBloxServer.Init()`
Initializes the server and creates the `SmartBloxEvent` RemoteEvent.

#### `SmartBloxServer:addRequestHandler(command: string, handler: (player: Player, params: SmartBloxObject) -> SmartBloxObject?)`
Registers a handler for a specific command.  
The handler receives the player and the request object and should return a response object (or `nil` for no reply).

#### `SmartBloxServer:send(command: string, sfsObj: SmartBloxObject, target: Player)`
(Internal) Sends a message to a specific client. Usually not called directly.

---

### SmartBloxObject

#### Constructor
```lua
SmartBloxObject.new(rawData: table?) -> SmartBloxObject
```

#### Putters (chainable)
- `putNull(key: string, value: nil)`
- `putBool(key: string, value: boolean)`
- `putByte(key: string, value: number)`
- `putShort(key: string, value: number)`
- `putInt(key: string, value: number)`
- `putLong(key: string, value: number)`
- `putFloat(key: string, value: number)`
- `putDouble(key: string, value: number)`
- `putUtfString(key: string, value: string)`
- `putIntArray(key: string, value: {number})`
- `putSFSArray(key: string, value: SmartBloxArray)`
- `putSFSObject(key: string, value: SmartBloxObject)`
- `putText(key: string, value: string)`

#### Getters
- `getNull(key: string): nil`
- `getBool(key: string): boolean?`
- `getByte(key: string): number?`
- `getShort(key: string): number?`
- `getInt(key: string): number?`
- `getLong(key: string): number?`
- `getFloat(key: string): number?`
- `getDouble(key: string): number?`
- `getUtfString(key: string): string?`
- `getText(key: string): string?`
- `getIntArray(key: string): {number}?`
- `getSFSObject(key: string): SmartBloxObject?`
- `getSFSArray(key: string): SmartBloxArray?`

#### Utility Methods
- `containsKey(key: string): boolean`
- `removeElement(key: string)`
- `getKeys(): {string}`
- `size(): number`
- `getRawData(): table` — Returns the raw packed data

---

### SmartBloxArray

#### Constructor
```lua
SmartBloxArray.new(rawListData: table?) -> SmartBloxArray
```

#### Adders (chainable)
- `addBool(value: boolean)`
- `addByte(value: number)`
- `addShort(value: number)`
- `addInt(value: number)`
- `addUtfString(value: string)`
- `addSFSObject(value: SmartBloxObject)`
- `addSFSArray(value: SmartBloxArray)`
- `addNull()`
- `addLong(value: number)`
- `addFloat(value: number)`
- `addDouble(value: number)`
- `addText(value: string)`
- `addIntArray(value: {number})`
- `addUtfStringArray(value: {string})`

#### Getters
- `getBool(index: number): boolean?`
- `getByte(index: number): number?`
- `getShort(index: number): number?`
- `getInt(index: number): number?`
- `getLong(index: number): number?`
- `getFloat(index: number): number?`
- `getDouble(index: number): number?`
- `getText(index: number): string?`
- `getIntArray(index: number): {number}?`
- `getUtfStringArray(index: number): {string}?`
- `getUtfString(index: number): string?`
- `getSFSObject(index: number): SmartBloxObject?`
- `getSFSArray(index: number): SmartBloxArray?`
- `get(index: number): any` — Returns the raw value or wrapped object/array

#### Utility Methods
- `size(): number`
- `removeElementAt(index: number)`
- `getRawData(): table`
- `contains(value: any): boolean`

---

## Important Notes

- **Nested objects/arrays** are fully supported (automatically wrapped into proper `SmartBloxObject` / `SmartBloxArray` instances).
- All `get*` methods return `nil` if the key/index doesn't exist or has the wrong type.
- `put*` and `add*` methods include runtime assertions for type safety (they will error in Studio if you pass wrong data).
- The system uses the exact same type codes as SFS2X for maximum compatibility.

---

## Stress Testing & Production Readiness

**This module has been heavily stress-tested** with the following scenarios:

- 100+ rapid-fire requests (pings)
- Deep nesting up to **50 levels**
- Large payloads with **2000 items** in arrays (including long strings and numeric values)
- Mixed nested objects + arrays
- Edge cases: missing keys, wrong-type getters (correctly returns `nil`)

All tests completed successfully with no serializer crashes, no assertion failures from the module, and stable callback behavior.  
Round-trip times remained reasonable even under heavy load (0.17–0.35 seconds for large payloads in Studio).

**Conclusion**: The core functionality (serializer, object/array system, RID callbacks, and single RemoteEvent handling) is stable and **ready for production use** in small to medium-sized games.
---

**Made for Roblox** — Inspired by SmartFoxServer 2X.

If you have any questions or want to contribute, open an issue or PR on GitHub!

Happy building! 🚀
