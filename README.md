-- CoinClicker v28 - Auto Golden com Clique de Mouse & Tempo Ajustado (1-4s)
-- [1] Auto Golden usa simulação física de clique de mouse via VirtualInputManager
-- [2] Intervalo do Auto Golden ajustado para 1 a 4 segundos
-- [3] AutoClick rápido + humanizado e Anti-AFK mantidos

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualUser = game:GetService("VirtualUser")
local VirtualInputManager = game:GetService("VirtualInputManager")
local GuiService = game:GetService("GuiService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

player.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)

local State = {
    Running = true,
    AutoClick = false,
    AutoItems = false,
    AutoBuff = false,
    AutoWrinkler = false,
    AutoGolden = false,
    CompactView = false,
    
    LastClick = 0,
    LastCompact = 0,
    
    StaminaStart = 0,
    TargetClickTime = 50,
    TargetRestTime = 70,
    NextClickDelay = 0.02,
    MicroPauseEnd = 0,
    
    NextItemTime = 0,
    NextBuffTime = 0,
    NextWrinklerTime = 0,
    NextGoldenTime = 0,
}

local function getRandomHumanDelay()
    return math.random(100, 900) / 100 -- 1.0s a 9.0s
end

local function getGoldenDelay()
    return math.random(100, 400) / 100 -- 1.0s a 4.0s
end

--==================================================
-- FUNÇÕES DE SUPORTE
--==================================================
local function parseNumber(text)
    if not text or text == "" then return 0 end
    text = string.lower(text)
    local valStr, suffix = string.match(text, "(%d+%.?%d*)%s*([kmbt]?)")
    if not valStr then return 0 end
    local num = tonumber(valStr) or 0
    if suffix == "k" then num *= 1e3
    elseif suffix == "m" then num *= 1e6
    elseif suffix == "b" then num *= 1e9
    elseif suffix == "t" then num *= 1e12
    end
    return num
end

local function getGui()
    return playerGui:FindFirstChild("CoinClickerGui")
end

local function visible(obj)
    if not obj or not obj:IsA("GuiObject") or not obj.Visible then return false end
    local p = obj.Parent
    while p and p ~= playerGui do
        if p:IsA("GuiObject") and not p.Visible then return false end
        p = p.Parent
    end
    return true
end

local function press(button, allowHidden)
    if not button or not button:IsA("GuiButton") then return false end
    if not allowHidden and not visible(button) then return false end

    if type(firesignal) == "function" then
        local ok1 = pcall(firesignal, button.Activated)
        local ok2 = pcall(firesignal, button.MouseButton1Click)
        return ok1 or ok2
    end

    if not visible(button) then return false end
    return pcall(function() button:Activate() end)
end

-- Clique Físico de Mouse na posição exata da UI
local function mouseClickButton(button)
    if not button or not visible(button) then return false end
    
    local inset = GuiService:GetGuiInset()
    local pos = button.AbsolutePosition
    local size = button.AbsoluteSize
    
    -- Calcula centro do botão com pequena variação humana (+- 3 pixels)
    local x = pos.X + (size.X / 2) + math.random(-3, 3)
    local y = pos.Y + (size.Y / 2) + inset.Y + math.random(-3, 3)
    
    local success = pcall(function()
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, true, game, 0)
        task.wait(math.random(35, 75) / 1000) -- tempo do toque
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, false, game, 0)
    end)
    
    if not success then
        return press(button)
    end
    return true
end

local function findFrame(name)
    local gui = getGui()
    if not gui then return nil end
    return gui:FindFirstChild(name, true)
end

local FrameCache = {}
local function getFrame(name)
    local cached = FrameCache[name]
    if cached and cached.Parent then return cached end
    cached = findFrame(name)
    FrameCache[name] = cached
    return cached
end

local function allButtons(root)
    local result = {}
    if not root then return result end
    if root:IsA("GuiButton") and visible(root) then table.insert(result, root) end
    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("GuiButton") and visible(obj) then table.insert(result, obj) end
    end
    return result
end

local function textOf(obj)
    local parts = {}
    if obj:IsA("TextButton") and obj.Text ~= "" then table.insert(parts, obj.Text) end
    for _, d in ipairs(obj:GetDescendants()) do
        if (d:IsA("TextLabel") or d:IsA("TextButton")) and d.Text ~= "" then
            table.insert(parts, d.Text)
        end
    end
    return string.lower(table.concat(parts, " "))
end

