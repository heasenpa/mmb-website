--========================================================
-- LIGHT SERVER HOP
-- Priority: 2 players > 3 players > Any available
--========================================================

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local Http = game:GetService("HttpService")
local TS = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")

local LP = Players.LocalPlayer
local PLACE = game.PlaceId
local JOB = game.JobId

--========================================================
-- CONFIG
--========================================================

local MAX_PAGES = 10
local TARGET_SERVERS = 30
local RETRY_DELAY = 1

--========================================================
-- LAST SERVER
--========================================================

local lastID

pcall(function()
    local data = TS:GetLocalPlayerTeleportData()

    if type(data) == "table" then
        lastID = data.id
    end
end)

--========================================================
-- REMOVE OLD GUI
--========================================================

pcall(function()
    local old = CoreGui:FindFirstChild("LightServerHop")

    if old then
        old:Destroy()
    end
end)

--========================================================
-- GUI
--========================================================

local gui = Instance.new("ScreenGui")
gui.Name = "LightServerHop"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Global
gui.DisplayOrder = 999999
gui.Parent = CoreGui

--========================================================
-- BUTTON
--========================================================

local btn = Instance.new("TextButton")

btn.Size = UDim2.new(0,90,0,38)
btn.Position = UDim2.new(1,-105,.5,-20)

btn.BackgroundColor3 =
    Color3.fromRGB(35,35,35)

btn.TextColor3 =
    Color3.fromRGB(255,255,255)

btn.Text = "Server Hop"
btn.TextSize = 13
btn.Font = Enum.Font.GothamBold

btn.AutoButtonColor = true

btn.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0,8)
corner.Parent = btn

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(130,0,255)
stroke.Thickness = 2
stroke.Parent = btn

--========================================================
-- STATUS
--========================================================

local status = Instance.new("TextLabel")

status.Size = UDim2.new(0,230,0,22)

status.Position =
    UDim2.new(
        1,-240,
        .5,23
    )

status.BackgroundTransparency = 1

status.Text = ""

status.TextColor3 =
    Color3.fromRGB(255,255,255)

status.TextSize = 11

status.Font = Enum.Font.Gotham

status.TextXAlignment =
    Enum.TextXAlignment.Right

status.Parent = gui

--========================================================
-- DRAG
--========================================================

local dragging = false
local dragStart
local startPos

btn.InputBegan:Connect(function(input)

    if input.UserInputType ==
        Enum.UserInputType.MouseButton1
        or input.UserInputType ==
        Enum.UserInputType.Touch then

        dragging = true

        dragStart =
            input.Position

        startPos =
            btn.Position

        input.Changed:Connect(function()

            if input.UserInputState ==
                Enum.UserInputState.End then

                dragging = false

            end

        end)

    end

end)

UIS.InputChanged:Connect(function(input)

    if not dragging then
        return
    end

    if input.UserInputType ==
        Enum.UserInputType.MouseMovement
        or input.UserInputType ==
        Enum.UserInputType.Touch then

        local delta =
            input.Position - dragStart

        btn.Position =
            UDim2.new(
                startPos.X.Scale,
                startPos.X.Offset + delta.X,
                startPos.Y.Scale,
                startPos.Y.Offset + delta.Y
            )

        status.Position =
            UDim2.new(
                btn.Position.X.Scale,
                btn.Position.X.Offset - 140,
                btn.Position.Y.Scale,
                btn.Position.Y.Offset + 43
            )

    end

end)

--========================================================
-- GET SERVERS
--========================================================

