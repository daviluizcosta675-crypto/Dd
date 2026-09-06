-- [[ OVERDRIVE HUB - CUSTOM GUI ]] --
-- PARTE 1: SERVIÇOS, CONFIGURAÇÕES E FUNÇÕES AUXILIARES

-- Serviços
local Players = game:GetService("Players")
local Player = Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local Camera = Workspace.CurrentCamera

-- Configurações e Estado
local cfg = {
    reach = 10,
    sphere = true,
    esp = true,
    touch = true,
    cd = 1.5,
    kick = 5,
    kickOn = false,
    spin = false,
    skillMode = false,
    spinSpeed = 3,
    skillSpeed = 5,
    autoFollow = false,
    magneticEnabled = false,
    magneticStrength = 50,
    fovEnabled = false,
    fovValue = 70,
    controlBallEnabled = false,
    controlBallTarget = nil,
    rgbPlayerEnabled = false,
    tpBallEnabled = false,
    autoCatch = false,
    catchIntensity = 5,
    trainingReach = 10,
    cooldown = 1.5,
    tpsAlert = false,
    espTrainingOnly = false,
    trainingMode = false
}

local balls = {}
local esps = {}
local lr = 0
local spinAngle = 0
local dt = 0
local sp = nil
local spectateEnabled = false
local spectateTarget = nil
local originalCameraCFrame = nil
local targetBall = nil
local char = nil
local humanoid = nil
local hrp = nil
local defaultFOV = 70
local rgbHue = 0
local rgbConnection = nil
local magneticWelds = {}
local bn = {TPS = true, ESA = true, MRS = true, PRS = true, MPS = true, SSS = true, AIFA = true, RBZ = true}
local tpsAlertActive = false
local tpsAlertGui = nil
local tpsDistance = 0
local floatBtn = nil
local autoFollowToggleObj = nil
local keybindKey = Enum.KeyCode.E
local keybindLabel = nil

-- Atualização de Personagem
local function updateChar()
    char = Player.Character
    if char then 
        humanoid = char:FindFirstChildOfClass("Humanoid")
        hrp = char:FindFirstChild("HumanoidRootPart")
    end
end

Player.CharacterAdded:Connect(function()
    task.wait(0.5)
    updateChar()
    if cfg.rgbPlayerEnabled and startRGBPlayer then
        startRGBPlayer()
    end
end)
updateChar()

-- Funções Auxiliares de Jogo
local function findClosestBall()
    if not hrp or not hrp.Parent then return nil end
    local closest = nil
    local closeDist = math.huge
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and (v.Name == "TPS" or (cfg.trainingMode and v.Name == "TrainingBall")) then
            local d = (v.Position - hrp.Position).Magnitude
            if d < closeDist then 
                closeDist = d
                closest = v 
            end
        end
    end
    return closest
end

local function findClosestTPS()
    if not hrp or not hrp.Parent then return nil end
    local closest = nil
    local closeDist = math.huge
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and v.Name == "TPS" then
            local d = (v.Position - hrp.Position).Magnitude
            if d < closeDist then 
                closeDist = d
                closest = v 
            end
        end
    end
    return closest
end

local function rb()
    if tick() - lr < cfg.cd then return end
    lr = tick()
    balls = {}
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and bn[v.Name] then 
            table.insert(balls, v) 
        end
    end
end

local function gp(c)
    local t = {}
    for _, v in ipairs(c:GetChildren()) do
        if v:IsA("BasePart") and v.Name ~= "HumanoidRootPart" then 
            table.insert(t, v) 
        end
    end
    return t
end

local function us()
    if not cfg.sphere then 
        if sp and sp.Parent then sp:Destroy() end
        sp = nil
        return 
    end
    if not sp or not sp.Parent then
        sp = Instance.new("Part")
        sp.Name = "CS"
        sp.Shape = Enum.PartType.Ball
        sp.Anchored = true
        sp.CanCollide = false
        sp.Transparency = 0.7
        sp.Material = Enum.Material.ForceField
        sp.Color = Color3.fromRGB(255, 50, 50)
        sp.Parent = Workspace
    end
    sp.Size = Vector3.new(cfg.reach * 2, cfg.reach * 2, cfg.reach * 2)
end

local function tp()
    if not Player.Character then return end
    local h = Player.Character:FindFirstChild("HumanoidRootPart")
    if not h then return end
    local cl = findClosestBall()
    if cl then 
        h.CFrame = cl.CFrame * CFrame.new(0, 3, 0)
    end
end
-- [[ OVERDRIVE HUB - CUSTOM GUI ]] --
-- PARTE 2: FUNÇÕES DE LÓGICA DAS FEATURES

-- Funções de Lógica das Features
local updateFloatBtnVisual

local function setAutoFollowState(state)
    cfg.autoFollow = state
    if cfg.autoFollow then 
        targetBall = findClosestBall() 
    else 
        targetBall = nil 
    end
    if updateFloatBtnVisual then updateFloatBtnVisual() end
end

local function doAutoFollow()
    if not cfg.autoFollow then return end
    if not humanoid or not hrp then updateChar() end
    if not humanoid or not hrp then return end
    if not targetBall or not targetBall.Parent then 
        targetBall = findClosestBall() 
    end
    if targetBall and targetBall.Parent then 
        humanoid:MoveTo(targetBall.Position) 
    end
end