local function greenStroke(obj)
    local function green(c)
        return c.G > c.R * 1.08 and c.G > c.B * 1.02 and c.G > 0.4
    end
    for _, d in ipairs(obj:GetDescendants()) do
        if d:IsA("UIStroke") and d.Enabled and green(d.Color) then return true end
    end
    local p = obj.Parent
    if p then
        for _, d in ipairs(p:GetChildren()) do
            if d:IsA("UIStroke") and d.Enabled and green(d.Color) then return true end
        end
    end
    return false
end

--==================================================
-- CACHE DA MOEDA
--==================================================
local cachedBigCoin = nil

local function nameLooksLikeCoin(obj)
    local n = string.lower(obj.Name)
    return string.find(n, "bigcoin", 1, true)
        or n == "coin"
        or n == "coinbutton"
        or string.find(n, "click", 1, true)
end

local function underBlockedSection(obj)
    local p = obj.Parent
    while p and p ~= playerGui do
        local n = string.lower(p.Name)
        if string.find(n, "wrinkler", 1, true)
            or string.find(n, "golden", 1, true)
            or string.find(n, "store", 1, true)
            or string.find(n, "upgrade", 1, true)
            or string.find(n, "generator", 1, true)
        then
            return true
        end
        p = p.Parent
    end
    return false
end

local function findBigCoin()
    local gui = getGui()
    if not gui then return nil end

    local middle = gui:FindFirstChild("MiddleColumn", true)

    if middle then
        for _, obj in ipairs(middle:GetDescendants()) do
            if obj:IsA("GuiButton") and nameLooksLikeCoin(obj) then return obj end
        end
    end

    for _, obj in ipairs(gui:GetDescendants()) do
        if obj:IsA("GuiButton") and nameLooksLikeCoin(obj) and not underBlockedSection(obj) then
            return obj
        end
    end

    local root = middle or gui
    local best, bestArea = nil, 0
    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("GuiButton") and not underBlockedSection(obj) then
            local s = obj.AbsoluteSize
            local area = s.X * s.Y
            if area > bestArea then
                best = obj
                bestArea = area
            end
        end
    end
    return best
end

local function getBigCoin()
    if cachedBigCoin and cachedBigCoin.Parent and cachedBigCoin:IsDescendantOf(playerGui) then
        return cachedBigCoin
    end
    cachedBigCoin = findBigCoin()
    return cachedBigCoin
end

--==================================================
-- FUNÇÕES DE AUTOMAÇÃO
--==================================================
local function buyBestGenerator()
    local generators = getFrame("Generators")
    if not generators then return false end
    local candidates = {}
    
    for _, button in ipairs(allButtons(generators)) do
        local s = button.AbsoluteSize
        if s.X >= 120 and s.Y >= 32 then
            local txt = textOf(button)
            if not string.find(txt, "robux", 1, true) then
                local price = parseNumber(txt)
                table.insert(candidates, {
                    button = button,
                    price = price,
                    posY = button.AbsolutePosition.Y,
                    canBuy = greenStroke(button)
                })
            end
        end
    end
    
    table.sort(candidates, function(a, b)
        if a.canBuy ~= b.canBuy then return a.canBuy end
        if a.price > 0 and b.price > 0 and a.price ~= b.price then return a.price > b.price end
        return a.posY > b.posY
    end)
    
    for _, item in ipairs(candidates) do
        if item.canBuy and press(item.button) then return true end
    end
    return false
end

local function buyAvailableUpgrade()
    local upgrades = getFrame("Upgrades")
    if not upgrades then return false end
    local candidates = allButtons(upgrades)
    
    table.sort(candidates, function(a, b)
        return a.AbsolutePosition.X < b.AbsolutePosition.X
    end)
    
    for _, button in ipairs(candidates) do
        if greenStroke(button) and press(button) then return true end
    end
    return false
end

local function popWrinklers()
    local wrinklers = getFrame("Wrinklers")
    if not wrinklers then return 0 end
    local count = 0
    for _, button in ipairs(allButtons(wrinklers)) do
        local s = button.AbsoluteSize
        if s.X >= 20 and s.Y >= 20 then
            if press(button) then count += 1 end
        end
    end
    return count
end

local function clickGoldens()
    local goldens = getFrame("Goldens")
    if not goldens then return 0 end
    local count = 0
    for _, button in ipairs(allButtons(goldens)) do
        if mouseClickButton(button) then
            count += 1
        end
    end
    return count
end

--==================================================
-- COMPACT VIEW
--==================================================
local saved = setmetatable({}, {__mode = "k"})

local function remember(obj)
    if not obj or not obj:IsA("GuiObject") or saved[obj] then return end
    saved[obj] = {
        Visible = obj.Visible,
        Size = obj.Size,
        Position = obj.Position,
        AnchorPoint = obj.AnchorPoint,
        BackgroundTransparency = obj.BackgroundTransparency,
        Active = obj.Active,
    }
