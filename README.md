-- ==============================================================================
-- RD MENU VIP — BY RD
-- Farm de Baús + ESP + Aimbot + Speed + Mini Painel
-- ==============================================================================

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local VirtualUser = game:GetService("VirtualUser")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local Lighting = game:GetService("Lighting")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera

local LocalPlayer = Players.LocalPlayer

-- ==============================================================================
-- OTIMIZAÇÃO
-- ==============================================================================
pcall(function()
    settings().Rendering.QualityLevel = 1
    Lighting.GlobalShadows = false
    Lighting.FogEnd = 9e9
    Lighting.Brightness = 0
    for _, v in pairs(Lighting:GetDescendants()) do
        if v:IsA("BlurEffect") or v:IsA("SunRaysEffect") or v:IsA("BloomEffect") then
            v.Enabled = false
        end
    end
end)

local SaveFileName = "RDMenuVIP_Config.json"

-- ==============================================================================
-- CONFIGURAÇÕES
-- ==============================================================================
local SETTINGS = {
    -- Farm
    ChestNames = {"Chest1", "Chest2", "Chest3", "Chest4", "Chest5", "Chest6", "Chest"},
    Speed = 350,
    AutoFarm = true,
    StopOnRare = true,
    CollectionRange = 15,
    Hotkey = Enum.KeyCode.RightShift,
    SpeedHack = false,
    SpeedHackValor = 500,
    ContadorBaus = 0,
    TempoFarm = 0,
    TrocasServidor = 0,
    -- ESP
    ESPAtivo = false,
    ESPLinha = false,
    ESPBox = false,
    ESPNome = false,
    ESPVida = false,
    ESPVidaPct = false,
    ESPEsqueleto = false,
    -- Aimbot
    AimbotAtivo = false,
    AimbotTipo = "Camera",
    AimbotFOV = 150,
    AimbotSuavidade = 25,
    AimbotParte = "Head",
    AimbotPredicao = 0.15,
    AimbotKey = Enum.KeyCode.E,
    AimbotToggle = false,
}

local STATE = {
    IsMoving = false,
    CurrentTarget = nil,
    MoveConnection = nil
}

-- ==============================================================================
-- CACHE DE BAÚS
-- ==============================================================================
local ChestCache = {}

local function AddToCache(instance)
    if table.find(SETTINGS.ChestNames, instance.Name) then
        ChestCache[instance] = instance
    end
end

local function RemoveFromCache(instance)
    if ChestCache[instance] then
        ChestCache[instance] = nil
        if STATE.CurrentTarget == instance then
            STATE.CurrentTarget = nil
            STATE.IsMoving = false
            if STATE.MoveConnection then STATE.MoveConnection:Disconnect() end
        end
    end
end

for _, v in pairs(workspace:GetDescendants()) do AddToCache(v) end
workspace.DescendantAdded:Connect(AddToCache)
workspace.DescendantRemoving:Connect(RemoveFromCache)

-- ==============================================================================
-- SAVE / LOAD
-- ==============================================================================
local function SaveConfig()
    local data = {
        AutoFarm = SETTINGS.AutoFarm,
        StopOnRare = SETTINGS.StopOnRare,
        Hotkey = SETTINGS.Hotkey and SETTINGS.Hotkey.Name or "RightShift",
        SpeedHack = SETTINGS.SpeedHack,
        SpeedHackValor = SETTINGS.SpeedHackValor,
        ESPAtivo = SETTINGS.ESPAtivo,
        AimbotAtivo = SETTINGS.AimbotAtivo,
    }
    pcall(function()
        writefile(SaveFileName, HttpService:JSONEncode(data))
    end)
end

local function LoadConfig()
    pcall(function()
        if isfile(SaveFileName) then
            local success, result = pcall(function() return HttpService:JSONDecode(readfile(SaveFileName)) end)
            if success and result then
                if result.AutoFarm ~= nil then SETTINGS.AutoFarm = result.AutoFarm end
                if result.StopOnRare ~= nil then SETTINGS.StopOnRare = result.StopOnRare end
                if result.Hotkey ~= nil then SETTINGS.Hotkey = Enum.KeyCode[result.Hotkey] or Enum.KeyCode.RightShift end
                if result.SpeedHack ~= nil then SETTINGS.SpeedHack = result.SpeedHack end
                if result.SpeedHackValor ~= nil then SETTINGS.SpeedHackValor = result.SpeedHackValor end
                if result.ESPAtivo ~= nil then SETTINGS.ESPAtivo = result.ESPAtivo end
                if result.AimbotAtivo ~= nil then SETTINGS.AimbotAtivo = result.AimbotAtivo end
            end
        end
    end)
end
LoadConfig()

-- ==============================================================================
-- ANTI-AFK
-- ==============================================================================
LocalPlayer.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)

local RareItems = {"Fist of Darkness", "God's Chalice"}
local function CheckRareItems()
    if not SETTINGS.StopOnRare then return false end
    local found = false
    for _, item in ipairs(LocalPlayer.Backpack:GetChildren()) do if table.find(RareItems, item.Name) then found = true; break end end
    if not found and LocalPlayer.Character then for _, item in ipairs(LocalPlayer.Character:GetChildren()) do if table.find(RareItems, item.Name) then found = true; break end end end
    return found
end

-- ==============================================================================
-- MOVIMENTAÇÃO (farm original)
-- ==============================================================================
local function Collect(chest)
    if not chest or not chest.Parent then return end
    local function fire(target)
        if target:IsA("BasePart") then
            firetouchinterest(LocalPlayer.Character.HumanoidRootPart, target, 0)
            firetouchinterest(LocalPlayer.Character.HumanoidRootPart, target, 1)
        end
    end
    if chest:IsA("Model") then for _, v in pairs(chest:GetChildren()) do fire(v) end else fire(chest) end
    VirtualInputManager:SendMouseButtonEvent(443, 275, 0, true, game, 1)
    VirtualInputManager:SendMouseButtonEvent(443, 275, 0, false, game, 1)
    SETTINGS.ContadorBaus = SETTINGS.ContadorBaus + 1
end

