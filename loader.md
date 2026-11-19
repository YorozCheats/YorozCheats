-- [[ YorozLua - Rivals Hub (Voidspam + Ragebot Peek Fix) ]] --

task.wait(1)

local repS = cloneref(game:GetService("ReplicatedStorage"))
local plrs = cloneref(game:GetService("Players"))
local runS = cloneref(game:GetService("RunService"))
local ws = cloneref(game:GetService("Workspace"))
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")
local lplr = plrs.LocalPlayer

-- ==================== SYSTEM KONFIGURACJI (CFG) ====================
local CFG_PATH = "yorozlua/rivals/cfgs/settings.json"

getgenv().Config = {
    Enabled = false,
    TriggerbotEnabled = false,
    UndergroundEnabled = false,
    RagebotEnabled = false,
    RagebotStatusEnabled = false,
    StatusFont = "GothamBold",
    VoidspamEnabled = false,
    TimeInVoid = 0.5,
    VoidPositionY = -14512312,
    FireRate = 0.0005,
    WeaponSlot = "Primary",
    EmoteID = 83194012641579,
    StopOnMove = false,
    FadeIn = 0.1,
    FadeOut = 0.1,
    Weight = 1,
    Speed = 1,
    AllowInvisible = true,
    RandomTeleport = true,
    MinTeleportRadius = 3,
    MaxTeleportRadius = 8,
    HeightOffsetMin = -1,
    HeightOffsetMax = 3,
    BehindDistance = 5,
    DeathDropThreshold = -5.5,
    MinBoundsY = 0,
    MaxBoundsY = 200,
    MaxDistanceFromTarget = 25,
    cameraStretch = false,
    stretchX = 100,
    stretchY = 100,
}

getgenv().ChinaHatSettings = {
    enabled = false, 
    hatColor = Color3.fromRGB(255, 105, 180), 
}

local underRef, stretchRef, voidspamRef, statusRef
local currentRageTargetName = nil

runS.RenderStepped:Connect(function()
    if getgenv().Config.cameraStretch then
        workspace.CurrentCamera.CFrame = workspace.CurrentCamera.CFrame * CFrame.new(
            0, 0, 0,
            getgenv().Config.stretchX/100, 0, 0,
            0, getgenv().Config.stretchY/100, 0,
            0, 0, 1
        )
    end
end)

local activeAnimationTrack = nil
local function playCustomEmote(animId)
    pcall(function()
        local character = lplr.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        local animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)

        if activeAnimationTrack then
            activeAnimationTrack:Stop()
            activeAnimationTrack = nil
        end

        local anim = Instance.new("Animation")
        anim.AnimationId = "rbxassetid://" .. tostring(animId)
        
        activeAnimationTrack = animator:LoadAnimation(anim)
        activeAnimationTrack.Priority = Enum.AnimationPriority.Action
        activeAnimationTrack:Play()
    end)
end

local function setCharacterInvisible(state)
    pcall(function()
        local char = lplr.Character
        if not char then return end
        for _, obj in ipairs(char:GetDescendants()) do
            if obj:IsA("BasePart") or obj:IsA("Decal") then
                if obj.Name ~= "HumanoidRootPart" and obj.Name ~= "ChinaHatPart" then
                    obj.Transparency = state and 1 or 0
                end
            end
        end
    end)
end

local function saveConfig()
    pcall(function()
        if not isfolder or not writefile then return end
        if not isfolder("yorozlua") then makefolder("yorozlua") end
        if not isfolder("yorozlua/rivals") then makefolder("yorozlua/rivals") end
        if not isfolder("yorozlua/rivals/cfgs") then makefolder("yorozlua/rivals/cfgs") end
        
        local json = HttpService:JSONEncode(getgenv().Config)
        writefile(CFG_PATH, json)
    end)
end

local util, enum, FighterController, SpectateController
pcall(function()
    util = require(repS.Modules.Utility)
    enum = require(repS.Modules.EnumLibrary)
    FighterController = require(lplr.PlayerScripts.Controllers.FighterController)
    SpectateController = require(lplr.PlayerScripts.Controllers:WaitForChild("SpectateController"))
end)

local slots = { Primary = 1, Secondary = 2, Melee = 3 }
local function getSlotNumber()
    return slots[getgenv().Config.WeaponSlot] or 1
end

-- ==================== TWORZENIE GUI ====================
local playerGui = lplr:WaitForChild("PlayerGui")
if playerGui:FindFirstChild("SmallVoidGui") then
    playerGui.SmallVoidGui:Destroy()
end
if playerGui:FindFirstChild("YorozKeyUI") then
    playerGui.YorozKeyUI:Destroy()
end
if playerGui:FindFirstChild("YorozLuaUI") then
    playerGui.YorozLuaUI:Destroy()
