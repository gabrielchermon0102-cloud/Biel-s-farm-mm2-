--============================================================
--  biel's auto farm!  |  Underground & Human-Movement Edition
--  Funcional: Coleta real, Noclip, Auto-Reset, Opacidade, Bolinha
--  Anti-Ban: Movimento dinâmico (acelera/desacelera) + Deitado
--============================================================

local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local Workspace          = game:GetService("Workspace")
local TweenService       = game:GetService("TweenService")
local UserInputService   = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

-- ▸ STATE ───────────────────────────────────────────────────
local farming          = false
local loopRunning      = false
local noclip           = false
local speed            = 40
local coinsCollected   = 0
local roundLimit       = 40
local autoResetEnabled = true
local bagFull          = false
local isUnderground    = false
local isMoving         = false

-- ▸ COLORS (Purple & Gray) ──────────────────────────────────
local C = {
    bg         = Color3.fromRGB(25, 25, 30),
    sidebar    = Color3.fromRGB(18, 18, 22),
    card       = Color3.fromRGB(35, 35, 45),
    cardHover  = Color3.fromRGB(45, 45, 58),
    accent     = Color3.fromRGB(139, 92, 246),
    text       = Color3.fromRGB(255, 255, 255),
    subtext    = Color3.fromRGB(160, 160, 170),
    border     = Color3.fromRGB(60, 60, 75),
    trackOff   = Color3.fromRGB(60, 60, 70),
    success    = Color3.fromRGB(34, 197, 94),
    danger     = Color3.fromRGB(239, 68, 68),
}

-- ▸ EXECUTOR SAFE PARENT ────────────────────────────────────
local function getGuiParent()
    local ok, hui = pcall(function() return gethui() end)
    if ok and hui then return hui end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local function protectGui(gui)
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
    pcall(function() if protectgui then protectgui(gui) end end)
end

-- ▸ UI HELPERS ──────────────────────────────────────────────
local function corner(inst, r)
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, r); c.Parent = inst
end
local function stroke(inst, col, th)
    local s = Instance.new("UIStroke")
    s.Color, s.Thickness, s.ApplyStrokeMode = col or C.border, th or 1, Enum.ApplyStrokeMode.Border
    s.Parent = inst
    return s
end

-- ▸ NOCLIP ──────────────────────────────────────────────────
local noclipConn
local function startNoclip()
    if noclipConn then return end
    noclip = true
    noclipConn = RunService.Stepped:Connect(function()
        if not noclip or not LocalPlayer.Character then return end
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end)
end

local function stopNoclip()
    noclip = false
    if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
    if LocalPlayer.Character then
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = true end
        end
    end
end

-- ▸ FARM LOGIC (Dynamic Human Movement) ─────────────────────
local function findNearestCoin()
    local char = LocalPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local myPos = root.Position
    local nearest, nearestDist = nil, math.huge

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") and obj.Parent and obj.Name:lower():find("coin") then
            if not obj:IsDescendantOf(char) then
                local d = (obj.Position - myPos).Magnitude
                if d < nearestDist then
                    nearest, nearestDist = obj, d
                end
            end
        end
    end
    return nearest
end

local function applyUndergroundPose()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    
    if isUnderground and hum and root then
        hum.HipHeight = -3.5 -- Faz o personagem afundar no chão
        -- Rotaciona o personagem para ficar deitado (90 graus)
        root.CFrame = CFrame.new(root.Position) * CFrame.Angles(math.rad(90), math.rad(root.Orientation.Y), 0)
    elseif hum then
        hum.HipHeight = 2 -- Altura normal
    end
end

