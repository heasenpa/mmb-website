--========================================================
-- SERVER HOP GUI
--========================================================
-- 1-5 players
-- Priority: 2 > 3 > 1 > 4 > 5
--
-- Có:
-- • GUI nhỏ
-- • Kéo được trên PC / Mobile
-- • Quét nhiều trang server
-- • Bỏ server hiện tại
-- • Bỏ server vừa ở
-- • Ưu tiên 2 -> 3 -> 1 -> 4 -> 5
-- • Tự xử lý TeleportInitFailed
-- • Xử lý server đầy / Error 772
-- • Tự tìm server mới khi teleport thất bại
--========================================================


--========================================================
-- SERVICES
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
-- Mỗi trang tối đa 100 server
-- Tối đa khoảng 2000 server
local MAX_PAGES = 20

-- Khi đã tìm được số server này thì dừng quét
-- Không cần quét hết 2000 server
local TARGET_SERVERS = 30

-- Số server tối đa thử trong một lần
local MAX_ATTEMPTS = 8

-- Thứ tự ưu tiên
--
-- 2 người
-- 3 người
-- 1 người
-- 4 người
-- 5 người
--
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

        if type(data.id) == "string" then
            LastServer = data
        end

    end

end)


--========================================================
-- XÓA GUI CŨ
--========================================================

pcall(function()

    local oldGui =
        CoreGui:FindFirstChild("ServerHopGUI")

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

HopButton.Size =
    UDim2.new(0, 90, 0, 40)

HopButton.Position =
    UDim2.new(1, -105, 0.5, -20)

HopButton.BackgroundColor3 =
    Color3.fromRGB(35, 35, 35)

HopButton.BackgroundTransparency = 0.05

HopButton.BorderSizePixel = 0

HopButton.Text =
    "Server Hop"

HopButton.TextColor3 =
    Color3.fromRGB(255, 255, 255)

HopButton.TextSize = 13

HopButton.Font =
    Enum.Font.GothamBold

HopButton.ZIndex = 100

HopButton.Parent = ScreenGui


--========================================================
-- BUTTON CORNER
--========================================================

local ButtonCorner =
    Instance.new("UICorner")

ButtonCorner.CornerRadius =
    UDim.new(0, 8)

ButtonCorner.Parent =
    HopButton


--========================================================
-- BUTTON STROKE
--========================================================

local ButtonStroke =
    Instance.new("UIStroke")

ButtonStroke.Thickness = 2

ButtonStroke.Color =
    Color3.fromRGB(120, 0, 255)

ButtonStroke.Transparency = 0

ButtonStroke.Parent =
    HopButton


--========================================================
-- STATUS
--========================================================

local Status =
    Instance.new("TextLabel")

Status.Name = "Status"

Status.Size =
    UDim2.new(0, 280, 0, 25)

Status.Position =
    UDim2.new(1, -290, 0.5, 27)

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

Status.Parent =
    ScreenGui


--========================================================
-- DRAG SYSTEM
--========================================================

local dragging = false
local dragStart = nil
local startPos = nil


HopButton.InputBegan:Connect(function(input)

    if input.UserInputType ==
        Enum.UserInputType.MouseButton1
        or input.UserInputType ==
        Enum.UserInputType.Touch then

        dragging = true

        dragStart =
            input.Position

        startPos =
            HopButton.Position

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


        HopButton.Position =
            UDim2.new(

                startPos.X.Scale,

                startPos.X.Offset
                + delta.X,

                startPos.Y.Scale,

                startPos.Y.Offset
                + delta.Y

            )


        Status.Position =
            UDim2.new(

                HopButton.Position.X.Scale,

                HopButton.Position.X.Offset
                - 190,

                HopButton.Position.Y.Scale,

                HopButton.Position.Y.Offset
                + 47

            )

    end

end)


--========================================================
-- GET SERVERS
--========================================================

local function GetServers()

    local suitableServers = {}

    local totalServers = 0

    local pagesChecked = 0

    local cursor = ""


    --====================================================
    -- QUÉT SERVER
    --====================================================

    for page = 1, MAX_PAGES do

        pagesChecked = page


        Status.Text =
            "Scanning "
            .. tostring(page)
            .. "/"
            .. tostring(MAX_PAGES)


        --================================================
        -- API URL
        --================================================

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
                "[ServerHop] HTTP request failed:",
                response
            )

            break

        end


        --================================================
        -- JSON DECODE
        --================================================

        local decodeSuccess, data =
            pcall(function()

                return HttpService:JSONDecode(
                    response
                )

            end)


        if not decodeSuccess
            or type(data) ~= "table" then

            warn(
                "[ServerHop] JSON decode failed"
            )

            break

        end


        --================================================
        -- SERVER DATA
        --================================================

        if type(data.data) == "table" then

            for _, server in ipairs(data.data) do

                totalServers += 1


                local serverId =
                    server.id

                local playerCount =
                    tonumber(server.playing)

                local maxPlayers =
                    tonumber(server.maxPlayers)


                --========================================
                -- SERVER ID HỢP LỆ
                --========================================

                if serverId
                    and type(serverId) == "string" then


                    --====================================
                    -- BỎ SERVER HIỆN TẠI
                    --====================================

                    if serverId ~= game.JobId then


                        --================================
                        -- BỎ SERVER TRƯỚC
                        --================================

                        if serverId ~= LastServer.id then


                            --================================
                            -- PLAYER COUNT
                            --================================

                            if playerCount then


                                --================================
                                -- 1-5 PLAYERS
                                --================================

                                if playerCount >= MIN_PLAYERS
                                    and playerCount <= MAX_PLAYERS then


                                    --================================
                                    -- SERVER CHƯA ĐẦY
                                    --================================

                                    if not maxPlayers
                                        or playerCount < maxPlayers then


                                        table.insert(
                                            suitableServers,
                                            server
                                        )

                                    end

                                end

                            end

                        end

                    end

                end

            end

        end


        --================================================
        -- ĐỦ SERVER
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


        --================================================
        -- DELAY NHẸ
        --================================================

        task.wait(0.12)

    end


    return
        suitableServers,
        totalServers,
        pagesChecked