function startRGBPlayer()
    if rgbConnection then rgbConnection:Disconnect() rgbConnection = nil end
    if not cfg.rgbPlayerEnabled then return end
    
    rgbConnection = RunService.RenderStepped:Connect(function()
        if not cfg.rgbPlayerEnabled then
            if rgbConnection then rgbConnection:Disconnect() rgbConnection = nil end
            return
        end
        rgbHue = (rgbHue + 0.5) % 360
        local color = Color3.fromHSV(rgbHue / 360, 1, 1)
        local ch = Player.Character
        if ch then
            for _, part in ipairs(ch:GetDescendants()) do
                if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                    part.Color = color
                end
            end
        end
    end)
end

local function stopRGBPlayer()
    if rgbConnection then rgbConnection:Disconnect() rgbConnection = nil end
end

local function doTPBall()
    if not cfg.tpBallEnabled then return end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp or not hrp.Parent then return end
    
    local closest = findClosestBall()
    if closest then
        hrp.CFrame = closest.CFrame * CFrame.new(0, 3, 0)
    end
end

local function doMagneticBall()
    if not cfg.magneticEnabled then return end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp or not hrp.Parent then return end
    
    for _, ball in ipairs(balls) do
        if ball and ball.Parent then
            local dist = (ball.Position - hrp.Position).Magnitude
            if dist <= cfg.reach + 15 then
                if magneticWelds[ball] then
                    local offset = magneticWelds[ball]
                    local targetPos = hrp.CFrame * CFrame.new(offset) * CFrame.new(0, 0, -3)
                    ball.CFrame = targetPos
                    ball.Velocity = Vector3.zero
                    ball.RotVelocity = Vector3.zero
                else
                    local direction = (hrp.Position - ball.Position).Unit
                    local strength = math.min(cfg.magneticStrength / math.max(dist, 3) * 2, 30)
                    pcall(function()
                        local bv = ball:FindFirstChild("CLB_Mag")
                        if not bv then
                            bv = Instance.new("BodyVelocity")
                            bv.Name = "CLB_Mag"
                            bv.MaxForce = Vector3.new(50000, 50000, 50000)
                            bv.Parent = ball
                        end
                        bv.Velocity = direction * strength
                    end)
                    
                    if dist < 5 then
                        pcall(function()
                            local bv = ball:FindFirstChild("CLB_Mag")
                            if bv then bv:Destroy() end
                        end)
                        local offset = (ball.Position - hrp.Position)
                        magneticWelds[ball] = offset
                        ball.CanCollide = false
                    end
                end
            else
                if magneticWelds[ball] then
                    magneticWelds[ball] = nil
                    ball.CanCollide = true
                end
            end
        end
    end
end

local function updateFOV()
    if cfg.fovEnabled then
        Camera.FieldOfView = cfg.fovValue
    else
        Camera.FieldOfView = defaultFOV
    end
end

local function doControlBall()
    if not cfg.controlBallEnabled then return end
    if not cfg.controlBallTarget or not cfg.controlBallTarget.Parent then
        cfg.controlBallTarget = findClosestBall()
        if not cfg.controlBallTarget then return end
    end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp then return end
    
    local ball = cfg.controlBallTarget
    local moveDirection = Vector3.zero
    
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDirection = moveDirection + Camera.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDirection = moveDirection - Camera.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDirection = moveDirection - Camera.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDirection = moveDirection + Camera.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.E) then moveDirection = moveDirection + Vector3.new(0, 1, 0) end
    if UserInputService:IsKeyDown(Enum.KeyCode.Q) then moveDirection = moveDirection - Vector3.new(0, 1, 0) end
    
    if moveDirection.Magnitude > 0 then
        moveDirection = moveDirection.Unit
        pcall(function()
            local bv = ball:FindFirstChild("CLB_Ctrl")
            if not bv then
                bv = Instance.new("BodyVelocity")
                bv.Name = "CLB_Ctrl"
                bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bv.Parent = ball
            end
            bv.Velocity = moveDirection * 50
            ball.RotVelocity = Vector3.zero
        end)
    end
end

local function spectatePlayer(targetPlayer)
    if not targetPlayer or targetPlayer == Player then return end
    if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("Head") then return end
    spectateEnabled = true
    spectateTarget = targetPlayer
    originalCameraCFrame = Camera.CFrame
    Camera.CameraSubject = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
    Camera.CameraType = Enum.CameraType.Custom
end

local function stopSpectate()
    spectateEnabled = false
    spectateTarget = nil
    Camera.CameraSubject = Player.Character and Player.Character:FindFirstChildOfClass("Humanoid")
    Camera.CameraType = Enum.CameraType.Custom
    if Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") then
        Camera.CFrame = originalCameraCFrame or Player.Character.HumanoidRootPart.CFrame * CFrame.new(0, 3, 10)
    end
end

local lastKickTime = 0
local function tryPowerKick(ball)
    if not cfg.kickOn or cfg.kick <= 0 then return end
    if tick() - lastKickTime < 0.5 then return end
    if not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp2 then return end
    local hum = Player.Character:FindFirstChildOfClass("Humanoid")
    if not hum or hum.MoveDirection.Magnitude < 0.1 then return end
    local dot = ((ball.Position - hrp2.Position).Unit):Dot(hum.MoveDirection.Unit)
    if dot > 0.5 then
        lastKickTime = tick()
        pcall(function()
            local old = ball:FindFirstChild("CLB_PK")
            if old then old:Destroy() end
            local bv = Instance.new("BodyVelocity")
            bv.Name = "CLB_PK"
            bv.Velocity = hrp2.CFrame.LookVector * cfg.kick * 45 + Vector3.new(0, cfg.kick * 9, 0)
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bv.Parent = ball
            task.delay(0.15, function() 
                pcall(function() if bv and bv.Parent then bv:Destroy() end end) 
            end)
        end)
    end
