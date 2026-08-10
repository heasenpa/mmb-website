--========================================================
-- SERVER HOP GUI - RETRY VERSION
--========================================================
-- 1-5 players
-- Priority: 2 > 3 > 1 > 4 > 5
-- Tìm nhiều server và thử lần lượt
-- Bỏ server hiện tại + server trước đó
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

-- Tối đa quét 10 trang
-- ~1000 server
local MAX_PAGES = 10

-- Tối đa thử 20 server
local MAX_ATTEMPTS = 20

-- Thứ tự ưu tiên
local PRIORITY = {
    2,
    3,
    1,
    4,
    5
}

--========================================================
-- LAST SERVER
--========================================================

local LastServer = {}

pcall(function()

    local data =
        TeleportService:GetLocalPlayerTeleportData()

    if type(data) == "table" then
        LastServer = data
    end

end)

--========================================================
-- REMOVE OLD GUI
--========================================================

pcall(function()

    local old =
        CoreGui:FindFirstChild("ServerHopGUI")

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

HopButton.Size =
    UDim2.new(0, 90, 0, 40)

HopButton.Position =
    UDim2.new(1, -105, 0.5, -20)

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

local ButtonCorner =
    Instance.new("UICorner")

ButtonCorner.CornerRadius =
    UDim.new(0, 8)

ButtonCorner.Parent = HopButton

local ButtonStroke =
    Instance.new("UIStroke")

ButtonStroke.Thickness = 2

ButtonStroke.Color =
    Color3.fromRGB(120, 0, 255)

ButtonStroke.Parent = HopButton

--========================================================
-- STATUS
--========================================================

local Status = Instance.new("TextLabel")

Status.Name = "Status"

Status.Size =
    UDim2.new(0, 250, 0, 25)

Status.Position =
    UDim2.new(1, -260, 0.5, 27)

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

    HopButton.Position =
        UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,

            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )

    Status.Position =
        UDim2.new(
            HopButton.Position.X.Scale,
            HopButton.Position.X.Offset - 160,

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

            url =
                url
                .. "&cursor="
                .. HttpService:UrlEncode(cursor)

        end

        local success, response =
            pcall(function()

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

        task.wait(0.12)

    end

    return servers

end

--========================================================
-- SORT SERVER
--========================================================

local function SortServers(servers)

    local result = {}

    for _, wantedPlayers in ipairs(PRIORITY) do

        for _, server in ipairs(servers) do

            if server.playing ==
                wantedPlayers then

                table.insert(
                    result,
                    server
                )

            end

        end

    end

    return result

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
        "Finding servers..."

    --====================================================
    -- GET SERVERS
    --====================================================

    local servers =
        GetServers()

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

    --====================================================
    -- SORT
    --====================================================

    local sorted =
        SortServers(servers)

    --====================================================
    -- GIỚI HẠN ATTEMPTS
    --====================================================

    local attempts =
        math.min(
            #sorted,
            MAX_ATTEMPTS
        )

    Status.Text =
        "Found "
        .. tostring(#sorted)
        .. " servers"

    task.wait(0.3)

    --====================================================
    -- TRY SERVER
    --====================================================

    for i = 1, attempts do

        local target =
            sorted[i]

        if not target then
            break
        end

        Status.Text =
            "Trying "
            .. tostring(i)
            .. "/"
            .. tostring(attempts)
            .. " - "
            .. tostring(target.playing)
            .. " players"

        HopButton.Text =
            "Teleport..."

        local success, errorMessage =
            pcall(function()

                TeleportService:
                    TeleportToPlaceInstance(
                        game.PlaceId,
                        target.id,
                        LocalPlayer,
                        {
                            id = game.JobId
                        }
                    )

            end)

        --================================================
        -- TELEPORT CALL ACCEPTED
        --================================================

        if success then

            -- Teleport đang được xử lý
            -- Không tiếp tục spam server khác
            return

        end

        warn(
            "[ServerHop] Attempt "
            .. tostring(i)
            .. " failed:",
            errorMessage
        )

        task.wait(0.3)

    end

    --====================================================
    -- TẤT CẢ THẤT BẠI
    --====================================================

    HopButton.Text =
        "Server Hop"

    HopButton.Active = true

    Status.Text =
        "All attempts failed"

    hopping = false

    task.delay(2, function()

        if Status then
            Status.Text = ""
        end

    end)

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

print("================================")
print(" Server Hop GUI loaded")
print(" Players: 1-5")
print(" Priority: 2 > 3 > 1 > 4 > 5")
print(" Max pages:", MAX_PAGES)
print(" Max attempts:", MAX_ATTEMPTS)
print("================================")

Status.Text =
    "Ready"

task.delay(2, function()

    if Status then
        Status.Text = ""
    end

end)