-- O Movimento Humano (BodyVelocity Dinâmico)
local moveConnection
local function startMovingToCoin(coin)
    local char = LocalPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root or not coin then return end

    isMoving = true
    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bv.Velocity = Vector3.zero
    bv.Parent = root

    if moveConnection then moveConnection:Disconnect() end
    
    moveConnection = RunService.Heartbeat:Connect(function(dt)
        if not farming or not isMoving or not coin.Parent or not char.Parent then
            bv.Velocity = Vector3.zero
            bv:Destroy()
            isMoving = false
            if moveConnection then moveConnection:Disconnect() end
            return
        end

        local currentPos = root.Position
        local targetPos = coin.Position
        local dist = (targetPos - currentPos).Magnitude

        if dist < 3 then
            bv.Velocity = Vector3.zero
            bv:Destroy()
            isMoving = false
            if moveConnection then moveConnection:Disconnect() end
            return
        end

        -- ▸ LÓGICA DE VELOCIDADE HUMANA ▸
        local currentSpeed = speed
        
        -- Se estiver longe, acelera (até 1.4x)
        if dist > 35 then
            currentSpeed = speed * 1.4
        -- Se estiver perto, freia (até 0.5x)
        elseif dist < 15 then
            currentSpeed = speed * 0.5
        end
        
        -- Adiciona variação (ruído) para não ser robótico
        local time = tick()
        local jitter = math.sin(time * 4) * 3 -- Oscilação suave
        local randomNoise = math.random(-2, 2) -- Aleatoriedade
        currentSpeed = math.max(5, currentSpeed + jitter + randomNoise)

        local dir = (targetPos - currentPos).Unit
        bv.Velocity = dir * currentSpeed

        -- Mantém a pose deitada se estiver no modo underground
        if isUnderground then
            applyUndergroundPose()
        end
    end)
end

--============================================================
--  BUILD THE UI
--============================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BielsAutoFarm"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.DisplayOrder = 999
protectGui(screenGui)
screenGui.Parent = getGuiParent()

-- MINIMIZED BALL
local ball = Instance.new("TextButton")
ball.Size = UDim2.new(0, 50, 0, 50)
ball.Position = UDim2.new(0, 50, 0, 50)
ball.BackgroundColor3 = C.accent
ball.Text = "⚡"
ball.TextColor3 = C.text
ball.Font = Enum.Font.GothamBold
ball.TextSize = 24
ball.Visible = false
ball.Parent = screenGui
corner(ball, 25)
stroke(ball, C.border, 2)

-- MAIN FRAME
local main = Instance.new("Frame")
main.Size = UDim2.new(0, 640, 0, 440)
main.Position = UDim2.new(0.5, -320, 0.5, -220)
main.BackgroundColor3 = C.bg
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Active = true
main.Parent = screenGui
corner(main, 10)
stroke(main, C.border, 1)

-- TOP BAR
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 40)
header.BackgroundColor3 = C.bg
header.BorderSizePixel = 0
header.Parent = main

local title = Instance.new("TextLabel")
title.Size = UDim2.new(0, 250, 1, 0)
title.Position = UDim2.new(0, 16, 0, 0)
title.BackgroundTransparency = 1
title.Text = "biel's auto farm!"
title.TextColor3 = C.text
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local minBtn = Instance.new("TextButton")
minBtn.Size = UDim2.new(0, 24, 0, 24)
minBtn.Position = UDim2.new(1, -60, 0, 8)
minBtn.BackgroundColor3 = C.card
minBtn.Text = "−"
minBtn.TextColor3 = C.text
minBtn.Font = Enum.Font.GothamBold
minBtn.TextSize = 18
minBtn.Parent = header
corner(minBtn, 6)

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 24, 0, 24)
closeBtn.Position = UDim2.new(1, -30, 0, 8)
closeBtn.BackgroundColor3 = C.card
closeBtn.Text = "✕"
closeBtn.TextColor3 = C.text
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 12
closeBtn.Parent = header
corner(closeBtn, 6)
closeBtn.MouseButton1Click:Connect(function() screenGui:Destroy() end)

-- SIDEBAR
local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.new(0, 180, 1, -40)
sidebar.Position = UDim2.new(0, 0, 0, 40)
sidebar.BackgroundColor3 = C.sidebar
sidebar.BorderSizePixel = 0
sidebar.Parent = main
stroke(sidebar, C.border, 1)

local navList = Instance.new("Frame")
navList.Size = UDim2.new(1, 0, 0, 200)
navList.Position = UDim2.new(0, 0, 0, 20)
navList.BackgroundTransparency = 1
navList.Parent = sidebar

local navLayout = Instance.new("UIListLayout")
navLayout.Padding = UDim.new(0, 4)
navLayout.Parent = navList

