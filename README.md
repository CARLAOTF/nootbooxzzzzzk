-- CoinClicker v33.3
-- Auto Buff com intervalo de 5 minutos

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualUser = game:GetService("VirtualUser")
local VirtualInputManager = game:GetService("VirtualInputManager")
local GuiService = game:GetService("GuiService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

--==================================================
-- ANTI-AFK
--==================================================

player.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)

--==================================================
-- STATE
--==================================================

local State = {
    Running = true,

    AutoClick = false,
    AutoItems = false,
    AutoBuff = false,
    AutoWrinkler = false,
    AutoGolden = false,
    AutoEquip = true,
    CompactView = false,

    LastClick = 0,
    LastCompact = 0,

    NextClickDelay = 0.02,
    MicroPauseEnd = 0,

    NextItemTime = 0,

    -- AUTO BUFF
    NextBuffTime = 0,
    BuffInterval = 300, -- 300 segundos = 5 minutos

    NextWrinklerTime = 0,
    NextGoldenTime = 0,
    NextEquipCheck = 0,
}

--==================================================
-- DELAYS
--==================================================

local function getRandomHumanDelay()
    return math.random(150, 600) / 1000
end

local function getGoldenDelay()
    return math.random(200, 500) / 1000
end

local function formatTime(seconds)
    seconds = math.max(0, math.floor(seconds))

    local minutes = math.floor(seconds / 60)
    local secs = seconds % 60

    return string.format("%02d:%02d", minutes, secs)
end

--==================================================
-- CACHE
--==================================================

local cachedBigCoin = nil
local FrameCache = {}

local function resetCoinCache()
    cachedBigCoin = nil
    FrameCache = {}
end

--==================================================
-- CLIQUE SEGURO
--==================================================

local function clickSafeScreenArea()
    task.wait(0.2)

    local camera = Workspace.CurrentCamera
    if not camera then
        return
    end

    local viewport = camera.ViewportSize

    local safeX = viewport.X / 2
    local safeY = viewport.Y * 0.15

    pcall(function()
        VirtualInputManager:SendMouseButtonEvent(
            safeX,
            safeY,
            0,
            true,
            game,
            0
        )

        task.wait(0.03)

        VirtualInputManager:SendMouseButtonEvent(
            safeX,
            safeY,
            0,
            false,
            game,
            0
        )
    end)
end

--==================================================
-- CHARACTER
--==================================================

local function setupCharacterListener(char)
    if not char then
        return
    end

    char.ChildAdded:Connect(function(child)
        if child:IsA("Tool") and child.Name == "CoinClicker" then
            resetCoinCache()
            task.spawn(clickSafeScreenArea)
        end
    end)

    char.ChildRemoved:Connect(function(child)
        if child:IsA("Tool") and child.Name == "CoinClicker" then
            resetCoinCache()
        end
    end)
end

if player.Character then
    setupCharacterListener(player.Character)
end

player.CharacterAdded:Connect(setupCharacterListener)

--==================================================
-- SUPORTE
--==================================================

local function parseNumber(text)
    if not text or text == "" then
        return 0
    end

    text = string.lower(text)

    local valStr, suffix =
        string.match(text, "(%d+%.?%d*)%s*([kmbt]?)")

    if not valStr then
        return 0
    end

    local num = tonumber(valStr) or 0

    if suffix == "k" then
        num *= 1e3
    elseif suffix == "m" then
        num *= 1e6
    elseif suffix == "b" then
        num *= 1e9
    elseif suffix == "t" then
        num *= 1e12
    end

    return num
end

local function getGui()
    return playerGui:FindFirstChild("CoinClickerGui")
        or playerGui:FindFirstChildWhichIsA("ScreenGui")
end

local function visible(obj)
    if not obj
        or not obj:IsA("GuiObject")
        or not obj.Visible
    then
        return false
    end

    local p = obj.Parent

    while p and p ~= playerGui do
        if p:IsA("GuiObject") and not p.Visible then
            return false
        end

        p = p.Parent
    end

    return true