end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "YorozLuaUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local statusLabel = Instance.new("TextLabel")
statusLabel.Name = "RagebotStatusLabel"
statusLabel.Size = UDim2.new(0, 420, 0, 34)
statusLabel.AnchorPoint = Vector2.new(0.5, 0.5)
statusLabel.Position = UDim2.new(0.5, 0, 0.64, 0)
statusLabel.BackgroundTransparency = 1
statusLabel.TextColor3 = Color3.fromRGB(0, 220, 255)
statusLabel.TextStrokeColor3 = Color3.fromRGB(10, 10, 14)
statusLabel.TextStrokeTransparency = 0.25
statusLabel.Font = Enum.Font[getgenv().Config.StatusFont] or Enum.Font.GothamBold
statusLabel.TextSize = 18
statusLabel.Text = ""
statusLabel.Visible = false
statusLabel.ZIndex = 200
statusLabel.Parent = screenGui

local fontList = {
    "GothamBold",
    "GothamBlack",
    "Code",
    "Arcade",
    "SciFi",
    "RobotoMono",
}
local currentFontIndex = table.find(fontList, getgenv().Config.StatusFont) or 1

local function applyStatusFont(fontName)
    local chosenFont = Enum.Font[fontName] or Enum.Font.GothamBold
    getgenv().Config.StatusFont = chosenFont.Name
    statusLabel.Font = chosenFont
end

local function updateRagebotStatusHUD()
    local shouldShow = getgenv().Config.RagebotEnabled and getgenv().Config.RagebotStatusEnabled
    statusLabel.Visible = shouldShow
    if not shouldShow then return end

    if currentRageTargetName and currentRageTargetName ~= "" then
        statusLabel.Text = "Ragebot: attacking " .. currentRageTargetName
    else
        statusLabel.Text = "Ragebot: searching target"
    end
end

local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "ToggleMenuButton"
toggleBtn.Size = UDim2.new(0, 75, 0, 45)
toggleBtn.Position = UDim2.new(0.5, -37.5, 0, 15)
toggleBtn.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
toggleBtn.BorderColor3 = Color3.fromRGB(60, 60, 70)
toggleBtn.TextColor3 = Color3.fromRGB(0, 220, 255)
toggleBtn.Text = "Menu"
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 14
toggleBtn.ZIndex = 999
toggleBtn.Parent = screenGui

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 10)
toggleCorner.Parent = toggleBtn

local toggleStroke = Instance.new("UIStroke")
toggleStroke.Color = Color3.fromRGB(0, 220, 255)
toggleStroke.Transparency = 0.5
toggleStroke.Parent = toggleBtn

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 480, 0, 340)
mainFrame.Position = UDim2.new(0.5, -240, 0.5, -170)
mainFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
mainFrame.BorderSizePixel = 0
mainFrame.Visible = true
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.ZIndex = 100
mainFrame.Parent = screenGui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 10)
mainCorner.Parent = mainFrame

local mainStroke = Instance.new("UIStroke")
mainStroke.Color = Color3.fromRGB(45, 45, 55)
mainStroke.Thickness = 1.5
mainStroke.Parent = mainFrame

local topBar = Instance.new("Frame")
topBar.Size = UDim2.new(1, 0, 0, 38)
topBar.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
topBar.BorderSizePixel = 0
topBar.ZIndex = 101
topBar.Parent = mainFrame

local titleText = Instance.new("TextLabel")
titleText.Size = UDim2.new(1, -50, 1, 0)
titleText.Position = UDim2.new(0, 15, 0, 0)
titleText.BackgroundTransparency = 1
titleText.TextColor3 = Color3.fromRGB(255, 255, 255)
titleText.Text = "YorozLua Hub | Rivals"
titleText.Font = Enum.Font.GothamBold
titleText.TextSize = 14
titleText.TextXAlignment = Enum.TextXAlignment.Left
titleText.ZIndex = 102
titleText.Parent = topBar

local closeButton = Instance.new("TextButton")
closeButton.Size = UDim2.new(0, 35, 0, 35)
closeButton.Position = UDim2.new(1, -38, 0, 2)
closeButton.BackgroundTransparency = 1
closeButton.TextColor3 = Color3.fromRGB(180, 180, 180)
closeButton.Text = "✕"
closeButton.Font = Enum.Font.GothamBold
closeButton.TextSize = 14
closeButton.ZIndex = 102
closeButton.Parent = topBar

closeButton.MouseButton1Click:Connect(function() mainFrame.Visible = false end)
toggleBtn.MouseButton1Click:Connect(function() mainFrame.Visible = not mainFrame.Visible end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == Enum.KeyCode.RightShift then
        mainFrame.Visible = not mainFrame.Visible
    end
end)

local tabListContainer = Instance.new("ScrollingFrame")
tabListContainer.Size = UDim2.new(0, 130, 1, -48)
tabListContainer.Position = UDim2.new(0, 6, 0, 44)
tabListContainer.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
tabListContainer.BorderSizePixel = 0
tabListContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
tabListContainer.ScrollBarThickness = 2
tabListContainer.ZIndex = 101
tabListContainer.Parent = mainFrame

local tabListLayout = Instance.new("UIListLayout")
tabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
tabListLayout.Padding = UDim.new(0, 4)
tabListLayout.Parent = tabListContainer

local contentContainer = Instance.new("Frame")
contentContainer.Size = UDim2.new(1, -146, 1, -48)
contentContainer.Position = UDim2.new(0, 140, 0, 44)
contentContainer.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
contentContainer.BorderSizePixel = 0
contentContainer.ZIndex = 101
contentContainer.Parent = mainFrame