local function createNavBtn(text, icon, active)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -16, 0, 32)
    btn.BackgroundColor3 = active and C.cardHover or C.sidebar
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.Parent = navList
    corner(btn, 6)

    local ico = Instance.new("TextLabel")
    ico.Size = UDim2.new(0, 32, 1, 0)
    ico.Position = UDim2.new(0, 8, 0, 0)
    ico.BackgroundTransparency = 1
    ico.Text = icon
    ico.TextColor3 = active and C.accent or C.subtext
    ico.TextSize = 14
    ico.Parent = btn

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, -40, 1, 0)
    txt.Position = UDim2.new(0, 40, 0, 0)
    txt.BackgroundTransparency = 1
    txt.Text = text
    txt.TextColor3 = active and C.text or C.subtext
    txt.Font = active and Enum.Font.GothamBold or Enum.Font.Gotham
    txt.TextSize = 13
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.Parent = btn
end
createNavBtn("Farm", "⚔️", true)
createNavBtn("Settings", "⚙️", false)

-- CONTENT AREA
local content = Instance.new("ScrollingFrame")
content.Size = UDim2.new(1, -180, 1, -40)
content.Position = UDim2.new(0, 180, 0, 40)
content.BackgroundColor3 = C.bg
content.BorderSizePixel = 0
content.CanvasSize = UDim2.new(0, 0, 0, 0)
content.AutomaticCanvasSize = Enum.AutomaticSize.Y
content.ScrollBarThickness = 2
content.ScrollBarImageColor3 = C.accent
content.Parent = main

local cPad = Instance.new("UIPadding")
cPad.PaddingTop = UDim.new(0, 16)
cPad.PaddingLeft = UDim.new(0, 16)
cPad.PaddingRight = UDim.new(0, 16)
cPad.PaddingBottom = UDim.new(0, 16)
cPad.Parent = content

local cLayout = Instance.new("UIListLayout")
cLayout.Padding = UDim.new(0, 12)
cLayout.SortOrder = Enum.SortOrder.LayoutOrder
cLayout.Parent = content

local function createCard(titleText, order)
    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, 0, 0, 0)
    card.AutomaticSize = Enum.AutomaticSize.Y
    card.BackgroundColor3 = C.card
    card.Parent = content
    corner(card, 8)
    stroke(card, C.border, 1)
    card.LayoutOrder = order

    local cLayoutInner = Instance.new("UIListLayout")
    cLayoutInner.Padding = UDim.new(0, 8)
    cLayoutInner.SortOrder = Enum.SortOrder.LayoutOrder
    cLayoutInner.Parent = card

    local cPadInner = Instance.new("UIPadding")
    cPadInner.PaddingTop = UDim.new(0, 12)
    cPadInner.PaddingBottom = UDim.new(0, 12)
    cPadInner.PaddingLeft = UDim.new(0, 12)
    cPadInner.PaddingRight = UDim.new(0, 12)
    cPadInner.Parent = card

    local tText = Instance.new("TextLabel")
    tText.Size = UDim2.new(1, 0, 0, 20)
    tText.BackgroundTransparency = 1
    tText.Text = titleText
    tText.TextColor3 = C.text
    tText.Font = Enum.Font.GothamBold
    tText.TextSize = 14
    tText.TextXAlignment = Enum.TextXAlignment.Left
    tText.LayoutOrder = 1
    tText.Parent = card

    return card
end

local function createToggle(parent, text, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 24)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = C.subtext
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local sw = Instance.new("TextButton")
    sw.Size = UDim2.new(0, 36, 0, 20)
    sw.Position = UDim2.new(1, -36, 0.5, -10)
    sw.BackgroundColor3 = default and C.accent or C.trackOff
    sw.Text = ""
    sw.AutoButtonColor = false
    sw.Parent = row
    corner(sw, 10)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = default and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    knob.BackgroundColor3 = C.text
    knob.Parent = sw
    corner(knob, 8)

    local state = default
    sw.MouseButton1Click:Connect(function()
        state = not state
        TweenService:Create(sw, TweenInfo.new(0.2), {BackgroundColor3 = state and C.accent or C.trackOff}):Play()
        TweenService:Create(knob, TweenInfo.new(0.2), {Position = state and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)}):Play()
        if callback then callback(state) end
    end)
end