end

local function press(button, allowHidden)
    if not button or not button:IsA("GuiButton") then
        return false
    end

    if not allowHidden and not visible(button) then
        return false
    end

    if type(firesignal) == "function" then
        local ok1 = pcall(firesignal, button.Activated)
        local ok2 = pcall(firesignal, button.MouseButton1Click)

        if ok1 or ok2 then
            return true
        end
    end

    return pcall(function()
        button:Activate()
    end)
end

local function mouseClickButton(button)
    if not button or not visible(button) then
        return false
    end

    local inset = GuiService:GetGuiInset()
    local pos = button.AbsolutePosition
    local size = button.AbsoluteSize

    local x =
        pos.X +
        (size.X / 2) +
        math.random(-2, 2)

    local y =
        pos.Y +
        (size.Y / 2) +
        inset.Y +
        math.random(-2, 2)

    local success = pcall(function()
        VirtualInputManager:SendMouseButtonEvent(
            x,
            y,
            0,
            true,
            game,
            0
        )

        task.wait(0.02)

        VirtualInputManager:SendMouseButtonEvent(
            x,
            y,
            0,
            false,
            game,
            0
        )
    end)

    if not success then
        return press(button)
    end

    return true
end

local function findFrame(name)
    local gui = getGui()

    if not gui then
        return nil
    end

    return gui:FindFirstChild(name, true)
end

local function getFrame(name)
    local cached = FrameCache[name]

    if cached and cached.Parent then
        return cached
    end

    cached = findFrame(name)
    FrameCache[name] = cached

    return cached
end

local function allButtons(root)
    local result = {}

    if not root then
        return result
    end

    if root:IsA("GuiButton") and visible(root) then
        table.insert(result, root)
    end

    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("GuiButton") and visible(obj) then
            table.insert(result, obj)
        end
    end

    return result
end

local function textOf(obj)
    local parts = {}

    if obj:IsA("TextButton") and obj.Text ~= "" then
        table.insert(parts, obj.Text)
    end

    for _, d in ipairs(obj:GetDescendants()) do
        if (d:IsA("TextLabel") or d:IsA("TextButton"))
            and d.Text ~= ""
        then
            table.insert(parts, d.Text)
        end
    end

    return string.lower(table.concat(parts, " "))
end

local function greenStroke(obj)
    local function green(c)
        return c.G > c.R * 1.05
            and c.G > c.B * 1.02
            and c.G > 0.35
    end

    for _, d in ipairs(obj:GetDescendants()) do
        if d:IsA("UIStroke")
            and d.Enabled
            and green(d.Color)
        then
            return true
        end
    end

    local p = obj.Parent

    if p then
        for _, d in ipairs(p:GetChildren()) do
            if d:IsA("UIStroke")
                and d.Enabled
                and green(d.Color)
            then
                return true
            end
        end
    end

    return false
end

--==================================================
-- MOEDA
--==================================================

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

    if not gui then
        return nil
    end

    local middle =
        gui:FindFirstChild("MiddleColumn", true)

    if middle then
        for _, obj in ipairs(middle:GetDescendants()) do
            if obj:IsA("GuiButton")
                and visible(obj)
                and nameLooksLikeCoin(obj)
            then
                return obj
            end
        end
    end

    for _, obj in ipairs(gui:GetDescendants()) do
        if obj:IsA("GuiButton")
            and visible(obj)
            and nameLooksLikeCoin(obj)
            and not underBlockedSection(obj)
        then
            return obj
        end
    end

    local root = middle or gui

    local best = nil
    local bestArea = 0

    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("GuiButton")
            and visible(obj)
            and not underBlockedSection(obj)
        then
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
    if cachedBigCoin
        and cachedBigCoin.Parent
        and cachedBigCoin:IsDescendantOf(playerGui)
        and visible(cachedBigCoin)
    then
        return cachedBigCoin
    end

    cachedBigCoin = findBigCoin()

    return cachedBigCoin