end

local function doSpin()
    if not cfg.spin then return end
    if not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp2 then return end
    spinAngle = spinAngle + (dt or 0.016) * cfg.spinSpeed * 2
    hrp2.CFrame = CFrame.new(hrp2.Position) * CFrame.Angles(0, spinAngle, 0)
end

local skillPhase = 0
local skillTimer = 0
local function doSkillMode()
    if not cfg.skillMode then return end
    if not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp2 then return end
    local closest = findClosestBall()
    if closest and (closest.Position - hrp2.Position).Magnitude < cfg.reach + 2 then
        skillTimer = skillTimer + (dt or 0.016) * cfg.skillSpeed
        if skillTimer >= 1 then 
            skillTimer = 0
            skillPhase = (skillPhase + 1) % 4 
        end
        local t = skillTimer
        local offset
        if skillPhase == 0 then offset = hrp2.CFrame.LookVector * math.sin(t * math.pi) * 2
        elseif skillPhase == 1 then offset = hrp2.CFrame.RightVector * math.sin(t * math.pi) * 2
        elseif skillPhase == 2 then offset = -hrp2.CFrame.RightVector * math.sin(t * math.pi) * 2
        else offset = -hrp2.CFrame.LookVector * math.sin(t * math.pi) * 2 end
        
        local targetPos = hrp2.Position + offset + Vector3.new(0, -1.5, 0)
        closest.CFrame = CFrame.new(closest.Position:Lerp(targetPos, 0.3), closest.Position)
        closest.Velocity = Vector3.zero
        closest.RotVelocity = Vector3.zero
    end
end
-- [[ OVERDRIVE HUB - CUSTOM GUI ]] --
-- PARTE 3: ALERTA TPS, SIMULAÇÃO E CRIAÇÃO DA GUI

-- Função para Alerta TPS
local function createTPSAlert()
    if tpsAlertGui then
        tpsAlertGui:Destroy()
        tpsAlertGui = nil
    end
    
    tpsAlertGui = Instance.new("Frame")
    tpsAlertGui.Name = "TPSAlert"
    tpsAlertGui.Size = UDim2.new(0, 300, 0, 80)
    tpsAlertGui.Position = UDim2.new(0.5, -150, 0.1, 0)
    tpsAlertGui.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
    tpsAlertGui.BackgroundTransparency = 0.15
    tpsAlertGui.BorderSizePixel = 0
    tpsAlertGui.Visible = false
    tpsAlertGui.Parent = Player:WaitForChild("PlayerGui")
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = tpsAlertGui
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255, 0, 0)
    stroke.Thickness = 3
    stroke.Parent = tpsAlertGui
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -20, 0.5, 0)
    title.Position = UDim2.new(0, 10, 0, 5)
    title.BackgroundTransparency = 1
    title.Text = "⚠️ TPS PRÓXIMA!"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextSize = 24
    title.Font = Enum.Font.SourceSansBold
    title.Parent = tpsAlertGui
    
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Name = "DistanceLabel"
    distanceLabel.Size = UDim2.new(1, -20, 0.5, 0)
    distanceLabel.Position = UDim2.new(0, 10, 0.5, 0)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Text = "Distância: 0.0 studs"
    distanceLabel.TextColor3 = Color3.fromRGB(255, 255, 200)
    distanceLabel.TextSize = 18
    distanceLabel.Font = Enum.Font.SourceSans
    distanceLabel.Parent = tpsAlertGui
    
    return tpsAlertGui
end

local function updateTPSAlert()
    if not cfg.tpsAlert then
        if tpsAlertGui then tpsAlertGui.Visible = false end
        tpsAlertActive = false
        return
    end
    
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp then return end
    
    local closestTPS = findClosestTPS()
    
    if closestTPS then
        local dist = (closestTPS.Position - hrp.Position).Magnitude
        tpsDistance = dist
        
        if dist <= 3 then
            if not tpsAlertActive then
                if not tpsAlertGui then createTPSAlert() end
                tpsAlertGui.Visible = true
                tpsAlertActive = true
            end
            
            if tpsAlertGui then
                local distanceLabel = tpsAlertGui:FindFirstChild("DistanceLabel")
                if distanceLabel then
                    distanceLabel.Text = string.format("Distância: %.1f studs", dist)
                end
                
                tpsAlertGui.BackgroundTransparency = 0.1 + math.sin(tick() * 8) * 0.1
            end
        else
            if tpsAlertActive then
                tpsAlertGui.Visible = false
                tpsAlertActive = false
            end
        end
    else
        if tpsAlertActive then
            tpsAlertGui.Visible = false
            tpsAlertActive = false
        end
    end
end

