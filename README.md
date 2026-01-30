local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local VirtualInputManager = game:GetService("VirtualInputManager")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local LP = game.Players.LocalPlayer
local BallShadow, RealBall
local WhiteColor = Color3.new(1, 1, 1)
local LastBallPos
local SpeedMulty = 3
local AutoParryEnabled = true

local ParryDistance = 15
local ReactionTime = 0.0001
local SafetyMargin = 1.3

local AutoClickerEnabled = false
local ClickSpeed = 1000
local LastClickTime = 0

local LastParryTime = 0
local ParryCooldown = 0.000001
local IsParrying = false

-- Настраиваемые пороги скорости и высоты
local SpeedHeightThresholds = {
    {speed = 7.5, maxHeight = 22},
    {speed = 10, maxHeight = 27},
    {speed = math.huge, maxHeight = 30}
}

local PlayerCoordsGui = nil
local PlayerCoordsLabel = nil

local IntermissionCoords = Vector3.new(567, 285, -783)
local AutoIntermissionEnabled = false
local LastIntermissionCheck = 0
local IntermissionCheckCooldown = 2

-- Бинды по умолчанию
local Binds = {
    ToggleGUI = Enum.KeyCode.K,
    AutoParry = nil,
    Clicker = Enum.KeyCode.E,
    Target = Enum.KeyCode.H,
    Teleport = Enum.KeyCode.T,
    AutoInter = nil,
    HVH = Enum.KeyCode.G
}

local BindingInProgress = false
local CurrentBindingButton = nil

-- Переменные для Target системы
local TargetEnabled = false
local currentTarget = nil
local currentRadius = 45
local rotationSpeed = 2
local targetLocked = false
local MAX_HEIGHT = 265
local currentAngle = 360
local targetConnection = nil
local noclipEnabled = false
local noclipConnection = nil
local isInGame = false

-- Переменные для HVH системы
local HVHEnabled = false
local HVHVisualBall = nil
local HVHConnection = nil
local OriginalHVHPosition = nil
local HVHState = "WAITING"
local LastBallColor = WhiteColor
local BallVelocity = Vector3.new(0, 0, 0)
local LastBallPosForVelocity = nil

-- Цвета для фиолетового градиента
local PurpleGradient = {
    Color3.fromRGB(100, 50, 200),
    Color3.fromRGB(150, 70, 250),
    Color3.fromRGB(200, 100, 255)
}

local AccentColor = Color3.fromRGB(180, 80, 255)
local TextColor = Color3.fromRGB(245, 245, 255)
local DarkTextColor = Color3.fromRGB(40, 20, 80)

local function IsInGame()
    local success, result = pcall(function()
        local healthBar = LP.PlayerGui.HUD.HolderBottom.HealthBar
        return healthBar.Visible == true
    end)
    return success and result
end

local function GetBallColor()
    if not RealBall then return WhiteColor end
    
    local highlight = RealBall:FindFirstChildOfClass("Highlight")
    if highlight then
        return highlight.FillColor
    end
    
    local surfaceGui = RealBall:FindFirstChildOfClass("SurfaceGui")
    if surfaceGui then
        local frame = surfaceGui:FindFirstChild("Frame")
        if frame and frame.BackgroundColor3 then
            return frame.BackgroundColor3
        end
    end
    
    if RealBall:IsA("Part") and RealBall.BrickColor ~= BrickColor.new("White") then
        return RealBall.Color
    end
    
    return WhiteColor
end

-- Функция для расчета высоты мяча по размеру тени (всегда +3 к расчету)
local function CalculateBallHeight(shadowSize)
    local baseShadowSize = 5
    local heightMultiplier = 20
    
    local shadowIncrease = math.max(0, shadowSize - baseShadowSize)
    local estimatedHeight = shadowIncrease * heightMultiplier
    
    -- Всегда добавляем +3 к высоте мяча
    return math.min(estimatedHeight + 3, 100)
end