local tabsFolder = {}
local firstTab = true

local function addTab(name)
    local tabButton = Instance.new("TextButton")
    tabButton.Size = UDim2.new(1, -8, 0, 32)
    tabButton.BackgroundColor3 = Color3.fromRGB(20, 20, 26)
    tabButton.TextColor3 = Color3.fromRGB(170, 170, 180)
    tabButton.Text = " " .. name
    tabButton.Font = Enum.Font.GothamMedium
    tabButton.TextSize = 13
    tabButton.TextXAlignment = Enum.TextXAlignment.Left
    tabButton.ZIndex = 102
    tabButton.Parent = tabListContainer

    local tCorner = Instance.new("UICorner")
    tCorner.CornerRadius = UDim.new(0, 6)
    tCorner.Parent = tabButton

    local tabContent = Instance.new("ScrollingFrame")
    tabContent.Size = UDim2.new(1, -10, 1, -10)
    tabContent.Position = UDim2.new(0, 5, 0, 5)
    tabContent.BackgroundTransparency = 1
    tabContent.CanvasSize = UDim2.new(0, 0, 0, 1200)
    tabContent.ScrollBarThickness = 3
    tabContent.Visible = false
    tabContent.ZIndex = 102
    tabContent.Parent = contentContainer

    local cLayout = Instance.new("UIListLayout")
    cLayout.SortOrder = Enum.SortOrder.LayoutOrder
    cLayout.Padding = UDim.new(0, 8)
    cLayout.Parent = tabContent

    if firstTab then
        tabContent.Visible = true
        tabButton.TextColor3 = Color3.fromRGB(0, 220, 255)
        tabButton.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
        firstTab = false
    end

    tabButton.MouseButton1Click:Connect(function()
        for _, t in pairs(tabsFolder) do
            t.content.Visible = false
            t.button.TextColor3 = Color3.fromRGB(170, 170, 180)
            t.button.BackgroundColor3 = Color3.fromRGB(20, 20, 26)
        end
        tabContent.Visible = true
        tabButton.TextColor3 = Color3.fromRGB(0, 220, 255)
        tabButton.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    end)

    table.insert(tabsFolder, {button = tabButton, content = tabContent})
    return tabContent
end

local function addToggle(tab, text, defaultConfig, callback)
    local toggleBtn = Instance.new("TextButton")
    toggleBtn.Size = UDim2.new(1, -10, 0, 36)
    toggleBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
    toggleBtn.TextColor3 = Color3.fromRGB(220, 220, 230)
    toggleBtn.Text = ""
    toggleBtn.AutoButtonColor = false
    toggleBtn.ZIndex = 103
    toggleBtn.Parent = tab

    local tCorner = Instance.new("UICorner")
    tCorner.CornerRadius = UDim.new(0, 6)
    tCorner.Parent = toggleBtn

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -50, 1, 0)
    label.Position = UDim2.new(0, 12, 0, 0)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.fromRGB(220, 220, 230)
    label.Text = text
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 104
    label.Parent = toggleBtn

    local statusBox = Instance.new("Frame")
    statusBox.Size = UDim2.new(0, 32, 0, 18)
    statusBox.Position = UDim2.new(1, -40, 0.5, -9)
    statusBox.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
    statusBox.ZIndex = 104
    statusBox.Parent = toggleBtn

    local sCorner = Instance.new("UICorner")
    sCorner.CornerRadius = UDim.new(1, 0)
    sCorner.Parent = statusBox

    local circle = Instance.new("Frame")
    circle.Size = UDim2.new(0, 14, 0, 14)
    circle.Position = UDim2.new(0, 2, 0.5, -7)
    circle.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
    circle.ZIndex = 105
    circle.Parent = statusBox

    local cCorner = Instance.new("UICorner")
    cCorner.CornerRadius = UDim.new(1, 0)
    cCorner.Parent = circle

    local enabled = defaultConfig
    
    local function updateVisual(state)
        if state then
            circle.Position = UDim2.new(1, -16, 0.5, -7)
            circle.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
            statusBox.BackgroundColor3 = Color3.fromRGB(0, 220, 255)
        else
            circle.Position = UDim2.new(0, 2, 0.5, -7)
            circle.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
            statusBox.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
        end
    end

    updateVisual(enabled)

    toggleBtn.MouseButton1Click:Connect(function()
        enabled = not enabled
        updateVisual(enabled)
        callback(enabled)
        saveConfig()
    end)
    
    return {
        SetState = function(state)
            enabled = state
            updateVisual(enabled)
            callback(enabled)
        end
    }
end

local function addButton(tab, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -10, 0, 36)
    btn.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    btn.TextColor3 = Color3.fromRGB(0, 220, 255)
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 13
    btn.AutoButtonColor = false
    btn.ZIndex = 103
    btn.Parent = tab

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn

    btn.MouseButton1Click:Connect(callback)
    return btn
end