-- Simulação de captura
local function simulateCatch(ball)
    if not cfg.autoCatch then return end
    if not ball or not ball.Parent then return end
    if not hrp or not hrp.Parent then return end
    
    local dist = (ball.Position - hrp.Position).Magnitude
    if dist <= cfg.trainingReach then
        pcall(function()
            local bv = ball:FindFirstChild("CatchSim")
            if not bv then
                bv = Instance.new("BodyVelocity")
                bv.Name = "CatchSim"
                bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bv.Parent = ball
            end
            local direction = (hrp.Position - ball.Position).Unit
            bv.Velocity = direction * cfg.catchIntensity * 2
            ball.RotVelocity = Vector3.zero
            
            task.delay(0.3, function()
                pcall(function()
                    if bv and bv.Parent then bv:Destroy() end
                end)
            end)
        end)
    end
end

-- ============================================
-- CRIAÇÃO DA GUI PERSONALIZADA (CUSTOM GUI)
-- ============================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "Overdrive_Hub_CustomGui"
ScreenGui.ResetOnSpawn = false
pcall(function() ScreenGui.Parent = Player:WaitForChild("PlayerGui") end)

-- Janela Principal
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 500, 0, 400)
MainFrame.Position = UDim2.new(0.5, -250, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(0, 150, 255)
MainStroke.Thickness = 2
MainStroke.Parent = MainFrame

-- Barra do Topo
local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 40)
TopBar.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
TopBar.BorderSizePixel = 0
TopBar.Parent = MainFrame

local TopBarCorner = Instance.new("UICorner")
TopBarCorner.CornerRadius = UDim.new(0, 10)
TopBarCorner.Parent = TopBar

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -90, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "OVERDRIVE HUB - V2.1"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 18
Title.Font = Enum.Font.SourceSansBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = TopBar

-- Botão de Minimizar
local MinimizeBtn = Instance.new("TextButton")
MinimizeBtn.Size = UDim2.new(0, 30, 0, 30)
MinimizeBtn.Position = UDim2.new(1, -70, 0, 5)
MinimizeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 75)
MinimizeBtn.Text = "-"
MinimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinimizeBtn.Font = Enum.Font.SourceSansBold
MinimizeBtn.TextSize = 18
MinimizeBtn.Parent = TopBar

local MinimizeCorner = Instance.new("UICorner")
MinimizeCorner.CornerRadius = UDim.new(0, 6)
MinimizeCorner.Parent = MinimizeBtn

MinimizeBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
end)

-- Botão de Fechar
local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0, 30, 0, 30)
CloseBtn.Position = UDim2.new(1, -35, 0, 5)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 40, 40)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.SourceSansBold
CloseBtn.TextSize = 16
CloseBtn.Parent = TopBar

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseBtn

CloseBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
end)

-- Barra Lateral (Abas)
local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, 130, 1, -40)
Sidebar.Position = UDim2.new(0, 0, 0, 40)
Sidebar.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
Sidebar.BorderSizePixel = 0
Sidebar.Parent = MainFrame

local TabContainer = Instance.new("Frame")
TabContainer.Size = UDim2.new(1, -130, 1, -40)
TabContainer.Position = UDim2.new(0, 130, 0, 40)
TabContainer.BackgroundTransparency = 1
TabContainer.Parent = MainFrame

local tabs = {}

local function createTab(name)
    local tabBtn = Instance.new("TextButton")
    tabBtn.Size = UDim2.new(1, -10, 0, 35)
    tabBtn.Position = UDim2.new(0, 5, 0, #tabs * 40 + 5)
    tabBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
    tabBtn.Text = name
    tabBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    tabBtn.Font = Enum.Font.SourceSansBold
    tabBtn.TextSize = 14
    tabBtn.Parent = Sidebar

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = tabBtn

    local tabContent = Instance.new("ScrollingFrame")
    tabContent.Size = UDim2.new(1, -10, 1, -10)
    tabContent.Position = UDim2.new(0, 5, 0, 5)
    tabContent.BackgroundTransparency = 1
    tabContent.BorderSizePixel = 0
    tabContent.ScrollBarThickness = 4
    tabContent.Visible = false
    tabContent.Parent = TabContainer

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 8)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = tabContent

    tabBtn.MouseButton1Click:Connect(function()
        for _, t in ipairs(tabs) do
            t.content.Visible = false
            t.btn.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
            t.btn.TextColor3 = Color3.fromRGB(200, 200, 200)
        end
        tabContent.Visible = true
        tabBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
        tabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end)

    local tabObj = {btn = tabBtn, content = tabContent, layout = layout}
    table.insert(tabs, tabObj)

    if #tabs == 1 then
        tabContent.Visible = true
        tabBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
        tabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end

    return tabContent
end

-- Criadores de Elementos da UI
local function addToggle(parent, text, defaultState, callback)
    local state = defaultState
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 35)
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -80, 1, 0)
    lbl.Position = UDim2.new(0, 10, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = Color3.fromRGB(220, 220, 220)
    lbl.Font = Enum.Font.SourceSans
    lbl.TextSize = 14
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 65, 0, 22)
    btn.Position = UDim2.new(1, -70, 0.5, -11)
    btn.BackgroundColor3 = state and Color3.fromRGB(50, 200, 50) or Color3.fromRGB(80, 80, 80)
    btn.Text = state and "LIGADO" or "DESLIGADO"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.SourceSansBold
    btn.TextSize = 10
    btn.Parent = frame

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 4)
    btnCorner.Parent = btn

    local function toggle()
        state = not state
        btn.BackgroundColor3 = state and Color3.fromRGB(50, 200, 50) or Color3.fromRGB(80, 80, 80)
        btn.Text = state and "LIGADO" or "DESLIGADO"
        callback(state)
    end

    btn.MouseButton1Click:Connect(toggle)
    return {
        Set = function(_, val)
            if state ~= val then toggle() end
        end
    }