-- Функция для создания визуального мяча (без пульсации)
local function CreateVisualBall(shadowPosition, shadowSize)
    if not shadowPosition then return nil end
    
    -- Удаляем старый мяч если есть
    if HVHVisualBall then
        HVHVisualBall:Destroy()
        HVHVisualBall = nil
    end
    
    -- Вычисляем высоту мяча с учетом +3
    local ballHeight = CalculateBallHeight(shadowSize)
    local ballYPosition = shadowPosition.Y + ballHeight
    local ballPosition = Vector3.new(shadowPosition.X, ballYPosition, shadowPosition.Z)
    
    -- Создаем новый визуальный мяч с размером 4,4,4
    HVHVisualBall = Instance.new("Part")
    HVHVisualBall.Name = "VisualBallTracker"
    HVHVisualBall.Anchored = true
    HVHVisualBall.CanCollide = false
    HVHVisualBall.Material = Enum.Material.Neon
    HVHVisualBall.Color = AccentColor
    HVHVisualBall.Transparency = 0.1
    HVHVisualBall.Size = Vector3.new(4, 4, 4)
    HVHVisualBall.Shape = Enum.PartType.Ball
    HVHVisualBall.Position = ballPosition
    HVHVisualBall.Parent = workspace
    
    -- Добавляем яркое свечение без пульсации
    local highlight = Instance.new("Highlight")
    highlight.FillColor = AccentColor
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.FillTransparency = 0.3
    highlight.OutlineTransparency = 0.1
    highlight.Parent = HVHVisualBall
    
    return HVHVisualBall, ballPosition
end

-- Функция для обновления позиции визуального мяча
local function UpdateVisualBall(shadowPosition, shadowSize)
    if not shadowPosition then return nil end
    
    if not HVHVisualBall then
        return CreateVisualBall(shadowPosition, shadowSize)
    else
        -- Вычисляем новую позицию с учетом +3
        local ballHeight = CalculateBallHeight(shadowSize)
        local ballYPosition = shadowPosition.Y + ballHeight
        local ballPosition = Vector3.new(shadowPosition.X, ballYPosition, shadowPosition.Z)
        
        HVHVisualBall.Position = ballPosition
        return ballPosition
    end
end

-- Функция для удаления визуального мяча
local function RemoveVisualBall()
    if HVHVisualBall then
        HVHVisualBall:Destroy()
        HVHVisualBall = nil
    end
end

