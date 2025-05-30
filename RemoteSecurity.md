## > RemoteSecurity.md
### Introduction
Remote events are an instance in Roblox that allow communication between client and server scripts.
Exploiters have the ability to view this communication and even modify data sent from the client using a remote event.
Let's say you wanted to obfuscate the communication of remote event traffic between the client and the server.

### Setup
1. Create a local script in StarterPlayerScripts
2. Create a server script in ServerScriptService
3. Create a remote storage folder in ReplicatedStorage
4. Populate the folder with remote events and remote functions that have an empty name

### Server
When the server is started, you have a small window to initialize everything before the client connects.
This is important because we want to avoid any possibility of the user being able to call a remote spy before the first initial call to the client.

Firstly, you would want a custom GUID module so you don't overload Roblox's servers with HttpService requests.
An example of that looks like this:
```lua
local GUID = {}

-- Formats a number into a two-character hexadecimal string
function GUID:Format(Number)
    if Number < 0 or Number > 255 then return nil end
    return string.upper(string.format("%02X", Number))
end

-- Generates a GUID-like string of the given length
function GUID:Generate(Length)
    math.randomseed(tick())  -- Seed the random generator for better randomness
    local Cache = {}
    for _ = 1, Length do
        table.insert(Cache, self:Format(math.random(0, 255)))
    end
    return table.concat(Cache)
end

-- Generates a full GUID in the format: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
function GUID:GenerateFull()
    math.randomseed(tick())
    local Sections = {4, 2, 2, 2, 6}  -- GUID format breakdown
    local Parts = {}
    
    for _, length in ipairs(Sections) do
        local segment = {}
        for _ = 1, length do
            table.insert(segment, self:Format(math.random(0, 255)))
        end
        table.insert(Parts, table.concat(segment))
    end
    
    return table.concat(Parts, "-")
end

return GUID
```

Using this custom GUID module, set the names of all remotes in the remote storage folder to a random GUID in a loop.
When the client invokes one of these remote functions with a predefined parameter length, create a remote event and return it to the client while parenting it to the remote storage folder with a random GUID name.
That would look something like this:
```lua
local ReplicatedStorage = game:GetService('ReplicatedStorage')

for Index, Remote in pairs(ReplicatedStorage.Remotes:GetChildren()) do
	if Remote.ClassName == "RemoteFunction" then
		Remote.OnServerInvoke = function(Client, ...)
			if #({...}) == 100 then
				local RemoteEvent = Instance.new("RemoteEvent")
				return RemoteEvent
			end
		end
	end
end
```

### Client
To prevent any viewing of the client source code, create a table containing some random hashes with the indexes being hexadecimal values.
Then, create another table to fetch these hashes with normal integer indexes.
When the source is decompiled, the hexadecimal will evaluate to what it's converted to, giving us the definition for *security by obscurity*.
```lua
local HT_90d64eeba8247d656ef6b4800ec0f52f = {
    [0x31]      = "31CEB6C0-888A-4CDB-ABD2-1980A1C455E1",
    [0x32]      = "E6493637-8F68-47DA-ACA4-FB604459A07C",
    [0x33]      = "28533C00-DEBB-44F3-91FA-14A088AEF4F0",
    [0x34]      = "728FAB66-2E30-421C-B3D5-746C5F95DA1A",
    [0x35]      = "61654309-F7AE-4666-9C92-5FB0E963C5E9",
    [0x36]      = "B20BE1FB-6490-4972-856F-A9ABEE8F0FA6",
    [0x37]      = "CAE1F5D2-600E-462E-8979-79A08C4574DE",
    [0x38]      = "42337869-8A7D-438F-8516-0BC4A5BC28A1",
    [0x39]      = "9BA1115F-FF64-4E78-8F41-7A28DF30293A",
    [0x3130]    = "521BABBC-70A3-4B35-B03C-A0BF54C8F10B",
    [0x3131]    = "4DBCD948-2755-4264-8ECA-65A81BFD35B8",
    [0x3132]    = "3E5B37FB-2041-4434-B0F3-9BFFBEA46CBC"
}

local IDX_6a992d5529f459a44fee58c733255e86 = {
    [1] = 0x31,
    [2] = 0x32,
    [3] = 0x33,
    [4] = 0x34,
    [5] = 0x35,
    [6] = 0x36,
    [7] = 0x37,
    [8] = 0x38,
    [9] = 0x39,
    [10] = 0x3130,
    [11] = 0x3131,
    [12] = 0x3132
}
```

Next, create a table that can be called as a function by virtue of the `__call` metamethod.
This table will be used to fetch a random hash in the hash table.
Now, in order to get that remote event from the server we need to invoke a remote function with a specific parameter size.
So, we will store that predefined amount of hashes in the same table we call to get the hashes.
```lua
shared.Key_eec89088ee408b80387155272b113256 = {}

setmetatable(shared.Key_eec89088ee408b80387155272b113256, {
    __call = function(tb1, ...)
        return HT_90d64eeba8247d656ef6b4800ec0f52f[IDX_6a992d5529f459a44fee58c733255e86[math.random(1, #IDX_6a992d5529f459a44fee58c733255e86)]]
    end
})

local Key_0fea6a13c52b4d4725368f24b045ca84 = {}

for i = 1, 100, 1 do
    local hash = shared.Key_eec89088ee408b80387155272b113256()
    table.insert(Key_0fea6a13c52b4d4725368f24b045ca84, #Key_0fea6a13c52b4d4725368f24b045ca84+1, hash)
end
```

If we called upon any remotes normally, an exploiter could use metamethod hijacking (also called namecall hijacking) to see exactly which remote is being fired, retrieve it from the Lua registry by searching function upvalues, and call it as they wish.
The code for that in an exploit looks something like this:
```lua
local gmt = getrawmetatable(game)
local o_namecall = gmt.__namecall
setreadonly(gmt, false)

gmt.__namecall = newcclosure(function(self, ...)
    local args = {...}
    local method = getnamecallmethod()


    if method == "FireServer" or method == "InvokeServer" then
        warn("Remote called!")
        warn("Location: ", self:GetFullName())
        warn("Type: ", method)
        warn("Args: ", unpack(args))
    end
    
    return o_namecall(self, unpack(args))
end)
```

To mitigate this, we can embed the true calls to the server in a table with the remote being the first argument.
This would prevent any automation scripts that call remotes automatically via that remote spy.
```lua
function shared.Key_eec89088ee408b80387155272b113256:InvokeServer(Key_cf51066f49e517f274b8173cc265c60b, ...)
    return Key_cf51066f49e517f274b8173cc265c60b:InvokeServer(...)
end

function shared.Key_eec89088ee408b80387155272b113256:FireServer(Key_cf51066f49e517f274b8173cc265c60b, ...)
    Key_cf51066f49e517f274b8173cc265c60b:FireServer(...)
end
```

Now we can invoke the server to get the remote event generated by the server.
Make sure to use CollectionService on the remote event so you don't lose it.
And finally, to finish thing off, we can start a loop that calls our 'fake' remote event table which has no impact on the client-to-server bandwidth and blocks the exploiter from even seeing any traffic in their developer console using a remote spy.

### Conclusion
The only real thing to worry about here are two things:
1. If anyone gets into your studio and views the code, they would be able to read what's happening and figure out your system. This is also possible yet slower with decompilation, but that's just because nothing is ever fully secure when there is a client involved.
2. If an exploiter did decompile your client script, they would determine that the key would be the parameter amount. You could prevent any automation scripts by changing that value every update, but a dedicated exploiter could bypass this.