end

local function addSlider(parent, text, min, max, defaultVal, suffix, callback)
    local val = defaultVal
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 45)
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -20, 0, 20)
    lbl.Position = UDim2.new(0, 10, 0, 2)
    lbl.BackgroundTransparency = 1
    lbl.Text = text .. ": " .. tostring(val) .. " " .. suffix
    lbl.TextColor3 = Color3.fromRGB(220, 220, 220)
    lbl.Font = Enum.Font.SourceSans
    lbl.TextSize = 14
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame

    local sliderBg = Instance.new("Frame")
    sliderBg.Size = UDim2.new(1, -20, 0, 8)
    sliderBg.Position = UDim2.new(0, 10, 0, 28)
    sliderBg.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    sliderBg.Parent = frame

    local sliderBgCorner = Instance.new("UICorner")
    sliderBgCorner.CornerRadius = UDim.new(0, 4)
    sliderBgCorner.Parent = sliderBg

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((val - min) / (max - min), 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
    fill.Parent = sliderBg

    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(0, 4)
    fillCorner.Parent = fill

    local dragging = false
    local function updateSlider(input)
        local pos = math.clamp((input.Position.X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X, 0, 1)
        val = math.floor(min + (max - min) * pos)
        fill.Size = UDim2.new(pos, 0, 1, 0)
        lbl.Text = text .. ": " .. tostring(val) .. " " .. suffix
        callback(val)
    end

    sliderBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            updateSlider(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            updateSlider(input)
        end
    end)
end

local function addButton(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -10, 0, 35)
    btn.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.SourceSansBold
    btn.TextSize = 14
    btn.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn

    btn.MouseButton1Click:Connect(callback)
end
-- [[ OVERDRIVE HUB - CUSTOM GUI ]] --
-- PARTE 4: ABAS, BOTÕES FLUTUANTES E LOOP PRINCIPAL

-- ============================================
-- CONSTRUÇÃO DAS ABAS DA GUI
-- ============================================
local mainTab = createTab("Principal")
local controlTab = createTab("Controles")
local trainingTab = createTab("Treino")
local visualTab = createTab("Personagem")
local spectateTab = createTab("Espectar")

-- ABA PRINCIPAL
autoFollowToggleObj = addToggle(mainTab, "Auto Seguir Bola", cfg.autoFollow, function(val)
    cfg.autoFollow = val
    if cfg.autoFollow then targetBall = findClosestBall() else targetBall = nil end
    if updateFloatBtnVisual then updateFloatBtnVisual() end
end)

addToggle(mainTab, "Mostrar Botão Flutuante", true, function(val)
    if floatBtn then floatBtn.Visible = val end
end)

addSlider(mainTab, "Reach (Alcance)", 1, 50, cfg.reach, "studs", function(val)
    cfg.reach = val
    us()
end)

addToggle(mainTab, "Mostrar Esfera Reach", cfg.sphere, function(val)
    cfg.sphere = val
    us()
end)

addToggle(mainTab, "Alerta TPS", cfg.tpsAlert, function(val)
    cfg.tpsAlert = val
    if not val and tpsAlertGui then
        tpsAlertGui.Visible = false
        tpsAlertActive = false
    end
end)

addToggle(mainTab, "ESP de Bolas", cfg.esp, function(val)
    cfg.esp = val
end)

addToggle(mainTab, "ESP Apenas Treino", cfg.espTrainingOnly, function(val)
    cfg.espTrainingOnly = val
end)

addButton(mainTab, "Teleportar para Bola", function()
    tp()
end)

addToggle(mainTab, "TP Bola Automático", cfg.tpBallEnabled, function(val)
    cfg.tpBallEnabled = val
end)

-- ABA CONTROLES
addToggle(controlTab, "Magnetic Ball", cfg.magneticEnabled, function(val)
    cfg.magneticEnabled = val
    if not val then
        for ball, _ in pairs(magneticWelds) do
            if ball and ball.Parent then ball.CanCollide = true end
        end
        magneticWelds = {}
    end
end)

addSlider(controlTab, "Força Magnética", 10, 200, cfg.magneticStrength, "Pwr", function(val)
    cfg.magneticStrength = val
end)

addToggle(controlTab, "Controlar Bola (WASD/E/Q)", cfg.controlBallEnabled, function(val)
    cfg.controlBallEnabled = val
    if val then
        cfg.controlBallTarget = findClosestBall()
    else
        if cfg.controlBallTarget then
            pcall(function() 
                local bv = cfg.controlBallTarget:FindFirstChild("CLB_Ctrl")
                if bv then bv:Destroy() end
            end)
        end
        cfg.controlBallTarget = nil
    end
end)

addToggle(controlTab, "Chute Potente", cfg.kickOn, function(val)
    cfg.kickOn = val
end)

addSlider(controlTab, "Força do Chute", 0, 10, cfg.kick, "Lvl", function(val)
    cfg.kick = val
end)

addToggle(controlTab, "Simular Captura", cfg.autoCatch, function(val)
    cfg.autoCatch = val
end)

addSlider(controlTab, "Intensidade Captura", 1, 10, cfg.catchIntensity, "x", function(val)
    cfg.catchIntensity = val
end)

addButton(controlTab, "Reset Configurações", function()
    cfg.reach = 10
    cfg.sphere = true
    cfg.esp = true
    cfg.autoFollow = false
    cfg.magneticEnabled = false
    cfg.controlBallEnabled = false
    cfg.kickOn = false
    cfg.tpBallEnabled = false
    cfg.autoCatch = false
    cfg.tpsAlert = false
    cfg.rgbPlayerEnabled = false
    cfg.fovEnabled = false
    cfg.spin = false
    cfg.skillMode = false
    
    if autoFollowToggleObj then autoFollowToggleObj:Set(false) end
    if tpsAlertGui then tpsAlertGui.Visible = false end
    tpsAlertActive = false
    stopRGBPlayer()
    updateFOV()
end)

-- ABA TREINO
addToggle(trainingTab, "Modo Treino", cfg.trainingMode, function(val)
    cfg.trainingMode = val
end)

addSlider(trainingTab, "Reach de Treino", 1, 50, cfg.trainingReach, "studs", function(val)
    cfg.trainingReach = val
end)

addSlider(trainingTab, "Cooldown", 0.5, 5, cfg.cooldown, "s", function(val)
    cfg.cooldown = val
    cfg.cd = val
end)

addButton(trainingTab, "Criar Bola de Treino", function()
    if not Workspace:FindFirstChild("TrainingBall") then
        local trainingBall = Instance.new("Part")
        trainingBall.Name = "TrainingBall"
        trainingBall.Size = Vector3.new(2, 2, 2)
        trainingBall.Shape = Enum.PartType.Ball
        trainingBall.BrickColor = BrickColor.new("Bright yellow")
        trainingBall.Material = Enum.Material.SmoothPlastic
        trainingBall.Anchored = false
        trainingBall.CanCollide = true
        trainingBall.Position = (hrp and hrp.Position or Vector3.new(0, 10, 0)) + Vector3.new(0, 5, 10)
        trainingBall.Parent = Workspace
        
        local hl = Instance.new("Highlight")
        hl.Adornee = trainingBall
        hl.FillColor = Color3.fromRGB(255, 255, 0)
        hl.FillTransparency = 0.3
        hl.OutlineColor = Color3.fromRGB(255, 200, 0)
        hl.Parent = trainingBall
    end
end)

addButton(trainingTab, "Destruir Bola de Treino", function()
    local tb = Workspace:FindFirstChild("TrainingBall")
    if tb then tb:Destroy() end
end)

-- ABA VISUAL
addToggle(visualTab, "Spin Mode", cfg.spin, function(val)
    cfg.spin = val
    if val then spinAngle = 0 end
end)

addSlider(visualTab, "Velocidade Spin", 1, 10, cfg.spinSpeed, "x", function(val)
    cfg.spinSpeed = val
end)

addToggle(visualTab, "Skill Mode", cfg.skillMode, function(val)
    cfg.skillMode = val
    if val then skillPhase = 0; skillTimer = 0 end
end)

addSlider(visualTab, "Velocidade Skill Mode", 1, 10, cfg.skillSpeed, "x", function(val)
    cfg.skillSpeed = val
end)

addToggle(visualTab, "RGB Player (Arco-Íris)", cfg.rgbPlayerEnabled, function(val)
    cfg.rgbPlayerEnabled = val
    if val then startRGBPlayer() else stopRGBPlayer() end
end)

addToggle(visualTab, "Modificar FOV", cfg.fovEnabled, function(val)
    cfg.fovEnabled = val
    updateFOV()
end)

addSlider(visualTab, "Valor do FOV", 30, 120, cfg.fovValue, "°", function(val)
    cfg.fovValue = val
    if cfg.fovEnabled then updateFOV() end
end)

-- ABA ESPECTAR
local specListFrame = Instance.new("Frame")
specListFrame.Size = UDim2.new(1, -10, 0, 200)
specListFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 32)
specListFrame.Parent = spectateTab

local specListCorner = Instance.new("UICorner")
specListCorner.CornerRadius = UDim.new(0, 6)
specListCorner.Parent = specListFrame

local specScroll = Instance.new("ScrollingFrame")
specScroll.Size = UDim2.new(1, -10, 1, -10)
specScroll.Position = UDim2.new(0, 5, 0, 5)
specScroll.BackgroundTransparency = 1
specScroll.ScrollBarThickness = 4
specScroll.Parent = specListFrame

local specLayout = Instance.new("UIListLayout")
specLayout.Padding = UDim.new(0, 4)
specLayout.Parent = specScroll

local function refreshPlayerList()
    for _, child in ipairs(specScroll:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= Player then
            local pBtn = Instance.new("TextButton")
            pBtn.Size = UDim2.new(1, -10, 0, 30)
            pBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
            pBtn.Text = plr.Name
            pBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            pBtn.Font = Enum.Font.SourceSans
            pBtn.TextSize = 14
            pBtn.Parent = specScroll

            local pCorner = Instance.new("UICorner")
            pCorner.CornerRadius = UDim.new(0, 4)
            pCorner.Parent = pBtn

            pBtn.MouseButton1Click:Connect(function()
                spectatePlayer(plr)
            end)
        end
    end
end

addButton(spectateTab, "Atualizar Lista", function()
    refreshPlayerList()
end)

addButton(spectateTab, "Parar de Espectar", function()
    stopSpectate()
end)

refreshPlayerList()

-- KEYBINDS
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == keybindKey then
        setAutoFollowState(not cfg.autoFollow)
        if autoFollowToggleObj then autoFollowToggleObj:Set(cfg.autoFollow) end
    end
end)

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.RightControl then
        MainFrame.Visible = not MainFrame.Visible
    end
end)