end

local function clickSmallCoins()
    local gui = getGui()

    if not gui then
        return
    end

    local middle =
        gui:FindFirstChild("MiddleColumn", true)
        or gui

    local big = getBigCoin()

    for _, obj in ipairs(middle:GetDescendants()) do
        if obj:IsA("GuiButton")
            and obj ~= big
            and visible(obj)
            and not underBlockedSection(obj)
        then
            local name = string.lower(obj.Name)

            if string.find(name, "coin")
                or string.find(name, "small")
                or string.find(name, "mini")
                or string.find(name, "spawn")
                or string.find(name, "bonus")
                or obj.AbsoluteSize.X < 140
            then
                press(obj, false)
            end
        end
    end
end

--==================================================
-- AUTO EQUIP
--==================================================

local function autoEquipTool()
    local char = player.Character

    if not char then
        return
    end

    if char:FindFirstChild("CoinClicker") then
        return
    end

    local backpack = player:FindFirstChild("Backpack")

    if backpack then
        local tool = backpack:FindFirstChild("CoinClicker")

        if tool then
            local humanoid =
                char:FindFirstChildOfClass("Humanoid")

            if humanoid then
                humanoid:EquipTool(tool)
            end
        end
    end
end

--==================================================
-- WRINKLERS
--==================================================

local function popWrinklers()
    local gui = getGui()

    if not gui then
        return 0
    end

    local count = 0

    local wrinklersFrame =
        getFrame("Wrinklers")
        or gui:FindFirstChild("Wrinklers", true)

    local targetList =
        wrinklersFrame
        and allButtons(wrinklersFrame)
        or {}

    if #targetList == 0 then
        for _, obj in ipairs(gui:GetDescendants()) do
            if obj:IsA("GuiButton")
                and visible(obj)
            then
                local n = string.lower(obj.Name)

                local pN =
                    obj.Parent
                    and string.lower(obj.Parent.Name)
                    or ""

                if string.find(n, "wrinkler")
                    or string.find(pN, "wrinkler")
                    or string.find(n, "wrs")
                then
                    table.insert(targetList, obj)
                end
            end
        end
    end

    for _, button in ipairs(targetList) do
        if mouseClickButton(button) or press(button) then
            count += 1
        end
    end

    return count
end

--==================================================
-- BUFF / UPGRADES
--==================================================

local function buyAvailableUpgrade()
    local upgrades =
        getFrame("Upgrades")
        or getFrame("Buffs")

    local gui = getGui()

    if not upgrades and gui then
        upgrades =
            gui:FindFirstChild("Upgrades", true)
            or gui:FindFirstChild("Buffs", true)
    end

    if not upgrades then
        return false
    end

    local candidates = allButtons(upgrades)

    table.sort(candidates, function(a, b)
        return a.AbsolutePosition.X < b.AbsolutePosition.X
    end)

    for _, button in ipairs(candidates) do
        local txt = textOf(button)

        if not string.find(txt, "robux", 1, true) then
            if greenStroke(button) or visible(button) then
                if press(button) or mouseClickButton(button) then
                    return true
                end
            end
        end
    end

    return false
end

--==================================================
-- GENERATORS
--==================================================

local function buyBestGenerator()
    local generators = getFrame("Generators")

    if not generators then
        return false
    end

    local candidates = {}

    for _, button in ipairs(allButtons(generators)) do
        local s = button.AbsoluteSize

        if s.X >= 100 and s.Y >= 25 then
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
        if a.canBuy ~= b.canBuy then
            return a.canBuy
        end

        if a.price > 0
            and b.price > 0
            and a.price ~= b.price
        then
            return a.price > b.price
        end

        return a.posY > b.posY
    end)

    for _, item in ipairs(candidates) do
        if item.canBuy
            and (
                press(item.button)
                or mouseClickButton(item.button)
            )
        then
            return true
        end
    end

    return false