local function MoveToTarget(target)
    if STATE.IsMoving then return end
    STATE.IsMoving = true
    STATE.CurrentTarget = target
    if STATE.MoveConnection then STATE.MoveConnection:Disconnect() end

    STATE.MoveConnection = RunService.Heartbeat:Connect(function(dt)
        if not SETTINGS.AutoFarm or CheckRareItems() then
            STATE.IsMoving = false
            STATE.MoveConnection:Disconnect()
            return
        end
        local Char = LocalPlayer.Character
        local Root = Char and Char:FindFirstChild("HumanoidRootPart")
        local Hum = Char and Char:FindFirstChildOfClass("Humanoid")
        if not Root or not Hum or Hum.Health <= 0 then
            STATE.IsMoving = false
            STATE.MoveConnection:Disconnect()
            return
        end
        if not target.Parent then
            STATE.IsMoving = false
            STATE.MoveConnection:Disconnect()
            return
        end
        local DestPos = (target:IsA("Model") and target:GetModelCFrame().Position) or target.Position
        local Dist = (DestPos - Root.Position).Magnitude
        if Dist <= SETTINGS.CollectionRange then
            Collect(target)
            STATE.IsMoving = false
            STATE.MoveConnection:Disconnect()
            ChestCache[target] = nil
            return
        end
        local Dir = (DestPos - Root.Position).Unit
        local Velocity = Dir * SETTINGS.Speed * dt
        if Velocity.Magnitude >= Dist then
            Root.CFrame = CFrame.new(DestPos)
        else
            Root.CFrame = Root.CFrame + Velocity
        end
        Root.Velocity = Vector3.new(0,0,0)
        for _, v in pairs(Char:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end)
end

LocalPlayer.CharacterAdded:Connect(function()
    if STATE.MoveConnection then STATE.MoveConnection:Disconnect() end
    STATE.IsMoving = false
    STATE.CurrentTarget = nil
    task.wait(1)
end)

-- ==============================================================================
-- SPEED (seguro - ativa/desativa por tempo)
-- ==============================================================================
local SpeedTimer = 0
local SpeedFase = "off"

RunService.Stepped:Connect(function(dt)
    if SETTINGS.SpeedHack then
        SpeedTimer = SpeedTimer + dt
        local char = LocalPlayer.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                if SpeedFase == "off" and SpeedTimer >= 3 then
                    hum.WalkSpeed = SETTINGS.SpeedHackValor
                    SpeedFase = "on"
                    SpeedTimer = 0
                elseif SpeedFase == "on" and SpeedTimer >= 5 then
                    hum.WalkSpeed = 16
                    SpeedFase = "off"
                    SpeedTimer = 0
                end
            end
        end
    else
        SpeedFase = "off"
        SpeedTimer = 0
    end
end)

-- ==============================================================================
-- ESP (Linha, Box, Nome, Vida, Porcentagem, Esqueleto)
-- ==============================================================================
local ESPDrawings = {}

local function CriarESP(p)
    if p == LocalPlayer or ESPDrawings[p] then return end
    local d = {}
    d.linha = Drawing.new("Line"); d.linha.Thickness = 1; d.linha.Color = Color3.fromRGB(0, 230, 255); d.linha.Visible = false
    d.box = Drawing.new("Square"); d.box.Thickness = 1; d.box.Filled = false; d.box.Color = Color3.fromRGB(0, 230, 255); d.box.Visible = false
    d.nome = Drawing.new("Text"); d.nome.Size = 14; d.nome.Center = true; d.nome.Outline = true; d.nome.Color = Color3.fromRGB(255, 255, 255); d.nome.Visible = false
    d.vida = Drawing.new("Line"); d.vida.Thickness = 4; d.vida.Color = Color3.fromRGB(0, 255, 0); d.vida.Visible = false
    d.vidaPct = Drawing.new("Text"); d.vidaPct.Size = 11; d.vidaPct.Center = true; d.vidaPct.Outline = true; d.vidaPct.Color = Color3.fromRGB(0, 255, 0); d.vidaPct.Visible = false
    d.esqueleto = {}
    for i = 1, 7 do
        local l = Drawing.new("Line"); l.Thickness = 1; l.Color = Color3.fromRGB(0, 230, 255); l.Visible = false
        table.insert(d.esqueleto, l)
    end
    ESPDrawings[p] = d
end

local function RemoverESP(p)
    if not ESPDrawings[p] then return end
    local d = ESPDrawings[p]
    for _, obj in pairs(d) do
        if type(obj) == "table" then for _, l in pairs(obj) do if l and l.Remove then l:Remove() end end
        elseif obj and obj.Remove then obj:Remove() end
    end
    ESPDrawings[p] = nil
end

local function AtualizarESP(p)
    local d = ESPDrawings[p]; if not d then return end
    local ch = p.Character
    if not ch or not ch:FindFirstChild("HumanoidRootPart") or not ch:FindFirstChild("Humanoid") then
        for _, obj in pairs(d) do
            if type(obj) == "table" then for _, l in pairs(obj) do if l then l.Visible = false end end
            elseif obj then obj.Visible = false end
        end
        return
    end
    local hum = ch:FindFirstChildOfClass("Humanoid")
    local hrp = ch.HumanoidRootPart
    if not hum or hum.Health <= 0 then
        for _, obj in pairs(d) do
            if type(obj) == "table" then for _, l in pairs(obj) do if l then l.Visible = false end end
            elseif obj then obj.Visible = false end
        end
        return
    end
    local rootPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
    if not onScreen then
        for _, obj in pairs(d) do
            if type(obj) == "table" then for _, l in pairs(obj) do if l then l.Visible = false end end
            elseif obj then obj.Visible = false end
        end
        return
    end
    local head = ch:FindFirstChild("Head")
    local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
    local legPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))
    local altura = math.abs(headPos.Y - legPos.Y)
    local largura = altura * 0.5
    local cx = rootPos.X
    local cy = rootPos.Y - altura * 0.2

    -- Linha
    if SETTINGS.ESPAtivo and SETTINGS.ESPLinha then
        d.linha.Visible = true
        d.linha.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
        d.linha.To = Vector2.new(cx, cy + altura / 2)
    else d.linha.Visible = false end

    -- Box
    if SETTINGS.ESPAtivo and SETTINGS.ESPBox then
        d.box.Visible = true
        d.box.Size = Vector2.new(largura, altura)
        d.box.Position = Vector2.new(cx - largura / 2, cy - altura / 2)
    else d.box.Visible = false end

    -- Nome
    if SETTINGS.ESPAtivo and SETTINGS.ESPNome then
        d.nome.Visible = true
        d.nome.Text = p.Name
        d.nome.Position = Vector2.new(cx, cy - altura / 2 - 16)
    else d.nome.Visible = false end

    -- Vida (barra)
    if SETTINGS.ESPAtivo and SETTINGS.ESPVida then
        local pct = hum.Health / hum.MaxHealth
        local bx = cx - largura / 2 - 6
        local by = cy - altura / 2
        d.vida.Visible = true
        d.vida.From = Vector2.new(bx, by + altura * (1 - pct))
        d.vida.To = Vector2.new(bx, by + altura)
        d.vida.Color = Color3.fromRGB(255 * (1 - pct), 255 * pct, 0)
    else d.vida.Visible = false end

    -- Vida porcentagem
    if SETTINGS.ESPAtivo and SETTINGS.ESPVidaPct then
        local pct = math.floor((hum.Health / hum.MaxHealth) * 100)
        d.vidaPct.Visible = true
        d.vidaPct.Text = pct .. "%"
        d.vidaPct.Position = Vector2.new(cx, cy - altura / 2 - 30)
    else d.vidaPct.Visible = false end

    -- Esqueleto
    if SETTINGS.ESPAtivo and SETTINGS.ESPEsqueleto and head then
        local ps = {head, ch:FindFirstChild("UpperTorso") or ch:FindFirstChild("Torso"), ch:FindFirstChild("LowerTorso") or ch:FindFirstChild("HumanoidRootPart"), ch:FindFirstChild("LeftUpperArm") or ch:FindFirstChild("Left Arm"), ch:FindFirstChild("RightUpperArm") or ch:FindFirstChild("Right Arm"), ch:FindFirstChild("LeftUpperLeg") or ch:FindFirstChild("Left Leg"), ch:FindFirstChild("RightUpperLeg") or ch:FindFirstChild("Right Leg")}
        local cs = {{1,2},{2,3},{2,4},{2,5},{3,6},{3,7}}
        for _, l in pairs(d.esqueleto) do l.Visible = false end
        for i, co in ipairs(cs) do
            local p1, p2 = ps[co[1]], ps[co[2]]
            if p1 and p2 then
                local s1, o1 = Camera:WorldToViewportPoint(p1.Position)
                local s2, o2 = Camera:WorldToViewportPoint(p2.Position)
                if o1 and o2 then
                    local l = d.esqueleto[i]
                    l.From = Vector2.new(s1.X, s1.Y)
                    l.To = Vector2.new(s2.X, s2.Y)
                    l.Visible = true
                end
            end
        end
    else for _, l in pairs(d.esqueleto) do l.Visible = false end end