-- ==================== TWORZENIE ZAKŁADEK ====================
local tabLegit = addTab("🎯 Legitbot")
local tabRage = addTab("🔥 Rage")
local tabPlayer = addTab("👤 Player")
local tabVisuals = addTab("👑 Visuals")
local tabHud = addTab("🖥️ Hud")
local tabSettings = addTab("⚙️ Settings")

local triggersRef, rageRef, chinaRef

local rageSettingsFrame = Instance.new("ScrollingFrame")
rageSettingsFrame.Size = UDim2.new(1, -10, 0, 85)
rageSettingsFrame.BackgroundTransparency = 1
rageSettingsFrame.Visible = getgenv().Config.RagebotEnabled
rageSettingsFrame.ScrollBarThickness = 2
rageSettingsFrame.CanvasSize = UDim2.new(0, 0, 0, 80)
rageSettingsFrame.Parent = tabRage

local rageLayout = Instance.new("UIListLayout")
rageLayout.SortOrder = Enum.SortOrder.LayoutOrder
rageLayout.Padding = UDim.new(0, 6)
rageLayout.Parent = rageSettingsFrame

local rageLabel = Instance.new("TextLabel")
rageLabel.Size = UDim2.new(1, 0, 0, 20)
rageLabel.BackgroundTransparency = 1
rageLabel.Text = "--- Rage Settings ---"
rageLabel.TextColor3 = Color3.fromRGB(0, 220, 255)
rageLabel.Font = Enum.Font.GothamBold
rageLabel.TextSize = 12
rageLabel.Parent = rageSettingsFrame

local slotsList = {"Primary", "Secondary", "Melee"}
local currentSlotIndex = table.find(slotsList, getgenv().Config.WeaponSlot) or 1

local weaponBtn = addButton(rageSettingsFrame, "Weapon: " .. getgenv().Config.WeaponSlot, function() end)

weaponBtn.MouseButton1Click:Connect(function()
    currentSlotIndex = currentSlotIndex % #slotsList + 1
    local newSlot = slotsList[currentSlotIndex]
    getgenv().Config.WeaponSlot = newSlot
    weaponBtn.Text = "Weapon: " .. newSlot
    saveConfig()
end)

local voidspamSettingsFrame = Instance.new("Frame")
voidspamSettingsFrame.Size = UDim2.new(1, -10, 0, 66)
voidspamSettingsFrame.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
voidspamSettingsFrame.BorderSizePixel = 0
voidspamSettingsFrame.Visible = getgenv().Config.RagebotEnabled and getgenv().Config.VoidspamEnabled
voidspamSettingsFrame.ZIndex = 103
voidspamSettingsFrame.Parent = tabRage

local vsCorner = Instance.new("UICorner")
vsCorner.CornerRadius = UDim.new(0, 6)
vsCorner.Parent = voidspamSettingsFrame

local vsTitle = Instance.new("TextLabel")
vsTitle.Size = UDim2.new(1, -16, 0, 20)
vsTitle.Position = UDim2.new(0, 8, 0, 4)
vsTitle.BackgroundTransparency = 1
vsTitle.Text = "--- Voidspam Settings ---"
vsTitle.TextColor3 = Color3.fromRGB(0, 220, 255)
vsTitle.Font = Enum.Font.GothamBold
vsTitle.TextSize = 11
vsTitle.TextXAlignment = Enum.TextXAlignment.Left
vsTitle.ZIndex = 104
vsTitle.Parent = voidspamSettingsFrame

local timeVoidLabel = Instance.new("TextLabel")
timeVoidLabel.Size = UDim2.new(0.65, -8, 0, 28)
timeVoidLabel.Position = UDim2.new(0, 8, 0, 28)
timeVoidLabel.BackgroundTransparency = 1
timeVoidLabel.Text = "Time in Void (0.1 - 5): " .. tostring(getgenv().Config.TimeInVoid)
timeVoidLabel.TextColor3 = Color3.fromRGB(220, 220, 230)
timeVoidLabel.Font = Enum.Font.GothamMedium
timeVoidLabel.TextSize = 12
timeVoidLabel.TextXAlignment = Enum.TextXAlignment.Left
timeVoidLabel.ZIndex = 104
timeVoidLabel.Parent = voidspamSettingsFrame

local timeVoidBox = Instance.new("TextBox")
timeVoidBox.Size = UDim2.new(0.32, -8, 0, 28)
timeVoidBox.Position = UDim2.new(0.67, 0, 0, 28)
timeVoidBox.BackgroundColor3 = Color3.fromRGB(32, 32, 42)
timeVoidBox.TextColor3 = Color3.fromRGB(0, 220, 255)
timeVoidBox.Text = tostring(getgenv().Config.TimeInVoid)
timeVoidBox.Font = Enum.Font.GothamBold
timeVoidBox.TextSize = 12
timeVoidBox.ClearTextOnFocus = false
timeVoidBox.ZIndex = 104
timeVoidBox.Parent = voidspamSettingsFrame

local tvCorner = Instance.new("UICorner")
tvCorner.CornerRadius = UDim.new(0, 5)
tvCorner.Parent = timeVoidBox

local function updateVoidspamVisibility()
    voidspamSettingsFrame.Visible = (getgenv().Config.RagebotEnabled and getgenv().Config.VoidspamEnabled)
