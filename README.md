local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local RS = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer
local pg = player:WaitForChild("PlayerGui", 10)
assert(pg, "PlayerGui indisponível")
assert(type(getconnections) == "function" and type(debug.getupvalues) == "function", "Potassium sem introspecção de conexões")

local feature = RS:WaitForChild("Features", 10)
feature = feature and feature:FindFirstChild("CoinClicker")
assert(feature, "Coin Clicker não encontrado")

local Catalog = require(feature.Catalog)
local Upgrades = require(feature.Upgrades)
local Fortunes = require(feature.Fortunes)
local Rules = require(feature.Rules)
local Achievements = require(feature.Achievements)
local Engine = require(feature.Engine)
local env = getgenv()

local function discover()
    local root = pg:FindFirstChild("CoinClickerGui")
    local main = root and root:FindFirstChild("Main")
    local body = main and main:FindFirstChild("Body")
    local right = body and body:FindFirstChild("RightColumn")
    local generators = right and right:FindFirstChild("Generators")
    local list = generators and generators:FindFirstChild("StoreGeneratorList")
    local row = list and list:FindFirstChild("StoreGeneratorRow")
    if row then
        for _, c in ipairs(getconnections(row.Activated)) do
            if type(c.Function) == "function" then
                for _, v in pairs(debug.getupvalues(c.Function)) do
                    if type(v) == "table" and type(v.buy) == "function" and type(v.click) == "function" and type(v.ready) == "function" then
                        return v, root
                    end
                end
            end
        end
    end
    return nil, root
end

if env.JaxCoinHub and type(env.JaxCoinHub.Stop)=="function" then env.JaxCoinHub.Stop() end

local vide=require(RS.Packages.vide)
local SettingsFactory=require(feature.Settings)
local ServerBackend=require(feature.Drivers.ServerBackend)

local function createBackend()
    local settingsInstance=SettingsFactory.new()
    local destroy,backend=vide.root(function()
        return ServerBackend({
            settings=settingsInstance, coinsPerClick=vide.source(1), timeScale=vide.source(1),
            particlesPerClick=settingsInstance.particles, goldenInterval=vide.source(20),
            wrinklerInterval=vide.source(25), achievementToasts=settingsInstance.achievementToasts,
            floatingNumbers=settingsInstance.floatingNumbers, clickSound=settingsInstance.clickSound,
            backgroundCoins=settingsInstance.backgroundCoins,
        })
    end)
    assert(type(destroy)=="function" and type(backend)=="table","Falha ao iniciar o backend do jogo")
    return backend,destroy,settingsInstance
end

local driver,privateDestroy,privateSettings=createBackend()
local root=pg:FindFirstChild("CoinClickerGui")

if env.JaxCoinFarm and type(env.JaxCoinFarm.Stop) == "function" then env.JaxCoinFarm.Stop() end
if env.JaxHubSyncProbe then env.JaxHubSyncProbe:Disconnect() env.JaxHubSyncProbe = nil end

-- CONFIGURAÇÕES DA FARM
local C = {
    AutoClick = false,         -- Desativado por padrão
    CompactView = false,       -- Modo Compacto (extraído do v32)
    Performance = false,       -- Desativado por padrão
    AutoBuy = false,           -- Desativado por padrão
    AutoUpgrade = false,       -- Desativado por padrão
    SaveMode = false,          -- Desativado por padrão
    
    AutoExtraUI = true, SmartStamina = false, Reserve = 0, Strategy = "Lucro",
    MaxSaveSeconds = 900, SaveAdvantage = 1.08,
    AutoGolden = true, AutoLump = true, AutoMarket = true, AutoWrinkler = true,
    WrinklerInterval = 30, AutoReward = true, AutoSpell = true, Spell = "auto",
    Paused = false, Verbose = false,
}

local hub = {
    Running = true, Version = "2.5 SMART BUYER + AUTOCLICK", Started = os.clock(), ServerRate = 0, ServerIncome = 0, History = {}, Pending = {}, Confirmed = 0, Rejected = 0,
    Counts = {ExtraUiClicks = 0, Buildings = 0, Upgrades = 0, Goldens = 0, Harvests = 0, Rares = 0, Wrinklers = 0, Spells = 0, Clicks = 0},
    Logs = {}, Connections = {}, Controls = {}, LastSync = nil, SyncCount = 0,
}
env.JaxCoinHub = hub
env.JaxHubBridge = nil

local lastRoot = root
local boundRow=root and root:FindFirstChild("StoreGeneratorRow",true)
local failures = {}

--==================================================
-- LÓGICA DE COMPACTAR A UI (DO SEGUNDO SCRIPT)
--==================================================
local savedCompact = setmetatable({}, {__mode = "k"})

local function rememberCompact(obj)
    if not obj or not obj:IsA("GuiObject") or savedCompact[obj] then return end
    savedCompact[obj] = {
        Visible = obj.Visible,
        Size = obj.Size,
        Position = obj.Position,
        AnchorPoint = obj.AnchorPoint,
        BackgroundTransparency = obj.BackgroundTransparency,
        Active = obj.Active,
    }
end

local function restoreCompact(obj)
    local old = savedCompact[obj]
    if not old or not obj or not obj.Parent then return end
    obj.Visible = old.Visible
    obj.Size = old.Size
    obj.Position = old.Position
    obj.AnchorPoint = old.AnchorPoint
    obj.BackgroundTransparency = old.BackgroundTransparency
    obj.Active = old.Active
end

local function applyCompact()
    local coinGui = pg:FindFirstChild("CoinClickerGui")
    if not coinGui then return end
    
    local main = coinGui:FindFirstChild("Main", true)
    local left = coinGui:FindFirstChild("LeftColumn", true)
    local middle = coinGui:FindFirstChild("MiddleColumn", true)
    local right = coinGui:FindFirstChild("RightColumn", true)
    
    for _, obj in ipairs({main, left, middle, right}) do
        if obj and obj:IsA("GuiObject") then rememberCompact(obj) end
    end
    
    if C.CompactView then
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
            if obj and obj:IsA("GuiObject") then restoreCompact(obj) end
        end
    end
end

local function log(message, isError)
    local stamp = string.format("%02d:%02d", math.floor((os.clock()-hub.Started)/60), math.floor((os.clock()-hub.Started)%60))
    table.insert(hub.Logs, 1, stamp.."  "..message)
    if #hub.Logs > 24 then table.remove(hub.Logs) end
    if isError then warn("[27K HUB] "..message) else print("[27K HUB] "..message) end
end

local function connect(signal, fn)
    local c = signal:Connect(fn)
    table.insert(hub.Connections, c)
    return c
end

local function attempt(name, fn)
    local ok, a, b = pcall(fn)
    if not ok then
        failures[name] = (failures[name] or 0) + 1
        if failures[name] == 1 then log(name..": "..tostring(a):sub(1,170), true) end
        return false, a
    end
    failures[name] = 0
    return true, a, b
end

local function ready()
    return driver~=nil and driver.ready()
end

local function fmt(n)
    n = tonumber(n) or 0
    if n ~= n then return "0" end
    if n == math.huge then return "∞" end
    local suffix = {"", "K", "M", "B", "T", "Qa", "Qi", "Sx"}
    local k = 1
    local x = math.abs(n)
    while x >= 1000 and k < #suffix do x = x / 1000 k = k + 1 end
    if k == #suffix and x >= 1000 then return string.format("%.2e", n) end
    local value = n < 0 and -x or x
    return k == 1 and string.format("%.1f",value):gsub("%.0$","") or string.format("%.2f%s",value,suffix[k])
end

--==================================================
-- LÓGICA DE DETECÇÃO DE BOTÃO DA MOEDA E AUTOCLICK
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
    while p and p ~= pg do
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
    local rootGui = pg:FindFirstChild("CoinClickerGui")
    if not rootGui then return nil end

    local middle = rootGui:FindFirstChild("MiddleColumn", true)
    if middle then
        for _, obj in ipairs(middle:GetDescendants()) do
            if obj:IsA("GuiButton") and nameLooksLikeCoin(obj) then return obj end
        end
    end

    for _, obj in ipairs(rootGui:GetDescendants()) do
        if obj:IsA("GuiButton") and nameLooksLikeCoin(obj) and not underBlockedSection(obj) then
            return obj
        end
    end
    return nil