end

for _, p in ipairs(Players:GetPlayers()) do if p ~= LocalPlayer then CriarESP(p) end end
Players.PlayerAdded:Connect(function(p) if p ~= LocalPlayer then CriarESP(p) end end)
Players.PlayerRemoving:Connect(RemoverESP)

RunService.RenderStepped:Connect(function()
    for p, _ in pairs(ESPDrawings) do
        if p.Parent then AtualizarESP(p) else RemoverESP(p) end
    end
end)

-- ==============================================================================
-- AIMBOT (Camera, Mouse, Silent)
-- ==============================================================================
local function GetParteAlvo(ch)
    if not ch then return nil end
    local p = SETTINGS.AimbotParte
    if p == "Head" then return ch:FindFirstChild("Head")
    elseif p == "Neck" then return ch:FindFirstChild("Neck") or ch:FindFirstChild("UpperTorso") or ch:FindFirstChild("Torso")
    elseif p == "Chest" then return ch:FindFirstChild("UpperTorso") or ch:FindFirstChild("Torso")
    elseif p == "Hip" then return ch:FindFirstChild("LowerTorso") or ch:FindFirstChild("Torso") end
    return ch:FindFirstChild("Head")
end

local function GetMelhorAlvo()
    local melhor, ms = nil, math.huge
    local ct = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local mp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and LocalPlayer.Character.HumanoidRootPart.Position or Camera.CFrame.Position
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local ch = p.Character
            local hum = ch:FindFirstChildOfClass("Humanoid")
            local hrp = ch:FindFirstChild("HumanoidRootPart")
            if hum and hum.Health > 0 and hrp then
                local pt = GetParteAlvo(ch)
                if pt then
                    local dr = (hrp.Position - mp).Magnitude
                    if dr <= 500 then
                        local sp, on = Camera:WorldToViewportPoint(pt.Position)
                        if on then
                            local dt = (Vector2.new(sp.X, sp.Y) - ct).Magnitude
                            if dt <= SETTINGS.AimbotFOV then
                                local sc = dt + dr * 0.05
                                if sc < ms then ms = sc; melhor = pt end
                            end
                        end
                    end
                end
            end
        end
    end
    return melhor
end

local function AplicarAimbot()
    if not SETTINGS.AimbotAtivo then return end
    local alvo = GetMelhorAlvo()
    if not alvo then return end
    local pp = alvo.Position + (alvo.AssemblyLinearVelocity or Vector3.zero) * SETTINGS.AimbotPredicao
    if SETTINGS.AimbotTipo == "Camera" then
        local cf = CFrame.new(Camera.CFrame.Position, pp)
        Camera.CFrame = Camera.CFrame:Lerp(cf, math.clamp(SETTINGS.AimbotSuavidade / 100, 0.01, 1))
    elseif SETTINGS.AimbotTipo == "Mouse" then
        local screenPos, on = Camera:WorldToViewportPoint(pp)
        if on and mousemoverel then
            local dx = screenPos.X - Camera.ViewportSize.X / 2
            local dy = screenPos.Y - Camera.ViewportSize.Y / 2
            local sens = (100 - SETTINGS.AimbotSuavidade) / 1000
            mousemoverel(dx * sens, dy * sens)
        end
    elseif SETTINGS.AimbotTipo == "Silent" then
        local cf = CFrame.new(Camera.CFrame.Position, pp)
        Camera.CFrame = Camera.CFrame:Lerp(cf, 0.5)
    end
end

RunService.RenderStepped:Connect(function()
    if SETTINGS.AimbotAtivo and (SETTINGS.AimbotToggle or UserInputService:IsKeyDown(SETTINGS.AimbotKey)) then
        AplicarAimbot()
    end
end)

-- ==============================================================================
-- SERVER HOP & PIRATES
-- ==============================================================================
local function JoinPirates()
    if LocalPlayer.Team and LocalPlayer.Team.Name == "Pirates" then return end
    task.spawn(function()
        local Start = tick()
        repeat
            if tick() - Start > 10 then break end
            pcall(function()
                local Net = ReplicatedStorage:WaitForChild("Modules"):WaitForChild("Net")
                local Remote = Net:WaitForChild("RE/OnEventServiceActivity")
                Remote:FireServer("TeamSelect/Team/Pirates")
            end)
            task.wait(1)
        until LocalPlayer.Team and LocalPlayer.Team.Name == "Pirates"
    end)
end