end

--==================================================
-- GOLDEN
--==================================================

local function clickGoldens()
    local gui = getGui()

    if not gui then
        return 0
    end

    local goldens =
        getFrame("Goldens")
        or gui:FindFirstChild("Goldens", true)

    if not goldens then
        return 0
    end

    local count = 0

    for _, button in ipairs(allButtons(goldens)) do
        if mouseClickButton(button) or press(button) then
            count += 1
        end
    end

    return count
end

--==================================================
-- COMPACT VIEW
--==================================================

local saved =
    setmetatable({}, {__mode = "k"})

local function remember(obj)
    if not obj
        or not obj:IsA("GuiObject")
        or saved[obj]
    then
        return
    end

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

    if not old
        or not obj
        or not obj.Parent
    then
        return
    end

    obj.Visible = old.Visible
    obj.Size = old.Size
    obj.Position = old.Position
    obj.AnchorPoint = old.AnchorPoint
    obj.BackgroundTransparency = old.BackgroundTransparency
    obj.Active = old.Active
end

local function applyCompact()
    local gui = getGui()

    if not gui then
        return
    end

    local main =
        gui:FindFirstChild("Main", true)

    local left =
        gui:FindFirstChild("LeftColumn", true)

    local middle =
        gui:FindFirstChild("MiddleColumn", true)

    local right =
        gui:FindFirstChild("RightColumn", true)

    for _, obj in ipairs({
        main,
        left,
        middle,
        right
    }) do
        if obj and obj:IsA("GuiObject") then
            remember(obj)
        end
    end

    if State.CompactView then
        if main then
            main.BackgroundTransparency = 1
            main.Active = false
        end

        if left then
            left.Visible = true
            left.AnchorPoint = Vector2.new(0, 0)
            left.Position = UDim2.fromOffset(0, 0)
            left.Size = UDim2.new(0, 270, 1, 0)
        end

        if middle then
            middle.Visible = false
            middle.Active = false
        end

        if right then
            right.Visible = true
            right.AnchorPoint = Vector2.new(1, 0)
            right.Position = UDim2.fromScale(1, 0)
            right.Size = UDim2.new(0, 300, 1, 0)
        end
    else
        for _, obj in ipairs({
            main,
            left,
            middle,
            right
        }) do
            if obj and obj:IsA("GuiObject") then
                restore(obj)
            end
        end
    end
end

--==================================================
-- HUB
--==================================================

local old =
    playerGui:FindFirstChild("CoinClickerV33")

if old then
    old:Destroy()
end

local Gui = Instance.new("ScreenGui")
Gui.Name = "CoinClickerV33"
Gui.ResetOnSpawn = false
Gui.IgnoreGuiInset = true
Gui.DisplayOrder = 1000
Gui.Parent = playerGui

local Main = Instance.new("Frame")
Main.Size = UDim2.fromOffset(265, 375)
Main.Position =
    UDim2.new(0.5, -132, 0.5, -187)

Main.BackgroundColor3 =
    Color3.fromRGB(18, 15, 27)

Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true
Main.ClipsDescendants = true
Main.Parent = Gui

Instance.new("UICorner", Main).CornerRadius =
    UDim.new(0, 12)

local Stroke = Instance.new("UIStroke")
Stroke.Thickness = 2
Stroke.Color = Color3.fromRGB(126, 79, 255)
Stroke.Parent = Main

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -74, 0, 34)
Title.Position = UDim2.fromOffset(12, 4)
Title.BackgroundTransparency = 1
Title.Text = "CoinClicker v33.3"
Title.TextColor3 = Color3.new(1, 1, 1)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Main

local Close = Instance.new("TextButton")
Close.Size = UDim2.fromOffset(28, 28)
Close.Position = UDim2.new(1, -34, 0, 5)
Close.BackgroundTransparency = 1
Close.Text = "×"
Close.TextColor3 = Color3.fromRGB(255, 110, 110)
Close.Font = Enum.Font.GothamBold
Close.TextSize = 20
Close.Parent = Main