end

local function getBigCoin()
    if cachedBigCoin and cachedBigCoin.Parent and cachedBigCoin:IsDescendantOf(pg) then
        return cachedBigCoin
    end
    cachedBigCoin = findBigCoin()
    return cachedBigCoin
end

local function press(button)
    if not button or not button:IsA("GuiButton") then return false end
    if type(firesignal) == "function" then
        local ok1 = pcall(firesignal, button.Activated)
        local ok2 = pcall(firesignal, button.MouseButton1Click)
        return ok1 or ok2
    end
    return pcall(function() button:Activate() end)
end

local function performHumanClick()
    if driver and type(driver.click) == "function" then
        pcall(driver.click)
    end
    local big = getBigCoin()
    if big then
        press(big)
    end
    hub.Counts.Clicks = hub.Counts.Clicks + 1
end

local clickState = {
    LastClick = 0,
    StaminaStart = 0,
    TargetClickTime = 50,
    TargetRestTime = 70,
    NextClickDelay = 0.02,
    MicroPauseEnd = 0,
}

local function findIndex(id)
    for i,g in ipairs(Catalog.all) do if g.id == id then return i end end
end

local translations = {
    Finger="Dedo", StickyNote="Nota pegajosa", Wallet="Carteira", Coffee="Café gelado",
    PlantPot="Vaso de plantas", SignBoard="Placa", Pizza="Festa da pizza", Boombox="Boombox",
    Dumbbell="Ginásio", Vacuum="Aspirador", Megaphone="Microfone", CrystalBall="Bola de cristal",
    Compass="Bússola", Camera="Cabine de fotos",
}

local function genName(g) return translations[g.id] or g.name end

local function oneBuy(id, rare)
    if not ready() then return false end
    local index = findIndex(id)
    if not index or driver.coins() < driver.priceAt(index) then return false end
    local amount, selling = driver.buyAmount(), driver.sellMode()
    local target=driver.ownedAt(index)+1
    if rare then
        local result=driver.marketBuy(id)
        if result then table.insert(hub.Pending,{Kind="building",Id=id,Target=target,At=os.clock(),Sync=hub.SyncCount}) end
        return result
    end
    driver.buyAmount(1)
    driver.sellMode(false)
    local ok, result = pcall(driver.buy, id)
    driver.buyAmount(amount)
    driver.sellMode(selling)
    if not ok then error(result) end
    if result==true then table.insert(hub.Pending,{Kind="building",Id=id,Target=target,At=os.clock(),Sync=hub.SyncCount}) end
    return result==true
end

local function availableUpgrades()
    local result = {}
    for _,u in ipairs(Upgrades.list) do
        if not driver.upgradePurchased(u.id) and driver.upgradeUnlocked(u) then table.insert(result,u) end
    end
    table.sort(result,function(a,b) return a.price < b.price end)
    return result
end

local viewFn,settings,savedEffects
local plans,lastPlanAt,lastPerformance={},0,nil

local function bindBackend()
    boundRow=root and root:FindFirstChild("StoreGeneratorRow",true)
    viewFn=nil
    if driver then
        for _,v in pairs(debug.getupvalues(driver.buy)) do
            if type(v)=="table" and type(v.hydrate)=="function" and type(v.buy)=="function" then
                for _,fn in pairs(debug.getupvalues(v.buy)) do
                    if type(fn)=="function" and debug.info(fn,"n")=="view" then viewFn=fn break end
                end
            end
        end
    end
    local prior=settings
    settings=privateSettings
    if settings~=prior then savedEffects=settings and settings.snapshot() or nil lastPerformance=nil end
    lastPlanAt=0
end

local function applyPerformance()
    if not settings or lastPerformance==C.Performance then return end
    local nextSettings=settings.snapshot()
    local baseline=savedEffects or nextSettings
    nextSettings.particles=C.Performance and 0 or baseline.particles
    nextSettings.backgroundCoins=C.Performance and 0 or baseline.backgroundCoins
    nextSettings.floatingNumbers=not C.Performance and baseline.floatingNumbers or false
    nextSettings.clickSound=not C.Performance and baseline.clickSound or false
    settings.apply(nextSettings)
    lastPerformance=C.Performance
    log(C.Performance and "Desempenho: partículas, números e som desligados" or "Efeitos do jogo restaurados")
end

local function putUpgrade(u)
    if not driver.upgradePurchased(u.id) and driver.upgradeUnlocked(u) and driver.coins()>=u.price and driver.buyUpgrade(u.id) then
        table.insert(hub.Pending,{Kind="upgrade",Id=u.id,At=os.clock(),Sync=hub.SyncCount})
        hub.Counts.Upgrades=hub.Counts.Upgrades+1
        log("Melhoria enviada: "..u.name)
        lastPlanAt=0
        return true
    end
    return false
end

local function planPurchases(force)
    if not ready() or not viewFn then plans={} return plans end
    local now=os.clock()
    if not force and now-lastPlanAt<0.45 then return plans end
    lastPlanAt=now
    local state=viewFn()
    local rate=0
    local _,clickMult=Engine.multipliers(state)
    local summary=Engine.summary(state)
    local baseClick=math.max(0,(driver.clickValue()/math.max(clickMult,0.000001)-Engine.baseCps(state)*summary.clickCpsShare)/2^summary.clickTiers)
    local baseIncome=Engine.coinsPerSecond(state)+Engine.clickValue(state,baseClick)*rate
    local result,stock={},{}
    if C.AutoMarket then for _,v in ipairs(driver.marketStock()) do stock[v.id]=v.left end end
    local function evaluate(kind,id,name,cost,mutate,unowned)
        local candidate=table.clone(state)
        candidate.owned=table.clone(state.owned)
        candidate.purchased=table.clone(state.purchased)
        mutate(candidate)
        local gain=Engine.coinsPerSecond(candidate)+Engine.clickValue(candidate,baseClick)*rate-baseIncome
        local roi=gain>0.000001 and cost/gain or math.huge
        table.insert(result,{Kind=kind,Id=id,Name=name,Cost=cost,Gain=gain,ROI=roi,Unowned=unowned})
    end
    for i,g in ipairs(Catalog.all) do
        if (not g.rare and C.AutoBuy) or (g.rare and C.AutoMarket and (stock[g.id] or 0)>0) then
            evaluate(g.rare and "rare" or "building",g.id,genName(g),driver.priceAt(i),function(v) v.owned[g.id]=(v.owned[g.id] or 0)+1 end,driver.ownedAt(i)==0)
        end
    end
    if C.AutoUpgrade then
        for _,u in ipairs(availableUpgrades()) do evaluate("upgrade",u.id,u.name,u.price,function(v) v.purchased[u.id]=true end,false) end
    end
    table.sort(result,function(a,b)
        if C.Strategy=="Mais barato" and a.Cost~=b.Cost then return a.Cost<b.Cost end
        if C.Strategy=="Desbloquear" and a.Unowned~=b.Unowned then return a.Unowned end
        if a.ROI~=b.ROI then return a.ROI<b.ROI end
        return a.Cost<b.Cost
    end)
    plans=result
    return result
end