-- ============================================
-- BOLHA FLUTUANTE OVERDRIVE HUB
-- ============================================
local toggleBubbleBtn = Instance.new("TextButton")
toggleBubbleBtn.Name = "OverdriveHubToggleBubble"
toggleBubbleBtn.Size = UDim2.new(0, 50, 0, 50)
toggleBubbleBtn.Position = UDim2.new(0.05, 0, 0.2, 0)
toggleBubbleBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
toggleBubbleBtn.Text = "OVER\nDRIVE"
toggleBubbleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleBubbleBtn.Font = Enum.Font.SourceSansBold
toggleBubbleBtn.TextSize = 10
toggleBubbleBtn.Parent = ScreenGui

local bubbleCorner = Instance.new("UICorner")
bubbleCorner.CornerRadius = UDim.new(1, 0)
bubbleCorner.Parent = toggleBubbleBtn

local bubbleStroke = Instance.new("UIStroke")
bubbleStroke.Color = Color3.fromRGB(0, 150, 255)
bubbleStroke.Thickness = 2
bubbleStroke.Parent = toggleBubbleBtn

local bubbleTextStroke = Instance.new("UIStroke")
bubbleTextStroke.Color = Color3.fromRGB(0, 0, 0)
bubbleTextStroke.Thickness = 1.5
bubbleTextStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual
bubbleTextStroke.Parent = toggleBubbleBtn