-- Функция для выполнения HVH с новой логикой
local function ExecuteHVH(shadowPosition, shadowSize)
    if not HVHEnabled or not IsInGame() or not shadowPosition then
        HVHState = "WAITING"
        OriginalHVHPosition = nil
        return
    end
    
    -- Получаем цвет мяча
    local ballColor = WhiteColor
    if RealBall then
        ballColor = GetBallColor()
    end
    
    -- Сохраняем текущий цвет мяча
    LastBallColor = ballColor
    
    -- Обновляем визуальный мяч
    local ballPosition = UpdateVisualBall(shadowPosition, shadowSize)
    if not ballPosition then return end
    
    -- Вычисляем высоту мяча
    local ballHeight = CalculateBallHeight(shadowSize)
    local ballYPosition = shadowPosition.Y + ballHeight
    ballPosition = Vector3.new(shadowPosition.X, ballYPosition, shadowPosition.Z)
    
    -- Вычисляем скорость мяча
    local ballSpeed = BallVelocity.Magnitude
    
    -- Если мяч не белый и скорость равна 0, не выполняем HVH
    if ballColor ~= WhiteColor and ballSpeed < 0.1 then
        HVHState = "WAITING"
        return
    end
    
    -- Логика состояний HVH
    if HVHState == "WAITING" then
        -- Ждем пока мяч станет не белым и имеет скорость больше 0
        if ballColor ~= WhiteColor and ballSpeed > 0.1 then
            -- Сохраняем оригинальную позицию
            if LP.Character and LP.Character.PrimaryPart then
                OriginalHVHPosition = LP.Character.PrimaryPart.Position
                HVHState = "TELEPORTING"
            end
        end
        
    elseif HVHState == "TELEPORTING" then
        -- Нажимаем F
        VirtualInputManager:SendKeyEvent(true, "F", false, game)
        task.wait(0.000001)
        VirtualInputManager:SendKeyEvent(false, "F", false, game)
        
        -- Уменьшенная задержка до 0.000001
        task.wait(0.000001)
        
        -- Телепортируемся прямо в мяч
        if LP.Character and LP.Character.PrimaryPart then
            LP.Character.PrimaryPart.CFrame = CFrame.new(ballPosition)
            HVHState = "FOLLOWING"
        end
        
    elseif HVHState == "FOLLOWING" then
        -- Проверяем, стал ли мяч белым или скорость стала 0
        if ballColor == WhiteColor or ballSpeed < 0.1 then
            HVHState = "RETURNING"
            return
        end
        
        -- Находимся прямо в мяче
        if LP.Character and LP.Character.PrimaryPart then
            LP.Character.PrimaryPart.CFrame = CFrame.new(ballPosition)
        end
        
    elseif HVHState == "RETURNING" then
        -- Возвращаемся в исходную позицию
        if OriginalHVHPosition and LP.Character and LP.Character.PrimaryPart then
            LP.Character.PrimaryPart.CFrame = CFrame.new(OriginalHVHPosition)
            
            -- Уменьшенная задержка до 0.000001
            task.wait(0.000001)
            
            -- Возвращаемся в состояние ожидания
            HVHState = "WAITING"
            OriginalHVHPosition = nil
        else
            HVHState = "WAITING"
        end
    end
end

-- Функция для запуска/остановки постоянной работы HVH
local function ToggleHVHContinuous()
    if HVHConnection then
        HVHConnection:Disconnect()
        HVHConnection = nil
    end
    
    if HVHEnabled then
        -- Сбрасываем состояние HVH
        HVHState = "WAITING"
        OriginalHVHPosition = nil
        LastBallColor = WhiteColor
        
        HVHConnection = RunService.Heartbeat:Connect(function()
            if BallShadow then
                ExecuteHVH(BallShadow.Position, BallShadow.Size.X)
            else
                -- Если мяч не найден, возвращаемся в состояние ожидания
                if HVHState ~= "WAITING" then
                    HVHState = "WAITING"
                    OriginalHVHPosition = nil
                end
            end
        end)
    else
        -- Выключаем HVH, сбрасываем состояние
        HVHState = "WAITING"
        OriginalHVHPosition = nil
        LastBallColor = WhiteColor
        RemoveVisualBall()
    end
end