local lastSavingsKey=nil
local function purchaseDecision(force)
    local list=planPurchases(force)
    local target=list[1]
    if not target then return nil,nil,"idle",nil end
    local funds=math.max(0,driver.coins()-(tonumber(C.Reserve) or 0))
    local affordable=nil
    for _,p in ipairs(list) do
        if p.Cost<=funds then affordable=p break end
    end
    local liveIncome=math.max(0.001,tonumber(hub.ServerIncome) or 0,driver.coinsPerSecond())
    if target.Cost<=funds then
        return target,target,"buy",{Funds=funds,Wait=0,AfterWait=0,Advantage=1,LiveIncome=liveIncome}
    end
    local waitNow=math.max(0,(target.Cost-funds)/liveIncome)
    if not affordable then
        return nil,target,"save",{Funds=funds,Wait=waitNow,AfterWait=math.huge,Advantage=math.huge,LiveIncome=liveIncome}
    end
    local afterIncome=liveIncome+math.max(0,affordable.Gain)
    local waitAfter=math.max(0,(target.Cost-(funds-affordable.Cost))/math.max(0.001,afterIncome))
    local advantage=affordable.ROI/math.max(0.000001,target.ROI)
    local returnBeforeTarget=affordable.ROI<=waitNow*0.90
    local delaysTarget=waitAfter>waitNow+0.05
    local roiMode=C.Strategy=="Lucro" or C.Strategy=="Retorno"
    local shouldSave=C.SaveMode and roiMode and waitNow<=C.MaxSaveSeconds and advantage>=C.SaveAdvantage and delaysTarget and not returnBeforeTarget
    local info={Funds=funds,Wait=waitNow,AfterWait=waitAfter,Advantage=advantage,LiveIncome=liveIncome,Alternative=affordable,ReturnBeforeTarget=returnBeforeTarget}
    if shouldSave then return nil,target,"save",info end
    return affordable,target,"buy_alternative",info
end

local function executePlan()
    local p,target,action,info=purchaseDecision(true)
    hub.PurchaseDecision={Buy=p,Target=target,Action=action,Info=info,At=os.clock()}
    if not p then
        if action=="save" and target then
            local key=target.Kind..":"..target.Id
            if key~=lastSavingsKey then
                lastSavingsKey=key
                local advantage=info and info.Advantage or 0
                local extra=advantage<math.huge and (" • "..string.format("%.2fx",advantage).." mais eficiente") or ""
                log("Guardando para "..target.Name.." • faltam "..fmt(math.max(0,target.Cost-(info and info.Funds or 0))).." • "..math.ceil(info and info.Wait or 0).."s"..extra)
            end
        end
        return
    end
    lastSavingsKey=nil
    if p.Kind=="upgrade" then putUpgrade(Upgrades.byId[p.Id])
    elseif oneBuy(p.Id,p.Kind=="rare") then
        if p.Kind=="rare" then hub.Counts.Rares=hub.Counts.Rares+1 else hub.Counts.Buildings=hub.Counts.Buildings+1 end
        log("Compra inteligente: "..p.Name.." • +"..fmt(p.Gain).."/s • retorno "..fmt(p.ROI).."s")
        lastPlanAt=0
    end
end

local function selectedSpell()
    if C.Spell~="auto" then return Fortunes.byId[C.Spell] end
    for _,id in ipairs({"handOfFate","sleightOfHand","packedHouse","conjure"}) do
        local spell=Fortunes.byId[id]
        if driver.spellReady(spell) and driver.mana()>=driver.spellCost(spell) then return spell end
    end
end

hub.PlanPurchases=planPurchases
hub.DecidePurchase=purchaseDecision
hub.Ready=ready
bindBackend()

-- INTERFACE GRÁFICA
local colors = {
    Base=Color3.fromRGB(12,17,24), Side=Color3.fromRGB(17,24,33), Card=Color3.fromRGB(23,33,45),
    Border=Color3.fromRGB(43,58,75), Text=Color3.fromRGB(240,242,250), Muted=Color3.fromRGB(151,161,185),
    Accent=Color3.fromRGB(27,139,216), Green=Color3.fromRGB(93,221,163), Danger=Color3.fromRGB(241,111,127),
}

local function new(class, parent, props)
    local item = Instance.new(class)
    for k,v in pairs(props or {}) do item[k]=v end
    item.Parent=parent
    return item
end

local function corner(parent, radius) new("UICorner",parent,{CornerRadius=UDim.new(0,radius or 8)}) end
local function stroke(parent) new("UIStroke",parent,{Color=colors.Border,Thickness=1,ApplyStrokeMode=Enum.ApplyStrokeMode.Border}) end
local function text(parent, value, pos, size, fontSize, color, bold)
    return new("TextLabel",parent,{
        BackgroundTransparency=1,Text=value,Position=pos,Size=size,
        Font=bold and Enum.Font.GothamBold or Enum.Font.Gotham,TextSize=fontSize or 13,
        TextColor3=color or colors.Text,TextXAlignment=Enum.TextXAlignment.Left,
        TextYAlignment=Enum.TextYAlignment.Center,TextTruncate=Enum.TextTruncate.AtEnd,
    })
end

local gui = new("ScreenGui",pg,{Name="JaxCoinHub",ResetOnSpawn=false,DisplayOrder=1200,ZIndexBehavior=Enum.ZIndexBehavior.Sibling})
hub.Gui=gui
local window=new("Frame",gui,{Name="Window",Size=UDim2.fromOffset(800,618),Position=UDim2.new(0.5,-400,0.5,-309),BackgroundColor3=colors.Base,BorderSizePixel=0,Active=true})
corner(window,14) stroke(window)
local scale = new("UIScale",window,{Scale=1})
local header=new("Frame",window,{Name="Header",Size=UDim2.new(1,0,0,56),BackgroundTransparency=1,Active=true})
text(header,"Vanta larp boy",UDim2.fromOffset(20,10),UDim2.fromOffset(360,24),20,colors.Text,true)
text(header,"Turbo de farm  •  feito por 27k",UDim2.fromOffset(20,33),UDim2.fromOffset(360,18),11,colors.Muted)

local function button(parent, name, label, position, size, callback, color)
    local b=new("TextButton",parent,{
        Name=name,Text=label,Position=position or UDim2.new(),Size=size or UDim2.fromOffset(100,30),
        BackgroundColor3=color or colors.Card,TextColor3=colors.Text,TextSize=12,
        Font=Enum.Font.GothamBold,AutoButtonColor=true,BorderSizePixel=0,
    })
    corner(b,7)
    if callback then connect(b.Activated,function() if hub.Running then attempt(name,callback) end end) end
    hub.Controls[name]=b
    return b
end

local badge=new("Frame",header,{Position=UDim2.fromOffset(393,15),Size=UDim2.fromOffset(147,28),BackgroundColor3=Color3.fromRGB(20,62,63),BorderSizePixel=0})
corner(badge,7)
local badgeText=text(badge,"AUTO FARM",UDim2.fromOffset(8,0),UDim2.new(1,-16,1,0),10,colors.Green,true)
local pause=button(header,"PauseAll","PAUSAR",UDim2.fromOffset(576,15),UDim2.fromOffset(100,29),function() C.Paused=not C.Paused log(C.Paused and "Automações pausadas" or "Automações retomadas") end)
button(header,"Minimize","—",UDim2.fromOffset(688,15),UDim2.fromOffset(40,29),function() window.Visible=false end)
button(header,"Unload","×",UDim2.fromOffset(742,15),UDim2.fromOffset(40,29),function() hub.Stop() end,colors.Danger)

local reopen=button(gui,"Reopen","27K",UDim2.fromOffset(20,160),UDim2.fromOffset(58,38),function() window.Visible=true end,colors.Accent)
reopen.Visible=false

local metrics={}
for i,label in ipairs({"SALDO CONFIRMADO","PRODUÇÃO / S","STAMINA DISPONÍVEL"}) do
    local x=156+(i-1)*207
    local card=new("Frame",window,{Name="Metric"..i,Position=UDim2.fromOffset(x,65),Size=UDim2.fromOffset(198,44),BackgroundColor3=colors.Card,BorderSizePixel=0})
    corner(card,8)
    text(card,label,UDim2.fromOffset(12,4),UDim2.fromOffset(174,14),10,colors.Muted,true)
    metrics[i]=text(card,"—",UDim2.fromOffset(12,20),UDim2.fromOffset(174,20),17,colors.Text,true)
end