local function createSlider(parent, text, min, max, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 40)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 0, 16)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = C.subtext
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0, 50, 0, 16)
    valLbl.Position = UDim2.new(1, -50, 0, 0)
    valLbl.BackgroundTransparency = 1
    valLbl.Text = tostring(default)
    valLbl.TextColor3 = C.text
    valLbl.Font = Enum.Font.GothamBold
    valLbl.TextSize = 13
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.Parent = row

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, 0, 0, 4)
    track.Position = UDim2.new(0, 0, 0, 28)
    track.BackgroundColor3 = C.trackOff
    track.Parent = row
    corner(track, 2)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((default-min)/(max-min), 0, 1, 0)
    fill.BackgroundColor3 = C.accent
    fill.Parent = track
    corner(fill, 2)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 12, 0, 12)
    knob.Position = UDim2.new((default-min)/(max-min), -6, 0.5, -6)
    knob.BackgroundColor3 = C.text
    knob.Parent = track
    corner(knob, 6)

    local dragging = false
    local function update(input)
        local pos = math.clamp((input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + (max - min) * pos)
        fill.Size = UDim2.new(pos, 0, 1, 0)
        knob.Position = UDim2.new(pos, -6, 0.5, -6)
        valLbl.Text = tostring(val)
        if callback then callback(val) end
    end

    track.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            update(input)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

--============================================================
--  CARDS
--============================================================
local cardFarm = createCard("Auto Farm", 1)
createToggle(cardFarm, "Coin Autofarm", false, function(state)
    farming = state
    if farming then
        startNoclip()
        if not loopRunning then task.spawn(function()
            loopRunning = true
            while farming do
                local coin = findNearestCoin()
                if coin and not isMoving then
                    startMovingToCoin(coin)
                    -- Espera o movimento até a moeda terminar
                    while isMoving and farming do task.wait(0.1) end
                    
                    if farming and coin.Parent == nil then
                        coinsCollected = coinsCollected + 1
                        if coinsCollected >= roundLimit then bagFull = true end
                    end
                else
                    task.wait(0.5)
                end
                task.wait(0.1)
            end
            loopRunning = false
        end) end
    else
        stopNoclip()
        isMoving = false
    end
end)

-- NOVO: Toggle para ficar deitado e no subsolo
createToggle(cardFarm, "Underground Mode (Lying)", false, function(state)
    isUnderground = state
    if farming then
        applyUndergroundPose()
    end
end)

createSlider(cardFarm, "Base Farm Speed", 10, 100, speed, function(val)
    speed = val
end)

local cardSettings = createCard("Settings", 2)
createToggle(cardSettings, "Auto-Reset When Bag Full", autoResetEnabled, function(state)
    autoResetEnabled = state
end)

createSlider(cardSettings, "Hub Opacity", 0, 90, 0, function(val)
    local trans = val / 100
    main.BackgroundTransparency = trans
    sidebar.BackgroundTransparency = trans
    header.BackgroundTransparency = trans
    for _, child in ipairs(content:GetChildren()) do
        if child:IsA("Frame") then
            child.BackgroundTransparency = trans
        end
    end
end)

--============================================================
--  DRAG LOGIC
--============================================================
local function makeDraggable(frame, isMain)
    local dragging, dragStart, startPos, dragInput
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                    if not isMain then
                        local dist = (input.Position - dragStart).Magnitude
                        if dist < 10 then
                            main.Visible = true
                            ball.Visible = false
                        end
                    end
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

makeDraggable(main, true)
makeDraggable(ball, false)

minBtn.MouseButton1Click:Connect(function()
    main.Visible = false
    ball.Visible = true
end)

--============================================================
--  AUTO RESET & INIT
--============================================================
local resetting = false
task.spawn(function()
    while task.wait(0.5) do
        if farming and bagFull and autoResetEnabled and not resetting then
            resetting = true
            isMoving = false
            stopNoclip()
            if LocalPlayer.Character then
                LocalPlayer.Character:BreakJoints()
            end
            coinsCollected = 0
            bagFull = false
            LocalPlayer.CharacterAdded:Wait()
            task.wait(1.5)
            if farming then 
                startNoclip() 
                if isUnderground then applyUndergroundPose() end
            end
            resetting = false
        end
    end
end)

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)
    if farming then
        startNoclip()
        if isUnderground then applyUndergroundPose() end
    end
end)

