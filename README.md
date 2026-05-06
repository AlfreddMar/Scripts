-- Oil Empire🛢️ Script Mejorado | Auto Farm, Auto Sell, Detección Universal
-- Créditos originales: dkxn[cite: 1]

local Players          = game:GetService("Players")
local TweenService     = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local lp       = Players.LocalPlayer
local username = lp.Name
local enabled    = false
local sellEnabled = false
local tweenSpeed = 0.1
local useTween   = true
local farmThread = nil
local sellThread  = nil
local antiAfkOn  = false
local antiAfkConn = nil

-- === FUNCIONES DE UTILIDAD ===

local function setAntiAfk(on)
    antiAfkOn = on
    if antiAfkConn then antiAfkConn:Disconnect() end
    if on then
        antiAfkConn = lp.Idled:Connect(function()
            local vp = game:GetService("VirtualInputManager")
            vp:SendKeyEvent(true, Enum.KeyCode.W, false, game)
            task.wait(0.1)
            vp:SendKeyEvent(false, Enum.KeyCode.W, false, game)
        end)
    end
end

local function getPlayerPlot()
    local plotsFolder = workspace:FindFirstChild("Plots")
    if not plotsFolder then return nil end
    for _, plot in ipairs(plotsFolder:GetChildren()) do
        local label = plot:FindFirstChild("Main", true)
        if label and label:IsA("TextLabel") and label.Text:find(username) then
            return plot
        end
    end
    -- Respaldo por OwnerTag[cite: 1]
    for _, plot in ipairs(plotsFolder:GetChildren()) do
        pcall(function()
            if plot.OwnerTag.BillboardGui.Main.TextLabel.Text:match("^(.+)'s") == username then
                return plot
            end
        end)
    end
    return nil
end

local function getBuildings()
    local plot = getPlayerPlot()
    return plot and plot:FindFirstChild("Buildings") or nil
end

-- MEJORA: Detección agresiva de Refinerías
local function getRefineries(buildings)
    local list = {}
    if not buildings then return list end
    for _, m in ipairs(buildings:GetChildren()) do
        if m:IsA("Model") then
            -- Detecta por Atributo, Nombre o presencia de motor de refinado[cite: 1]
            local isRefinery = (m:GetAttribute("Type") == "Refinery") or 
                               (m.Name:lower():find("refinery")) or
                               (m:FindFirstChild("RefineProcess", true) ~= nil)
            
            if isRefinery then
                list[#list + 1] = m
            end
        end
    end
    return list
end

-- MEJORA: Extracción robusta de valores (10/10)
local function getValues(model)
    local text = ""
    pcall(function()
        local info = model:FindFirstChild("Info", true)
        local valObj = info and info:FindFirstChild("Value", true)
        text = valObj and (valObj.Text or valObj.Value) or ""
    end)
    
    local c, m = text:gsub(" ",""):match("^(%d+)/(%d+)$")
    return tonumber(c) or 0, tonumber(m) or 0
end

local function getPrimary(model)
    return model:FindFirstChild("Primary") or model.PrimaryPart
end

local function teleport(targetCF)
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    hrp.Anchored = true
    if useTween and tweenSpeed > 0.05 then
        local startCF = hrp.CFrame
        local elapsed = 0
        while elapsed < tweenSpeed and enabled do
            elapsed = elapsed + task.wait()
            hrp.CFrame = startCF:Lerp(targetCF, math.min(elapsed/tweenSpeed, 1))
        end
    end
    hrp.CFrame = targetCF
    hrp.Anchored = false
end

-- === BUCLES PRINCIPALES ===

local function farmLoop()
    while enabled do
        local buildings = getBuildings()
        local list = getRefineries(buildings)
        
        if #list > 0 then
            -- Ordenar por las más llenas[cite: 1]
            table.sort(list, function(a, b)
                local ca, ma = getValues(a)
                local cb, mb = getValues(b)
                return (ca/(ma>0 and ma or 1)) > (cb/(mb>0 and mb or 1))
            end)

            for _, model in ipairs(list) do
                if not enabled then break end
                local cur, max = getValues(model)
                if max > 0 and cur >= (max * 0.98) then -- Recolectar al 98%+
                    local primary = getPrimary(model)
                    if primary then
                        teleport(primary.CFrame)
                        task.wait(0.1)
                    end
                end
            end
        end
        task.wait(0.5)
    end
end

-- === LÓGICA DE AUTO SELL[cite: 1] ===

local sellPrice = 10
local minGasoline = 10000

local function trySell()
    local sellRemote = nil
    for _, v in ipairs(game:GetService("ReplicatedStorage"):GetDescendants()) do
        if v:IsA("RemoteEvent") and v.Name:lower():find("sell") then
            sellRemote = v; break
        end
    end
    if sellRemote then 
        pcall(function() sellRemote:FireServer() end) 
        return true
    end
    return false
end

local function sellLoop()
    while sellEnabled do
        local price = 0
        pcall(function() price = game:GetService("ReplicatedStorage").GasPrice.Value end)
        local gas = 0
        pcall(function() gas = lp.leaderstats.Gasoline.Value end)

        if price >= sellPrice and gas >= minGasoline then
            local oldEnabled = enabled
            enabled = false -- Pausa farm para vender
            
            local sellPart = workspace:FindFirstChild("Sell", true)
            if sellPart and sellPart:IsA("BasePart") then
                teleport(sellPart.CFrame * CFrame.new(0,3,0))
                task.wait(0.5)
                trySell()
            end
            
            enabled = oldEnabled
            if enabled then task.spawn(farmLoop) end
            task.wait(10)
        end
        task.wait(1)
    end
end

-- === CONSTRUCCIÓN DE LA GUI (Simplificada y funcional) ===

pcall(function() if game.CoreGui:FindFirstChild("OilEmpireGui") then game.CoreGui.OilEmpireGui:Destroy() end end)

local gui = Instance.new("ScreenGui", game.CoreGui)
gui.Name = "OilEmpireGui"

local main = Instance.new("Frame", gui)
main.Size = UDim2.new(0, 280, 0, 320)
main.Position = UDim2.new(0.5, -140, 0.4, -160)
main.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
main.BorderSizePixel = 0
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 10)