local function ServerHop()
    SETTINGS.TrocasServidor = SETTINGS.TrocasServidor + 1
    local PlaceId = game.PlaceId
    local function ExecHop()
        local req = game:HttpGet("https://games.roblox.com/v1/games/" .. PlaceId .. "/servers/Public?sortOrder=Asc&limit=100")
        local data = HttpService:JSONDecode(req)
        local servers = {}
        for _, s in pairs(data.data) do
            if s.playing < s.maxPlayers and s.id ~= game.JobId then table.insert(servers, s.id) end
        end
        if #servers > 0 then
            TeleportService:TeleportToPlaceInstance(PlaceId, servers[math.random(1, #servers)], LocalPlayer)
        else
            task.wait(1); ExecHop()
        end
    end
    pcall(ExecHop)
end

-- ==============================================================================
-- LOOP PRINCIPAL DO FARM
-- ==============================================================================
local function GetNearestChest()
    local Root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not Root then return nil end
    local Nearest = nil
    local MinDist = math.huge
    for chest, _ in pairs(ChestCache) do
        if chest and chest.Parent then
            local Pos = (chest:IsA("Model") and chest:GetModelCFrame().Position) or chest.Position
            local Dist = (Pos - Root.Position).Magnitude
            if Dist < MinDist then
                MinDist = Dist
                Nearest = chest
            end
        else
            ChestCache[chest] = nil
        end
    end
    return Nearest
end

task.spawn(function()
    while true do
        task.wait()
        if CheckRareItems() then
            SETTINGS.AutoFarm = false
            SaveConfig()
            continue
        end
        if SETTINGS.AutoFarm then
            SETTINGS.TempoFarm = SETTINGS.TempoFarm + 0.1
            local Char = LocalPlayer.Character
            if not Char or not Char:FindFirstChild("HumanoidRootPart") or (Char:FindFirstChildOfClass("Humanoid") and Char.Humanoid.Health <= 0) then
                task.wait(1)
                continue
            end
            if not STATE.IsMoving then
                local Chest = GetNearestChest()
                if Chest then
                    MoveToTarget(Chest)
                else
                    task.wait(1.5)
                    ServerHop()
                    task.wait(10)
                end
            end
        else
            if STATE.MoveConnection then STATE.MoveConnection:Disconnect() end
            STATE.IsMoving = false
            task.wait(0.5)
        end
    end
end)

task.spawn(function()
    while true do
        if SETTINGS.AutoFarm then
            VirtualInputManager:SendMouseButtonEvent(443, 275, 0, true, game, 1)
            task.wait()
            VirtualInputManager:SendMouseButtonEvent(443, 275, 0, false, game, 1)
        end
        task.wait(0.05)
    end
end)

-- ==============================================================================
-- INTERFACE RD MENU VIP
-- ==============================================================================
local Viewport = game.CoreGui
pcall(function() if gethui then Viewport = gethui() end end)

if Viewport:FindFirstChild("RDMenuVIP") then Viewport.RDMenuVIP:Destroy() end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "RDMenuVIP"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = Viewport

local TEMA = {
    Fundo = Color3.fromRGB(10, 12, 18),
    Header = Color3.fromRGB(14, 16, 24),
    Card = Color3.fromRGB(20, 24, 34),
    CardHover = Color3.fromRGB(28, 34, 48),
    CardAtivo = Color3.fromRGB(0, 80, 110),
    Azul = Color3.fromRGB(0, 230, 255),
    AzulClaro = Color3.fromRGB(120, 240, 255),
    Branco = Color3.fromRGB(235, 240, 250),
    Cinza = Color3.fromRGB(130, 140, 160),
    Borda = Color3.fromRGB(0, 60, 90),
}

-- TÍTULO FIXO RGB
local TituloFixo = Instance.new("TextLabel")
TituloFixo.Size = UDim2.new(0, 300, 0, 30)
TituloFixo.Position = UDim2.new(0.5, -150, 0, 5)
TituloFixo.BackgroundColor3 = Color3.fromRGB(10, 12, 18)
TituloFixo.BackgroundTransparency = 0.5
TituloFixo.Text = "👑 RD CHEATS 00"
TituloFixo.TextColor3 = Color3.fromRGB(0, 230, 255)
TituloFixo.Font = Enum.Font.GothamBlack
TituloFixo.TextSize = 16
TituloFixo.Parent = ScreenGui
Instance.new("UICorner", TituloFixo).CornerRadius = UDim.new(0, 8)
local TituloStroke = Instance.new("UIStroke")
TituloStroke.Color = Color3.fromRGB(0, 230, 255)
TituloStroke.Thickness = 1.5
TituloStroke.Parent = TituloFixo

task.spawn(function()
    while true do
        task.wait(0.1)
        if TituloFixo and TituloFixo.Parent then
            local hue = (tick() * 0.5) % 1
            local cor = Color3.fromHSV(hue, 1, 1)
            TituloFixo.TextColor3 = cor
            TituloStroke.Color = cor
        end
    end
end)

-- PAINEL PRINCIPAL
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 520, 0, 420)
MainFrame.Position = UDim2.new(0.5, -260, 0.5, -210)
MainFrame.BackgroundColor3 = TEMA.Fundo
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Visible = true
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 12)
local Stroke = Instance.new("UIStroke")
Stroke.Color = TEMA.Azul
Stroke.Thickness = 1.5
Stroke.Transparency = 0.3
Stroke.Parent = MainFrame

local TopLine = Instance.new("Frame")
TopLine.Size = UDim2.new(1, 0, 0, 2)
TopLine.BackgroundColor3 = TEMA.Azul
TopLine.BorderSizePixel = 0
TopLine.Parent = MainFrame

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 56)
Header.BackgroundColor3 = TEMA.Header
Header.BorderSizePixel = 0
Header.Parent = MainFrame
Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 12)

local LogoBox = Instance.new("Frame")
LogoBox.Size = UDim2.new(0, 36, 0, 36)
LogoBox.Position = UDim2.new(0, 14, 0, 10)
LogoBox.BackgroundColor3 = TEMA.Azul
LogoBox.BorderSizePixel = 0
LogoBox.Parent = Header
Instance.new("UICorner", LogoBox).CornerRadius = UDim.new(0, 9)

local LogoText = Instance.new("TextLabel")
LogoText.Size = UDim2.new(1, 0, 1, 0)
LogoText.BackgroundTransparency = 1
LogoText.Text = "RD"
LogoText.TextColor3 = TEMA.Fundo
LogoText.Font = Enum.Font.GothamBlack
LogoText.TextSize = 16
LogoText.Parent = LogoBox

local Titulo = Instance.new("TextLabel")
Titulo.Size = UDim2.new(0, 300, 0, 20)
Titulo.Position = UDim2.new(0, 58, 0, 10)
Titulo.BackgroundTransparency = 1
Titulo.Text = "RD MENU VIP"
Titulo.TextColor3 = TEMA.AzulClaro
Titulo.Font = Enum.Font.GothamBlack
Titulo.TextSize = 16
Titulo.TextXAlignment = Enum.TextXAlignment.Left
Titulo.Parent = Header

local Assinatura = Instance.new("TextLabel")
Assinatura.Size = UDim2.new(0, 300, 0, 14)
Assinatura.Position = UDim2.new(0, 58, 0, 30)
Assinatura.BackgroundTransparency = 1
Assinatura.Text = "BY RD"
Assinatura.TextColor3 = TEMA.Cinza
Assinatura.Font = Enum.Font.GothamMedium
Assinatura.TextSize = 10
Assinatura.TextXAlignment = Enum.TextXAlignment.Left
Assinatura.Parent = Header

local BtnFechar = Instance.new("TextButton")
BtnFechar.Size = UDim2.new(0, 26, 0, 26)
BtnFechar.Position = UDim2.new(1, -36, 0, 15)
BtnFechar.BackgroundColor3 = TEMA.Card
BtnFechar.Text = "X"
BtnFechar.TextColor3 = TEMA.Cinza
BtnFechar.Font = Enum.Font.GothamBold
BtnFechar.TextSize = 12
BtnFechar.BorderSizePixel = 0
BtnFechar.Parent = Header
Instance.new("UICorner", BtnFechar).CornerRadius = UDim.new(0, 6)

local BtnOcultar = Instance.new("TextButton")
BtnOcultar.Size = UDim2.new(0, 60, 0, 26)
BtnOcultar.Position = UDim2.new(1, -102, 0, 15)
BtnOcultar.BackgroundColor3 = TEMA.Card
BtnOcultar.Text = "OCULTAR"
BtnOcultar.TextColor3 = TEMA.Cinza
BtnOcultar.Font = Enum.Font.GothamBold
BtnOcultar.TextSize = 9
BtnOcultar.BorderSizePixel = 0
BtnOcultar.Parent = Header
Instance.new("UICorner", BtnOcultar).CornerRadius = UDim.new(0, 6)

