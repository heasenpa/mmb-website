--========================================================
-- SERVER HOP GUI
-- 1-5 PLAYERS
-- Priority: 2 > 3 > 1 > 4 > 5
--========================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer

--========================================================
-- CONFIG
--========================================================

local MIN_PLAYERS = 1
local MAX_PLAYERS = 5

-- 10 trang x tối đa 100 server
local MAX_PAGES = 10

-- Ưu tiên số người
local PRIORITY = {
    2,
    3,
    1,
    4,
    5
}

--========================================================
-- LẤY SERVER TRƯỚC ĐÓ AN TOÀN
--========================================================

local LastServer = {}

pcall(function()

    local data = TeleportService:GetLocalPlayerTeleportData()

    if type(data) == "table" then
        LastServer = data
    end

end)

--========================================================
-- XÓA GUI CŨ
--========================================================

pcall(function()

    local old = CoreGui:FindFirstChild("ServerHopGUI")

    if old then
        old:Destroy()
    end

end)

--========================================================
-- GUI
--========================================================

local ScreenGui = Instance.new("ScreenGui")

ScreenGui.Name = "ServerHopGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
ScreenGui.DisplayOrder = 999999

ScreenGui.Parent = CoreGui

--========================================================
-- BUTTON
--========================================================

local HopButton = Instance.new("TextButton")

HopButton.Name = "ServerHopButton"

HopButton.Size = UDim2.new(
    0,
    90,
    0,
    40
)

HopButton.Position = UDim2.new(
    1,
    -105,
    0.5,
    -20
)

HopButton.BackgroundColor3 =
    Color3.fromRGB(35, 35, 35)

HopButton.BackgroundTransparency = 0.05

HopButton.BorderSizePixel = 0

HopButton.Text = "Server Hop"

HopButton.TextColor3 =
    Color3.fromRGB(255, 255, 255)

HopButton.TextSize = 13

HopButton.Font =
    Enum.Font.GothamBold

HopButton.ZIndex = 100

HopButton.Parent = ScreenGui

local ButtonCorner = Instance.new("UICorner")

ButtonCorner.CornerRadius =
    UDim.new(0, 8)

ButtonCorner.Parent = HopButton

local ButtonStroke = Instance.new("UIStroke")

ButtonStroke.Thickness = 2

ButtonStroke.Color =
    Color3.fromRGB(120, 0, 255)

ButtonStroke.Parent = HopButton

--========================================================
-- STATUS
--========================================================

local Status = Instance.new("TextLabel")

Status.Name = "Status"

Status.Size = UDim2.new(
    0,
    230,
    0,
    25
)

Status.Position = UDim2.new(
    1,
    -240,
    0.5,
    27
)

Status.BackgroundTransparency = 1

Status.Text = ""

Status.TextColor3 =
    Color3.fromRGB(255, 255, 255)

Status.TextSize = 12

Status.Font =
    Enum.Font.Gotham

Status.TextXAlignment =
    Enum.TextXAlignment.Right

Status.ZIndex = 100

Status.Parent = ScreenGui

--========================================================
-- DRAG BUTTON
--========================================================

local dragging = false
local dragStart
local startPos

local function updateButton(input)

    local delta =
        input.Position - dragStart

    HopButton.Position = UDim2.new(
        startPos.X.Scale,
        startPos.X.Offset + delta.X,

        startPos.Y.Scale,
        startPos.Y.Offset + delta.Y
    )

    Status.Position = UDim2.new(
        HopButton.Position.X.Scale,
        HopButton.Position.X.Offset - 140,

        HopButton.Position.Y.Scale,
        HopButton.Position.Y.Offset + 47
    )

end

HopButton.InputBegan:Connect(function(input)

    if input.UserInputType ==
        Enum.UserInputType.MouseButton1
        or input.UserInputType ==
        Enum.UserInputType.Touch then

        dragging = true

        dragStart = input.Position
        startPos = HopButton.Position

        input.Changed:Connect(function()

            if input.UserInputState ==
                Enum.UserInputState.End then

                dragging = false

            end

        end)

    end

end)

UserInputService.InputChanged:Connect(function(input)

    if not dragging then
        return
    end

    if input.UserInputType ==
        Enum.UserInputType.MouseMovement
        or input.UserInputType ==
        Enum.UserInputType.Touch then

        updateButton(input)

    end

end)