end

local function setTimeInVoid(val)
    local num = tonumber(val)
    if not num then
        timeVoidBox.Text = tostring(getgenv().Config.TimeInVoid)
        return
    end
    num = math.clamp(num, 0.1, 5)
    num = math.floor(num * 100 + 0.5) / 100
    getgenv().Config.TimeInVoid = num
    timeVoidBox.Text = tostring(num)
    timeVoidLabel.Text = "Time in Void (0.1 - 5): " .. tostring(num)
    saveConfig()
end

timeVoidBox.FocusLost:Connect(function()
    setTimeInVoid(timeVoidBox.Text)
end)

rageRef = addToggle(tabRage, "Ragebot & Fake Position", getgenv().Config.RagebotEnabled, function(val)
    getgenv().Config.RagebotEnabled = val
    getgenv().Config.Enabled = val
    rageSettingsFrame.Visible = val
    updateVoidspamVisibility()
    setCharacterInvisible(val)
    updateRagebotStatusHUD()
end)

voidspamRef = addToggle(tabRage, "Voidspam", getgenv().Config.VoidspamEnabled, function(val)
    getgenv().Config.VoidspamEnabled = val
    updateVoidspamVisibility()
end)

underRef = addToggle(tabRage, "Underground", getgenv().Config.UndergroundEnabled, function(val)
    getgenv().Config.UndergroundEnabled = val
    if not val then
        pcall(function()
            local char = lplr.Character
            if char and char:FindFirstChild("HumanoidRootPart") then
                char.HumanoidRootPart.AssemblyLinearVelocity = Vector3.zero
            end
        end)
    end
end)

triggersRef = addToggle(tabLegit, "Triggerbot", getgenv().Config.TriggerbotEnabled, function(val)
    getgenv().Config.TriggerbotEnabled = val
end)

-- ==================== SEKCJA PLAYER & ANIMACJE ====================
local playerSectionLabel = Instance.new("TextLabel")
playerSectionLabel.Size = UDim2.new(1, 0, 0, 20)
playerSectionLabel.BackgroundTransparency = 1
playerSectionLabel.Text = "--- Camera & Visual ---"
playerSectionLabel.TextColor3 = Color3.fromRGB(0, 220, 255)
playerSectionLabel.Font = Enum.Font.GothamBold
playerSectionLabel.TextSize = 12
playerSectionLabel.Parent = tabPlayer

stretchRef = addToggle(tabPlayer, "Camera Stretch", getgenv().Config.cameraStretch, function(val)
    getgenv().Config.cameraStretch = val
    saveConfig()
end)

local emoteSectionLabel = Instance.new("TextLabel")
emoteSectionLabel.Size = UDim2.new(1, 0, 0, 20)
emoteSectionLabel.BackgroundTransparency = 1
emoteSectionLabel.Text = "--- Emotes & Animations ---"
emoteSectionLabel.TextColor3 = Color3.fromRGB(0, 220, 255)
emoteSectionLabel.Font = Enum.Font.GothamBold
emoteSectionLabel.TextSize = 12
emoteSectionLabel.Parent = tabPlayer

addButton(tabPlayer, "🕺 Taniec: Gangnam Style", function()
    playCustomEmote(33380097)
end)

addButton(tabPlayer, "🌪️ Taniec: Tornado", function()
    playCustomEmote(35767120)
end)

addButton(tabPlayer, "💃 Taniec: Line Dance", function()
    playCustomEmote(318977779)
end)

addButton(tabPlayer, "🎸 Taniec: Air Guitar", function()
    playCustomEmote(33380184)
end)

addButton(tabPlayer, "⏹️ Zatrzymaj Animację", function()
    if activeAnimationTrack then
        activeAnimationTrack:Stop()
        activeAnimationTrack = nil
    end
end)

local animTextBox = Instance.new("TextBox")
animTextBox.Size = UDim2.new(1, -10, 0, 36)
animTextBox.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
animTextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
animTextBox.PlaceholderText = "Wpisz ID animacji..."
animTextBox.PlaceholderColor3 = Color3.fromRGB(120, 120, 130)
animTextBox.Text = ""
animTextBox.Font = Enum.Font.GothamMedium
animTextBox.TextSize = 13
animTextBox.Parent = tabPlayer

local boxCorner = Instance.new("UICorner")
boxCorner.CornerRadius = UDim.new(0, 6)
boxCorner.Parent = animTextBox

addButton(tabPlayer, "▶️ Odtwórz własną animację (z ID)", function()
    local id = tonumber(animTextBox.Text)
    if id then
        playCustomEmote(id)
    end
end)

-- ==================== SEKCJA HUD ====================
local hudSectionLabel = Instance.new("TextLabel")
hudSectionLabel.Size = UDim2.new(1, 0, 0, 20)
hudSectionLabel.BackgroundTransparency = 1
hudSectionLabel.Text = "--- HUD Elements ---"
hudSectionLabel.TextColor3 = Color3.fromRGB(0, 220, 255)
hudSectionLabel.Font = Enum.Font.GothamBold
hudSectionLabel.TextSize = 12
hudSectionLabel.Parent = tabHud