print("[MM2] biel's auto farm! (Underground & Human Edition) Loaded.")--============================================================
--  biel's auto farm!  |  Underground & Human-Movement Edition
--  Funcional: Coleta real, Noclip, Auto-Reset, Opacidade, Bolinha
--  Anti-Ban: Movimento dinâmico (acelera/desacelera) + Deitado
--============================================================

local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local Workspace          = game:GetService("Workspace")
local TweenService       = game:GetService("TweenService")
local UserInputService   = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

-- ▸ STATE ───────────────────────────────────────────────────
local farming          = false
local loopRunning      = false
local noclip           = false
local speed            = 40
local coinsCollected   = 0
local roundLimit       = 40
local autoResetEnabled = true
local bagFull          = false
local isUnderground    = false
local isMoving         = false

-- ▸ COLORS (Purple & Gray) ──────────────────────────────────
local C = {
    bg         = Color3.fromRGB(25, 25, 30),
    sidebar    = Color3.fromRGB(18, 18, 22),
    card       = Color3.fromRGB(35, 35, 45),
    cardHover  = Color3.fromRGB(45, 45, 58),
    accent     = Color3.fromRGB(139, 92, 246),
    text       = Color3.fromRGB(255, 255, 255),
    subtext    = Color3.fromRGB(160, 160, 170),
    border     = Color3.fromRGB(60, 60, 75),
    trackOff   = Color3.fromRGB(60, 60, 70),
    success    = Color3.fromRGB(34, 197, 94),
    danger     = Color3.fromRGB(239, 68, 68),
}

-- ▸ EXECUTOR SAFE PARENT ────────────────────────────────────
local function getGuiParent()
    local ok, hui = pcall(function() return gethui() end)
    if ok and hui then return hui end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local function protectGui(gui)
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
    pcall(function() if protectgui then protectgui(gui) end end)
end

-- ▸ UI HELPERS ──────────────────────────────────────────────
local function corner(inst, r)
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, r); c.Parent = inst
end
local function stroke(inst, col, th)
    local s = Instance.new("UIStroke")
    s.Color, s.Thickness, s.ApplyStrokeMode = col or C.border, th or 1, Enum.ApplyStrokeMode.Border
    s.Parent = inst
    return s
end

-- ▸ NOCLIP ──────────────────────────────────────────────────
local noclipConn
local function startNoclip()
    if noclipConn then return end
    noclip = true
    noclipConn = RunService.Stepped:Connect(function()
        if not noclip or not LocalPlayer.Character then return end
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end)
end

local function stopNoclip()
    noclip = false
    if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
    if LocalPlayer.Character then
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = true end
        end
    end
end

-- ▸ FARM LOGIC (Dynamic Human Movement) ─────────────────────
local function findNearestCoin()
    local char = LocalPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local myPos = root.Position
    local nearest, nearestDist = nil, math.huge

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") and obj.Parent and obj.Name:lower():find("coin") then
            if not obj:IsDescendantOf(char) then
                local d = (obj.Position - myPos).Magnitude
                if d < nearestDist then
                    nearest, nearestDist = obj, d
                end
            end
        end
    end
    return nearest
end

local function applyUndergroundPose()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    
    if isUnderground and hum and root then
        hum.HipHeight = -3.5 -- Faz o personagem afundar no chão
        -- Rotaciona o personagem para ficar deitado (90 graus)
        root.CFrame = CFrame.new(root.Position) * CFrame.Angles(math.rad(90), math.rad(root.Orientation.Y), 0)
    elseif hum then
        hum.HipHeight = 2 -- Altura normal
    end
end

