local API_URL = "https://roblox-monitor.malocsenpai.workers.dev/"
local HEARTBEAT = 300 -- 5 phút

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")
local MarketplaceService = game:GetService("MarketplaceService")

local LocalPlayer = Players.LocalPlayer

-- ========================================
-- HTTP REQUEST
-- ========================================

local httpRequest =
    request
    or http_request
    or (syn and syn.request)

if not httpRequest then
    warn("Executor không hỗ trợ HTTP request!")
    return
end

-- ========================================
-- LẤY TÊN GAME
-- ========================================

local function getGameName()

    local success, result = pcall(function()

        local info = MarketplaceService:GetProductInfo(game.PlaceId)

        return info.Name

    end)

    if success and result then
        return result
    end

    return "Unknown Game"
end

-- ========================================
-- LẤY COINS
-- ========================================

local function getCoins()

    local success, result = pcall(function()

        local playerGui = LocalPlayer:WaitForChild("PlayerGui", 3)

        local crossPlatform = playerGui:WaitForChild("CrossPlatform", 3)
        local shop = crossPlatform:WaitForChild("Shop", 3)
        local small = shop:WaitForChild("Small", 3)

        local container1 = small:WaitForChild("Container", 3)
        local title = container1:WaitForChild("Title", 3)
        local container2 = title:WaitForChild("Container", 3)

        local coins = container2:WaitForChild("Coins", 3)
        local container3 = coins:WaitForChild("Container", 3)
        local amount = container3:WaitForChild("Amount", 3)

        local text = tostring(amount.Text)

        -- Loại bỏ dấu phẩy
        text = text:gsub(",", "")

        -- Loại bỏ khoảng trắng
        text = text:gsub("%s+", "")

        return tonumber(text)

    end)

    if success and result ~= nil then
        return result
    end

    -- Game không có Coins
    return nil
end

-- ========================================
-- XÓA GUI CŨ
-- ========================================

pcall(function()

    local old = CoreGui:FindFirstChild("RobloxMonitor")

    if old then
        old:Destroy()
    end

end)

-- ========================================
-- TẠO SCREEN GUI
-- ========================================

local gui = Instance.new("ScreenGui")

gui.Name = "RobloxMonitor"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true

-- Ưu tiên hiển thị trên các GUI khác
gui.DisplayOrder = 999999

gui.Parent = CoreGui

-- ========================================
-- NÚT O NHỎ
-- ========================================

local openButton = Instance.new("TextButton")

openButton.Name = "OpenButton"

openButton.Size = UDim2.new(0, 45, 0, 45)

-- ========================================
-- GIỮA CẠNH PHẢI MÀN HÌNH
-- ========================================

openButton.Position = UDim2.new(
    1,
    -60,
    0.5,
    -22
)

openButton.BackgroundColor3 =
    Color3.fromRGB(25, 25, 25)

openButton.BorderSizePixel = 0

openButton.Text = "O"

openButton.TextColor3 =
    Color3.fromRGB(255, 255, 255)

openButton.TextSize = 20

openButton.Font =
    Enum.Font.GothamBold

openButton.AutoButtonColor = true

openButton.Parent = gui

-- Bo góc nút O

local openCorner = Instance.new("UICorner")

openCorner.CornerRadius =
    UDim.new(0, 10)

openCorner.Parent = openButton

-- Viền nút O

local openStroke = Instance.new("UIStroke")

openStroke.Color =
    Color3.fromRGB(80, 80, 80)

openStroke.Thickness = 1

openStroke.Parent = openButton

-- ========================================
-- GUI LỚN
-- ========================================

local frame = Instance.new("Frame")

frame.Name = "MonitorFrame"

frame.Size = UDim2.new(
    0,
    250,
    0,
    100
)

frame.Position = UDim2.new(
    1,
    -260,
    0,
    80
)

frame.BackgroundColor3 =
    Color3.fromRGB(25, 25, 25)

frame.BorderSizePixel = 0

frame.Visible = false

frame.Parent = gui

-- Bo góc

local frameCorner = Instance.new("UICorner")

frameCorner.CornerRadius =
    UDim.new(0, 10)

frameCorner.Parent = frame

-- Viền

local frameStroke = Instance.new("UIStroke")

frameStroke.Color =
    Color3.fromRGB(60, 60, 60)

frameStroke.Thickness = 1

frameStroke.Parent = frame

-- ========================================
-- TITLE
-- ========================================

local title = Instance.new("TextLabel")

title.Name = "Title"

title.Size = UDim2.new(
    1,
    -55,
    0,
    30
)

title.Position = UDim2.new(
    0,
    10,
    0,
    5
)

title.BackgroundTransparency = 1

title.Text = "Roblox Monitor"

title.TextColor3 =
    Color3.fromRGB(255, 255, 255)

