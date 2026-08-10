--========================================================
-- SERVER HOP GUI - DEBUG / SMART VERSION
--========================================================
-- Server 1-5 người
-- Ưu tiên: 2 > 3 > 1 > 4 > 5
-- Quét rộng nhưng dừng sớm khi đủ server
-- Có DEBUG trên GUI
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

-- Tối đa 20 trang
-- 20 x 100 = khoảng 2000 server
local MAX_PAGES = 20

-- Khi tìm được số này server phù hợp thì dừng quét
-- Không cần quét đủ 2000 nếu đã có nhiều server tốt
local TARGET_SERVERS = 30

-- Số server tối đa sẽ thử teleport
local MAX_ATTEMPTS = 10

-- Ưu tiên
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

    local data = TeleportService:GetLocalPlayerTeleportData()

    if type(data) == "table" then

        if type(data.id) == "string" then
            LastServer = data
        end

    end

end)

--========================================================
-- XÓA GUI CŨ
--========================================================

pcall(function()

    local oldGui = CoreGui:FindFirstChild("ServerHopGUI")

    if oldGui then
        oldGui:Destroy()
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
    280,
    0,
    25
)

Status.Position = UDim2.new(
    1,
    -290,
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
            HopButton.Position.X.Offset - 190,

            HopButton.Position.Y.Scale,
            HopButton.Position.Y.Offset + 47
        )

    end

end)

--========================================================
-- GET SERVERS
--========================================================

local function GetServers()

    local suitableServers = {}

    local totalServers = 0
    local lowServers = 0
    local pagesChecked = 0

    local cursor = ""

    for page = 1, MAX_PAGES do

        pagesChecked = page

        Status.Text =
            "Scanning page "
            .. tostring(page)
            .. "/"
            .. tostring(MAX_PAGES)

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

        --================================================
        -- REQUEST
        --================================================

        local requestSuccess, response =
            pcall(function()

                return game:HttpGet(url)

            end)

        if not requestSuccess then

            warn(
                "[ServerHop] HTTP error:",
                response
            )

            break

        end

        --================================================
        -- JSON
        --================================================

        local decodeSuccess, data =
            pcall(function()

                return HttpService:JSONDecode(response)

            end)

        if not decodeSuccess or type(data) ~= "table" then

            warn(
                "[ServerHop] JSON decode error"
            )

            break

        end

        --================================================
        -- SERVER DATA
        --================================================

        if type(data.data) == "table" then

            for _, server in ipairs(data.data) do

                totalServers += 1

                local playerCount =
                    tonumber(server.playing)

                local maxPlayers =
                    tonumber(server.maxPlayers)

                local serverId =
                    server.id

                --========================================
                -- KIỂM TRA SERVER
                --========================================

                if serverId
                and type(serverId) == "string"
                and serverId ~= game.JobId then

                    -- Không quay lại server trước
                    if serverId ~= LastServer.id then

                        if playerCount then

                            if playerCount >= MIN_PLAYERS
                            and playerCount <= MAX_PLAYERS then

                                -- MaxPlayers cũng phải hợp lệ
                                if not maxPlayers
                                or playerCount < maxPlayers then

                                    table.insert(
                                        suitableServers,
                                        server
                                    )

                                    lowServers += 1

                                end

                            end

                        end

                    end

                end

            end

        end

        --================================================
        -- ĐỦ SERVER THÌ DỪNG
        --================================================

        if #suitableServers >= TARGET_SERVERS then
            break
        end

        --================================================
        -- NEXT PAGE
        --================================================

        cursor =
            data.nextPageCursor or ""

        if cursor == "" then
            break
        end

        -- Nghỉ nhỏ giữa request
        task.wait(0.12)

    end

    return suitableServers, totalServers, pagesChecked

end

--========================================================
-- SORT THEO PRIORITY
--========================================================

local function SortServers(servers)

    local sorted = {}

    -- 2 > 3 > 1 > 4 > 5

    for _, wantedPlayers in ipairs(PRIORITY) do

        for _, server in ipairs(servers) do

            if tonumber(server.playing)
                == wantedPlayers then

                table.insert(
                    sorted,
                    server
                )

            end

        end

    end

    return sorted

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
        "Searching servers..."

    --====================================================
    -- GET SERVERS
    --====================================================

    local servers, totalServers, pagesChecked =
        GetServers()

    --====================================================
    -- DEBUG
    --====================================================

    print("==============================")
    print("[ServerHop]")
    print("API servers:", totalServers)
    print("Suitable 1-5:", #servers)
    print("Pages checked:", pagesChecked)
    print("Current JobId:", game.JobId)
    print("Last JobId:", LastServer.id or "None")
    print("==============================")

    --====================================================
    -- KHÔNG CÓ SERVER
    --====================================================

    if #servers == 0 then

        HopButton.Text =
            "Server Hop"

        HopButton.Active = true

        if totalServers == 0 then

            Status.Text =
                "API returned 0 servers"

        else

            Status.Text =
                "API:"
                .. tostring(totalServers)
                .. " | 1-5: 0"

        end

        hopping = false

        task.delay(4, function()

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

    local attempts =
        math.min(
            #sorted,
            MAX_ATTEMPTS
        )

    Status.Text =
        "Found "
        .. tostring(#sorted)
        .. " | Trying..."

    task.wait(0.2)

    --====================================================
    -- TRY SERVER
    --====================================================

    for i = 1, attempts do

        local target =
            sorted[i]

        if not target then
            break
        end

        local count =
            tonumber(target.playing) or 0

        Status.Text =
            "Try "
            .. tostring(i)
            .. "/"
            .. tostring(attempts)
            .. " | "
            .. tostring(count)
            .. " players"

        HopButton.Text =
            "Teleport..."

        print(
            "[ServerHop] Trying server:",
            target.id,
            "Players:",
            count
        )

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

        if success then

            print(
                "[ServerHop] Teleport requested"
            )

            Status.Text =
                "Teleporting..."

            return

        end

        warn(
            "[ServerHop] Attempt "
            .. tostring(i)
            .. " failed:",
            errorMessage
        )

        task.wait(0.25)

    end

    --====================================================
    -- ALL FAILED
    --====================================================

    HopButton.Text =
        "Server Hop"

    HopButton.Active = true

    Status.Text =
        "All teleport attempts failed"

    hopping = false

    task.delay(4, function()

        if Status then
            Status.Text = ""
        end

    end)

end

--========================================================
-- CLICK
--========================================================

HopButton.MouseButton1Click:Connect(function()

    ServerHop()

end)

--========================================================
-- READY
--========================================================

print("================================")
print(" SERVER HOP GUI READY")
print(" Range: 1-5 players")
print(" Priority: 2 > 3 > 1 > 4 > 5")
print(" Max pages:", MAX_PAGES)
print(" Target servers:", TARGET_SERVERS)
print(" Max attempts:", MAX_ATTEMPTS)
print("================================")

Status.Text =
    "Ready"

task.delay(2, function()

    if Status then
        Status.Text = ""
    end

end)