local draggingBubble = false
local dragStartBubble, startPosBubble, dragInputBubble

toggleBubbleBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingBubble = true
        dragStartBubble = input.Position
        startPosBubble = toggleBubbleBtn.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                draggingBubble = false
            end
        end)
    end
end)

toggleBubbleBtn.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInputBubble = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInputBubble and draggingBubble then
        local delta = input.Position - dragStartBubble
        toggleBubbleBtn.Position = UDim2.new(startPosBubble.X.Scale, startPosBubble.X.Offset + delta.X, startPosBubble.Y.Scale, startPosBubble.Y.Offset + delta.Y)
    end
end)

local bubbleClickTime = 0
toggleBubbleBtn.MouseButton1Down:Connect(function()
    bubbleClickTime = tick()
end)

toggleBubbleBtn.MouseButton1Up:Connect(function()
    if tick() - bubbleClickTime < 0.3 then
        MainFrame.Visible = not MainFrame.Visible
    end
end)

-- ============================================
-- BOTÃO FLUTUANTE RETANGULAR
-- ============================================
local floatGui = Instance.new("ScreenGui")
floatGui.Name = "OverdriveHubFloatingGui"
floatGui.ResetOnSpawn = false
pcall(function() floatGui.Parent = Player:WaitForChild("PlayerGui") end)

floatBtn = Instance.new("TextButton")
floatBtn.Name = "AutoFollowFloatBtn"
floatBtn.Size = UDim2.new(0, 150, 0, 42)
floatBtn.Position = UDim2.new(0.85, 0, 0.2, 0)
floatBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
floatBtn.Text = "SEGUIR: OFF"
floatBtn.TextColor3 = Color3.fromRGB(255, 60, 60)
floatBtn.TextScaled = true
floatBtn.Font = Enum.Font.GothamBold
floatBtn.Parent = floatGui

local floatCorner = Instance.new("UICorner")
floatCorner.CornerRadius = UDim.new(0, 8)
floatCorner.Parent = floatBtn

local floatStroke = Instance.new("UIStroke")
floatStroke.Color = Color3.fromRGB(0, 150, 255)
floatStroke.Thickness = 1.5
floatStroke.Parent = floatBtn

local draggingFloat = false
local dragInputFloat, dragStartFloat, startPosFloat

floatBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingFloat = true
        dragStartFloat = input.Position
        startPosFloat = floatBtn.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                draggingFloat = false
            end
        end)
    end
end)

floatBtn.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInputFloat = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInputFloat and draggingFloat then
        local delta = input.Position - dragStartFloat
        floatBtn.Position = UDim2.new(startPosFloat.X.Scale, startPosFloat.X.Offset + delta.X, startPosFloat.Y.Scale, startPosFloat.Y.Offset + delta.Y)
    end
end)

function updateFloatBtnVisual()
    if cfg.autoFollow then
        floatBtn.Text = "SEGUIR: ON"
        floatBtn.TextColor3 = Color3.fromRGB(50, 255, 50)
        floatStroke.Color = Color3.fromRGB(50, 255, 50)
    else
        floatBtn.Text = "SEGUIR: OFF"
        floatBtn.TextColor3 = Color3.fromRGB(255, 60, 60)
        floatStroke.Color = Color3.fromRGB(0, 150, 255)
    end
end