title.TextSize = 17

title.Font =
    Enum.Font.GothamBold

title.TextXAlignment =
    Enum.TextXAlignment.Left

title.Parent = frame

-- ========================================
-- NÚT X
-- ========================================

local closeButton = Instance.new("TextButton")

closeButton.Name = "CloseButton"

closeButton.Size = UDim2.new(
    0,
    30,
    0,
    30
)

closeButton.Position = UDim2.new(
    1,
    -35,
    0,
    5
)

closeButton.BackgroundTransparency = 1

closeButton.Text = "X"

closeButton.TextColor3 =
    Color3.fromRGB(220, 220, 220)

closeButton.TextSize = 16

closeButton.Font =
    Enum.Font.GothamBold

closeButton.Parent = frame

-- ========================================
-- ONLINE
-- ========================================

local online = Instance.new("TextLabel")

online.Name = "Online"

online.Size = UDim2.new(
    1,
    -20,
    0,
    25
)

online.Position = UDim2.new(
    0,
    10,
    0,
    35
)

online.BackgroundTransparency = 1

online.Text = "🟢 Online"

online.TextColor3 =
    Color3.fromRGB(100, 255, 100)

online.TextSize = 15

online.Font =
    Enum.Font.GothamBold

online.TextXAlignment =
    Enum.TextXAlignment.Left

online.Parent = frame

-- ========================================
-- COUNTDOWN
-- ========================================

local countdown = Instance.new("TextLabel")

countdown.Name = "Countdown"

countdown.Size = UDim2.new(
    1,
    -20,
    0,
    25
)

countdown.Position = UDim2.new(
    0,
    10,
    0,
    62
)

countdown.BackgroundTransparency = 1

countdown.Text =
    "Gửi tiếp theo: 05:00"

countdown.TextColor3 =
    Color3.fromRGB(220, 220, 220)

countdown.TextSize = 14

countdown.Font =
    Enum.Font.Gotham

countdown.TextXAlignment =
    Enum.TextXAlignment.Left

countdown.Parent = frame

-- ========================================
-- MỞ GUI
-- ========================================

openButton.MouseButton1Click:Connect(function()

    openButton.Visible = false

    frame.Visible = true

end)

-- ========================================
-- ĐÓNG GUI
-- ========================================

closeButton.MouseButton1Click:Connect(function()

    frame.Visible = false

    openButton.Visible = true

end)

-- ========================================
-- FORMAT THỜI GIAN
-- ========================================

local function formatTime(seconds)

    seconds = math.max(
        0,
        math.floor(seconds)
    )

    local minutes =
        math.floor(seconds / 60)

    local secs =
        seconds % 60

    return string.format(
        "%02d:%02d",
        minutes,
        secs
    )
end

-- ========================================
-- GỬI DỮ LIỆU
-- ========================================

local function sendData()

    local coins = getCoins()

    local gameName = getGameName()

    -- ====================================
    -- DỮ LIỆU CƠ BẢN
    -- ====================================

    local data = {

        username = LocalPlayer.Name,

        displayName =
            LocalPlayer.DisplayName,

        userId =
            LocalPlayer.UserId,

        -- Giữ "map" để Worker
        -- nhận đúng dữ liệu

        map = gameName,

        jobId =
            game.JobId,

        placeId =
            game.PlaceId
    }

    -- ====================================
    -- CHỈ GỬI COINS NẾU CÓ
    -- ====================================

    if coins ~= nil then

        data.coins = coins

    end

    -- ====================================
    -- HTTP REQUEST
    -- ====================================

    local success, response =
        pcall(function()

            return httpRequest({

                Url = API_URL,

                Method = "POST",

                Headers = {

                    ["Content-Type"] =
                        "application/json"

                },

                Body =
                    HttpService:JSONEncode(data)

            })

        end)

    -- ====================================
    -- KIỂM TRA RESPONSE
    -- ====================================

    if success and response then

        if tonumber(response.StatusCode) == 200 then

            return true

        end

    end

    return false
end

-- ========================================
-- COUNTDOWN + HEARTBEAT
-- ========================================

task.spawn(function()

    local nextSend =
        os.time() + HEARTBEAT

    while true do

        local remaining =
            nextSend - os.time()

        -- =================================
        -- ĐẾN THỜI GIAN GỬI
        -- =================================

        if remaining <= 0 then

            -- Gửi dữ liệu

            sendData()

            -- Reset 5 phút

            nextSend =
                os.time() + HEARTBEAT

            remaining =
                HEARTBEAT

        end

        -- =================================
        -- CẬP NHẬT GUI
        -- =================================

        countdown.Text =
            "Gửi tiếp theo: "
            .. formatTime(remaining)

        task.wait(1)

    end

end)