local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, 150, 1, -76)
Sidebar.Position = UDim2.new(0, 10, 0, 64)
Sidebar.BackgroundColor3 = TEMA.Header
Sidebar.BorderSizePixel = 0
Sidebar.Parent = MainFrame
Instance.new("UICorner", Sidebar).CornerRadius = UDim.new(0, 8)
local SidebarStroke = Instance.new("UIStroke")
SidebarStroke.Color = TEMA.Borda
SidebarStroke.Thickness = 1
SidebarStroke.Parent = Sidebar
local SidebarLayout = Instance.new("UIListLayout")
SidebarLayout.Padding = UDim.new(0, 5)
SidebarLayout.Parent = Sidebar
local SidebarPadding = Instance.new("UIPadding")
SidebarPadding.PaddingTop = UDim.new(0, 8)
SidebarPadding.PaddingLeft = UDim.new(0, 6)
SidebarPadding.PaddingRight = UDim.new(0, 6)
SidebarPadding.Parent = Sidebar

local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, -180, 1, -76)
Content.Position = UDim2.new(0, 168, 0, 64)
Content.BackgroundColor3 = TEMA.Header
Content.BorderSizePixel = 0
Content.Parent = MainFrame
Instance.new("UICorner", Content).CornerRadius = UDim.new(0, 8)
local ContentStroke = Instance.new("UIStroke")
ContentStroke.Color = TEMA.Borda
ContentStroke.Thickness = 1
ContentStroke.Parent = Content

local abas = {}
local botoesAbas = {}

local function NovaAba()
    local scroll = Instance.new("ScrollingFrame")
    scroll.Size = UDim2.new(1, -10, 1, -10)
    scroll.Position = UDim2.new(0, 5, 0, 5)
    scroll.BackgroundTransparency = 1
    scroll.ScrollBarThickness = 3
    scroll.ScrollBarImageColor3 = TEMA.Azul
    scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    scroll.Visible = false
    scroll.Parent = Content
    local layout = Instance.new("UIListLayout")
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 5)
    layout.Parent = scroll
    return scroll
end

local function Secao(parent, txt)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, -8, 0, 20)
    l.BackgroundTransparency = 1
    l.Text = txt:upper()
    l.TextColor3 = TEMA.AzulClaro
    l.Font = Enum.Font.GothamBold
    l.TextSize = 10
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = parent
    return l
end

local function Toggle(parent, nome, key, cb)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -8, 0, 30)
    b.BackgroundColor3 = TEMA.Card
    b.Text = ""
    b.BorderSizePixel = 0
    b.AutoButtonColor = false
    b.Parent = parent
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
    local stroke = Instance.new("UIStroke")
    stroke.Color = TEMA.Borda
    stroke.Thickness = 1
    stroke.Parent = b
    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0, 8, 0, 8)
    dot.Position = UDim2.new(0, 10, 0.5, -4)
    dot.BackgroundColor3 = TEMA.Cinza
    dot.BorderSizePixel = 0
    dot.Parent = b
    Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)
    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, -40, 1, 0)
    txt.Position = UDim2.new(0, 26, 0, 0)
    txt.BackgroundTransparency = 1
    txt.Text = nome
    txt.TextColor3 = TEMA.Cinza
    txt.Font = Enum.Font.GothamMedium
    txt.TextSize = 11
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.Parent = b
    local estado = SETTINGS[key] or false
    local function aplicar()
        if estado then
            b.BackgroundColor3 = TEMA.CardAtivo
            stroke.Color = TEMA.Azul
            dot.BackgroundColor3 = TEMA.AzulClaro
            txt.TextColor3 = TEMA.Branco
        else
            b.BackgroundColor3 = TEMA.Card
            stroke.Color = TEMA.Borda
            dot.BackgroundColor3 = TEMA.Cinza
            txt.TextColor3 = TEMA.Cinza
        end
    end
    aplicar()
    b.MouseButton1Click:Connect(function()
        estado = not estado
        SETTINGS[key] = estado
        aplicar()
        if cb then cb(estado) end
        SaveConfig()
    end)
    return b
end

local function Botao(parent, nome, cb)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -8, 0, 30)
    b.BackgroundColor3 = TEMA.Card
    b.Text = nome
    b.TextColor3 = TEMA.Branco
    b.Font = Enum.Font.GothamBold
    b.TextSize = 11
    b.BorderSizePixel = 0
    b.Parent = parent
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
    local stroke = Instance.new("UIStroke")
    stroke.Color = TEMA.Borda
    stroke.Thickness = 1
    stroke.Parent = b
    b.MouseButton1Click:Connect(function() if cb then cb() end end)
    return b
end

local abaMAIN = NovaAba()
local abaFARM = NovaAba()
local abaESP = NovaAba()
local abaAIM = NovaAba()
local abaMOVE = NovaAba()
local abaSERVER = NovaAba()
local abaSETTINGS = NovaAba()

abas["MAIN"] = abaMAIN
abas["FARM"] = abaFARM
abas["ESP"] = abaESP
abas["AIMBOT"] = abaAIM
abas["MOVIMENTO"] = abaMOVE
abas["SERVIDOR"] = abaSERVER
abas["CONFIG"] = abaSETTINGS

-- MAIN
Secao(abaMAIN, "Status")
local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -8, 0, 22)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Aguardando..."
StatusLabel.TextColor3 = TEMA.AzulClaro
StatusLabel.Font = Enum.Font.Code
StatusLabel.TextSize = 11
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Parent = abaMAIN

Secao(abaMAIN, "Controles")
Toggle(abaMAIN, "FARM ATIVAR/DESATIVAR", "AutoFarm")
Toggle(abaMAIN, "STOP ON RARE", "StopOnRare")

Secao(abaMAIN, "Estatísticas")
local ContadorLabel = Instance.new("TextLabel")
ContadorLabel.Size = UDim2.new(1, -8, 0, 20)
ContadorLabel.BackgroundTransparency = 1
ContadorLabel.Text = "Baús Coletados: 0"
ContadorLabel.TextColor3 = TEMA.AzulClaro
ContadorLabel.Font = Enum.Font.GothamMedium
ContadorLabel.TextSize = 11
ContadorLabel.TextXAlignment = Enum.TextXAlignment.Left
ContadorLabel.Parent = abaMAIN

local TempoLabel = Instance.new("TextLabel")
TempoLabel.Size = UDim2.new(1, -8, 0, 20)
TempoLabel.BackgroundTransparency = 1
TempoLabel.Text = "Tempo de Farm: 0s"
TempoLabel.TextColor3 = TEMA.AzulClaro
TempoLabel.Font = Enum.Font.GothamMedium
TempoLabel.TextSize = 11
TempoLabel.TextXAlignment = Enum.TextXAlignment.Left
TempoLabel.Parent = abaMAIN

