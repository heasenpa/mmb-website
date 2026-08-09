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
-- TẠO GUI
-- ========================================

local gui = Instance.new("ScreenGui")
gui.Name = "RobloxMonitor"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.DisplayOrder = 999999
gui.Parent = CoreGui

-- ========================================
-- NÚT O NHỎ
-- ========================================

local openButton = Instance.new("TextButton")
openButton.Name = "OpenButton"
openButton.Size = UDim2.new(0, 45, 0, 45)

-- Ở giữa màn hình
openButton.Position = UDim2.new(0.5, -22, 0.5, -22)

openButton.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
openButton.BorderSizePixel = 0
openButton.Text = "O"
openButton.TextColor3 = Color3.fromRGB(255, 255, 255)
openButton.TextSize = 20
openButton.Font = Enum.Font.GothamBold
openButton.Parent = gui

local openCorner = Instance.new("UICorner")
openCorner.CornerRadius = UDim.new(0, 10)
openCorner.Parent = openButton

-- Viền
local openStroke = Instance.new("UIStroke")
openStroke.Color = Color3.fromRGB(80, 80, 80)
openStroke.Thickness = 1
openStroke.Parent = openButton

-- ========================================
-- GUI LỚN
-- ========================================

local frame = Instance.new("Frame")
frame.Name = "MonitorFrame"
frame.Size = UDim2.new(0, 250, 0, 100)
frame.Position = UDim2.new(1, -260, 0, 80)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
frame.BorderSizePixel = 0
frame.Visible = false
frame.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = frame

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(60, 60, 60)
stroke.Thickness = 1
stroke.Parent = frame

-- ========================================
-- TITLE
-- ========================================

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, -55, 0, 30)
title.Position = UDim2.new(0, 10, 0, 5)
title.BackgroundTransparency = 1
title.Text = "Roblox Monitor"
title.TextColor3 = Color3.new(1, 1, 1)
title.TextSize = 17
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = frame

-- ========================================
-- NÚT X
-- ========================================

local closeButton = Instance.new("TextButton")
closeButton.Name = "CloseButton"
closeButton.Size = UDim2.new(0, 30, 0, 30)
closeButton.Position = UDim2.new(1, -35, 0, 5)
closeButton.BackgroundTransparency = 1
closeButton.Text = "X"
closeButton.TextColor3 = Color3.fromRGB(220, 220, 220)
closeButton.TextSize = 16
closeButton.Font = Enum.Font.GothamBold
closeButton.Parent = frame

-- ========================================
-- ONLINE
-- ========================================

local online = I