local Minimize = Instance.new("TextButton")
Minimize.Size = UDim2.fromOffset(28, 28)
Minimize.Position = UDim2.new(1, -64, 0, 5)
Minimize.BackgroundTransparency = 1
Minimize.Text = "-"
Minimize.TextColor3 = Color3.new(1, 1, 1)
Minimize.Font = Enum.Font.GothamBold
Minimize.TextSize = 24
Minimize.Parent = Main

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, -24, 0, 18)
Status.Position = UDim2.fromOffset(12, 38)
Status.BackgroundTransparency = 1
Status.Text = "Pronto"
Status.TextColor3 = Color3.fromRGB(150, 225, 165)
Status.Font = Enum.Font.GothamMedium
Status.TextSize = 10
Status.TextXAlignment = Enum.TextXAlignment.Left
Status.Parent = Main

local Holder = Instance.new("Frame")
Holder.Size = UDim2.new(1, -20, 1, -66)
Holder.Position = UDim2.fromOffset(10, 60)
Holder.BackgroundTransparency = 1
Holder.Parent = Main

local Layout = Instance.new("UIListLayout")
Layout.Padding = UDim.new(0, 7)
Layout.Parent = Holder

--==================================================
-- MINIMIZAR
--==================================================

local isMinimized = false

Minimize.Activated:Connect(function()
    isMinimized = not isMinimized

    if isMinimized then
        Main.Size = UDim2.fromOffset(265, 42)
        Minimize.Text = "+"
    else
        Main.Size = UDim2.fromOffset(265, 375)
        Minimize.Text = "-"
    end
end)

--==================================================
-- TOGGLE
--==================================================

local function toggle(label, key)
    local B = Instance.new("TextButton")

    B.Size = UDim2.new(1, 0, 0, 36)
    B.BackgroundColor3 =
        Color3.fromRGB(31, 28, 43)

    B.BorderSizePixel = 0
    B.Text = ""
    B.Parent = Holder

    Instance.new("UICorner", B).CornerRadius =
        UDim.new(0, 9)

    local T = Instance.new("TextLabel")
    T.Size = UDim2.new(1, -62, 1, 0)
    T.Position = UDim2.fromOffset(10, 0)
    T.BackgroundTransparency = 1
    T.Text = label
    T.TextColor3 = Color3.new(1, 1, 1)
    T.Font = Enum.Font.GothamMedium
    T.TextSize = 12
    T.TextXAlignment = Enum.TextXAlignment.Left
    T.Parent = B

    local F = Instance.new("TextLabel")
    F.Size = UDim2.fromOffset(44, 24)
    F.Position = UDim2.new(1, -52, 0.5, -12)
    F.TextColor3 = Color3.new(1, 1, 1)
    F.Font = Enum.Font.GothamBold
    F.TextSize = 10
    F.Parent = B

    Instance.new("UICorner", F).CornerRadius =
        UDim.new(1, 0)

    local function refresh()
        F.Text = State[key] and "ON" or "OFF"

        F.BackgroundColor3 =
            State[key]
            and Color3.fromRGB(126, 79, 255)
            or Color3.fromRGB(66, 63, 78)
    end

    B.Activated:Connect(function()
        State[key] = not State[key]

        refresh()

        if key == "AutoClick"
            and not State[key]
        then
            Status.Text = "Pronto"
        end

        --==========================================
        -- AUTO BUFF
        --==========================================

        if key == "AutoBuff" then
            State.NextBuffTime = 0

            if State.AutoBuff then
                Status.Text =
                    "Buff ativado • aplicando..."
            else
                Status.Text =
                    "Buff automático OFF"
            end
        end

        if key == "CompactView" then
            pcall(applyCompact)
        end
    end)

    refresh()
end

