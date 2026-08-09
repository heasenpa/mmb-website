```lua
--========================================================
-- SERVER HOP GUI
-- 1-5 PLAYERS
-- Ưu tiên: 2 > 3 > 1 > 4 > 5
-- Không vào server hiện tại
-- Không quay lại server vừa ở
--========================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")

local LocalPlayer = Players.LocalPlayer

--========================================================
-- CONFIG
--========================================================

local MIN_PLAYERS = 1
local MAX_PLAYERS = 5

-- Số trang tối đa:
-- 10 trang x tối đa 100 server = khoảng 1000 server
local MAX_PAGES = 10

-- Thứ tự ưu tiên server
local PRIORITY = {
    2,
    3,
    1,
    4,
    5
}

--========================================================
-- SERVER TRƯỚC ĐÓ
-- Lấy từ TeleportData giống cơ chế script farm của bạn
--========================================================

local LastServer = TeleportService:GetLocalPlayerTeleportData()

if type(LastServer) ~= "table" then
    LastServer = {}
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

    local PlaceId = game.PlaceId
    local JobId = game.JobId

    Status.Text = "Đang tìm server..."
    HopButton.Text = "Searching..."
    HopButton.Active = false

    local allServers = {}
    local cursor = ""

    --====================================================
    -- QUÉT SERVER
    --====================================================

    for page = 1, MAX_PAGES do

        local url =
            "https://games.roblox.com/v1/games/"
            .. PlaceId
            .. "/servers/Public?sortOrder=Asc&limit=100"

        if cursor ~= "" then
            url = url .. "&cursor=" .. HttpService:UrlEncode(cursor)
        end

        local success, result = pcall(function()

            return HttpService:JSONDecode(
                game:HttpGet(url)
            )

        end)

        if not success or not result then

            warn("Không thể lấy danh sách server:", result)

            break
        end

        if result.data then

            for _, server in ipairs(result.data) do

                local playerCount = server.playing

                if server.id
                and server.id ~= JobId
                and server.id ~= LastServer.id
                and playerCount
                and playerCount >= MIN_PLAYERS
                and playerCount <= MAX_PLAYERS then

                    table.insert(allServers, server)

                end

            end

        end

        cursor = result.nextPageCursor or ""

        if cursor == "" then
            break
        end

        -- Nghỉ nhẹ giữa các request
        task.wait(0.15)
    end

    --====================================================
    -- KHÔNG CÓ SERVER
    --====================================================

    if #allServers == 0 then

        warn("Không tìm thấy server 1-5 người.")

        Status.Text = "Không tìm thấy server"

        HopButton.Text = "Server Hop"
        HopButton.Active = true

        hopping = false

        task.delay(2, function()

            if Status then
                Status.Text = ""
            end

        end)

        return
    end

    --====================================================
    -- CHIA SERVER THEO SỐ NGƯỜI
    --====================================================

    local groups = {
        [1] = {},
        [2] = {},
        [3] = {},
        [4] = {},
        [5] = {}
    }

    for _, server in ipairs(allServers) do

        local count = server.playing

        if groups[count] then
            table.insert(groups[count], server)
        end

    end

    --====================================================
    -- CHỌN SERVER THEO PRIORITY
    --====================================================

    local target = nil

    for _, playerCount in ipairs(PRIORITY) do

        local available = groups[playerCount]

        if available and #available > 0 then

            target = available[
                math.random(1, #available)
            ]

            break
        end

    end

    --====================================================
    -- KHÔNG CHỌN ĐƯỢC
    --====================================================

    if not target then

        Status.Text = "Không có server phù hợp"

        HopButton.Text = "Server Hop"
        HopButton.Active = true

        hopping = false

        return
    end

    --====================================================
    -- TELEPORT
    --====================================================

    Status.Text =
        "Đang vào server "
        .. tostring(target.playing)
        .. " người..."

    HopButton.Text = "Teleporting..."

    task.wait(0.2)

    local success, errorMessage = pcall(function()

        TeleportService:TeleportToPlaceInstance(
            PlaceId,
            target.id,
            LocalPlayer,

            -- Giống cơ chế trong script farm
            -- Server kế tiếp sẽ biết JobId hiện tại
            {
                id = JobId
            }
        )

    end)

    if not success then

        warn("Teleport failed:", errorMessage)

        Status.Text = "Teleport thất bại"

        HopButton.Text = "Server Hop"
        HopButton.Active = true

        hopping = false

        task.delay(2, function()

            if Status then
                Status.Text = ""
            end

        end)
    end
end

--========================================================
-- GUI
--========================================================

local oldGui = nil

pcall(function()
    oldGui = game:GetService("CoreGui"):FindFirstChild("ServerHopGUI")
end)

if oldGui then
    oldGui:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ServerHopGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
ScreenGui.DisplayOrder = 999999

-- Dùng PlayerGui để tương thích mobile tốt hơn
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

--========================================================
-- NÚT SERVER HOP
--========================================================

HopButton = Instance.new("TextButton")

HopButton.Name = "ServerHopButton"
HopButton.Size = UDim2.new(0, 90, 0, 40)
HopButton.Position = UDim2.new(
    1,
    -100,
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
HopButton.Font = Enum.Font.GothamBold

HopButton.Parent = ScreenGui

local ButtonCorner = Instance.new("UICorner")

ButtonCorner.CornerRadius =
    UDim.new(0, 8)

ButtonCorner.Parent = HopButton

local ButtonStroke = Instance.new("UIStroke")

ButtonStroke.Thickness = 1.5
ButtonStroke.Color =
    Color3.fromRGB(120, 0, 255)

ButtonStroke.Transparency = 0.2

ButtonStroke.Parent = HopButton

--========================================================
-- STATUS
--========================================================

Status = Instance.new("TextLabel")

Status.Name = "Status"

Status.Size =
    UDim2.new(0, 220, 0, 24)

Status.Position =
    UDim2.new(
        1,
        -230,
        0.5,
        28
    )

Status.BackgroundTransparency = 1

Status.Text = ""

Status.TextColor3 =
    Color3.fromRGB(255, 255, 255)

Status.TextSize = 12
Status.Font = Enum.Font.Gotham

Status.TextXAlignment =
    Enum.TextXAlignment.Right

Status.Parent = ScreenGui

--========================================================
-- DRAG NÚT
--========================================================

local dragging = false
local dragStart
local startPos
local dragInput

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

HopButton.InputChanged:Connect(function(input)

    if input.UserInputType ==
        Enum.UserInputType.MouseMovement
        or input.UserInputType ==
        Enum.UserInputType.Touch then

        dragInput = input

    end

end)

UserInputService.InputChanged:Connect(function(input)

    if input == dragInput and dragging then

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
                HopButton.Position.X.Offset - 130,

                HopButton.Position.Y.Scale,
                HopButton.Position.Y.Offset + 47
            )
    end
end)

--========================================================
-- CLICK
--========================================================

HopButton.MouseButton1Click:Connect(function()

    ServerHop()

end)

print("====================================")
print(" Server Hop GUI Loaded")
print(" Search: 1-5 Players")
print(" Priority: 2 > 3 > 1 > 4 > 5")
print(" Previous server will be skipped")
print(" Max pages:", MAX_PAGES)
print("====================================")
```