-- FARM
Secao(abaFARM, "Informações")
local BausLabel = Instance.new("TextLabel")
BausLabel.Size = UDim2.new(1, -8, 0, 20)
BausLabel.BackgroundTransparency = 1
BausLabel.Text = "Baús no Mapa: 0"
BausLabel.TextColor3 = TEMA.Cinza
BausLabel.Font = Enum.Font.GothamMedium
BausLabel.TextSize = 11
BausLabel.TextXAlignment = Enum.TextXAlignment.Left
BausLabel.Parent = abaFARM

local DistanciaLabel = Instance.new("TextLabel")
DistanciaLabel.Size = UDim2.new(1, -8, 0, 20)
DistanciaLabel.BackgroundTransparency = 1
DistanciaLabel.Text = "Baú Mais Próximo: --"
DistanciaLabel.TextColor3 = TEMA.Cinza
DistanciaLabel.Font = Enum.Font.GothamMedium
DistanciaLabel.TextSize = 11
DistanciaLabel.TextXAlignment = Enum.TextXAlignment.Left
DistanciaLabel.Parent = abaFARM

-- ESP
Secao(abaESP, "ESP Principal")
Toggle(abaESP, "ESP ATIVAR", "ESPAtivo")

Secao(abaESP, "Elementos")
Toggle(abaESP, "ESP LINHA", "ESPLinha")
Toggle(abaESP, "ESP BOX", "ESPBox")
Toggle(abaESP, "ESP NOME", "ESPNome")
Toggle(abaESP, "ESP VIDA", "ESPVida")
Toggle(abaESP, "ESP VIDA PORCENTAGEM", "ESPVidaPct")
Toggle(abaESP, "ESP ESQUELETO", "ESPEsqueleto")

-- AIMBOT
Secao(abaAIM, "Aimbot")
Toggle(abaAIM, "AIMBOT ATIVAR", "AimbotAtivo")
Toggle(abaAIM, "AIMBOT TOGGLE (SEM SEGURAR)", "AimbotToggle")

Secao(abaAIM, "Tipo de Aimbot")
local tipos = {"Camera", "Mouse", "Silent"}
local tipoBtns = {}
local tipoFrame = Instance.new("Frame")
tipoFrame.Size = UDim2.new(1, -8, 0, 32)
tipoFrame.BackgroundColor3 = TEMA.Card
tipoFrame.BorderSizePixel = 0
tipoFrame.Parent = abaAIM
Instance.new("UICorner", tipoFrame).CornerRadius = UDim.new(0, 6)
local tfl = Instance.new("UIListLayout")
tfl.FillDirection = Enum.FillDirection.Horizontal
tfl.Padding = UDim.new(0, 4)
tfl.Parent = tipoFrame
local tfp = Instance.new("UIPadding")
tfp.PaddingLeft = UDim.new(0, 4)
tfp.PaddingTop = UDim.new(0, 4)
tfp.Parent = tipoFrame
local function atualizarTipos()
    for t, b in pairs(tipoBtns) do
        if SETTINGS.AimbotTipo == t then
            b.BackgroundColor3 = TEMA.CardAtivo
            b.TextColor3 = TEMA.Branco
        else
            b.BackgroundColor3 = TEMA.CardHover
            b.TextColor3 = TEMA.Cinza
        end
    end
end
for _, t in ipairs(tipos) do
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, 80, 0, 24)
    b.BackgroundColor3 = TEMA.CardHover
    b.Text = t
    b.TextColor3 = TEMA.Cinza
    b.Font = Enum.Font.GothamBold
    b.TextSize = 10
    b.BorderSizePixel = 0
    b.Parent = tipoFrame
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 4)
    tipoBtns[t] = b
    b.MouseButton1Click:Connect(function()
        SETTINGS.AimbotTipo = t
        atualizarTipos()
        SaveConfig()
    end)
end
atualizarTipos()

Secao(abaAIM, "Configurações")
local FOVLabel = Instance.new("TextLabel")
FOVLabel.Size = UDim2.new(1, -8, 0, 20)
FOVLabel.BackgroundTransparency = 1
FOVLabel.Text = "FOV: " .. SETTINGS.AimbotFOV
FOVLabel.TextColor3 = TEMA.Cinza
FOVLabel.Font = Enum.Font.GothamMedium
FOVLabel.TextSize = 11
FOVLabel.TextXAlignment = Enum.TextXAlignment.Left
FOVLabel.Parent = abaAIM

local FOVBtn = Instance.new("TextButton")
FOVBtn.Size = UDim2.new(1, -8, 0, 30)
FOVBtn.BackgroundColor3 = TEMA.Card
FOVBtn.Text = "AUMENTAR FOV (+50)"
FOVBtn.TextColor3 = TEMA.Branco
FOVBtn.Font = Enum.Font.GothamBold
FOVBtn.TextSize = 11
FOVBtn.BorderSizePixel = 0
FOVBtn.Parent = abaAIM
Instance.new("UICorner", FOVBtn).CornerRadius = UDim.new(0, 6)
FOVBtn.MouseButton1Click:Connect(function()
    SETTINGS.AimbotFOV = math.min(SETTINGS.AimbotFOV + 50, 600)
    FOVLabel.Text = "FOV: " .. SETTINGS.AimbotFOV
    SaveConfig()
end)

local SuavLabel = Instance.new("TextLabel")
SuavLabel.Size = UDim2.new(1, -8, 0, 20)
SuavLabel.BackgroundTransparency = 1
SuavLabel.Text = "Suavidade: " .. SETTINGS.AimbotSuavidade
SuavLabel.TextColor3 = TEMA.Cinza
SuavLabel.Font = Enum.Font.GothamMedium
SuavLabel.TextSize = 11
SuavLabel.TextXAlignment = Enum.TextXAlignment.Left
SuavLabel.Parent = abaAIM

local SuavBtn = Instance.new("TextButton")
SuavBtn.Size = UDim2.new(1, -8, 0, 30)
SuavBtn.BackgroundColor3 = TEMA.Card
SuavBtn.Text = "AUMENTAR SUAVIDADE (+5)"
SuavBtn.TextColor3 = TEMA.Branco
SuavBtn.Font = Enum.Font.GothamBold
SuavBtn.TextSize = 11
SuavBtn.BorderSizePixel = 0
SuavBtn.Parent = abaAIM
Instance.new("UICorner", SuavBtn).CornerRadius = UDim.new(0, 6)
SuavBtn.MouseButton1Click:Connect(function()
    SETTINGS.AimbotSuavidade = math.min(SETTINGS.AimbotSuavidade + 5, 100)
    SuavLabel.Text = "Suavidade: " .. SETTINGS.AimbotSuavidade
    SaveConfig()
end)

-- MOVIMENTO
Secao(abaMOVE, "Speed")
Toggle(abaMOVE, "SPEED (RISCO)", "SpeedHack")
local SpeedValorLabel = Instance.new("TextLabel")
SpeedValorLabel.Size = UDim2.new(1, -8, 0, 20)
SpeedValorLabel.BackgroundTransparency = 1
SpeedValorLabel.Text = "Velocidade: " .. SETTINGS.SpeedHackValor
SpeedValorLabel.TextColor3 = TEMA.Cinza
SpeedValorLabel.Font = Enum.Font.GothamMedium
SpeedValorLabel.TextSize = 11
SpeedValorLabel.TextXAlignment = Enum.TextXAlignment.Left
SpeedValorLabel.Parent = abaMOVE