statusRef = addToggle(tabHud, "Ragebot Status", getgenv().Config.RagebotStatusEnabled, function(val)
    getgenv().Config.RagebotStatusEnabled = val
    updateRagebotStatusHUD()
end)

local fontBtn = addButton(tabHud, "Status Font: " .. getgenv().Config.StatusFont, function() end)
fontBtn.MouseButton1Click:Connect(function()
    currentFontIndex = currentFontIndex % #fontList + 1
    local chosenFont = fontList[currentFontIndex]
    applyStatusFont(chosenFont)
    fontBtn.Text = "Status Font: " .. chosenFont
    saveConfig()
end)

chinaRef = addToggle(tabVisuals, "China Hat", getgenv().ChinaHatSettings.enabled, function(val)
    getgenv().ChinaHatSettings.enabled = val
    pcall(function()
        local character = lplr.Character
        if not character then return end
        if not val then
            local existingHat = character:FindFirstChild("ChinaHatPart")
            if existingHat then existingHat:Destroy() end
        else
            local head = character:FindFirstChild("Head")
            if head and not character:FindFirstChild("ChinaHatPart") then
                local Cone = Instance.new("Part")
                Cone.Name = "ChinaHatPart"
                Cone.Size = Vector3.new(1, 1, 1)
                Cone.Material = Enum.Material.Neon
                Cone.Transparency = 0.2
                Cone.CanCollide = false
                Cone.Color = getgenv().ChinaHatSettings.hatColor 

                local Mesh = Instance.new("SpecialMesh")
                Mesh.MeshType = Enum.MeshType.FileMesh
                Mesh.MeshId = "rbxassetid://1033714"
                Mesh.Scale = Vector3.new(1.7, 1.1, 1.7)
                Mesh.Parent = Cone

                local Weld = Instance.new("Weld")
                Weld.Part0 = head
                Weld.Part1 = Cone
                Weld.C0 = CFrame.new(0, 0.9, 0)
                Weld.Parent = Cone

                Cone.Parent = character
            end
        end
    end)
end)

addButton(tabSettings, "💾 Zapisz CFG", function() saveConfig() end)
addButton(tabSettings, "📂 Załaduj CFG", function()
    pcall(function()
        if not isfile or not readfile then return end
        if isfile(CFG_PATH) then
            local content = readfile(CFG_PATH)
            local data = HttpService:JSONDecode(content)
            if data and type(data) == "table" then
                for k, v in pairs(data) do
                    if getgenv().Config[k] ~= nil then
                        getgenv().Config[k] = v
                    end
                end
            end
        end
    end)
    if triggersRef then triggersRef.SetState(getgenv().Config.TriggerbotEnabled) end
    if rageRef then rageRef.SetState(getgenv().Config.RagebotEnabled) end
    if voidspamRef then voidspamRef.SetState(getgenv().Config.VoidspamEnabled) end
    if underRef then underRef.SetState(getgenv().Config.UndergroundEnabled) end
    if stretchRef then stretchRef.SetState(getgenv().Config.cameraStretch) end
    if statusRef then statusRef.SetState(getgenv().Config.RagebotStatusEnabled) end
    currentFontIndex = table.find(fontList, getgenv().Config.StatusFont) or 1
    applyStatusFont(fontList[currentFontIndex])
    fontBtn.Text = "Status Font: " .. fontList[currentFontIndex]
    setTimeInVoid(getgenv().Config.TimeInVoid)
    weaponBtn.Text = "Weapon: " .. getgenv().Config.WeaponSlot
    updateVoidspamVisibility()
    updateRagebotStatusHUD()
end)

-- ==================== LOGIKA GRY ====================
local function handleDangerousItems()
    local descendants = {}
    pcall(function() descendants = ws:GetDescendants() end)
    for _, obj in ipairs(descendants) do
        local name = obj.Name:lower()
        if name:find("kill") or name:find("lava") or name:find("death") or name:find("hazard") or name:find("damage") or name:find("chainsaw") or name:find("subspace tripmine") then
            pcall(function()
                if obj:IsA("BasePart") then
                    obj.CFrame = CFrame.new(obj.Position.X, -500, obj.Position.Z)
                    obj.AssemblyLinearVelocity = Vector3.zero
                    obj.AssemblyAngularVelocity = Vector3.zero
                    obj.Anchored = true
                elseif obj:IsA("Model") then
                    local primary = obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
                    if primary then
                        primary.CFrame = CFrame.new(primary.Position.X, -500, primary.Position.Z)
                        primary.Anchored = true
                    end
                end
            end)
        end
    end
end

task.spawn(function()
    while true do
        handleDangerousItems()
        task.wait(0.5)
    end
end)

task.spawn(function()
    while true do
        task.wait(0.2)
        if getgenv().Config.UndergroundEnabled then
            pcall(function()
                local char = lplr.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    local root = char.HumanoidRootPart
                    if root.Position.Y > -20 then
                        root.CFrame = root.CFrame - Vector3.new(0, 4, 0)
                    end
                    root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                end
            end)
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(1)
        if not getgenv().Config.RagebotEnabled then continue end
        if FighterController and FighterController.LocalFighter then
            pcall(function() FighterController.LocalFighter:EquipItem(getSlotNumber()) end)
        end
    end
end)

