--// Server Hop 1-5 Players
--// Ưu tiên: 2 người > 3 người > 1 người > 4 người > 5 người
--// GUI nhỏ + hỗ trợ Mobile + kéo được

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local PlaceId = game.PlaceId
local CurrentJobId = game.JobId

--==================================================
-- CONFIG
--==================================================

local MIN_PLAYERS = 1
local MAX_PLAYERS = 5

-- Số trang server tối đa muốn quét
-- 1 trang ~= tối đa 100 server
local MAX_PAGES = 10

-- Thứ tự ưu tiên
local PRIORITY = {
    2,
    3,
    1,
    4,
    5
}

--==================================================
-- GUI
--==================================================

local oldGui = CoreGui:FindFirstChild("ServerHopGUI")

if oldGui then
    oldGui:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ServerHopGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
ScreenGui.Parent = CoreGui

local Button = Instance.new("TextButton")
Button.Name = "ServerHopButton"
Button.Size = UDim2.new(0, 90, 0, 40)
Button.Position = UDim2.new(1, -100, 0.5, -20)
Button.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
Button.BackgroundTransparency = 0.08
Button.BorderSizePixel = 0
Button.Text = "Server Hop"
Button.TextColor3 = Color3.fromRGB(255, 255, 255)
Button.TextSize = 13
Button.Font = Enum.Font.GothamBold
Button.AutoButtonColor = true
Button.Parent = ScreenGui

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 8)
Corner.Parent = Button

local Stroke = Instance.new("UIStroke")
Stroke.Thickness = 1
Stroke.Transparency = 0.5
Stroke.Parent = Button

local Status = Instance.new("TextLabel")
Status.Name = "Status"
Status.Size = UDim2.new(0, 220, 0, 24)
Status.Position = UDim2.new(1, -230, 0.5, 28)
Status.BackgroundTransparency = 1
Status.Text = ""
Status.TextColor3 = Color3.fromRGB(255, 255, 255)
Status.TextSize = 12
Status.Font = Enum.Font.Gotham
Status.TextXAlignment = Enum.TextXAlignment.Right
Status.Parent = ScreenGui

--==================================================
-- DRAG MOBILE / PC
--==================================================

local dragging = false
local dragStart
local startPos
local dragInput

Button.InputBegan:Connect(function(input)

    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragStart = input.Position
        startPos = Button.Position

        input.Changed:Connect(function()

            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end

        end)
    end
end)

Button.InputChanged:Connect(function(input)

    if input.UserInputType == Enum.UserInputType.MouseMovement
    or input.UserInputType == Enum.UserInputType.Touch then

        dragInput = input

    end

end)

UserInputService.InputChanged:Connect(function(input)

    if input == dragInput and dragging then

        local delta = input.Position - dragStart

        Button.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )

        Status.Position = UDim2.new(
            Button.Position.X.Scale,
            Button.Position.X.Offset - 130,
            Button.Position.Y.Scale,
            Button.Position.Y.Offset + 47
        )

    end

end)

--==================================================
-- GET SERVERS
--==================================================

local function GetServers()

    local servers = {}
    local cursor = ""

    for page = 1, MAX_PAGES do

        local url =
            "https://games.roblox.com/v1/games/"
            .. PlaceId
            .. "/servers/Public?sortOrder=Asc&limit=100"

        if cursor ~= "" then
            url = url .. "&cursor=" .. HttpService:UrlEncode(cursor)
        end

        local success, result = pcall(function()
            return game:HttpGet(url)
        end)

        if not success then
            warn("Server API Error:", result)
            break
        end

        local data

        local decodeSuccess = pcall(function()
            data = HttpService:JSONDecode(result)
        end)

        if not decodeSuccess or not data then
            warn("Failed to decode server data")
            break
        end

        if data.data then

            for _, server in ipairs(data.data) do

                if server.id
                and server.id ~= CurrentJobId
                and server.playing
                and server.maxPlayers
                and server.playing >= MIN_PLAYERS
                and server.playing <= MAX_PLAYERS then

                    table.insert(servers, server)

                end

            end

        end

        cursor = data.nextPageCursor or ""

        if cursor == "" then
            break
        end

        -- Nghỉ nhẹ giữa các request
        task.wait(0.15)
    end

    return servers
end

--==================================================
-- CHOOSE BEST SERVER
--==================================================

local function ChooseServer(servers)

    if #servers == 0 then
        return nil
    end

    -- Tạo nhóm theo số người
    local groups = {
        [1] = {},
        [2] = {},
        [3] = {},
        [4] = {},
        [5] = {}
    }

    for _, server in ipairs(servers) do

        local players = server.playing

        if groups[players] then
            table.insert(groups[players], server)
        end

    end

    -- Ưu tiên theo:
    -- 2 người
    -- 3 người
    -- 1 người
    -- 4 người
    -- 5 người

    for _, playerCount in ipairs(PRIORITY) do

        local list = groups[playerCount]

        if #list > 0 then

            return list[
                math.random(1, #list)
            ]

        end

    end

    return nil
end

--==================================================
-- SERVER HOP
--==================================================

local hopping = false

local function ServerHop()

    if hopping then
        return
    end

    hopping = true

    Button.Active = false
    Button.AutoButtonColor = false
    Button.Text = "Searching..."

    Status.Text = "Finding 1-5 player server..."

    local servers = GetServers()

    if #servers == 0 then

        Button.Text = "Server Hop"
        Button.Active = true
        Button.AutoButtonColor = true

        Status.Text = "No server found"

        task.delay(2, function()

            if Status then
                Status.Text = ""
            end

        end)

        hopping = false
        return
    end

    Status.Text =
        "Found "
        .. tostring(#servers)
        .. " suitable servers"

    task.wait(0.2)

    local target = ChooseServer(servers)

    if not target then

        Button.Text = "Server Hop"
        Button.Active = true
        Button.AutoButtonColor = true

        Status.Text = "No suitable server"

        hopping = false
        return
    end

    Status.Text =
        "Joining: "
        .. tostring(target.playing)
        .. " players"

    Button.Text = "Teleporting..."

    task.wait(0.3)

    local success, errorMessage = pcall(function()

        TeleportService:TeleportToPlaceInstance(
            PlaceId,
            target.id,
            LocalPlayer
        )

    end)

    if not success then

        warn("Teleport failed:", errorMessage)

        Button.Text = "Server Hop"
        Button.Active = true
        Button.AutoButtonColor = true

        Status.Text = "Teleport failed"

        task.delay(2, function()

            if Status then
                Status.Text = ""
            end

        end)

        hopping = false
    end
end

--==================================================
-- BUTTON
--==================================================

Button.MouseButton1Click:Connect(function()
    ServerHop()
end)

print("Server Hop 1-5 loaded")
print("Priority: 2 > 3 > 1 > 4 > 5")