end


--========================================================
-- SORT SERVER
--========================================================

local function SortServers(servers)

    local sorted = {}


    --====================================================
    -- PRIORITY
    --====================================================

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
-- STATE
--========================================================

local hopping = false

local failedServers = {}


--========================================================
-- TRY TELEPORT
--========================================================

local function TryTeleport(target)

    if not target then
        return false
    end


    if not target.id then
        return false
    end


    --====================================================
    -- SERVER ĐÃ FAIL TRƯỚC ĐÓ
    --====================================================

    if failedServers[target.id] then
        return false
    end


    failedServers[target.id] = true


    --====================================================
    -- TELEPORT
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
            "[ServerHop] Teleport call failed:",
            errorMessage
        )

        return false

    end


    return true

end


--========================================================
-- TELEPORT FAILED EVENT
--========================================================
-- Đây là phần quan trọng để bắt lỗi 772.
--
-- Khi Roblox báo:
--
-- "Dịch Chuyển Thất Bại"
-- "Máy chủ đã đầy"
-- Error 772
--
-- script sẽ tự tìm server mới.
--========================================================

TeleportService.TeleportInitFailed:Connect(

    function(
        player,
        teleportResult,
        errorMessage,
        placeId,
        teleportOptions
    )


        --================================================
        -- CHỈ ACC HIỆN TẠI
        --================================================

        if player ~= LocalPlayer then
            return
        end


        if not hopping then
            return
        end


        warn(
            "[ServerHop] Teleport failed:",
            tostring(teleportResult),
            tostring(errorMessage)
        )


        --================================================
        -- STATUS
        --================================================

        Status.Text =
            "Teleport failed - retrying..."

        HopButton.Text =
            "Retry..."


        --================================================
        -- ĐỢI ROBLOX XỬ LÝ
        --================================================

        task.wait(0.5)


        --================================================
        -- QUÉT SERVER MỚI
        --================================================

        Status.Text =
            "Finding another server..."


        local servers,
            totalServers,
            pagesChecked =
            GetServers()


        --================================================
        -- KHÔNG CÓ SERVER
        --================================================

        if #servers == 0 then

            HopButton.Text =
                "Server Hop"

            HopButton.Active = true

            Status.Text =
                "No other server"

            hopping = false


            task.delay(3, function()

                if Status then
                    Status.Text = ""
                end

            end)


            return

        end


        --================================================
        -- SORT
        --================================================

        local sorted =
            SortServers(servers)


        local attempts =
            math.min(
                #sorted,
                MAX_ATTEMPTS
            )


        --================================================
        -- THỬ SERVER MỚI
        --================================================

        for i = 1, attempts do

            local target =
                sorted[i]


            if target
                and not failedServers[target.id] then


                local playerCount =
                    tonumber(
                        target.playing
                    ) or 0


                Status.Text =
                    "Retry "
                    .. tostring(i)
                    .. "/"
                    .. tostring(attempts)
                    .. " | "
                    .. tostring(playerCount)
                    .. " players"


                HopButton.Text =
                    "Teleport..."


                print(
                    "[ServerHop] Retry:",
                    i,
                    "Server:",
                    target.id,
                    "Players:",
                    playerCount
                )


                local result =
                    TryTeleport(target)


                if result then

                    Status.Text =
                        "Teleporting..."

                    return

                end


                task.wait(0.3)

            end

        end


        --================================================
        -- RETRY THẤT BẠI
        --================================================

        HopButton.Text =
            "Server Hop"

        HopButton.Active = true

        Status.Text =
            "All retries failed"

        hopping = false


        task.delay(4, function()

            if Status then
                Status.Text = ""
            end

        end)

    end

)