-- SERVIDOR
Secao(abaSERVER, "Ações")
Botao(abaSERVER, "SERVER HOP (RISCO)", function() ServerHop() end)
Botao(abaSERVER, "ENTRAR NO TIME PIRATES", function() JoinPirates() end)

Secao(abaSERVER, "Informações")
local TrocasLabel = Instance.new("TextLabel")
TrocasLabel.Size = UDim2.new(1, -8, 0, 20)
TrocasLabel.BackgroundTransparency = 1
TrocasLabel.Text = "Trocas de Servidor: 0"
TrocasLabel.TextColor3 = TEMA.Cinza
TrocasLabel.Font = Enum.Font.GothamMedium
TrocasLabel.TextSize = 11
TrocasLabel.TextXAlignment = Enum.TextXAlignment.Left
TrocasLabel.Parent = abaSERVER

local TimeLabel = Instance.new("TextLabel")
TimeLabel.Size = UDim2.new(1, -8, 0, 20)
TimeLabel.BackgroundTransparency = 1
TimeLabel.Text = "Time Atual: --"
TimeLabel.TextColor3 = TEMA.Cinza
TimeLabel.Font = Enum.Font.GothamMedium
TimeLabel.TextSize = 11
TimeLabel.TextXAlignment = Enum.TextXAlignment.Left
TimeLabel.Parent = abaSERVER

-- CONFIG
Secao(abaSETTINGS, "Configurações")
Botao(abaSETTINGS, "SALVAR CONFIG", function() SaveConfig() end)
Botao(abaSETTINGS, "CARREGAR CONFIG", function() LoadConfig() end)

Secao(abaSETTINGS, "Tecla de Atalho")
local HotkeyLabel = Instance.new("TextLabel")
HotkeyLabel.Size = UDim2.new(1, -8, 0, 20)
HotkeyLabel.BackgroundTransparency = 1
HotkeyLabel.Text = "Atual: " .. (SETTINGS.Hotkey and SETTINGS.Hotkey.Name or "RightShift")
HotkeyLabel.TextColor3 = TEMA.AzulClaro
HotkeyLabel.Font = Enum.Font.GothamMedium
HotkeyLabel.TextSize = 10
HotkeyLabel.TextXAlignment = Enum.TextXAlignment.Left
HotkeyLabel.Parent = abaSETTINGS

Botao(abaSETTINGS, "DEFINIR TECLA", function()
    HotkeyLabel.Text = "Pressione uma tecla..."
    local conn
    conn = UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.UserInputType == Enum.UserInputType.Keyboard then
            SETTINGS.Hotkey = input.KeyCode
            HotkeyLabel.Text = "Atual: " .. input.KeyCode.Name
            SaveConfig()
            conn:Disconnect()
        end
    end)
end)

Secao(abaSETTINGS, "Menu")
Botao(abaSETTINGS, "OCULTAR MENU", function()
    MainFrame.Visible = false
    LogoFlutuante.Visible = true
    TituloFixo.Visible = true
end)
Botao(abaSETTINGS, "FECHAR MENU", function()
    if ScreenGui then ScreenGui:Destroy() end
end)

local function MostrarAba(nome)
    for n, s in pairs(abas) do s.Visible = (n == nome) end
    for n, b in pairs(botoesAbas) do
        if n == nome then
            b.btn.BackgroundColor3 = TEMA.CardAtivo
            b.ind.Visible = true
            b.txt.TextColor3 = TEMA.Branco
            b.stroke.Color = TEMA.Azul
        else
            b.btn.BackgroundColor3 = TEMA.Card
            b.ind.Visible = false
            b.txt.TextColor3 = TEMA.Cinza
            b.stroke.Color = TEMA.Borda
        end
    end
end

local function CriarBotaoAba(nome)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, 0, 0, 28)
    b.BackgroundColor3 = TEMA.Card
    b.Text = ""
    b.BorderSizePixel = 0
    b.AutoButtonColor = false
    b.Parent = Sidebar
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 5)
    local stroke = Instance.new("UIStroke")
    stroke.Color = TEMA.Borda
    stroke.Thickness = 1
    stroke.Parent = b
    local ind = Instance.new("Frame")
    ind.Size = UDim2.new(0, 3, 0.6, 0)
    ind.Position = UDim2.new(0, 0, 0.2, 0)
    ind.BackgroundColor3 = TEMA.Azul
    ind.BorderSizePixel = 0
    ind.Visible = false
    ind.Parent = b
    Instance.new("UICorner", ind).CornerRadius = UDim.new(0, 2)
    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, -14, 1, 0)
    txt.Position = UDim2.new(0, 10, 0, 0)
    txt.BackgroundTransparency = 1
    txt.Text = nome
    txt.TextColor3 = TEMA.Cinza
    txt.Font = Enum.Font.GothamMedium
    txt.TextSize = 10
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.Parent = b
    botoesAbas[nome] = {btn = b, ind = ind, txt = txt, stroke = stroke}
    b.MouseButton1Click:Connect(function() MostrarAba(nome) end)
end

CriarBotaoAba("MAIN")
CriarBotaoAba("FARM")
CriarBotaoAba("ESP")
CriarBotaoAba("AIMBOT")
CriarBotaoAba("MOVIMENTO")
CriarBotaoAba("SERVIDOR")
CriarBotaoAba("CONFIG")

MostrarAba("MAIN")

-- PAINEL MINI SPEED
local MiniFrame = Instance.new("Frame")
MiniFrame.Size = UDim2.new(0, 180, 0, 110)
MiniFrame.Position = UDim2.new(1, -195, 0.5, -55)
MiniFrame.BackgroundColor3 = TEMA.Fundo
MiniFrame.BorderSizePixel = 0
MiniFrame.Parent = ScreenGui
MiniFrame.Active = true
MiniFrame.Draggable = true
MiniFrame.Visible = true
Instance.new("UICorner", MiniFrame).CornerRadius = UDim.new(0, 10)
local MiniStroke = Instance.new("UIStroke")
MiniStroke.Color = TEMA.Azul
MiniStroke.Thickness = 1.5
MiniStroke.Transparency = 0.4
MiniStroke.Parent = MiniFrame

local MiniTitulo = Instance.new("TextLabel")
MiniTitulo.Size = UDim2.new(1, 0, 0, 22)
MiniTitulo.BackgroundTransparency = 1
MiniTitulo.Text = "⚡ SPEED"
MiniTitulo.TextColor3 = TEMA.AzulClaro
MiniTitulo.Font = Enum.Font.GothamBlack
MiniTitulo.TextSize = 12
MiniTitulo.Parent = MiniFrame