--========================================================
-- GET SERVER LIST
--========================================================

local function GetServers()

    local servers = {}

    local cursor = ""

    for page = 1, MAX_PAGES do

        local url =
            "https://games.roblox.com/v1/games/"
            .. tostring(game.PlaceId)
            .. "/servers/Public?sortOrder=Asc&limit=100"

        if cursor ~= "" then

            url = url ..
                "&cursor=" ..
                HttpService:UrlEncode(cursor)

        end

        local success, response = pcall(function()

            return game:HttpGet(url)

        end)

        if not success then

            warn(
                "[ServerHop] HttpGet failed:",
                response
            )

            break

        end

        local decodeSuccess, data =
            pcall(function()

                return HttpService:JSONDecode(response)

            end)

        if not decodeSuccess or not data then

            warn(
                "[ServerHop] JSON decode failed"
            )

            break

        end

        if data.data then

            for _, server in ipairs(data.data) do

                if server.id
                and server.id ~= game.JobId
                and server.id ~= LastServer.id
                and server.playing
                and server.playing >= MIN_PLAYERS
                and server.playing <= MAX_PLAYERS then

                    table.insert(
                        servers,
                        server
                    )

                end

            end

        end

        cursor =
            data.nextPageCursor or ""

        if cursor == "" then
            break
        end

        task.wait(0.15)

    end

    return servers

end

--========================================================
-- CHOOSE SERVER
--========================================================

local function ChooseServer(servers)

    if #servers == 0 then
        return nil
    end

    local groups = {
        [1] = {},
        [2] = {},
        [3] = {},
        [4] = {},
        [5] = {}
    }

    for _, server in ipairs(servers) do

        local count = server.playing

        if groups[count] then

            table.insert(
                groups[count],
                server
            )

        end

    end

    -- 2 > 3 > 1 > 4 > 5

    for _, count in ipairs(PRIORITY) do

        local group = groups[count]

        if group and #group > 0 then

            return group[
                math.random(1, #group)
            ]

        end

    end

    return nil

end

--========================================================
-- SERVER HOP
--========================================================

local hopping = false

local function ServerHop()

    if hopping then
        return
    end

    hopping = true

    HopButton.Active = false

    HopButton.Text =
        "Searching..."

    Status.Text =
        "Finding 1-5 player server..."

    local servers = GetServers()

    if #servers == 0 then

        HopButton.Text =
            "Server Hop"

        HopButton.Active = true

        Status.Text =
            "No suitable server"

        hopping = false

        task.delay(2, function()

            if Status then
                Status.Text = ""
            end

        end)

        return

    end

    Status.Text =
        "Found "
        .. tostring(#servers)
        .. " servers"

    task.wait(0.2)

    local target =
        ChooseServer(servers)

    if not target then

        HopButton.Text =
            "Server Hop"

        HopButton.Active = true

        Status.Text =
            "No server found"

        hopping = false

        return

    end

    Status.Text =
        "Joining "
        .. tostring(target.playing)
        .. " players..."

    HopButton.Text =
        "Teleporting..."

    task.wait(0.2)

    --====================================================
    -- TELEPORT
    -- Giống cơ chế script farm của bạn
    --====================================================

    local success, errorMessage =
        pcall(function()

            TeleportService:TeleportToPlaceInstance(
                game.PlaceId,
                target.id,
                LocalPlayer,
                {
                    id = game.JobId
                }
            )

        end)

    if not success then

        warn(
            "[ServerHop] Teleport failed:",
            errorMessage
        )

        HopButton.Text =
            "Server Hop"

        HopButton.Active = true

        Status.Text =
            "Teleport failed"

        hopping = false

        task.delay(2, function()

            if Status then
                Status.Text = ""
            end

        end)

    end

end

--========================================================
-- BUTTON CLICK
--========================================================

HopButton.MouseButton1Click:Connect(function()

    ServerHop()

end)

--========================================================
-- READY
--========================================================

print("==============================")
print(" Server Hop GUI loaded")
print(" Range: 1-5 players")
print(" Priority: 2 > 3 > 1 > 4 > 5")
print(" Max pages:", MAX_PAGES)
print("==============================")

Status.Text = "Ready"

task.delay(2, function()

    if Status then
        Status.Text = ""
    end

end)