end

local function restore(obj)
    local old = saved[obj]
    if not old or not obj or not obj.Parent then return end
    obj.Visible = old.Visible
    obj.Size = old.Size
    obj.Position = old.Position
    obj.AnchorPoint = old.AnchorPoint
    obj.BackgroundTransparency = old.BackgroundTransparency
    obj.Active = old.Active
end

local function applyCompact()
    local gui = getGui()
    if not gui then return end
    
    local main = gui:FindFirstChild("Main", true)
    local left = gui:FindFirstChild("LeftColumn", true)
    local middle = gui:FindFirstChild("MiddleColumn", true)
    local right = gui:FindFirstChild("RightColumn", true)
    
    for _, obj in ipairs({main, left, middle, right}) do
        if obj and obj:IsA("GuiObject") then remember(obj) end
    end
    
    if State.CompactView then
        if main then main.BackgroundTransparency = 1 main.Active = false end
        if left then
            left.Visible = true
            left.AnchorPoint = Vector2.new(0,0)
            left.Position = UDim2.fromOffset(0,0)
            left.Size = UDim2.new(0,270,1,0)
        end
        if middle then middle.Visible = false middle.Active = false end
        if right then
            right.Visible = true
            right.AnchorPoint = Vector2.new(1,0)
            right.Position = UDim2.fromScale(1,0)
            right.Size = UDim2.new(0,300,1,0)
        end
    else
        for _, obj in ipairs({main, left, middle, right}) do
            if obj and obj:IsA("GuiObject") then restore(obj) end
        end
    end
end

--==================================================
-- INTERFACE (HUB)
--==================================================
local old = playerGui:FindFirstChild("CoinClickerV28")
if old then old:Destroy() end

local Gui = Instance.new("ScreenGui")
Gui.Name = "CoinClickerV28"
Gui.ResetOnSpawn = false
Gui.IgnoreGuiInset = true
Gui.DisplayOrder = 1000
Gui.Parent = playerGui

local Main = Instance.new("Frame")
Main.Size = UDim2.fromOffset(265, 330)
Main.Position = UDim2.new(0.5, -132, 0.5, -165)
Main.BackgroundColor3 = Color3.fromRGB(18,15,27)
Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true
Main.Parent = Gui
Instance.new("UICorner", Main).CornerRadius = UDim.new(0,12)

local Stroke = Instance.new("UIStroke")
Stroke.Thickness = 2
Stroke.Color = Color3.fromRGB(126,79,255)
Stroke.Parent = Main

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1,-44,0,34)
Title.Position = UDim2.fromOffset(12,4)
Title.BackgroundTransparency = 1
Title.Text = "CoinClicker v28"
Title.TextColor3 = Color3.new(1,1,1)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Main

local Close = Instance.new("TextButton")
Close.Size = UDim2.fromOffset(28,28)
Close.Position = UDim2.new(1,-34,0,5)
Close.BackgroundTransparency = 1
Close.Text = "×"
Close.TextColor3 = Color3.fromRGB(255,110,110)
Close.Font = Enum.Font.GothamBold
Close.TextSize = 20
Close.Parent = Main

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1,-24,0,18)
Status.Position = UDim2.fromOffset(12,38)
Status.BackgroundTransparency = 1
Status.Text = "Pronto"
Status.TextColor3 = Color3.fromRGB(150,225,165)
Status.Font = Enum.Font.GothamMedium
Status.TextSize = 10
Status.TextXAlignment = Enum.TextXAlignment.Left
Status.Parent = Main

local Holder = Instance.new("Frame")
Holder.Size = UDim2.new(1,-20,1,-66)
Holder.Position = UDim2.fromOffset(10,60)
Holder.BackgroundTransparency = 1
Holder.Parent = Main

local Layout = Instance.new("UIListLayout")
Layout.Padding = UDim.new(0,7)
Layout.Parent = Holder