local sidebar=new("Frame",window,{Name="Sidebar",Position=UDim2.fromOffset(12,65),Size=UDim2.fromOffset(128,513),BackgroundColor3=colors.Side,BorderSizePixel=0})
corner(sidebar,10)
text(sidebar,"COIN CLICKER",UDim2.fromOffset(12,12),UDim2.fromOffset(106,20),10,colors.Muted,true)

local pages,tabButtons={},{}
local tabNames={"Painel","Farm","Loja","Eventos","Estatísticas","Opções"}
local selected="Painel"

function hub.SelectTab(name)
    assert(pages[name],"Aba desconhecida: "..tostring(name))
    selected=name
    for n,p in pairs(pages) do p.Visible=n==name end
    for n,b in pairs(tabButtons) do b.BackgroundColor3=n==name and colors.Accent or colors.Side end
end

for i,name in ipairs(tabNames) do
    local page=new("ScrollingFrame",window,{
        Name="Page_"..name,Position=UDim2.fromOffset(156,119),Size=UDim2.fromOffset(629,459),
        BackgroundTransparency=1,BorderSizePixel=0,ScrollBarThickness=4,
        ScrollBarImageColor3=colors.Accent,AutomaticCanvasSize=Enum.AutomaticSize.Y,
        CanvasSize=UDim2.new(),ScrollingDirection=Enum.ScrollingDirection.Y,Visible=i==1,
    })
    new("UIPadding",page,{PaddingRight=UDim.new(0,9),PaddingBottom=UDim.new(0,8)})
    new("UIListLayout",page,{Padding=UDim.new(0,8),SortOrder=Enum.SortOrder.LayoutOrder})
    pages[name]=page
    tabButtons[name]=button(sidebar,"Tab_"..name,name,UDim2.fromOffset(8,44+(i-1)*44),UDim2.fromOffset(112,36),function() hub.SelectTab(name) end)
end

text(sidebar,"F6  pausar tudo\nF8  mostrar hub\n\nArraste pelo topo",UDim2.fromOffset(12,388),UDim2.fromOffset(106,100),11,colors.Muted).TextWrapped=true
local footer=text(window,"Conectando…",UDim2.fromOffset(18,588),UDim2.fromOffset(760,20),11,colors.Muted)

local order={}
local function row(page, height, name)
    order[page]=(order[page] or 0)+1
    local r=new("Frame",page,{Name=name or "Card",Size=UDim2.new(1,0,0,height),BackgroundColor3=colors.Card,BorderSizePixel=0,LayoutOrder=order[page]})
    corner(r,8)
    return r
end

local function section(page,title,desc)
    local r=row(page,desc and 50 or 29,"Section")
    r.BackgroundTransparency=1
    text(r,title,UDim2.fromOffset(1,0),UDim2.new(1,-4,0,25),16,colors.Text,true)
    if desc then text(r,desc,UDim2.fromOffset(1,24),UDim2.new(1,-4,0,24),11,colors.Muted) end
end

local refreshers={}
local function toggle(page,key,title,desc)
    local r=row(page,60,key)
    text(r,title,UDim2.fromOffset(12,5),UDim2.new(1,-108,0,23),13,colors.Text,true)
    text(r,desc,UDim2.fromOffset(12,28),UDim2.new(1,-108,0,24),11,colors.Muted)
    local b
    b=button(r,key.."Toggle","",UDim2.new(1,-82,0,15),UDim2.fromOffset(70,30),function()
        C[key]=not C[key]
        b.Text=C[key] and "LIGADO" or "OFF"
        b.BackgroundColor3=C[key] and colors.Accent or colors.Border
        
        if key == "CompactView" then
            pcall(applyCompact)
        end
        
        log(title..": "..(C[key] and "ligado" or "desligado"))
    end)
    table.insert(refreshers,function()
        b.Text=C[key] and "LIGADO" or "OFF"
        b.BackgroundColor3=C[key] and colors.Accent or colors.Border
    end)
    return b
end

local function numeric(page,key,title,desc,min,max)
    local r=row(page,60,key)
    text(r,title,UDim2.fromOffset(12,5),UDim2.new(1,-137,0,23),13,colors.Text,true)
    text(r,desc,UDim2.fromOffset(12,28),UDim2.new(1,-137,0,24),11,colors.Muted)
    local box=new("TextBox",r,{Name=key.."Input",Text=tostring(C[key]),PlaceholderText=tostring(C[key]),Position=UDim2.new(1,-121,0,15),Size=UDim2.fromOffset(108,31),BackgroundColor3=colors.Side,TextColor3=colors.Text,Font=Enum.Font.GothamBold,TextSize=13,ClearTextOnFocus=false,BorderSizePixel=0})
    corner(box,6)
    connect(box.FocusLost,function()
        local raw=box.Text:gsub("%s",""):gsub(",",".")
        local n=tonumber(raw)
        if not n or n~=n or n==math.huge or n==-math.huge then box.Text=tostring(C[key]) log("Número inválido: "..title) return end
        C[key]=math.clamp(n,min,max)
        box.Text=tostring(C[key])
        log(title.." = "..tostring(C[key]))
    end)
    hub.Controls[key.."Input"]=box
end

local function actionRow(page,key,title,desc,label,fn)
    local r=row(page,60,key)
    text(r,title,UDim2.fromOffset(12,5),UDim2.new(1,-126,0,23),13,colors.Text,true)
    local detail=text(r,desc,UDim2.fromOffset(12,28),UDim2.new(1,-126,0,24),11,colors.Muted)
    local b=button(r,key,label,UDim2.new(1,-111,0,15),UDim2.fromOffset(98,30),fn)
    return detail,b,r
end

-- ABA FARM
local farm=pages.Farm
section(farm,"Farm automático","Ative os recursos individualmente. F6 pausa todas as automações.")
toggle(farm,"AutoClick","Auto Click humanizado","Clica por ~50s e descansa por ~70s com micro-pausas humanas.")
toggle(farm,"CompactView","Modo compacto (UI)","Reduz o tamanho e oculta partes da interface do jogo.")
toggle(farm,"Performance","Modo desempenho","Remove partículas, números flutuantes e som de clique.")
toggle(farm,"AutoBuy","Comprar geradores","Reinveste todo o saldo nas melhores oportunidades.")
toggle(farm,"AutoUpgrade","Comprar melhorias","Compara melhorias e geradores pelo ganho de renda.")
toggle(farm,"SaveMode","Guardar para compra melhor","Evita compras baratas quando elas atrasariam uma opção superior.")
numeric(farm,"MaxSaveSeconds","Limite para guardar","Tempo máximo, em segundos, que o planejador aceita esperar por uma compra superior.",30,3600)