floatBtn.MouseButton1Click:Connect(function()
    setAutoFollowState(not cfg.autoFollow)
    if autoFollowToggleObj then autoFollowToggleObj:Set(cfg.autoFollow) end
end)

-- Inicializações
us()
rb()
createTPSAlert()

-- ============================================
-- MAIN RENDER LOOP
-- ============================================
RunService.RenderStepped:Connect(function(delta)
    dt = delta or 0.016

    if spectateEnabled and spectateTarget then
        if not spectateTarget.Character or not spectateTarget.Character:FindFirstChild("Head") then 
            stopSpectate()
        else
            local head = spectateTarget.Character:FindFirstChild("Head")
            if head then 
                Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(head.Position + Vector3.new(0, 3, 5), head.Position), 0.1) 
            end
        end
    end
    
    if cfg.autoFollow then doAutoFollow() end
    if cfg.magneticEnabled then doMagneticBall() end
    if cfg.controlBallEnabled then doControlBall() end
    if cfg.tpBallEnabled then doTPBall() end
    
    updateTPSAlert()
    
    if cfg.autoCatch then
        for _, ball in ipairs(balls) do
            if ball and ball.Parent then
                simulateCatch(ball)
            end
        end
    end

    if cfg.touch then
        local ch = Player.Character
        if ch then
            for _, pt in ipairs(gp(ch)) do
                for _, b in ipairs(balls) do
                    if b and b.Parent and (b.Position - pt.Position).Magnitude <= cfg.reach then
                        pcall(function() firetouchinterest(b, pt, 0); firetouchinterest(b, pt, 1) end)
                        if cfg.kickOn and cfg.kick > 0 then tryPowerKick(b) end
                    end
                end
            end
        end
    end

    doSpin()
    doSkillMode()

    if sp and sp.Parent then
        local h = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")
        if h then sp.Position = h.Position end
    end

    -- ESP
    if cfg.esp then
        local h = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")
        for _, b in ipairs(balls) do
            if cfg.espTrainingOnly and b.Name ~= "TrainingBall" then
                if esps[b] then
                    pcall(function() esps[b].bb:Destroy() end)
                    pcall(function() esps[b].hl:Destroy() end)
                    esps[b] = nil
                end
                continue
            end
            
            if b and b.Parent and not esps[b] then
                pcall(function()
                    local bb = Instance.new("BillboardGui")
                    bb.Name = "CE"
                    bb.Adornee = b
                    bb.Size = UDim2.new(0, 60, 0, 35)
                    bb.StudsOffset = Vector3.new(0, 3, 0)
                    bb.AlwaysOnTop = true
                    bb.Parent = Player:WaitForChild("PlayerGui")
                    
                    local n = Instance.new("TextLabel")
                    n.Size = UDim2.new(1, 0, 0.5, 0)
                    n.BackgroundTransparency = 1
                    n.Text = " " .. b.Name
                    n.TextColor3 = b.Name == "TPS" and Color3.fromRGB(255, 50, 50) or Color3.fromRGB(50, 255, 50)
                    n.TextStrokeTransparency = 0.4
                    n.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                    n.TextScaled = true
                    n.Font = Enum.Font.SourceSansBold
                    n.Parent = bb
                    
                    local dd = Instance.new("TextLabel")
                    dd.Name = "D"
                    dd.Size = UDim2.new(1, 0, 0.5, 0)
                    dd.Position = UDim2.new(0, 0, 0.5, 0)
                    dd.BackgroundTransparency = 1
                    dd.Text = "0m"
                    dd.TextColor3 = Color3.fromRGB(255, 255, 100)
                    dd.TextStrokeTransparency = 0.4
                    dd.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                    dd.TextScaled = true
                    dd.Font = Enum.Font.SourceSans
                    dd.Parent = bb
                    
                    local hl = Instance.new("Highlight")
                    hl.Adornee = b
                    hl.FillColor = b.Name == "TPS" and Color3.fromRGB(255, 30, 30) or Color3.fromRGB(50, 255, 50)
                    hl.FillTransparency = 0.5
                    hl.OutlineColor = Color3.fromRGB(255, 200, 100)
                    hl.Parent = Camera
                    
                    esps[b] = {bb = bb, d = dd, hl = hl}
                end)
            end
            if h and esps[b] and esps[b].d then
                pcall(function()
                    esps[b].d.Text = math.floor((b.Position - h.Position).Magnitude) .. "m"
                end)
            end
        end
    else
        for b, o in pairs(esps) do
            pcall(function() o.bb:Destroy() end)
            pcall(function() o.hl:Destroy() end)
        end
        esps = {}
    end

    for b, o in pairs(esps) do
        if not b or not b.Parent then
            pcall(function() o.bb:Destroy() end)
            pcall(function() o.hl:Destroy() end)
            esps[b] = nil
        end
    end
end)

-- TASK DE LIMPEZA
task.spawn(function()
    while true do
        task.wait(0.5)
        rb()
        if not cfg.magneticEnabled then
            for _, ball in ipairs(balls) do
                pcall(function()
                    local bv = ball:FindFirstChild("CLB_Mag")
                    if bv then bv:Destroy() end
                end)
            end
        end
        if not cfg.controlBallEnabled and cfg.controlBallTarget then
            pcall(function()
                local bv = cfg.controlBallTarget:FindFirstChild("CLB_Ctrl")
                if bv then bv:Destroy() end
            end)
        end
    end
end)