local MiniSpeedBtn = Instance.new("TextButton")
MiniSpeedBtn.Size = UDim2.new(1, -16, 0, 28)
MiniSpeedBtn.Position = UDim2.new(0, 8, 0, 26)
MiniSpeedBtn.BackgroundColor3 = SETTINGS.SpeedHack and TEMA.CardAtivo or TEMA.Card
MiniSpeedBtn.Text = SETTINGS.SpeedHack and "SPEED: ON" or "SPEED: OFF"
MiniSpeedBtn.TextColor3 = SETTINGS.SpeedHack and TEMA.Branco or TEMA.Cinza
MiniSpeedBtn.Font = Enum.Font.GothamBold
MiniSpeedBtn.TextSize = 11
MiniSpeedBtn.BorderSizePixel = 0
MiniSpeedBtn.Parent = MiniFrame
Instance.new("UICorner", MiniSpeedBtn).CornerRadius = UDim.new(0, 6)

local MiniVelLabel = Instance.new("TextLabel")
MiniVelLabel.Size = UDim2.new(1, -16, 0, 18)
MiniVelLabel.Position = UDim2.new(0, 8, 0, 58)
MiniVelLabel.BackgroundTransparency = 1
MiniVelLabel.Text = "Velocidade: " .. SETTINGS.SpeedHackValor
MiniVelLabel.TextColor3 = TEMA.Cinza
MiniVelLabel.Font = Enum.Font.GothamMedium
MiniVelLabel.TextSize = 10
MiniVelLabel.TextXAlignment = Enum.TextXAlignment.Left
MiniVelLabel.Parent = MiniFrame

local MiniMaisBtn = Instance.new("TextButton")
MiniMaisBtn.Size = UDim2.new(0, 80, 0, 24)
MiniMaisBtn.Position = UDim2.new(0, 8, 0, 78)
MiniMaisBtn.BackgroundColor3 = TEMA.Card
MiniMaisBtn.Text = "+100"
MiniMaisBtn.TextColor3 = TEMA.Branco
MiniMaisBtn.Font = Enum.Font.GothamBold
MiniMaisBtn.TextSize = 11
MiniMaisBtn.BorderSizePixel = 0
MiniMaisBtn.Parent = MiniFrame
Instance.new("UICorner", MiniMaisBtn).CornerRadius = UDim.new(0, 5)

local MiniMenosBtn = Instance.new("TextButton")
MiniMenosBtn.Size = UDim2.new(0, 80, 0, 24)
MiniMenosBtn.Position = UDim2.new(0, 92, 0, 78)
MiniMenosBtn.BackgroundColor3 = TEMA.Card
MiniMenosBtn.Text = "-100"
MiniMenosBtn.TextColor3 = TEMA.Branco
MiniMenosBtn.Font = Enum.Font.GothamBold
MiniMenosBtn.TextSize = 11
MiniMenosBtn.BorderSizePixel = 0
MiniMenosBtn.Parent = MiniFrame
Instance.new("UICorner", MiniMenosBtn).CornerRadius = UDim.new(0, 5)

local function AtualizarMiniSpeed()
    MiniSpeedBtn.Text = SETTINGS.SpeedHack and "SPEED: ON" or "SPEED: OFF"
    MiniSpeedBtn.BackgroundColor3 = SETTINGS.SpeedHack and TEMA.CardAtivo or TEMA.Card
    MiniSpeedBtn.TextColor3 = SETTINGS.SpeedHack and TEMA.Branco or TEMA.Cinza
    MiniVelLabel.Text = "Velocidade: " .. SETTINGS.SpeedHackValor
    SpeedValorLabel.Text = "Velocidade: " .. SETTINGS.SpeedHackValor
end

MiniSpeedBtn.MouseButton1Click:Connect(function()
    SETTINGS.SpeedHack = not SETTINGS.SpeedHack
    AtualizarMiniSpeed()
    if not SETTINGS.SpeedHack then
        local char = LocalPlayer.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then hum.WalkSpeed = 16 end
        end
    end
    SaveConfig()
end)

MiniMaisBtn.MouseButton1Click:Connect(function()
    SETTINGS.SpeedHackValor = math.min(SETTINGS.SpeedHackValor + 100, 2000)
    AtualizarMiniSpeed()
    SaveConfig()
end)

MiniMenosBtn.MouseButton1Click:Connect(function()
    SETTINGS.SpeedHackValor = math.max(SETTINGS.SpeedHackValor - 100, 16)
    AtualizarMiniSpeed()
    SaveConfig()
end)

-- LOGO FLUTUANTE
local LogoFlutuante = Instance.new("TextButton")
LogoFlutuante.Size = UDim2.new(0, 50, 0, 50)
LogoFlutuante.Position = UDim2.new(0, 20, 0.5, -25)
LogoFlutuante.BackgroundColor3 = TEMA.Header
LogoFlutuante.Text = "RD"
LogoFlutuante.TextColor3 = TEMA.Azul
LogoFlutuante.Font = Enum.Font.GothamBlack
LogoFlutuante.TextSize = 16
LogoFlutuante.BorderSizePixel = 0
LogoFlutuante.Visible = false
LogoFlutuante.Parent = ScreenGui
Instance.new("UICorner", LogoFlutuante).CornerRadius = UDim.new(1, 0)
local LogoStroke = Instance.new("UIStroke")
LogoStroke.Color = TEMA.Azul
LogoStroke.Thickness = 2
LogoStroke.Parent = LogoFlutuante

LogoFlutuante.MouseButton1Click:Connect(function()
    MainFrame.Visible = true
    LogoFlutuante.Visible = false
    TituloFixo.Visible = true
end)

BtnOcultar.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
    LogoFlutuante.Visible = true
    TituloFixo.Visible = true
end)

BtnFechar.MouseButton1Click:Connect(function()
    if ScreenGui then ScreenGui:Destroy() end
end)

UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if SETTINGS.Hotkey and input.KeyCode == SETTINGS.Hotkey then
        MainFrame.Visible = not MainFrame.Visible
        if not MainFrame.Visible then
            LogoFlutuante.Visible = true
            TituloFixo.Visible = true
        else
            LogoFlutuante.Visible = false
            TituloFixo.Visible = true
        end
    end
end)

-- ATUALIZAÇÃO DE ESTATÍSTICAS
task.spawn(function()
    while true do
        task.wait(1)
        pcall(function()
            if ContadorLabel then ContadorLabel.Text = "Baús Coletados: " .. SETTINGS.ContadorBaus end
            if TempoLabel then TempoLabel.Text = "Tempo de Farm: " .. math.floor(SETTINGS.TempoFarm) .. "s" end
            if BausLabel then BausLabel.Text = "Baús no Mapa: " .. (function() local c = 0; for _ in pairs(ChestCache) do c = c + 1 end; return c end)() end
            if TrocasLabel then TrocasLabel.Text = "Trocas de Servidor: " .. SETTINGS.TrocasServidor end
            if TimeLabel then TimeLabel.Text = "Time Atual: " .. (LocalPlayer.Team and LocalPlayer.Team.Name or "--") end
        end)
    end
end)

game:GetService("StarterGui"):SetCore("SendNotification", {
    Title = "RD MENU VIP — BY RD",
    Text = "Farm + ESP + Aimbot + Speed abertos",
    Duration = 5,
})

JoinPirates()