local strategyDetail,strategyButton=actionRow(farm,"Strategy","Estratégia de compra","Compara o ganho passivo e o ganho dos cliques.","Retorno",function()
    local modes={"Lucro","Retorno","Mais barato","Desbloquear"}
    for i,v in ipairs(modes) do if v==C.Strategy then C.Strategy=modes[i%#modes+1] break end end
    log("Estratégia: "..C.Strategy)
end)

local recommendation=row(farm,50,"Recommendation")
local recommendationText=text(recommendation,"",UDim2.fromOffset(12,5),UDim2.new(1,-24,1,-10),12,colors.Muted)
recommendationText.TextWrapped=true

table.insert(refreshers,function()
    if selected~="Farm" then return end
    strategyButton.Text=C.Strategy
    strategyDetail.Text=(C.Strategy=="Lucro" or C.Strategy=="Retorno") and "Compara o ganho passivo e o ganho dos cliques." or (C.Strategy=="Mais barato" and "Prioriza o gerador com menor preço." or "Prioriza tipos ainda não comprados; depois, retorno.")
    local p,target,action,info=ready() and purchaseDecision()
    if action=="save" and target then
        local alt=info and info.Alternative
        local comparison=alt and (" • evita "..alt.Name.." ("..string.format("%.2fx",info.Advantage).." pior/moeda)") or ""
        recommendationText.Text="Guardando para "..target.Name.." • faltam "..fmt(math.max(0,target.Cost-(info and info.Funds or 0))).." • ETA "..math.ceil(info and info.Wait or 0).."s"..comparison
    elseif p then
        recommendationText.Text="Próxima ação: "..p.Name.." • "..fmt(p.Cost).." moedas • +"..fmt(p.Gain).."/s • retorno "..fmt(p.ROI).."s"
    else
        recommendationText.Text="Aguardando uma oportunidade com ganho real."
    end
end)

-- ABA LOJA
local shop=pages.Loja
section(shop,"Geradores","Os botões compram 1 unidade; a seleção da loja do jogo é preservada.")
local buildingRows={}
for i,g in ipairs(Catalog.generators) do
    local r=row(shop,57,"Buy_"..g.id)
    new("ImageLabel",r,{Image=g.icon,Position=UDim2.fromOffset(10,9),Size=UDim2.fromOffset(38,38),BackgroundTransparency=1})
    text(r,genName(g),UDim2.fromOffset(59,5),UDim2.new(1,-175,0,23),13,colors.Text,true)
    local detail=text(r,"",UDim2.fromOffset(59,28),UDim2.new(1,-175,0,21),11,colors.Muted)
    local b=button(r,"Buy_"..g.id,"Comprar 1",UDim2.new(1,-108,0,13),UDim2.fromOffset(96,31),function()
        if oneBuy(g.id,false) then hub.Counts.Buildings=hub.Counts.Buildings+1 log("Comprou "..genName(g))
        else log("Saldo insuficiente ou gerador indisponível") end
    end)
    table.insert(buildingRows,{Index=i,Detail=detail,Button=b})
end

section(shop,"Melhorias disponíveis","Lista das 12 melhorias liberadas com menor preço; atualiza automaticamente.")
local upgradeRows={}
for i=1,12 do
    local data={}
    local r=row(shop,57,"UpgradeSlot"..i)
    data.Row=r
    data.Name=text(r,"",UDim2.fromOffset(12,5),UDim2.new(1,-128,0,23),13,colors.Text,true)
    data.Detail=text(r,"",UDim2.fromOffset(12,28),UDim2.new(1,-128,0,21),11,colors.Muted)
    data.Button=button(r,"UpgradeSlot"..i,"Comprar",UDim2.new(1,-108,0,13),UDim2.fromOffset(96,31),function()
        local u=data.Upgrade
        if ready() and u and not driver.upgradePurchased(u.id) and driver.upgradeUnlocked(u) and driver.coins()>=u.price and putUpgrade(u) then
        else log("Melhoria indisponível ou saldo insuficiente") end
    end)
    table.insert(upgradeRows,data)
end
local upgradesEmpty=row(shop,40,"NoUpgrades")
text(upgradesEmpty,"Nenhuma melhoria liberada no momento.",UDim2.fromOffset(12,0),UDim2.new(1,-24,1,0),12,colors.Muted)

table.insert(refreshers,function()
    if selected~="Loja" then return end
    if not ready() then return end
    for _,r in ipairs(buildingRows) do
        local cost=driver.priceAt(r.Index)
        r.Detail.Text="Possui "..driver.ownedAt(r.Index).." • "..fmt(cost).." moedas • +"..fmt(driver.marginalCpsAt(r.Index)).."/s • retorno "..fmt(driver.paybackAt(r.Index)).."s"
        r.Button.BackgroundColor3=driver.coins()>=cost and colors.Accent or colors.Border
    end
    local available=availableUpgrades()
    upgradesEmpty.Visible=#available==0
    for i,r in ipairs(upgradeRows) do
        local u=available[i] r.Upgrade=u r.Row.Visible=u~=nil
        if u then
            r.Name.Text=u.name r.Detail.Text=fmt(u.price).." moedas • "..u.kind
            r.Button.BackgroundColor3=driver.coins()>=u.price and colors.Accent or colors.Border
        end
    end
end)

-- ABA EVENTOS
local events=pages.Eventos
section(events,"Eventos e coleta","Recursos aguardam o desbloqueio e a disponibilidade no servidor.")
toggle(events,"AutoExtraUI","Clicar extras da interface","Clica imediatamente em GoldenCoin e nos Wrinklers que aparecem ao redor.")
toggle(events,"AutoGolden","Coletar moedas douradas","Inclui as moedas douradas que surgem em tempestades.")
toggle(events,"AutoLump","Colher lumps maduros","Colhe apenas no estágio ripe, com chance garantida.")
toggle(events,"AutoMarket","Comprar raros do mercado","Compra assim que há estoque e saldo suficiente.")
toggle(events,"AutoReward","Resgatar recompensa","Tenta o resgate apenas após cumprir os requisitos.")
toggle(events,"AutoWrinkler","Coletar wrinklers","Estoura os disponíveis no intervalo configurado.")
numeric(events,"WrinklerInterval","Intervalo de wrinklers","Segundos entre as coletas automáticas.",10,3600)

local lumpDetail=actionRow(events,"HarvestNow","Colher lump agora","","Colher",function()
    if ready() and driver.lumpStage()=="ripe" and driver.harvestLump() then hub.Counts.Harvests=hub.Counts.Harvests+1 log("Colheita enviada")
    else log("Colheita garantida ainda indisponível") end
end)
local wrinklerDetail=actionRow(events,"WrinklersNow","Coletar wrinklers agora","","Coletar",function()
    if not ready() then return end
    local list=table.clone(driver.wrinklers())
    local count=0
    for _,w in ipairs(list) do if driver.popWrinkler(w.id) then count=count+1 end end
    hub.Counts.Wrinklers=hub.Counts.Wrinklers+count
    log("Wrinklers coletados: "..count)
end)
local rewardDetail=actionRow(events,"RewardNow","Recompensa de conquistas","","Resgatar",function()
    if ready() and driver.rewardReady() and not driver.rewardOwned() and driver.claimReward() then log("Resgate enviado ao servidor")
    else log("Recompensa já recebida ou requisitos incompletos") end
end)

section(events,"Feitiços","Liberados pela Bola de Cristal; podem falhar e aplicar penalidades do jogo.")
toggle(events,"AutoSpell","Conjuração automática","Combo inteligente ou feitiço escolhido; aguarda mana.")
local spellNames={auto="Combo inteligente",conjure="Conjurar moedas",stretchTime="Estender buffs",resurrect="Invocar wrinkler",handOfFate="Moeda dourada",sleightOfHand="Clique x25",packedHouse="Produção x5",edifice="Criar edifício"}
local selectedSpellDetail,selectedSpellButton=actionRow(events,"SelectSpell","Feitiço automático","","Trocar",function()
    local ids={"auto"}
    for _,spell in ipairs(Fortunes.list) do table.insert(ids,spell.id) end
    for i,id in ipairs(ids) do if id==C.Spell then C.Spell=ids[i%#ids+1] break end end
    log("Feitiço escolhido: "..(spellNames[C.Spell] or C.Spell))
end)

local spellRows={}
for _,s in ipairs(Fortunes.list) do
    local detail,b=actionRow(events,"Cast_"..s.id,spellNames[s.id] or s.name,"","Conjurar",function()
        if ready() and driver.spellReady(s) and driver.mana()>=driver.spellCost(s) and driver.castSpell(s.id) then
            hub.Counts.Spells=hub.Counts.Spells+1 log("Conjuração enviada: "..s.name)
        else log("Feitiço bloqueado ou mana insuficiente") end
    end)
    table.insert(spellRows,{Spell=s,Detail=detail,Button=b})
end

section(events,"Mercado de raros","Estoque só aparece durante o evento de mercado.")
local rareRows={}
for _,g in ipairs(Catalog.rares) do
    local detail,b=actionRow(events,"Rare_"..g.id,g.name,"","Comprar 1",function()
        local available=false
        if ready() then for _,s in ipairs(driver.marketStock()) do if s.id==g.id and s.left>0 then available=true break end end end
        if available and oneBuy(g.id,true) then hub.Counts.Rares=hub.Counts.Rares+1 log("Raro comprado: "..g.name)
        else log("Raro sem estoque ou saldo insuficiente") end
    end)
    table.insert(rareRows,{Generator=g,Detail=detail,Button=b})
end

table.insert(refreshers,function()
    if selected~="Eventos" then return end
    if not ready() then return end
    local stage=driver.lumpStage()
    lumpDetail.Text=stage=="locked" and "Desbloqueia com progresso no jogo." or ("Estágio: "..stage.." • possui "..fmt(driver.lumps()).." • "..math.ceil(driver.lumpRipeIn()/60).." min")
    wrinklerDetail.Text=tostring(#driver.wrinklers()).." wrinklers disponíveis."
    rewardDetail.Text=driver.rewardOwned() and "Recompensa recebida." or (driver.rewardReady() and "Recompensa pronta para resgatar." or (driver.achievementCount().."/"..#Achievements.list.." conquistas."))
    selectedSpellDetail.Text=spellNames[C.Spell] or C.Spell
    for _,r in ipairs(spellRows) do
        local s=r.Spell local unlocked=driver.spellReady(s)
        local cost=driver.spellCost(s)
        r.Detail.Text=unlocked and ("Mana: "..fmt(driver.mana()).."/"..fmt(driver.manaCap()).." • custo "..fmt(cost)) or "Bloqueado: aumente a capacidade de mana."
        r.Button.BackgroundColor3=unlocked and driver.mana()>=cost and colors.Accent or colors.Border
    end
    local stock={}
    for _,s in ipairs(driver.marketStock()) do stock[s.id]=s.left end
    for _,r in ipairs(rareRows) do
        local index=findIndex(r.Generator.id)
        local price=driver.priceAt(index)
        local left=stock[r.Generator.id] or 0
        r.Detail.Text="Estoque "..left.." • "..fmt(price).." moedas • possui "..driver.ownedAt(index)
        r.Button.BackgroundColor3=left>0 and driver.coins()>=price and colors.Accent or colors.Border
    end
end)

-- ABA ESTATÍSTICAS
local stats=pages["Estatísticas"]
section(stats,"Sessão e progresso","Contadores do hub e dados atuais recebidos do jogo.")
local statsCard=row(stats,300,"Statistics")
local statsText=text(statsCard,"",UDim2.fromOffset(14,10),UDim2.new(1,-28,1,-20),13,colors.Text)
statsText.TextYAlignment=Enum.TextYAlignment.Top statsText.TextWrapped=true

section(stats,"Atalhos do jogo")
local function gameTab(wanted)
    local rt=pg:FindFirstChild("CoinClickerGui")
    local buttons=rt and rt:FindFirstChild("Buttons",true)
    if buttons then
        for _,b in ipairs(buttons:GetChildren()) do
            if b:IsA("TextButton") and b.Text==wanted then
                firesignal(b.Activated) window.Visible=false return
            end
        end
    end
    log("Painel do jogo indisponível: "..wanted)
end

actionRow(stats,"GameStats","Estatísticas completas","Abre o painel de estatísticas do próprio jogo.","Abrir",function() gameTab("Stats") end)
actionRow(stats,"GameLegacy","Legado e prestígio","Abre o painel do jogo para consultar a ascensão.","Abrir",function() gameTab("Legacy") end)

section(stats,"Registro de ações")
local logsCard=row(stats,193,"Logs")
local logsText=text(logsCard,"",UDim2.fromOffset(12,10),UDim2.new(1,-24,1,-20),11,colors.Muted)
logsText.TextYAlignment=Enum.TextYAlignment.Top logsText.TextWrapped=true

table.insert(refreshers,function()
    if selected~="Estatísticas" then return end
    if not ready() then statsText.Text="Aguardando o Coin Clicker…" return end
    local n=hub.Counts
    local duration=os.clock()-hub.Started
    local buffs={} for _,b in ipairs(driver.buffs()) do table.insert(buffs,b.name or b.kind or tostring(b.id)) end
    statsText.Text=table.concat({
        "Sessão: "..math.floor(duration/60).." min "..math.floor(duration%60).." s",
        "Cliques manuais/auto: "..n.Clicks.."   |   Golden/Wrinkler clicados: "..n.ExtraUiClicks,
        "Pedidos: "..n.Buildings.." geradores • "..n.Upgrades.." melhorias • "..n.Rares.." raros",
        "Douradas: "..n.Goldens.."   |   Lumps: "..n.Harvests.."   |   Wrinklers: "..n.Wrinklers.."   |   Feitiços: "..n.Spells,
        "",
        "Valor do clique: "..fmt(driver.clickValue()).."   |   Geradores totais: "..driver.totalOwned(),
        "Ganho total: "..fmt(driver.totalEarned()).."   |   Gasto total: "..fmt(driver.totalSpent()),
        "Conquistas: "..driver.achievementCount().."/"..#Achievements.list.."   |   Cliques totais: "..fmt(driver.totalClicks()),
        "Prestígio: "..fmt(driver.prestigeLevel()).."   |   Ganho de ascensão: "..fmt(driver.prestigeGain()),
        "Mana: "..fmt(driver.mana()).."/"..fmt(driver.manaCap()).."   |   Lumps: "..fmt(driver.lumps()),
        "Buffs ativos: "..(#buffs>0 and table.concat(buffs,", ") or "nenhum"),
        "Servidor: "..string.format("%.1f",hub.ServerRate).." cliques/s • "..fmt(hub.ServerIncome).." moedas/s",
        "Compras confirmadas: "..hub.Confirmed.." • pendentes: "..#hub.Pending.." • sem confirmação: "..hub.Rejected,
    },"\n")
    local lines={} for i=1,math.min(9,#hub.Logs) do table.insert(lines,hub.Logs[i]) end
    logsText.Text=table.concat(lines,"\n")
end)

-- ABA OPÇÕES
local options=pages["Opções"]
section(options,"Controles do hub","Configuração mantida enquanto esta sessão do hub estiver ativa.")
toggle(options,"Paused","Pausar automações","Os botões de ações manuais continuam disponíveis.")
toggle(options,"Verbose","Diagnóstico no console","Exibe os contadores uma vez por minuto.")
actionRow(options,"PauseFeatures","Desligar automações","Desativa os recursos automáticos individuais.","Desligar",function()
    local keysToDisable = {"AutoClick","CompactView","AutoExtraUI","AutoBuy","AutoUpgrade","AutoGolden","AutoLump","AutoMarket","AutoWrinkler","AutoReward","AutoSpell"}
    for _,k in ipairs(keysToDisable) do C[k]=false end
    pcall(applyCompact)
    log("Todos os recursos automáticos desligados")
end)
actionRow(options,"ClearCounts","Zerar contadores da sessão","Zera apenas os contadores mostrados pelo hub.","Zerar",function()
    for k in pairs(hub.Counts) do hub.Counts[k]=0 end
    log("Contadores da sessão zerados")
end)
actionRow(options,"Reconnect","Reconectar ao jogo","Refaz a ligação com a interface do Coin Clicker.","Reconectar",function()
    if privateDestroy then privateDestroy() end
    driver,privateDestroy,privateSettings=createBackend()
    bindBackend()
    lastPerformance=nil
    hub.LastSync=nil hub.LastSyncAt=nil hub.ServerRate=0 hub.ServerIncome=0
    table.clear(hub.Pending)
    log("Backend reiniciado; aguardando confirmação do servidor")
end)
actionRow(options,"HideHub","Minimizar interface","F8 ou botão 27K reabre; o farm continua ativo.","Minimizar",function() window.Visible=false end)
actionRow(options,"StopHub","Encerrar hub","Para as tarefas e remove a interface.","Encerrar",function() hub.Stop() end)

local info=row(options,112,"Keybinds")
local infoText=text(info,"F6: pausar/retomar tudo  •  F8: mostrar/ocultar\nArraste a barra superior para mover o hub.\nOpções da Farm começam desativadas.\nCompras, melhorias e auto-click podem ser ligados na Farm.\nO servidor controla a taxa aceita.",UDim2.fromOffset(12,7),UDim2.new(1,-24,1,-14),12,colors.Muted)
infoText.TextWrapped=true

-- ABA PAINEL
local overview=pages.Painel
section(overview,"Ritmo de ganho","Cada barra representa uma atualização real do servidor.")
local liveCard=row(overview,105,"LiveOverview")
local speedText=text(liveCard,"",UDim2.fromOffset(14,8),UDim2.new(0.5,-22,0,27),21,colors.Green,true)
local incomeText=text(liveCard,"",UDim2.new(0.5,8,0,27),UDim2.new(0.5,-22,0,27),21,colors.Text,true)
text(liveCard,"Cliques/s do servidor • média de 8s",UDim2.fromOffset(14,37),UDim2.new(0.5,-22,0,18),10,colors.Muted)
text(liveCard,"Ganho/s do servidor • média de 8s",UDim2.new(0.5,8,0,37),UDim2.new(0.5,-22,0,18),10,colors.Muted)
local confirmedText=text(liveCard,"",UDim2.fromOffset(14,69),UDim2.new(1,-28,0,24),12,colors.Muted)

local chart=row(overview,90,"IncomeChart")
local bars={}
for i=1,24 do
    bars[i]=new("Frame",chart,{Name="Sample"..i,AnchorPoint=Vector2.new(0,1),Position=UDim2.new((i-1)/24,6,1,-9),Size=UDim2.new(1/24,-5,0,2),BackgroundColor3=colors.Accent,BorderSizePixel=0})
    corner(bars[i],3)
end

section(overview,"Motor de compras","Compara o ganho passivo e o valor dos cliques; não reserva saldo.")
local nextCard=row(overview,77,"NextPurchase")
local nextTitle=text(nextCard,"",UDim2.fromOffset(13,5),UDim2.new(1,-26,0,26),14,colors.Text,true)
local nextDetail=text(nextCard,"",UDim2.fromOffset(13,33),UDim2.new(1,-26,0,36),12,colors.Muted)
nextDetail.TextWrapped=true

actionRow(overview,"QuickFarm","Configurar automações","Estratégia de investimento e compras automáticas.","Abrir",function() hub.SelectTab("Farm") end)
actionRow(overview,"QuickShop","Compras manuais","Geradores e melhorias com preços atualizados.","Abrir",function() hub.SelectTab("Loja") end)

table.insert(refreshers,function()
    badgeText.Text=C.Paused and "PAUSADO" or "AUTO FARM"
    if selected~="Painel" or not ready() then return end
    speedText.Text=string.format("%.1f cliques/s",hub.ServerRate)
    incomeText.Text=fmt(hub.ServerIncome).." moedas/s"
    confirmedText.Text=hub.Confirmed.." compras confirmadas • "..#hub.Pending.." aguardando • "..(hub.LastSyncAt and string.format("sync há %.1fs",os.clock()-hub.LastSyncAt) or "aguardando servidor")
    local maximum=1
    for _,value in ipairs(hub.History) do maximum=math.max(maximum,value) end
    for i,bar in ipairs(bars) do
        local sample=hub.History[i]
        bar.Size=UDim2.new(1/24,-5,0,sample and math.max(3,math.floor(sample/maximum*66)) or 2)
        bar.BackgroundColor3=sample and colors.Accent or colors.Border
    end
    local best,target,action,info=purchaseDecision()
    if action=="save" and target then
        nextTitle.Text="Meta inteligente: "..target.Name
        local comparison=info and info.Alternative and (" • "..info.Alternative.Name.." atrasaria a meta") or ""
        nextDetail.Text="Guardando "..fmt(target.Cost).." • faltam "..fmt(math.max(0,target.Cost-(info and info.Funds or 0))).." • ETA "..math.ceil(info and info.Wait or 0).."s • +"..fmt(target.Gain).."/s"..comparison
    else
        local shown=best or target
        nextTitle.Text=shown and ("Próxima compra: "..shown.Name) or "Compras aguardando disponibilidade"
        nextDetail.Text=shown and ("Custo "..fmt(shown.Cost).." • ganho estimado +"..fmt(shown.Gain).."/s • retorno em "..fmt(shown.ROI).."s") or "Ative geradores, melhorias ou mercado na aba Farm/Eventos."
    end
end)

function hub.Stop()
    if not hub.Running then return end
    hub.Running=false
    C.CompactView = false
    pcall(applyCompact)
    
    if settings and savedEffects then pcall(function()
        local current=settings.snapshot()
        current.particles=savedEffects.particles current.backgroundCoins=savedEffects.backgroundCoins
        current.floatingNumbers=savedEffects.floatingNumbers current.clickSound=savedEffects.clickSound
        settings.apply(current)
        if driver then driver.flushSettings() end
    end) end
    if privateDestroy then privateDestroy() privateDestroy=nil end
    for _,c in ipairs(hub.Connections) do c:Disconnect() end
    gui:Destroy()
    if env.JaxCoinHub==hub then env.JaxCoinHub=nil end
    print("[27K HUB] Encerrado. Todas as tarefas e conexões removidas.")
end

function hub.Snapshot()
    return {Running=hub.Running,Paused=C.Paused,Buildings=hub.Counts.Buildings,Upgrades=hub.Counts.Upgrades,Coins=ready() and driver.coins() or 0,ServerCoins=hub.LastSync and hub.LastSync.Coins,SyncCount=hub.SyncCount,SelectedTab=selected,ExtraUiClicks=hub.Counts.ExtraUiClicks,ServerRate=hub.ServerRate,ServerIncome=hub.ServerIncome,Confirmed=hub.Confirmed,Rejected=hub.Rejected,Stamina=ready() and driver.stamina() or 0,Production=ready() and driver.coinsPerSecond() or 0,ClickValue=ready() and driver.clickValue() or 0,Version=hub.Version}
end

local samples, chartHistory = {}, {}
local lastChart=0
local function sampleServer(s)
    if type(s)~="table" then return end
    local now=os.clock()
    table.insert(samples,{At=now,Clicks=s.TotalClicks or 0,Earned=s.TotalEarned or 0})
    while #samples>2 and now-samples[2].At>8 do table.remove(samples,1) end
    local first=samples[1]
    local span=now-first.At
    if span>=1 then
        hub.ServerRate=math.max(0,((s.TotalClicks or 0)-first.Clicks)/span)
        hub.ServerIncome=math.max(0,((s.TotalEarned or 0)-first.Earned)/span)
    end
    if now-lastChart>=1 then
        lastChart=now
        table.insert(chartHistory,hub.ServerIncome)
        if #chartHistory>24 then table.remove(chartHistory,1) end
    end
    hub.History=table.clone(chartHistory)
end

local remotes=require(feature.Remotes).get()
connect(remotes.Sync.OnClientEvent,function(s)
    if type(s)~="table" then return end
    local now=os.clock()
    sampleServer(s)
    hub.LastSync=s hub.LastSyncAt=now hub.SyncCount=hub.SyncCount+1
    for i=#hub.Pending,1,-1 do
        local p=hub.Pending[i]
        local confirmed=p.Kind=="upgrade" and (s.Purchased or {})[p.Id]==true
            or p.Kind=="building" and ((s.Owned or {})[p.Id] or 0)>=p.Target
        if confirmed then hub.Confirmed=hub.Confirmed+1 table.remove(hub.Pending,i)
        elseif now-p.At>8 and hub.SyncCount>p.Sync+1 then hub.Rejected=hub.Rejected+1 table.remove(hub.Pending,i) end
    end
    lastPlanAt=0
end)

connect(UIS.InputBegan,function(input,processed)
    if processed or UIS:GetFocusedTextBox() then return end
    if input.KeyCode==Enum.KeyCode.F6 then C.Paused=not C.Paused log(C.Paused and "Automações pausadas" or "Automações retomadas")
    elseif input.KeyCode==Enum.KeyCode.F8 then window.Visible=not window.Visible end
end)

local dragging, dragStart, original
connect(header.InputBegan,function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
        dragging=true dragStart=input.Position original=window.Position
    end
end)
connect(UIS.InputEnded,function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then dragging=false end
end)
connect(UIS.InputChanged,function(input)
    if dragging and (input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch) then
        local delta=input.Position-dragStart
        local cam=workspace.CurrentCamera
        local view=cam and cam.ViewportSize or Vector2.new(1920,1080)
        local s=scale.Scale
        local x=original.X.Scale*view.X+original.X.Offset+delta.X
        local y=original.Y.Scale*view.Y+original.Y.Offset+delta.Y
        window.Position=UDim2.fromOffset(math.clamp(x,0,math.max(0,view.X-800*s)),math.clamp(y,0,math.max(0,view.Y-56*s)))
    end
end)

local function refresh()
    local cam=workspace.CurrentCamera
    if cam then scale.Scale=math.min(1,math.max(0.3,math.min((cam.ViewportSize.X-24)/800,(cam.ViewportSize.Y-58)/618))) end
    reopen.Visible=not window.Visible
    pause.Text=C.Paused and "RETOMAR" or "PAUSAR"
    pause.BackgroundColor3=C.Paused and colors.Accent or colors.Card
    if ready() then
        metrics[1].Text=hub.LastSync and fmt(hub.LastSync.Coins) or fmt(driver.coins())
        metrics[2].Text=fmt(driver.coinsPerSecond())
        metrics[3].Text=string.format("%d%%",math.floor(driver.stamina()*100))
        metrics[3].TextColor3=driver.stamina()<0.2 and colors.Danger or colors.Green
    else for _,m in ipairs(metrics) do m.Text="—" end end
    footer.Text=(ready() and "CONECTADO" or "AGUARDANDO JOGO").."  •  "..(C.Paused and "PAUSADO" or "ATIVO").."  •  F6 pausa  /  F8 interface"
    for _,fn in ipairs(refreshers) do fn() end
end

hub.SelectTab("Painel")
hub.Refresh=refresh
refresh()

local lastPurchase,lastEvents,lastWrinklers,lastSpell,lastReward,lastRefresh,lastBind,lastVerbose,lastExtraUI,lastCompact=0,0,os.clock(),0,0,0,0,os.clock(),0,0

local function clickExtraUi()
    local gui=pg:FindFirstChild("CoinClickerGui")
    if not gui then return 0 end
    local clicked=0
    local targets={}
    for _,item in ipairs(gui:GetDescendants()) do
        if item:IsA("GuiButton") and item.Visible and (item.Name=="GoldenCoin" or item.Name=="Wrinkler") then
            table.insert(targets,item)
        end
    end
    for _,item in ipairs(targets) do
        local conns=getconnections(item.Activated)
        if #conns>0 and item.Parent then
            firesignal(item.Activated)
            clicked=clicked+1
        end
    end
    return clicked
end

local function automatic(key,fn)
    if not C[key] then return end
    local ok=attempt(key,fn)
    if not ok and (failures[key] or 0)>=3 then C[key]=false log("Desativado após 3 falhas: "..key,true) end
end

-- LOOP PRINCIPAL
connect(RunService.Heartbeat,function(dt)
    if not hub.Running then return end
    local now=os.clock()
    
    if now-lastBind>=1 then
        lastBind=now
        attempt("Conexão",function()
            if ready() then applyPerformance() end
        end)
    end
    
    -- ATUALIZAÇÃO DA COMPACTAÇÃO DA UI
    if now-lastCompact>=0.4 then
        lastCompact=now
        pcall(applyCompact)
    end
    
    if not C.Paused and ready() then
        -- EXECUÇÃO DO AUTOCLICK HUMANIZADO
        if C.AutoClick then
            if clickState.StaminaStart == 0 then
                clickState.StaminaStart = now
                clickState.TargetClickTime = math.random(48, 52)
                clickState.TargetRestTime = math.random(67, 73)
            end
            
            local totalCycle = clickState.TargetClickTime + clickState.TargetRestTime
            local elapsed = (now - clickState.StaminaStart) % totalCycle
            
            if elapsed < clickState.TargetClickTime then
                if now >= clickState.MicroPauseEnd then
                    if math.random(1, 1000) <= 10 then
                        clickState.MicroPauseEnd = now + (math.random(15, 35) / 100)
                    end
                    
                    if now - clickState.LastClick >= clickState.NextClickDelay then
                        clickState.LastClick = now
                        local baseDelay = math.random(18, 48) / 1000
                        local fatigue = (elapsed / clickState.TargetClickTime) * 0.012
                        clickState.NextClickDelay = baseDelay + fatigue
                        
                        performHumanClick()
                    end
                end
            else
                if math.ceil(totalCycle - elapsed) == 1 then
                    clickState.TargetClickTime = math.random(48, 52)
                    clickState.TargetRestTime = math.random(67, 73)
                end
            end
        else
            clickState.StaminaStart = 0
        end

        -- DEMAIS AUTOMAÇÕES
        if C.AutoExtraUI and now-lastExtraUI>=0.04 then
            lastExtraUI=now
            local clicked=clickExtraUi()
            if clicked>0 then hub.Counts.ExtraUiClicks=hub.Counts.ExtraUiClicks+clicked end
        end
        
        if now-lastPurchase>=0.3 then
            lastPurchase=now
            if C.AutoBuy or C.AutoUpgrade or C.AutoMarket then
                local ok=attempt("Compras",executePlan)
                if not ok and (failures.Compras or 0)>=3 then
                    C.AutoBuy=false C.AutoUpgrade=false C.AutoMarket=false
                    log("Compras pausadas após 3 falhas",true)
                end
            end
        end
        
        if now-lastEvents>=0.08 then
            lastEvents=now
            automatic("AutoGolden",function()
                for _,g in ipairs(table.clone(driver.goldens())) do
                    if driver.clickGolden(g.id) then hub.Counts.Goldens=hub.Counts.Goldens+1 log("Moeda dourada coletada") end
                end
            end)
        end
        
        if now-lastReward>=2 then
            lastReward=now
            automatic("AutoLump",function()
                if driver.lumpStage()=="ripe" and driver.harvestLump() then hub.Counts.Harvests=hub.Counts.Harvests+1 log("Colheita enviada") end
            end)
            automatic("AutoReward",function()
                if not driver.rewardOwned() and driver.rewardReady() and driver.claimReward() then log("Resgate enviado") end
            end)
        end
        
        if now-lastWrinklers>=C.WrinklerInterval then
            lastWrinklers=now
            automatic("AutoWrinkler",function()
                local count=0
                for _,w in ipairs(table.clone(driver.wrinklers())) do if driver.popWrinkler(w.id) then count=count+1 end end
                if count>0 then hub.Counts.Wrinklers=hub.Counts.Wrinklers+count log("Wrinklers coletados: "..count) end
            end)
        end
        
        if now-lastSpell>=1 then
            lastSpell=now
            automatic("AutoSpell",function()
                local spell=selectedSpell()
                if spell and driver.spellReady(spell) and driver.mana()>=driver.spellCost(spell) and driver.castSpell(spell.id) then
                    hub.Counts.Spells=hub.Counts.Spells+1 log("Feitiço enviado: "..spell.name)
                end
            end)
        end
    end
    
    if now-lastRefresh>=0.35 then lastRefresh=now attempt("Interface",refresh) end
    if C.Verbose and now-lastVerbose>=60 then
        lastVerbose=now print("[27K HUB] Servidor:",hub.ServerRate,"cliques/s",hub.ServerIncome,"moedas/s; compras confirmadas:",hub.Confirmed,"sem confirmação:",hub.Rejected)
    end
end)

if ready() then applyPerformance() end
log("Hub pronto! Opções da Farm iniciadas desativadas por padrão.")
print("[27K HUB] 2.5 SMART BUYER + AUTOCLICK carregado.")
