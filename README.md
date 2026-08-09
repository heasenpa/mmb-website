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

        text = text:gsub(",", "")
        text = text:gsub("%s+", "")

        return tonumber(text)

    end)

    if success and result ~= nil then
        return result
    end

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
-- TẠO GUI NHỎ
-- ========================================

local gui = Instance.new("ScreenGui")
gui.Name = "RobloxMonitor"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.Parent = CoreGui

local frame = Instance.new("Frame")

-- Ô nhỏ
frame.Size = UDim2.new(0, 65, 0, 30)

-- Góc phải + chính giữa màn hình
frame.Position = UDim2.new(1, -70, 0.5, -15)

frame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
frame.BackgroundTransparency = 0.15
frame.BorderSizePixel = 0
frame.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 6)
corner.Parent = frame

-- ========================================
-- COUNTDOWN TEXT
-- ========================================

local countdown = Instance.new("TextLabel")

countdown.Size = UDim2.new(1, 0, 1, 0)
countdown.Position = UDim2.new(0, 0, 0, 0)

countdown.BackgroundTransparency = 1

countdown.Text = "05:00"

countdown.TextColor3 = Color3.fromRGB(255, 255, 255)
countdown.TextSize = 14
countdown.Font = Enum.Font.GothamBold

countdown.TextXAlignment = Enum.TextXAlignment.Center
countdown.TextYAlignment = Enum.TextYAlignment.Center

countdown.Parent = frame

-- ========================================
-- FORMAT THỜI GIAN
-- ========================================

local function formatTime(seconds)

    seconds = math.max(0, math.floor(seconds))

    local minutes = math.floor(seconds / 60)
    local secs = seconds % 60

    return string.format("%02d:%02d", minutes, secs)
end

-- ========================================
-- GỬI DỮ LIỆU
-- ========================================

local function sendData()

    local coins = getCoins()
    local gameName = getGameName()

    local data = {
        username = LocalPlayer.Name,
        displayName = LocalPlayer.DisplayName,
        userId = LocalPlayer.UserId,

        map = gameName,

        jobId = game.JobId,
        placeId = game.PlaceId
    }

    -- Chỉ gửi Coins nếu game có Coins
    if coins ~= nil then
        data.coins = coins
    end

    -- ========================================
    -- HTTP REQUEST
    -- ========================================

    local success, response = pcall(function()

        return httpRequest({

            Url = API_URL,

            Method = "POST",

            Headers = {
                ["Content-Type"] = "application/json"
            },

            Body = HttpService:JSONEncode(data)

        })

    end)

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

    -- Bắt đầu từ 5 phút
    local nextSend = os.time() + HEARTBEAT

    while true do

        -- Tính số giây còn lại
        local remaining = nextSend - os.time()

        -- ====================================
        -- HẾT 5 PHÚT
        -- ====================================

        if remaining <= 0 then

            -- Gửi dữ liệu lên Worker
            sendData()

            -- Reset lại 5 phút
            nextSend = os.time() + HEARTBEAT

            remaining = HEARTBEAT
        end

        -- ====================================
        -- CẬP NHẬT GUI
        -- ====================================

        countdown.Text = formatTime(remaining)

        -- Đợi đúng khoảng 1 giây
        task.wait(1)

    end

end)