local function CreatePlayerCoordsGUI()
    if PlayerCoordsGui then
        PlayerCoordsGui:Destroy()
    end
    
    PlayerCoordsGui = Instance.new("ScreenGui")
    PlayerCoordsGui.Name = "PlayerCoordsGUI"
    PlayerCoordsGui.Parent = LP:WaitForChild("PlayerGui")
    
    local CoordsFrame = Instance.new("Frame")
    CoordsFrame.Size = UDim2.new(0, 220, 0, 90)
    CoordsFrame.Position = UDim2.new(0, 10, 0, 10)
    CoordsFrame.BackgroundColor3 = PurpleGradient[1]
    CoordsFrame.BackgroundTransparency = 0.15
    CoordsFrame.BorderSizePixel = 0
    CoordsFrame.Parent = PlayerCoordsGui
    
    local CoordsCorner = Instance.new("UICorner")
    CoordsCorner.CornerRadius = UDim.new(0, 12)
    CoordsCorner.Parent = CoordsFrame
    
    local CoordsGradient = Instance.new("UIGradient")
    CoordsGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, PurpleGradient[1]),
        ColorSequenceKeypoint.new(1, PurpleGradient[2])
    })
    CoordsGradient.Rotation = 90
    CoordsGradient.Parent = CoordsFrame
    
    local CoordsTitle = Instance.new("TextLabel")
    CoordsTitle.Size = UDim2.new(1, 0, 0, 25)
    CoordsTitle.Position = UDim2.new(0, 0, 0, 0)
    CoordsTitle.BackgroundTransparency = 1
    CoordsTitle.Text = "📊 PLAYER COORDINATES"
    CoordsTitle.TextColor3 = TextColor
    CoordsTitle.TextSize = 14
    CoordsTitle.Font = Enum.Font.GothamBold
    CoordsTitle.Parent = CoordsFrame
    
    PlayerCoordsLabel = Instance.new("TextLabel")
    PlayerCoordsLabel.Size = UDim2.new(1, 0, 0, 60)
    PlayerCoordsLabel.Position = UDim2.new(0, 0, 0, 25)
    PlayerCoordsLabel.BackgroundTransparency = 1
    PlayerCoordsLabel.Text = "X: 0.0\nY: 0.0\nZ: 0.0\nStatus: Loading..."
    PlayerCoordsLabel.TextColor3 = TextColor
    PlayerCoordsLabel.TextSize = 12
    PlayerCoordsLabel.Font = Enum.Font.Gotham
    PlayerCoordsLabel.TextXAlignment = Enum.TextXAlignment.Left
    PlayerCoordsLabel.Parent = CoordsFrame
    
    local dragging = false
    local dragStart = nil
    local startPos = nil
    
    CoordsTitle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = CoordsFrame.Position
        end
    end)
    
    CoordsTitle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            if dragging then
                local delta = input.Position - dragStart
                CoordsFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
            end
        end
    end)
    
    CoordsTitle.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
end

local function UpdatePlayerCoordinates()
    if not PlayerCoordsLabel then
        return
    end
    
    local inGame = IsInGame()
    local statusText = inGame and "IN GAME" or "NOT IN GAME"
    local statusColor = inGame and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
    
    if LP.Character and LP.Character.PrimaryPart then
        local playerPos = LP.Character.PrimaryPart.Position
        PlayerCoordsLabel.Text = string.format("X: %.1f\nY: %.1f\nZ: %.1f\nStatus: %s", 
            playerPos.X, playerPos.Y, playerPos.Z, statusText)
        PlayerCoordsLabel.TextColor3 = statusColor
    else
        PlayerCoordsLabel.Text = string.format("X: 0.0\nY: 0.0\nZ: 0.0\nStatus: %s", statusText)
        PlayerCoordsLabel.TextColor3 = statusColor
    end
    
    return inGame
end

local function TeleportToIntermission()
    if not LP.Character or not LP.Character.PrimaryPart then
        return false
    end
    
    local safePosition = IntermissionCoords + Vector3.new(0, 5, 0)
    LP.Character.PrimaryPart.CFrame = CFrame.new(safePosition)
    
    return true
end

local function CheckAutoIntermission()
    if not AutoIntermissionEnabled then
        return
    end
    
    local currentTime = tick()
    if currentTime - LastIntermissionCheck < IntermissionCheckCooldown then
        return
    end
    
    LastIntermissionCheck = currentTime
    
    if not IsInGame() then
        if TeleportToIntermission() then
            print("Auto Intermission: Teleported to intermission coordinates")
        else
            print("Auto Intermission: Teleport failed")
        end
    end
end

local function FindNearestPlayer()
    local nearestPlayer = nil
    local nearestDistance = math.huge
    
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LP and player.Character and player.Character.PrimaryPart and player.Character.Humanoid and player.Character.Humanoid.Health > 0 then
            if LP.Character and LP.Character.PrimaryPart then
                local distance = (player.Character.PrimaryPart.Position - LP.Character.PrimaryPart.Position).Magnitude
                if distance < nearestDistance then
                    nearestDistance = distance
                    nearestPlayer = player
                end
            end
        end
    end
    
    return nearestPlayer
end