local title = Instance.new("TextLabel", main)
title.Size = UDim2.new(1, 0, 0, 40)
title.Text = "OIL EMPIRE 🛢️ MEJORADO"
title.TextColor3 = Color3.new(1, 1, 1)
title.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
title.Font = Enum.Font.GothamBold
Instance.new("UICorner", title).CornerRadius = UDim.new(0, 10)

local function createToggle(name, pos, callback)
    local btn = Instance.new("TextButton", main)
    btn.Size = UDim2.new(0, 240, 0, 40)
    btn.Position = pos
    btn.Text = name .. ": OFF"
    btn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.Gotham
    Instance.new("UICorner", btn)
    
    local state = false
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.Text = name .. (state and ": ON" or ": OFF")
        btn.BackgroundColor3 = state and Color3.fromRGB(60, 120, 60) or Color3.fromRGB(40, 40, 40)
        callback(state)
    end)
end

createToggle("Auto Farm", UDim2.new(0, 20, 0, 60), function(v)
    enabled = v
    if v then task.spawn(farmLoop) end
end)

createToggle("Auto Sell", UDim2.new(0, 20, 0, 110), function(v)
    sellEnabled = v
    if v then task.spawn(sellLoop) end
end)

createToggle("Anti-AFK", UDim2.new(0, 20, 0, 160), function(v)
    setAntiAfk(v)
end)

local status = Instance.new("TextLabel", main)
status.Size = UDim2.new(0, 240, 0, 60)
status.Position = UDim2.new(0, 20, 0, 220)
status.Text = "Buscando Refinerías..."
status.TextColor3 = Color3.fromRGB(150, 150, 150)
status.BackgroundTransparency = 1
status.TextWrapped = true

-- Actualizador de estado[cite: 1]
task.spawn(function()
    while gui.Parent do
        local count = #getRefineries(getBuildings())
        status.Text = "Refinerías detectadas: " .. count .. "\nGasolina: " .. (lp.leaderstats.Gasoline.Value or 0)
        task.wait(1)
    end
end)

-- Arrastrar GUI
local dragging, dragInput, dragStart, startPos
main.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true; dragStart = input.Position; startPos = main.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
end)