-- O Movimento Humano (BodyVelocity Dinâmico)
local moveConnection
local function startMovingToCoin(coin)
    local char = LocalPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root or not coin then return end

    isMoving = true
    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bv.Velocity = Vector3.zero
    bv.Parent = root

    if moveConnection then moveConnection:Disconnect() end
    
    moveConnection = RunService.Heartbeat:Connect(function(dt)
        if not farming or not isMoving or not coin.Parent or not char.Parent then
            bv.Velocity = Vector3.zero
            bv:Destroy()
            isMoving = false
            if moveConnection then moveConnection:Disconnect() end
            return
        end

        local currentPos = root.Position
        local targetPos = coin.Position
        local dist = (targetPos - currentPos).Magnitude

        if dist < 3 then
            bv.Velocity = Vector3.zero
            bv:Destroy()
            isMoving = false
            if moveConnection then moveConnection:Disconnect() end
            return
        end

        -- ▸ LÓGICA DE VELOCIDADE HUMANA ▸
        local currentSpeed = speed
        
        -- Se estiver longe, acelera (até 1.4x)
        if dist > 35 then
            currentSpeed = speed * 1.4
        -- Se estiver perto, freia (até 0.5x)
        elseif dist < 15 then
            currentSpeed = speed * 0.5
        end
        
        -- Adiciona variação (ruído) para não ser robótico
        local time = tick()
        local jitter = math.sin(time * 4) * 3 -- Oscilação suave
        local randomNoise = math.random(-2, 2) -- Aleatoriedade
        currentSpeed = math.max(5, currentSpeed + jitter + randomNoise)

        local dir = (targetPos - currentPos).Unit
        bv.Velocity = dir * currentSpeed

        -- Mantém a pose deitada se estiver no modo underground
        if isUnderground then
            applyUndergroundPose()
        end
    end)
end

--============================================================
--  BUILD THE UI
--============================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BielsAutoFarm"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.DisplayOrder = 999
protectGui(screenGui)
screenGui.Parent = getGuiParent()

-- MINIMIZED BALL
local ball = Instance.new("TextButton")
ball.Size = UDim2.new(0, 50, 0, 50)
ball.Position = UDim2.new(0, 50, 0, 50)
ball.BackgroundColor3 = C.accent
ball.Text = "⚡"
ball.TextColor3 = C.text
ball.Font = Enum.Font.GothamBold
ball.TextSize = 24
ball.Visible = false
ball.Parent = screenGui
corner(ball, 25)
stroke(ball, C.border, 2)

-- MAIN FRAME
local main = Instance.new("Frame")
main.Size = UDim2.new(0, 640, 0, 440)
main.Position = UDim2.new(0.5, -320, 0.5, -220)
main.BackgroundColor3 = C.bg
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Active = true
main.Parent = screenGui
corner(main, 10)
stroke(main, C.border, 1)

-- TOP BAR
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 40)
header.BackgroundColor3 = C.bg
header.BorderSizePixel = 0
header.Parent = main

local title = Instance.new("TextLabel")
title.Size = UDim2.new(0, 250, 1, 0)
title.Position = UDim2.new(0, 16, 0, 0)
title.BackgroundTransparency = 1
title.Text = "biel's auto farm!"
title.TextColor3 = C.text
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local minBtn = Instance.new("TextButton")
minBtn.Size = UDim2.new(0, 24, 0, 24)
minBtn.Position = UDim2.new(1, -60, 0, 8)
minBtn.BackgroundColor3 = C.card
minBtn.Text = "−"
minBtn.TextColor3 = C.text
minBtn.Font = Enum.Font.GothamBold
minBtn.TextSize = 18
minBtn.Parent = header
corner(minBtn, 6)

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 24, 0, 24)
closeBtn.Position = UDim2.new(1, -30, 0, 8)
closeBtn.BackgroundColor3 = C.card
closeBtn.Text = "✕"
closeBtn.TextColor3 = C.text
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 12
closeBtn.Parent = header
corner(closeBtn, 6)
closeBtn.MouseButton1Click:Connect(function() screenGui:Destroy() end)

-- SIDEBAR
local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.new(0, 180, 1, -40)
sidebar.Position = UDim2.new(0, 0, 0, 40)
sidebar.BackgroundColor3 = C.sidebar
sidebar.BorderSizePixel = 0
sidebar.Parent = main
stroke(sidebar, C.border, 1)

local navList = Instance.new("Frame")
navList.Size = UDim2.new(1, 0, 0, 200)
navList.Position = UDim2.new(0, 0, 0, 20)
navList.BackgroundTransparency = 1
navList.Parent = sidebar

local navLayout = Instance.new("UIListLayout")
navLayout.Padding = UDim.new(0, 4)
navLayout.Parent = navList