local character = lplr.Character or lplr.CharacterAdded:Wait()
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")

lplr.CharacterAdded:Connect(function(newChar)
    character = newChar
    humanoidRootPart = character:WaitForChild("HumanoidRootPart")
    if getgenv().Config.RagebotEnabled then
        task.wait(0.2)
        setCharacterInvisible(true)
    end
end)

local deflecting = {}
plrs.PlayerRemoving:Connect(function(player) deflecting[player] = nil end)

local function updateDeflection()
    if not FighterController or not FighterController.Objects then return end
    for _, fighterObj in FighterController.Objects do
        local player = fighterObj.Player
        if not player then continue end
        if not fighterObj.Entity or not fighterObj.Entity:IsAlive() or fighterObj:Get("IsSpectating") then
            deflecting[player] = false
            continue
        end
        local equipped = fighterObj.EquippedItem
        local itemName = equipped and equipped.ViewModel and equipped.ViewModel.Name or ""
        local isDeflecting = (itemName == "Katana" or itemName == "Chainsaw") and (equipped._attack_cooldown and equipped._attack_cooldown > tick()) or false
        deflecting[player] = isDeflecting
    end
end

local function isEnemy(player)
    if player == lplr then return false end
    if SpectateController then
        local duel = SpectateController.CurrentDuelSubject
        local localDueler = duel and duel:GetDueler(lplr)
        local localTeam = localDueler and localDueler:Get("TeamID") or nil
        if localTeam and duel and duel.Duelers then
            for _, dueler in duel.Duelers do
                if dueler.Player == player then
                    return dueler:Get("TeamID") ~= localTeam
                end
            end
        end
    end
    local pTeam = player:GetAttribute("TeamID")
    local lTeam = lplr:GetAttribute("TeamID")
    if pTeam and lTeam then return pTeam ~= lTeam end
    return true
end

local function getClosestTarget()
    local char = lplr.Character
    if not char then return nil, nil, nil end
    local myRoot = char:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil, nil, nil end
    
    local closestPlayer, closestRoot, closestHead = nil, nil, nil
    local closestDist = math.huge
    for _, player in plrs:GetPlayers() do
        if not isEnemy(player) then continue end
        local pChar = player.Character
        if not pChar then continue end
        local pRoot = pChar:FindFirstChild("HumanoidRootPart")
        local pHead = pChar:FindFirstChild("Head")
        local pHum = pChar:FindFirstChildWhichIsA("Humanoid")
        if not (pRoot and pHead and pHum) then continue end
        local dist = (myRoot.Position - pRoot.Position).Magnitude
        if dist < closestDist then
            closestDist = dist
            closestPlayer = player
            closestRoot = pRoot
            closestHead = pHead
        end
    end
    return closestPlayer, closestRoot, closestHead
end

local lastFire = 0
local voidCycleStart = 0
local inVoidPhase = true
local lastVoidModeActive = false