local function toggle(label, key)
    local B = Instance.new("TextButton")
    B.Size = UDim2.new(1,0,0,36)
    B.BackgroundColor3 = Color3.fromRGB(31,28,43)
    B.BorderSizePixel = 0
    B.Text = ""
    B.Parent = Holder
    Instance.new("UICorner", B).CornerRadius = UDim.new(0,9)
    
    local T = Instance.new("TextLabel")
    T.Size = UDim2.new(1,-62,1,0)
    T.Position = UDim2.fromOffset(10,0)
    T.BackgroundTransparency = 1
    T.Text = label
    T.TextColor3 = Color3.new(1,1,1)
    T.Font = Enum.Font.GothamMedium
    T.TextSize = 12
    T.TextXAlignment = Enum.TextXAlignment.Left
    T.Parent = B
    
    local F = Instance.new("TextLabel")
    F.Size = UDim2.fromOffset(44,24)
    F.Position = UDim2.new(1,-52,0.5,-12)
    F.TextColor3 = Color3.new(1,1,1)
    F.Font = Enum.Font.GothamBold
    F.TextSize = 10
    F.Parent = B
    Instance.new("UICorner", F).CornerRadius = UDim.new(1,0)
    
    local function refresh()
        F.Text = State[key] and "ON" or "OFF"
        F.BackgroundColor3 = State[key] and Color3.fromRGB(126,79,255) or Color3.fromRGB(66,63,78)
    end
    
    B.Activated:Connect(function()
        State[key] = not State[key]
        refresh()
        if key == "AutoClick" and not State[key] then
            State.StaminaStart = 0
            Status.Text = "Pronto"
        end
        if key == "CompactView" then pcall(applyCompact) end
    end)
    refresh()
end

toggle("Auto Click", "AutoClick")
toggle("Auto Comprar Itens", "AutoItems")
toggle("Auto Buff / Upgrades", "AutoBuff")
toggle("Auto Wrinklers", "AutoWrinkler")
toggle("Auto Golden", "AutoGolden")
toggle("Compact View", "CompactView")

--==================================================
-- LOOP PRINCIPAL
--==================================================
local connection
connection = RunService.Heartbeat:Connect(function()
    if not State.Running then return end
    local now = os.clock()
    
    -- AUTOCLICK RÁPIDO & HUMANIZADO
    if State.AutoClick then
        if State.StaminaStart == 0 then
            State.StaminaStart = now
            State.TargetClickTime = math.random(48, 52)
            State.TargetRestTime = math.random(67, 73)
        end
        
        local totalCycle = State.TargetClickTime + State.TargetRestTime
        local elapsed = (now - State.StaminaStart) % totalCycle
        
        if elapsed < State.TargetClickTime then
            local remainingClick = math.ceil(State.TargetClickTime - elapsed)
            
            if now < State.MicroPauseEnd then
                Status.Text = "Clicando... (Pausa rápida)"
            else
                if math.random(1, 1000) <= 10 then
                    State.MicroPauseEnd = now + (math.random(15, 35) / 100)
                end
                
                if now - State.LastClick >= State.NextClickDelay then
                    State.LastClick = now
                    local baseDelay = math.random(18, 48) / 1000
                    local fatigue = (elapsed / State.TargetClickTime) * 0.012
                    State.NextClickDelay = baseDelay + fatigue
                    
                    local big = getBigCoin()
                    if big then
                        press(big, true)
                        Status.Text = "Clicando Rápido... (" .. remainingClick .. "s restantes)"
                    else
                        cachedBigCoin = nil
                        Status.Text = "Procurando moeda..."
                    end
                end
            end
        else
            local remainingRest = math.ceil(totalCycle - elapsed)
            Status.Text = "Stamina: Descansando (" .. remainingRest .. "s)"
            
            if remainingRest == 1 then
                State.TargetClickTime = math.random(48, 52)
                State.TargetRestTime = math.random(67, 73)
            end
        end
    end
    
    -- COMPRA DE ITENS (Atraso 1 a 9s)
    if State.AutoItems and now >= State.NextItemTime then
        State.NextItemTime = now + getRandomHumanDelay()
        buyBestGenerator()
    end
    
    -- COMPRA DE BUFFS (Atraso 1 a 9s)
    if State.AutoBuff and now >= State.NextBuffTime then
        State.NextBuffTime = now + getRandomHumanDelay()
        buyAvailableUpgrade()
    end
    
    -- ESTOURAR WRINKLERS (Atraso 1 a 9s)
    if State.AutoWrinkler and now >= State.NextWrinklerTime then
        State.NextWrinklerTime = now + getRandomHumanDelay()
        popWrinklers()
    end
    
    -- AUTO GOLDEN (Atraso de 1 a 4s + Clique Físico de Mouse)
    if State.AutoGolden and now >= State.NextGoldenTime then
        State.NextGoldenTime = now + getGoldenDelay()
        clickGoldens()
    end
    
    if now - State.LastCompact >= 0.4 then
        State.LastCompact = now
        pcall(applyCompact)
    end
end)

local function stop()
    State.Running = false
    State.CompactView = false
    pcall(applyCompact)
    if connection then connection:Disconnect() end
    if Gui then Gui:Destroy() end
end

Close.Activated:Connect(stop)
print("[CoinClicker v28] Auto Golden seguro + Mouse Clicks carregado.")