local function createNavBtn(text, icon, active)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -16, 0, 32)
    btn.BackgroundColor3 = active and C.cardHover or C.sidebar
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.Parent = navList
    corner(btn, 6)

    local ico = Instance.new("TextLabel")
    ico.Size = UDim2.new(0, 32, 1, 0)
    ico.Position = UDim2.new(0, 8, 0, 0)
    ico.BackgroundTransparency = 1
    ico.Text = icon
    ico.TextColor3 = active and C.accent or C.subtext
    ico.TextSize = 14
    ico.Parent = btn

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, -40, 1, 0)
    txt.Position = UDim2.new(0, 40, 0, 0)
    txt.BackgroundTransparency = 1
    txt.Text = text
    txt.TextColor3 = active and C.text or C.subtext
    txt.Font = active and Enum.Font.GothamBold or Enum.Font.Gotham
    txt.TextSize = 13
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.Parent = btn
end
createNavBtn("Farm", "⚔️", true)
createNavBtn("Settings", "⚙️", false)

-- CONTENT AREA
local content = Instance.new("ScrollingFrame")
content.Size = UDim2.new(1, -180, 1, -40)
content.Position = UDim2.new(0, 180, 0, 40)
content.BackgroundColor3 = C.bg
content.BorderSizePixel = 0
content.CanvasSize = UDim2.new(0, 0, 0, 0)
content.AutomaticCanvasSize = Enum.AutomaticSize.Y
content.ScrollBarThickness = 2
content.ScrollBarImageColor3 = C.accent
content.Parent = main

local cPad = Instance.new("UIPadding")
cPad.PaddingTop = UDim.new(0, 16)
cPad.PaddingLeft = UDim.new(0, 16)
cPad.PaddingRight = UDim.new(0, 16)
cPad.PaddingBottom = UDim.new(0, 16)
cPad.Parent = content

local cLayout = Instance.new("UIListLayout")
cLayout.Padding = UDim.new(0, 12)
cLayout.SortOrder = Enum.SortOrder.LayoutOrder
cLayout.Parent = content

local function createCard(titleText, order)
    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, 0, 0, 0)
    card.AutomaticSize = Enum.AutomaticSize.Y
    card.BackgroundColor3 = C.card
    card.Parent = content
    corner(card, 8)
    stroke(card, C.border, 1)
    card.LayoutOrder = order

    local cLayoutInner = Instance.new("UIListLayout")
    cLayoutInner.Padding = UDim.new(0, 8)
    cLayoutInner.SortOrder = Enum.SortOrder.LayoutOrder
    cLayoutInner.Parent = card

    local cPadInner = Instance.new("UIPadding")
    cPadInner.PaddingTop = UDim.new(0, 12)
    cPadInner.PaddingBottom = UDim.new(0, 12)
    cPadInner.PaddingLeft = UDim.new(0, 12)
    cPadInner.PaddingRight = UDim.new(0, 12)
    cPadInner.Parent = card

    local tText = Instance.new("TextLabel")
    tText.Size = UDim2.new(1, 0, 0, 20)
    tText.BackgroundTransparency = 1
    tText.Text = titleText
    tText.TextColor3 = C.text
    tText.Font = Enum.Font.GothamBold
    tText.TextSize = 14
    tText.TextXAlignment = Enum.TextXAlignment.Left
    tText.LayoutOrder = 1
    tText.Parent = card

    return card
end

local function createToggle(parent, text, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 24)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = C.subtext
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local sw = Instance.new("TextButton")
    sw.Size = UDim2.new(0, 36, 0, 20)
    sw.Position = UDim2.new(1, -36, 0.5, -10)
    sw.BackgroundColor3 = default and C.accent or C.trackOff
    sw.Text = ""
    sw.AutoButtonColor = false
    sw.Parent = row
    corner(sw, 10)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = default and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    knob.BackgroundColor3 = C.text
    knob.Parent = sw
    corner(knob, 8)

    local state = default
    sw.MouseButton1Click:Connect(function()
        state = not state
        TweenService:Create(sw, TweenInfo.new(0.2), {BackgroundColor3 = state and C.accent or C.trackOff}):Play()
        TweenService:Create(knob, TweenInfo.new(0.2), {Position = state and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)}):Play()
        if callback then callback(state) end
    end)
end