toggle("Auto Click", "AutoClick")
toggle("Auto Comprar Itens", "AutoItems")
toggle("Auto Buff / Upgrades", "AutoBuff")
toggle("Auto Wrinklers", "AutoWrinkler")
toggle("Auto Golden", "AutoGolden")
toggle("Auto Equip Item", "AutoEquip")
toggle("Compact View", "CompactView")

--==================================================
-- LOOP PRINCIPAL
--==================================================

local connection

connection = RunService.Heartbeat:Connect(function()
    if not State.Running then
        return
    end

    local now = os.clock()

    --==============================================
    -- AUTO EQUIP
    --==============================================

    if (State.AutoClick or State.AutoEquip)
        and now >= State.NextEquipCheck
    then
        State.NextEquipCheck = now + 0.5
        autoEquipTool()
    end

    --==============================================
    -- AUTO CLICK
    --==============================================

    if State.AutoClick then
        if now < State.MicroPauseEnd then

            Status.Text =
                "Clicando... (Micro Pausa)"

        else

            if math.random(1, 100) <= 1 then
                State.MicroPauseEnd =
                    now +
                    (math.random(10, 30) / 100)
            end

            if now - State.LastClick
                >= State.NextClickDelay
            then

                State.LastClick = now

                State.NextClickDelay =
                    math.random(15, 35) / 1000

                local big = getBigCoin()

                if big then

                    if not press(big, true) then
                        mouseClickButton(big)
                    end

                    clickSmallCoins()

                    Status.Text =
                        "Ativo (Clicando...)"

                else

                    resetCoinCache()

                    Status.Text =
                        "Aguardando Moeda..."
                end
            end
        end
    end

    --==============================================
    -- AUTO ITEMS
    --==============================================

    if State.AutoItems
        and now >= State.NextItemTime
    then
        State.NextItemTime =
            now + getRandomHumanDelay()

        buyBestGenerator()
    end

    --==============================================
    -- AUTO BUFF
    -- 5 MINUTOS
    --==============================================

    if State.AutoBuff then

        if State.NextBuffTime <= 0 then

            local success =
                pcall(buyAvailableUpgrade)

            if success then
                State.NextBuffTime =
                    now + State.BuffInterval

                Status.Text =
                    "Buff aplicado • próximo em 05:00"
            else
                State.NextBuffTime =
                    now + 5

                Status.Text =
                    "Buff não encontrado • tentando..."
            end

        elseif now >= State.NextBuffTime then

            local success =
                pcall(buyAvailableUpgrade)

            if success then
                State.NextBuffTime =
                    now + State.BuffInterval

                Status.Text =
                    "Buff aplicado • próximo em 05:00"
            else
                State.NextBuffTime =
                    now + 5

                Status.Text =
                    "Buff não encontrado • tentando..."
            end

        else

            local remaining =
                State.NextBuffTime - now

            Status.Text =
                "Buff • próximo em "
                .. formatTime(remaining)
        end
    end

    --==============================================
    -- AUTO WRINKLER
    --==============================================

    if State.AutoWrinkler
        and now >= State.NextWrinklerTime
    then
        State.NextWrinklerTime =
            now + getRandomHumanDelay()

        popWrinklers()
    end

    --==============================================
    -- AUTO GOLDEN
    --==============================================

    if State.AutoGolden
        and now >= State.NextGoldenTime
    then
        State.NextGoldenTime =
            now + getGoldenDelay()

        clickGoldens()
    end

    --==============================================
    -- COMPACT
    --==============================================

    if now - State.LastCompact >= 0.4 then
        State.LastCompact = now
        pcall(applyCompact)
    end
end)

--==================================================
-- STOP
--==================================================

local function stop()
    State.Running = false
    State.CompactView = false

    pcall(applyCompact)

    if connection then
        connection:Disconnect()
        connection = nil
    end

    if Gui then
        Gui:Destroy()
    end
end

Close.Activated:Connect(stop)

print("================================")
print(" CoinClicker v33.3")
print(" Auto Buff: 5 minutos")
print("================================")
