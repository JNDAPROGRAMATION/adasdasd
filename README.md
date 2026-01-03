-- Serverscript (LocalScript no StarterPlayerScripts)
-- ESP/Wallhack e Aimbot para Roblox

-- Configurações
local ESP_SETTINGS = {
    ENABLED = true,
    WALLHACK = true,
    SHOW_NAMES = true,
    SHOW_DISTANCE = true,
    SHOW_HEALTH = true,
    SHOW_BOX = true,
    SHOW_TRACER = true,
    
    -- Configurações do Aimbot
    AIMBOT_ENABLED = false, -- Aimbot começa desativado
    AIMBOT_KEY = Enum.KeyCode.Q, -- Tecla para ativar aimbot
    AIMBOT_SMOOTHNESS = 10, -- Suavidade do movimento
    AIMBOT_MAX_DISTANCE = 500, -- Distância máxima
    AIMBOT_FOV = 100, -- Campo de visão
    AIMBOT_TARGET_PART = "Head", -- Parte para mirar
    AIMBOT_PRIORITY = "Closest", -- Prioridade
}

-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Camera = workspace.CurrentCamera

-- Variáveis
local localPlayer = Players.LocalPlayer
local playerGui = localPlayer:WaitForChild("PlayerGui")
local espFolder = Instance.new("Folder")
espFolder.Name = "ESPFolder"
espFolder.Parent = playerGui

local espCache = {}
local connections = {}
local controlGui = nil

-- Variáveis do Aimbot
local aimbotTarget = nil
local isAimbotActive = false
local aimbotConnection = nil

-- Função para mover o mouse (alternativa para mousemoverel)
local function moveMouse(deltaX, deltaY)
    local mouse = localPlayer:GetMouse()
    mouse.TargetFilter = workspace
    
    -- Mover o mouse usando UserInputService
    local currentPos = Vector2.new(mouse.X, mouse.Y)
    local newPos = currentPos + Vector2.new(deltaX, deltaY)
    
    pcall(function()
        mousemoverel(deltaX, deltaY)
    end)
end

-- Função para verificar se um jogador é válido
local function isValidTarget(player)
    if player == localPlayer then return false end
    if not player.Character then return false end
    
    local character = player.Character
    local humanoid = character:FindFirstChild("Humanoid")
    local targetPart = character:FindFirstChild(ESP_SETTINGS.AIMBOT_TARGET_PART)
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    
    if not humanoid or humanoid.Health <= 0 then return false end
    if not targetPart or not rootPart then return false end
    
    -- Verificar time
    if localPlayer.Team and player.Team then
        if localPlayer.Team == player.Team then
            return false
        end
    end
    
    return true, character, targetPart
end

-- Função para obter o melhor alvo
local function getBestTarget()
    local bestTarget = nil
    local bestDistance = math.huge
    
    local localCharacter = localPlayer.Character
    if not localCharacter then return nil end
    
    local localRootPart = localCharacter:FindFirstChild("HumanoidRootPart")
    if not localRootPart then return nil end
    
    for _, player in pairs(Players:GetPlayers()) do
        local isValid, character, targetPart = isValidTarget(player)
        
        if isValid then
            local distance = (targetPart.Position - localRootPart.Position).Magnitude
            
            if distance <= ESP_SETTINGS.AIMBOT_MAX_DISTANCE then
                if distance < bestDistance then
                    bestDistance = distance
                    bestTarget = player
                end
            end
        end
    end
    
    return bestTarget
end

-- Função do aimbot
local function aimbotUpdate()
    if not ESP_SETTINGS.AIMBOT_ENABLED or not isAimbotActive then
        aimbotTarget = nil
        return
    end
    
    local targetPlayer = getBestTarget()
    
    if targetPlayer and targetPlayer.Character then
        local targetPart = targetPlayer.Character:FindFirstChild(ESP_SETTINGS.AIMBOT_TARGET_PART)
        if targetPart then
            -- Calcular posição na tela
            local targetScreenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
            
            if onScreen then
                local mouse = localPlayer:GetMouse()
                local currentMousePos = Vector2.new(mouse.X, mouse.Y)
                local targetPos = Vector2.new(targetScreenPos.X, targetScreenPos.Y)
                
                -- Calcular direção
                local direction = targetPos - currentMousePos
                
                -- Aplicar suavidade
                local smoothDirection = direction / ESP_SETTINGS.AIMBOT_SMOOTHNESS
                
                -- Mover mouse
                moveMouse(smoothDirection.X, smoothDirection.Y)
                
                aimbotTarget = targetPlayer
            else
                aimbotTarget = nil
            end
        else
            aimbotTarget = nil
        end
    else
        aimbotTarget = nil
    end
end