local function createSlider(parent, text, min, max, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 40)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 0, 16)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = C.subtext
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0, 50, 0, 16)
    valLbl.Position = UDim2.new(1, -50, 0, 0)
    valLbl.BackgroundTransparency = 1
    valLbl.Text = tostring(default)
    valLbl.TextColor3 = C.text
    valLbl.Font = Enum.Font.GothamBold
    valLbl.TextSize = 13
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.Parent = row

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, 0, 0, 4)
    track.Position = UDim2.new(0, 0, 0, 28)
    track.BackgroundColor3 = C.trackOff
    track.Parent = row
    corner(track, 2)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((default-min)/(max-min), 0, 1, 0)
    fill.BackgroundColor3 = C.accent
    fill.Parent = track
    corner(fill, 2)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 12, 0, 12)
    knob.Position = UDim2.new((default-min)/(max-min), -6, 0.5, -6)
    knob.BackgroundColor3 = C.text
    knob.Parent = track
    corner(knob, 6)

    local dragging = false
    local function update(input)
        local pos = math.clamp((input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + (max - min) * pos)
        fill.Size = UDim2.new(pos, 0, 1, 0)
        knob.Position = UDim2.new(pos, -6, 0.5, -6)
        valLbl.Text = tostring(val)
        if callback then callback(val) end
    end

    track.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            update(input)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

--============================================================
--  CARDS
--============================================================
local cardFarm = createCard("Auto Farm", 1)
createToggle(cardFarm, "Coin Autofarm", false, function(state)
    farming = state
    if farming then
        startNoclip()
        if not loopRunning then task.spawn(function()
            loopRunning = true
            while farming do
                local coin = findNearestCoin()
                if coin and not isMoving then
                    startMovingToCoin(coin)
                    -- Espera o movimento até a moeda terminar
                    while isMoving and farming do task.wait(0.1) end
                    
                    if farming and coin.Parent == nil then
                        coinsCollected = coinsCollected + 1
                        if coinsCollected >= roundLimit then bagFull = true end
                    end
                else
                    task.wait(0.5)
                end
                task.wait(0.1)
            end
            loopRunning = false
        end) end
    else
        stopNoclip()
        isMoving = false
    end
end)

-- NOVO: Toggle para ficar deitado e no subsolo
createToggle(cardFarm, "Underground Mode (Lying)", false, function(state)
    isUnderground = state
    if farming then
        applyUndergroundPose()
    end
end)

createSlider(cardFarm, "Base Farm Speed", 10, 100, speed, function(val)
    speed = val
end)

local cardSettings = createCard("Settings", 2)
createToggle(cardSettings, "Auto-Reset When Bag Full", autoResetEnabled, function(state)
    autoResetEnabled = state
end)

createSlider(cardSettings, "Hub Opacity", 0, 90, 0, function(val)
    local trans = val / 100
    main.BackgroundTransparency = trans
    sidebar.BackgroundTransparency = trans
    header.BackgroundTransparency = trans
    for _, child in ipairs(content:GetChildren()) do
        if child:IsA("Frame") then
            child.BackgroundTransparency = trans
        end
    end
end)

--============================================================
--  DRAG LOGIC
--============================================================
local function makeDraggable(frame, isMain)
    local dragging, dragStart, startPos, dragInput
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                    if not isMain then
                        local dist = (input.Position - dragStart).Magnitude
                        if dist < 10 then
                            main.Visible = true
                            ball.Visible = false
                        end
                    end
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

makeDraggable(main, true)
makeDraggable(ball, false)

minBtn.MouseButton1Click:Connect(function()
    main.Visible = false
    ball.Visible = true
end)

--============================================================
--  AUTO RESET & INIT
--============================================================
local resetting = false
task.spawn(function()
    while task.wait(0.5) do
        if farming and bagFull and autoResetEnabled and not resetting then
            resetting = true
            isMoving = false
            stopNoclip()
            if LocalPlayer.Character then
                LocalPlayer.Character:BreakJoints()
            end
            coinsCollected = 0
            bagFull = false
            LocalPlayer.CharacterAdded:Wait()
            task.wait(1.5)
            if farming then 
                startNoclip() 
                if isUnderground then applyUndergroundPose() end
            end
            resetting = false
        end
    end
end)

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)
    if farming then
        startNoclip()
        if isUnderground then applyUndergroundPose() end
    end
end)

print("[MM2] biel's auto farm! (Underground & Human Edition) Loaded.")