--========================================================
-- SERVER HOP
--========================================================

local function ServerHop()

    --====================================================
    -- ĐANG HOP
    --====================================================

    if hopping then
        return
    end


    hopping = true


    --====================================================
    -- RESET FAILED SERVER
    --====================================================

    failedServers = {}


    --====================================================
    -- GUI
    --====================================================

    HopButton.Active = false

    HopButton.Text =
        "Searching..."

    Status.Text =
        "Finding servers..."


    --====================================================
    -- GET SERVERS
    --====================================================

    local servers,
        totalServers,
        pagesChecked =
        GetServers()


    --====================================================
    -- DEBUG
    --====================================================

    print("================================")
    print("[ServerHop]")
    print("API servers:", totalServers)
    print("Suitable 1-5:", #servers)
    print("Pages checked:", pagesChecked)
    print("Current JobId:", game.JobId)
    print("Last JobId:", LastServer.id or "None")
    print("================================")


    --====================================================
    -- KHÔNG CÓ SERVER
    --====================================================

    if #servers == 0 then

        HopButton.Text =
            "Server Hop"

        HopButton.Active = true


        if totalServers == 0 then

            Status.Text =
                "API returned 0"

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
        .. " servers"


    task.wait(0.2)


    --====================================================
    -- TRY SERVER
    --====================================================

    for i = 1, attempts do

        local target =
            sorted[i]


        if target then

            local playerCount =
                tonumber(
                    target.playing
                ) or 0


            Status.Text =
                "Try "
                .. tostring(i)
                .. "/"
                .. tostring(attempts)
                .. " | "
                .. tostring(playerCount)
                .. " players"


            HopButton.Text =
                "Teleport..."


            print(
                "[ServerHop] Trying:",
                target.id,
                "Players:",
                playerCount
            )


            local result =
                TryTeleport(target)


            if result then

                Status.Text =
                    "Teleporting..."


                --================================================
                -- QUAN TRỌNG
                --
                -- Không đặt hopping = false ở đây.
                --
                -- Nếu teleport thành công:
                -- script sẽ chuyển sang server mới.
                --
                -- Nếu teleport thất bại:
                -- TeleportInitFailed sẽ được gọi.
                --================================================

                return

            end


            task.wait(0.25)

        end

    end


    --====================================================
    -- TẤT CẢ SERVER THẤT BẠI NGAY TỪ ĐẦU
    --====================================================

    HopButton.Text =
        "Server Hop"

    HopButton.Active = true

    Status.Text =
        "Could not teleport"

    hopping = false


    task.delay(4, function()

        if Status then
            Status.Text = ""
        end

    end)

end


--========================================================
-- BUTTON CLICK
--========================================================

HopButton.MouseButton1Click:Connect(

    function()

        ServerHop()

    end

)


--========================================================
-- READY
--========================================================

print("================================")
print(" SERVER HOP GUI READY")
print(" Players: 1-5")
print(" Priority: 2 > 3 > 1 > 4 > 5")
print(" Max pages:", MAX_PAGES)
print(" Target servers:", TARGET_SERVERS)
print(" Max attempts:", MAX_ATTEMPTS)
print(" TeleportInitFailed: ENABLED")
print("================================")


Status.Text =
    "Ready"


task.delay(2, function()

    if Status then
        Status.Text = ""
    end

end)