-- Função para criar interface SIMPLES
local function createSimpleGUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SimpleESP_GUI"
    screenGui.ResetOnSpawn = false
    screenGui.IgnoreGuiInset = true
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    screenGui.Parent = playerGui
    
    -- Botão ESP
    local espButton = Instance.new("TextButton")
    espButton.Name = "ESP_Button"
    espButton.Size = UDim2.new(0, 100, 0, 40)
    espButton.Position = UDim2.new(1, -110, 0, 20)
    espButton.BackgroundColor3 = ESP_SETTINGS.ENABLED and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
    espButton.BackgroundTransparency = 0.3
    espButton.Text = ESP_SETTINGS.ENABLED and "ESP: ON" or "ESP: OFF"
    espButton.TextColor3 = ESP_SETTINGS.ENABLED and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 50, 50)
    espButton.TextSize = 16
    espButton.Font = Enum.Font.GothamBold
    espButton.Parent = screenGui
    
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 8)
    espCorner.Parent = espButton
    
    local espStroke = Instance.new("UIStroke")
    espStroke.Color = Color3.fromRGB(80, 80, 80)
    espStroke.Thickness = 2
    espStroke.Parent = espButton
    
    -- Botão Aimbot
    local aimbotButton = Instance.new("TextButton")
    aimbotButton.Name = "Aimbot_Button"
    aimbotButton.Size = UDim2.new(0, 100, 0, 40)
    aimbotButton.Position = UDim2.new(1, -110, 0, 70)
    aimbotButton.BackgroundColor3 = ESP_SETTINGS.AIMBOT_ENABLED and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
    aimbotButton.BackgroundTransparency = 0.3
    aimbotButton.Text = ESP_SETTINGS.AIMBOT_ENABLED and "AIMBOT: ON" or "AIMBOT: OFF"
    aimbotButton.TextColor3 = ESP_SETTINGS.AIMBOT_ENABLED and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 50, 50)
    aimbotButton.TextSize = 16
    aimbotButton.Font = Enum.Font.GothamBold
    aimbotButton.Parent = screenGui
    
    local aimbotCorner = Instance.new("UICorner")
    aimbotCorner.CornerRadius = UDim.new(0, 8)
    aimbotCorner.Parent = aimbotButton
    
    local aimbotStroke = Instance.new("UIStroke")
    aimbotStroke.Color = Color3.fromRGB(80, 80, 80)
    aimbotStroke.Thickness = 2
    aimbotStroke.Parent = aimbotButton
    
    -- Eventos dos botões
    espButton.MouseButton1Click:Connect(function()
        ESP_SETTINGS.ENABLED = not ESP_SETTINGS.ENABLED
        espButton.Text = ESP_SETTINGS.ENABLED and "ESP: ON" or "ESP: OFF"
        espButton.TextColor3 = ESP_SETTINGS.ENABLED and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 50, 50)
        espButton.BackgroundColor3 = ESP_SETTINGS.ENABLED and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
        
        print("ESP: " .. (ESP_SETTINGS.ENABLED and "ATIVADO" or "DESATIVADO"))
    end)
    
    aimbotButton.MouseButton1Click:Connect(function()
        ESP_SETTINGS.AIMBOT_ENABLED = not ESP_SETTINGS.AIMBOT_ENABLED
        aimbotButton.Text = ESP_SETTINGS.AIMBOT_ENABLED and "AIMBOT: ON" or "AIMBOT: OFF"
        aimbotButton.TextColor3 = ESP_SETTINGS.AIMBOT_ENABLED and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 50, 50)
        aimbotButton.BackgroundColor3 = ESP_SETTINGS.AIMBOT_ENABLED and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
        
        print("AIMBOT: " .. (ESP_SETTINGS.AIMBOT_ENABLED and "ATIVADO" or "DESATIVADO"))
    end)
    
    return screenGui
end

-- Função para criar ESP object (versão simplificada)
local function createESPObject(player)
    local espObject = {
        Player = player,
        Box = nil,
        NameLabel = nil,
        Connections = {}
    }
    
    -- Criar ScreenGui para textos
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = player.Name .. "_ESPGui"
    screenGui.ResetOnSpawn = false
    screenGui.IgnoreGuiInset = true
    screenGui.Parent = espFolder
    
    -- Criar label para nome
    espObject.NameLabel = Instance.new("TextLabel")
    espObject.NameLabel.Name = "Name"
    espObject.NameLabel.BackgroundTransparency = 1
    espObject.NameLabel.Text = player.Name
    espObject.NameLabel.TextColor3 = Color3.new(1, 1, 1)
    espObject.NameLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    espObject.NameLabel.TextStrokeTransparency = 0.5
    espObject.NameLabel.TextSize = 16
    espObject.NameLabel.Visible = false
    espObject.NameLabel.Parent = screenGui
    
    espCache[player] = espObject
    
    return espObject
end