local function getServers()

    local list = {}
    local cursor = ""

    for page = 1, MAX_PAGES do

        local url =
            "https://games.roblox.com/v1/games/"
            .. PLACE
            .. "/servers/Public?sortOrder=Asc&limit=100"

        if cursor ~= "" then

            url =
                url
                .. "&cursor="
                .. Http:UrlEncode(cursor)

        end

        local ok, response =
            pcall(
                game.HttpGet,
                game,
                url
            )

        if not ok then
            break
        end

        local decoded, data =
            pcall(
                Http.JSONDecode,
                Http,
                response
            )

        if not decoded
            or type(data) ~= "table" then

            break
        end

        for _, server in ipairs(data.data or {}) do

            local playing =
                tonumber(server.playing)

            local maxPlayers =
                tonumber(server.maxPlayers)

            if server.id
                and server.id ~= JOB
                and server.id ~= lastID
                and playing
                and maxPlayers
                and playing < maxPlayers then

                table.insert(
                    list,
                    server
                )

            end

        end

        -- Có đủ server để lựa chọn
        if #list >= TARGET_SERVERS then
            break
        end

        cursor =
            data.nextPageCursor or ""

        if cursor == "" then
            break
        end

        task.wait(0.1)

    end

    return list

end

--========================================================
-- CHOOSE SERVER
--
-- 2 người
--   ↓
-- 3 người
--   ↓
-- bất kỳ server còn chỗ
--========================================================

local function chooseServer(list)

    local three = {}
    local fallback = {}

    for _, server in ipairs(list) do

        local players =
            tonumber(server.playing) or 0

        if players == 2 then

            return server

        elseif players == 3 then

            table.insert(
                three,
                server
            )

        else

            table.insert(
                fallback,
                server
            )

        end

    end

    -- Không có 2 người -> lấy 3 người

    if #three > 0 then

        return three[
            math.random(#three)
        ]

    end

    -- Không có 2/3 -> server bất kỳ

    if #fallback > 0 then

        return fallback[
            math.random(#fallback)
        ]

    end

    return nil

end

--========================================================
-- STATE
--========================================================

local running = false
local failed = {}

--========================================================
-- TELEPORT
--========================================================

local function teleport(server)

    if not server
        or not server.id then

        return false

    end

    if failed[server.id] then
        return false
    end

    -- Đánh dấu server đã thử
    failed[server.id] = true

    local ok =
        pcall(function()

            TS:TeleportToPlaceInstance(

                PLACE,

                server.id,

                LP,

                {
                    id = JOB
                }

            )

        end)

    return ok

end

--========================================================
-- MAIN SERVER HOP
--========================================================

local function serverHop()

    if running then
        return
    end

    running = true
    failed = {}

    btn.Active = false

    while running do

        btn.Text = "Searching"
        status.Text = "Finding server..."

        --================================================
        -- SCAN
        --================================================

        local servers =
            getServers()

        --================================================
        -- NO SERVER
        --================================================

        if #servers == 0 then

            status.Text =
                "No server - retrying"

            task.wait(
                RETRY_DELAY
            )

        else

            --================================================
            -- CHOOSE
            --================================================

            local target =
                chooseServer(servers)

            if target then

                local count =
                    tonumber(
                        target.playing
                    ) or 0

                status.Text =
                    "Joining "
                    .. count
                    .. " players"

                btn.Text =
                    "Teleport"

                local ok =
                    teleport(target)

                if ok then

                    status.Text =
                        "Teleporting..."

                    -- Không tắt running.
                    -- Nếu fail 772, event bên dưới
                    -- sẽ cho vòng lặp tiếp tục.

                    task.wait(2)

                else

                    task.wait(
                        RETRY_DELAY
                    )

                end

            else

                task.wait(
                    RETRY_DELAY
                )

            end

        end

    end

end

--========================================================
-- TELEPORT FAILED
--========================================================

TS.TeleportInitFailed:Connect(function(player)

    if player ~= LP then
        return
    end

    if not running then
        return
    end

    status.Text =
        "Teleport failed - retrying"

    btn.Text =
        "Retry"

    -- Không cần gọi serverHop lại.
    -- Vòng while đang chạy sẽ tự quét lại.

    task.wait(0.5)

end)

--========================================================
-- BUTTON
--========================================================

btn.MouseButton1Click:Connect(function()

    serverHop()

end)

--========================================================
-- READY
--========================================================

status.Text = "Ready"

task.delay(2,function()

    if status then
        status.Text = ""
    end

end)