local function TeleportToPlayer(player)
    if not player or not player.Character or not player.Character.PrimaryPart or not LP.Character or not LP.Character.PrimaryPart then
        return false
    end
    
    local targetPos = player.Character.PrimaryPart.Position
    local safePosition = targetPos + Vector3.new(0, 5, 0)
    
    LP.Character.PrimaryPart.CFrame = CFrame.new(safePosition)
    return true
end

local function GetMaxHeightBySpeed(speedStuds)
    for _, threshold in ipairs(SpeedHeightThresholds) do
        if speedStuds <= threshold.speed then
            return threshold.maxHeight
        end
    end
    return 30
end

-- Функция для получения названия клавиши
local function GetKeyName(keyCode)
    if not keyCode then return "NONE" end
    local keyName = tostring(keyCode):gsub("Enum.KeyCode.", "")
    if keyName == "LeftControl" then return "LCTRL"
    elseif keyName == "RightControl" then return "RCTRL"
    elseif keyName == "LeftShift" then return "LSHIFT"
    elseif keyName == "RightShift" then return "RSHIFT"
    elseif keyName == "LeftAlt" then return "LALT"
    elseif keyName == "RightAlt" then return "RALT"
    else return keyName end
end

-- Функция для обновления текста кнопок с биндами
local function UpdateButtonText(button, baseText, bindKey)
    if bindKey then
        button.Text = baseText .. " (" .. GetKeyName(bindKey) .. ")"
    else
        button.Text = baseText
    end
end

-- Функция для установки бинда
local function StartBinding(button, bindType, baseText)
    if BindingInProgress then return end
    
    BindingInProgress = true
    CurrentBindingButton = button
    
    local originalText = button.Text
    button.Text = "Press any key..."
    button.BackgroundColor3 = AccentColor
    button.TextColor3 = DarkTextColor
    
    local connection
    connection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.UserInputType == Enum.UserInputType.Keyboard then
            Binds[bindType] = input.KeyCode
            
            UpdateButtonText(button, baseText, input.KeyCode)
            
            -- Устанавливаем цвет в зависимости от состояния функции
            if bindType == "AutoParry" then
                button.BackgroundColor3 = AutoParryEnabled and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
            elseif bindType == "Clicker" then
                button.BackgroundColor3 = AutoClickerEnabled and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
            elseif bindType == "Target" then
                button.BackgroundColor3 = TargetEnabled and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
            elseif bindType == "HVH" then
                button.BackgroundColor3 = HVHEnabled and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
            elseif bindType == "AutoInter" then
                button.BackgroundColor3 = AutoIntermissionEnabled and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
            else
                button.BackgroundColor3 = PurpleGradient[2]
            end
            
            button.TextColor3 = DarkTextColor
            BindingInProgress = false
            CurrentBindingButton = nil
            connection:Disconnect()
        elseif input.UserInputType == Enum.UserInputType.MouseButton1 then
            Binds[bindType] = nil
            UpdateButtonText(button, baseText, nil)
            button.BackgroundColor3 = PurpleGradient[2]
            button.TextColor3 = DarkTextColor
            
            BindingInProgress = false
            CurrentBindingButton = nil
            connection:Disconnect()
        end
    end)
end

-- Функции для Target системы
local function toggleNoclip(state)
    if noclipConnection then
        noclipConnection:Disconnect()
        noclipConnection = nil
    end
    
    if state then
        noclipEnabled = true
        
        if LP.Character then
            for _, part in pairs(LP.Character:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        end
        
        noclipConnection = RunService.Stepped:Connect(function()
            if LP.Character and noclipEnabled then
                for _, part in pairs(LP.Character:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = false
                    end
                end
            end
        end)
    else
        noclipEnabled = false
        if LP.Character then
            for _, part in pairs(LP.Character:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = true
                end
            end
        end
    end
end

-- Оптимизированный поиск ближайшего игрока НИЖЕ максимальной высоты
local lastPlayerSearch = 0
local playerSearchInterval = 0.5
lo
