--// SERVER HOP - LIGHT
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local Http = game:GetService("HttpService")
local TS = game:GetService("TeleportService")
local CG = game:GetService("CoreGui")

local LP = Players.LocalPlayer
local PLACE = game.PlaceId
local JOB = game.JobId

local MAX_PAGES = 10
local RETRY_DELAY = 5
local PRIORITY = {2,3,1,4,5}

--// Last server
local lastID
pcall(function()
    local d = TS:GetLocalPlayerTeleportData()
    if type(d) == "table" then
        lastID = d.id
    end
end)

--// GUI
pcall(function()
    CG:FindFirstChild("LightServerHop"):Destroy()
end)

local gui = Instance.new("ScreenGui")
gui.Name = "LightServerHop"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Global
gui.Parent = CG

local btn = Instance.new("TextButton")
btn.Size = UDim2.new(0,90,0,38)
btn.Position = UDim2.new(1,-105,.5,-20)
btn.BackgroundColor3 = Color3.fromRGB(35,35,35)
btn.TextColor3 = Color3.new(1,1,1)
btn.Text = "Server Hop"
btn.TextSize = 13
btn.Font = Enum.Font.GothamBold
btn.Parent = gui

Instance.new("UICorner",btn).CornerRadius = UDim.new(0,8)

local stroke = Instance.new("UIStroke",btn)
stroke.Color = Color3.fromRGB(130,0,255)
stroke.Thickness = 2

local status = Instance.new("TextLabel")
status.Size = UDim2.new(0,220,0,22)
status.Position = UDim2.new(1,-230,.5,23)
status.BackgroundTransparency = 1
status.TextColor3 = Color3.new(1,1,1)
status.TextSize = 11
status.Font = Enum.Font.Gotham
status.TextXAlignment = Enum.TextXAlignment.Right
status.Parent = gui

--// Drag
local drag, start, pos
btn.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1
    or i.UserInputType == Enum.UserInputType.Touch then
        drag = true
        start = i.Position
        pos = btn.Position
        i.Changed:Connect(function()
            if i.UserInputState == Enum.UserInputState.End then
                drag = false
            end
        end)
    end
end)

UIS.InputChanged:Connect(function(i)
    if drag and (
        i.UserInputType == Enum.UserInputType.MouseMovement
        or i.UserInputType == Enum.UserInputType.Touch) then

        local d = i.Position - start

        btn.Position = UDim2.new(
            pos.X.Scale,pos.X.Offset+d.X,
            pos.Y.Scale,pos.Y.Offset+d.Y
        )

        status.Position = UDim2.new(
            btn.Position.X.Scale,
            btn.Position.X.Offset-130,
            btn.Position.Y.Scale,
            btn.Position.Y.Offset+43
        )
    end
end)

--// Get servers
local function getServers()
    local list = {}
    local cursor = ""

    for page = 1,MAX_PAGES do
        local url =
            "https://games.roblox.com/v1/games/"
            ..PLACE.."/servers/Public?sortOrder=Asc&limit=100"

        if cursor ~= "" then
            url = url.."&cursor="..Http:UrlEncode(cursor)
        end

        local ok,res = pcall(game.HttpGet,game,url)
        if not ok then break end

        local good,data = pcall(Http.JSONDecode,Http,res)
        if not good or type(data) ~= "table" then break end

        for _,s in ipairs(data.data or {}) do
            local n = tonumber(s.playing)

            if s.id
            and s.id ~= JOB
            and s.id ~= lastID
            and n
            and n >= 1
            and n <= 5
            and (not s.maxPlayers or n < s.maxPlayers) then
                table.insert(list,s)
            end
        end

        if #list >= 30 then break end

        cursor = data.nextPageCursor or ""
        if cursor == "" then break end

        task.wait(.1)
    end

    return list
end

--// Sort 2 > 3 > 1 > 4 > 5
local function sortServers(list)
    local out = {}

    for _,wanted in ipairs(PRIORITY) do
        for _,s in ipairs(list) do
            if tonumber(s.playing) == wanted then
                table.insert(out,s)
            end
        end
    end

    return out
end

local running = false
local failed = {}

--// Teleport
local function teleport(s)
    if not s or failed[s.id] then
        return
    end

    failed[s.id] = true

    pcall(function()
        TS:TeleportToPlaceInstance(
            PLACE,
            s.id,
            LP,
            {id = JOB}
        )
    end)
end

--// Main
local function hop()
    if running then return end
    running = true
    failed = {}

    btn.Active = false

    while running do
        status.Text = "Searching..."
        btn.Text = "Searching"

        local servers = sortServers(getServers())

        if #servers == 0 then
            status.Text = "No server - retrying"
            task.wait(RETRY_DELAY)
        else
            local success = false

            for _,s in ipairs(servers) do
                if not failed[s.id] then
                    local n = tonumber(s.playing) or 0

                    status.Text =
                        "Trying: "..n.." players"
                    btn.Text = "Teleport"

                    teleport(s)
                    success = true
                    break
                end
            end

            if not success then
                task.wait(RETRY_DELAY)
            end
        end
    end
end

--// 772 / Teleport failed
TS.TeleportInitFailed:Connect(function(player)
    if player ~= LP or not running then return end

    status.Text = "Failed - searching again"
    btn.Text = "Retry"

    task.wait(.5)
    -- vòng while sẽ tự tìm server mới
end)

btn.MouseButton1Click:Connect(hop)

status.Text = "Ready"