-- Função para atualizar ESP
local function updateESP(player, espObject)
    if not ESP_SETTINGS.ENABLED then
        if espObject.NameLabel then
            espObject.NameLabel.Visible = false
        end
        return
    end
    
    if not player or not player.Character then
        if espObject.NameLabel then
            espObject.NameLabel.Visible = false
        end
        return
    end
    
    local character = player.Character
    local head = character:FindFirstChild("Head")
    
    if not head then return end
    
    -- Converter posição para tela
    local screenPosition, onScreen = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 2, 0))
    
    if onScreen and espObject.NameLabel then
        espObject.NameLabel.Visible = true
        espObject.NameLabel.Position = UDim2.new(0, screenPosition.X, 0, screenPosition.Y)
        
        -- Cor baseada no time
        if player.Team then
            espObject.NameLabel.TextColor3 = player.Team.TeamColor.Color
        else
            espObject.NameLabel.TextColor3 = Color3.new(1, 1, 1)
        end
    elseif espObject.NameLabel then
        espObject.NameLabel.Visible = false
    end
end

-- Função para atualizar todos os ESPs
local function updateAllESP()
    for player, espObject in pairs(espCache) do
        if player ~= localPlayer then
            updateESP(player, espObject)
        end
    end
end

-- Função principal de inicialização
local function initialize()
    -- Criar interface
    controlGui = createSimpleGUI()
    
    -- Inicializar ESP para jogadores existentes
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= localPlayer then
            createESPObject(player)
        end
    end
    
    -- Conectar eventos
    table.insert(connections, Players.PlayerAdded:Connect(function(player)
        if player ~= localPlayer then
            createESPObject(player)
        end
    end))
    
    table.insert(connections, Players.PlayerRemoving:Connect(function(player)
        local espObject = espCache[player]
        if espObject then
            if espObject.NameLabel then
                espObject.NameLabel.Parent:Destroy()
            end
            espCache[player] = nil
        end
    end))
    
    -- Loop de atualização do ESP
    table.insert(connections, RunService.RenderStepped:Connect(function()
        updateAllESP()
    end))
    
    -- Loop do Aimbot
    aimbotConnection = RunService.RenderStepped:Connect(function()
        aimbotUpdate()
    end)
    table.insert(connections, aimbotConnection)
    
    -- Teclas de atalho
    table.insert(connections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if not gameProcessed then
            -- F1 para ESP
            if input.KeyCode == Enum.KeyCode.F1 then
                ESP_SETTINGS.ENABLED = not ESP_SETTINGS.ENABLED
                
                local espButton = controlGui:FindFirstChild("ESP_Button")
                if espButton then
                    espButton.Text = ESP_SETTINGS.ENABLED and "ESP: ON" or "ESP: OFF"
                    espButton.TextColor3 = ESP_SETTINGS.ENABLED and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 50, 50)
                    espButton.BackgroundColor3 = ESP_SETTINGS.ENABLED and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
                end
                
                print("ESP: " .. (ESP_SETTINGS.ENABLED and "ATIVADO (F1)" or "DESATIVADO (F1)"))
            end
            
            -- F2 para Aimbot
            if input.KeyCode == Enum.KeyCode.F2 then
                ESP_SETTINGS.AIMBOT_ENABLED = not ESP_SETTINGS.AIMBOT_ENABLED
                
                local aimbotButton = controlGui:FindFirstChild("Aimbot_Button")
                if aimbotButton then
                    aimbotButton.Text = ESP_SETTINGS.AIMBOT_ENABLED and "AIMBOT: ON" or "AIMBOT: OFF"
                    aimbotButton.TextColor3 = ESP_SETTINGS.AIMBOT_ENABLED and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 50, 50)
                    aimbotButton.BackgroundColor3 = ESP_SETTINGS.AIMBOT_ENABLED and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
                end
                
                print("AIMBOT: " .. (ESP_SETTINGS.AIMBOT_ENABLED and "ATIVADO (F2)" or "DESATIVADO (F2)"))
            end
            
            -- Tecla do aimbot (Q)
            if input.KeyCode == ESP_SETTINGS.AIMBOT_KEY then
                isAimbotActive = true
                print("Aimbot ativado (segure " .. ESP_SETTINGS.AIMBOT_KEY.Name .. ")")
            end
        end
    end))
    
    table.insert(connections, UserInputService.InputEnded:Connect(function(input, gameProcessed)
        if input.KeyCode == ESP_SETTINGS.AIMBOT_KEY then
            isAimbotActive = false
            aimbotTarget = nil
        end
    end))
    
    print("=== ESP + AIMBOT CARREGADO ===")
    print("Controles:")
    print("  F1 - Ativar/Desativar ESP")
    print("  F2 - Ativar/Desativar Aimbot")
    print("  " .. ESP_SETTINGS.AIMBOT_KEY.Name .. " - Segurar para usar Aimbot")
    print("  Botões no canto superior direito")
    print("================================")
end

-- Iniciar
initialize()

-- Limpeza
local function cleanup()
    for _, connection in ipairs(connections) do
        if connection then
            pcall(function() connection:Disconnect() end)
        end
    end
    
    if controlGui then
        controlGui:Destroy()
    end
    
    if espFolder then
        espFolder:Destroy()
    end
end

-- Conectar evento de saída
game:GetService("Players").LocalPlayer.AncestryChanged:Connect(function()
    cleanup()
end)