runS.Heartbeat:Connect(function()
    local isVoidspamRageActive = getgenv().Config.RagebotEnabled and getgenv().Config.VoidspamEnabled

    if isVoidspamRageActive and not lastVoidModeActive then
        voidCycleStart = tick()
        inVoidPhase = true
    elseif not isVoidspamRageActive then
        inVoidPhase = false
    end
    lastVoidModeActive = isVoidspamRageActive

    if lplr.Character and lplr.Character:FindFirstChild("HumanoidRootPart") then
        local myRootPart = lplr.Character.HumanoidRootPart
        if myRootPart.Position.Y < -40 and not getgenv().Config.UndergroundEnabled and not isVoidspamRageActive then
            myRootPart.CFrame = CFrame.new(myRootPart.Position.X, 50, myRootPart.Position.Z)
            myRootPart.Velocity = Vector3.zero
        end
    end

    if not getgenv().Config.RagebotEnabled then
        currentRageTargetName = nil
        updateRagebotStatusHUD()
        return
    end

    setCharacterInvisible(true)

    updateDeflection()
    local targetPlayer, targetRoot, targetHead = getClosestTarget()
    currentRageTargetName = targetPlayer and (targetPlayer.DisplayName or targetPlayer.Name) or nil
    updateRagebotStatusHUD()

    local desyncCF = nil

    if targetPlayer and targetRoot and targetHead then
        local pHum = targetPlayer.Character and targetPlayer.Character:FindFirstChildWhichIsA("Humanoid")
        local isTargetDead = (not pHum or pHum.Health <= 0 or pHum.Health <= -150)
        
        if isTargetDead then
            local dropPos = targetRoot.Position + Vector3.new(0, getgenv().Config.DeathDropThreshold, 0)
            desyncCF = CFrame.new(dropPos)
        else
            local targetPos = targetRoot.Position
            if getgenv().Config.RandomTeleport then
                local angle = math.random() * math.pi * 2
                local radius = math.random(getgenv().Config.MinTeleportRadius * 10, getgenv().Config.MaxTeleportRadius * 10) / 10
                local height = math.random(getgenv().Config.HeightOffsetMin * 10, getgenv().Config.HeightOffsetMax * 10) / 10
                local randomOffset = Vector3.new(math.cos(angle) * radius, height, math.sin(angle) * radius)
                targetPos = targetRoot.Position + randomOffset
            else
                local behindOffset = Vector3.new(0, 0, getgenv().Config.BehindDistance)
                targetPos = targetRoot.CFrame:PointToWorldSpace(behindOffset)
            end

            local validPos = true
            if targetPos.Y < getgenv().Config.MinBoundsY or targetPos.Y > getgenv().Config.MaxBoundsY then validPos = false end
            if (targetPos - targetRoot.Position).Magnitude > getgenv().Config.MaxDistanceFromTarget then validPos = false end

            if not validPos then targetPos = targetRoot.Position + Vector3.new(0, 2, 2) end
            desyncCF = CFrame.lookAt(targetPos, targetHead.Position) * CFrame.Angles(math.rad(180), 0, 0)
        end
    else
        local myRoot = lplr.Character and lplr.Character:FindFirstChild("HumanoidRootPart")
        if myRoot then desyncCF = myRoot.CFrame end
    end

    local effectiveCF = desyncCF
    if isVoidspamRageActive then
        local elapsed = tick() - voidCycleStart
        local voidDuration = math.clamp(tonumber(getgenv().Config.TimeInVoid) or 0.5, 0.1, 5)
        local peekDuration = 0.25

        if elapsed <= voidDuration then
            inVoidPhase = true
            local basePos = (desyncCF and desyncCF.Position) or (lplr.Character and lplr.Character:FindFirstChild("HumanoidRootPart") and lplr.Character.HumanoidRootPart.Position) or Vector3.zero
            effectiveCF = CFrame.new(basePos.X, getgenv().Config.VoidPositionY, basePos.Z)
        else
            inVoidPhase = false
            effectiveCF = desyncCF
            if elapsed >= (voidDuration + peekDuration) then
                voidCycleStart = tick()
                inVoidPhase = true
            end
        end
    end

    if effectiveCF and lplr.Character then
        local myRoot = lplr.Character:FindFirstChild("HumanoidRootPart")
        if myRoot then
            local oldCF = myRoot.CFrame
            local oldVel = myRoot.Velocity
            local oldRotVel = myRoot.RotVelocity
            myRoot.CFrame = effectiveCF
            runS:BindToRenderStep("__restore", 101, function()
                if myRoot then
                    myRoot.CFrame = oldCF
                    myRoot.Velocity = oldVel
                    myRoot.RotVelocity = oldRotVel
                end
                runS:UnbindFromRenderStep("__restore")
            end)
        end
    end

    if not targetPlayer or not targetHead or not targetRoot then return end
    if isVoidspamRageActive and inVoidPhase then return end

    local pHum = targetPlayer.Character and targetPlayer.Character:FindFirstChildWhichIsA("Humanoid")
    if not pHum or pHum.Health <= 0 or pHum.Health <= -150 then return end
    if deflecting[targetPlayer] then return end
    if not lplr.Character or not lplr.Character:FindFirstChild("HumanoidRootPart") then return end
    if not FighterController or not FighterController.LocalFighter then return end
    local item = FighterController.LocalFighter.EquippedItem
    if not item then return end
    if tick() - lastFire < getgenv().Config.FireRate then return end
    lastFire = tick()
    
    local originPos = (desyncCF and desyncCF.Position) or (effectiveCF and effectiveCF.Position) or targetRoot.Position
    local targetPos = targetHead.Position
    local aimCF = CFrame.lookAt(originPos, targetPos)
    local targetCF = targetHead.CFrame
    local randomOffset = Vector3.new((math.random() - 0.5) * 0.1, (math.random() - 0.5) * 0.1, (math.random() - 0.5) * 0.1)
    local aimedPos = targetPos + randomOffset
    local objSpaceHeadOffset = targetHead.CFrame:ToObjectSpace(CFrame.new(aimedPos))
    local cameradata = {}

    cameradata[string.char(1)] = {
        [string.char(0)] = util:EncodeCFrame(aimCF),
        [string.char(1)] = util:EncodeCFrame(targetCF),
        [string.char(2)] = targetHead,
        [string.char(3)] = util:EncodeCFrame(objSpaceHeadOffset)
    }
    repS.Remotes.Replication.Fighter.UseItem:FireServer(
        item:Get("ObjectID"),
        enum:ToEnum("StartShooting"),
        cameradata,
        nil
    )
end)

runS.RenderStepped:Connect(function()
    if not getgenv().Config.TriggerbotEnabled then return end
    pcall(function()
        local mouse = lplr:GetMouse()
        local target = mouse and mouse.Target
        if target and target.Parent then
            local enemyChar = target.Parent
            local enemyPlr = plrs:GetPlayerFromCharacter(enemyChar)
            if enemyPlr and enemyPlr ~= lplr then
                local hum = enemyChar:FindFirstChildOfClass("Humanoid")
                if hum and hum.Health > 0 and mouse1click then
                    mouse1click()
                end
            end
        end
    end)
end)

print("YorozLua Hub - Test peekDuration 0.25 przy Voidspamie!")
