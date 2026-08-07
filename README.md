-- NOVO SCRIPT - LINEAR DASHBOARD (TELA DEITADA / MÓVEL / MINIMIZÁVEL)
print("Linear Dashboard: Visual Neutro & Interface Otimizada")

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local localPlayer = Players.LocalPlayer
local Cam = workspace.CurrentCamera

-- =======================================================
-- ESTADOS E VARIÁVEIS GERAIS
-- =======================================================
local FlyAtivado = false
local NoclipAtivado = false
local AntiRagdollAtivado = false
local LockAtivado = false
local ShiftLockAtivado = false
local ESPAtivado = false

local VelocidadeVoo = 50
local ForcaGruda = 0.98
local AlvoAtual = nil
local MovendoTela = false

local bodyVel, bodyGyro

-- =======================================================
-- 1. VOO (FLY)
-- =======================================================
local function AtivarVoo()
    local char = localPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    bodyVel = Instance.new("BodyVelocity")
    bodyVel.MaxForce = Vector3.new(1e9, 1e9, 1e9)
    bodyVel.Velocity = Vector3.zero
    bodyVel.Parent = root

    bodyGyro = Instance.new("BodyGyro")
    bodyGyro.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
    bodyGyro.CFrame = root.CFrame
    bodyGyro.Parent = root
end

local function DesativarVoo()
    if bodyVel then bodyVel:Destroy() bodyVel = nil end
    if bodyGyro then bodyGyro:Destroy() bodyGyro = nil end
end

-- =======================================================
-- 2. ESP PONTO AZUL
-- =======================================================
local function CriarESP(plr)
    if plr == localPlayer then return end
    
    local function Aplicar(char)
        local root = char:WaitForChild("HumanoidRootPart", 10)
        if root then
            if root:FindFirstChild("LinearESP") then root.LinearESP:Destroy() end
            
            local bbg = Instance.new("BillboardGui")
            bbg.Name = "LinearESP"
            bbg.AlwaysOnTop = true
            bbg.Size = UDim2.new(0, 7, 0, 7)
            bbg.Enabled = ESPAtivado
            bbg.Parent = root

            local pnt = Instance.new("Frame", bbg)
            pnt.Size = UDim2.new(1, 0, 1, 0)
            pnt.BackgroundColor3 = Color3.fromRGB(0, 120, 215) -- Azul sóbrio (Sem Neon)
            Instance.new("UICorner", pnt).CornerRadius = UDim.new(1, 0)
        end
    end

    plr.CharacterAdded:Connect(Aplicar)
    if plr.Character then Aplicar(plr.Character) end
end

local function ToggleESP(estado)
    ESPAtivado = estado
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= localPlayer and plr.Character then
            local root = plr.Character:FindFirstChild("HumanoidRootPart")
            if root and root:FindFirstChild("LinearESP") then
                root.LinearESP.Enabled = estado
            end
        end
    end
end

-- =======================================================
-- 3. TOQUE NA TELA (MOBILE SENS)
-- =======================================================
UserInputService.TouchStarted:Connect(function() MovendoTela = true end)
UserInputService.TouchEnded:Connect(function() MovendoTela = false end)

-- =======================================================
-- 4. MOTORES PRINCIPAIS (PHYSICS & RENDER)
-- =======================================================

-- STEPPED: Física, Anti-Ragdoll, Noclip e Fly
RunService.Stepped:Connect(function()
    local char = localPlayer.Character
    local hum = char and char:FindFirstChild("Humanoid")
    local root = char and char:FindFirstChild("HumanoidRootPart")

    if hum and root then
        -- ANTI-RAGDOLL
        if AntiRagdollAtivado then
            hum.PlatformStand = false
            if hum.Sit then hum.Sit = false end

            local estado = hum:GetState()
            if estado == Enum.HumanoidStateType.Physics or estado == Enum.HumanoidStateType.FallingDown or estado == Enum.HumanoidStateType.Ragdoll then
                hum:ChangeState(Enum.HumanoidStateType.GettingUp)
                if root.Velocity.Y > 2 then
                    root.Velocity = Vector3.new(root.Velocity.X, -45, root.Velocity.Z)
                end
            end
        end

        -- NOCLIP
        if NoclipAtivado then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
        end

        -- VOO (Movimentação)
        if FlyAtivado and bodyVel and bodyGyro then
            bodyGyro.CFrame = Cam.CFrame
            local moveDir = hum.MoveDirection
            if moveDir.Magnitude > 0 then
                bodyVel.Velocity = Cam.CFrame:VectorToWorldSpace(CFrame.new(0, 0, -1).VectorToWorldSpace(moveDir)) * VelocidadeVoo
            else
                bodyVel.Velocity = Vector3.zero
            end
        end
    end
end)

-- RENDERSTEPPED: Aim Bot Supremo e Shift Lock
RunService.RenderStepped:Connect(function()
    local char = localPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChild("Humanoid")

    if not root or not hum or hum.Health <= 0 then 
        AlvoAtual = nil 
        return 
    end

    -- SHIFT LOCK
    if ShiftLockAtivado and not FlyAtivado then
        local camLook = Cam.CFrame.LookVector
        root.CFrame = CFrame.new(root.Position, root.Position + Vector3.new(camLook.X, 0, camLook.Z))
        hum.AutoRotate = false
    elseif not ShiftLockAtivado and not LockAtivado then
        hum.AutoRotate = true
    end

    -- AIM BOT SUPREMO (Destrava ao mover a câmera)
    if LockAtivado then
        if MovendoTela then
            AlvoAtual = nil
        else
            if not AlvoAtual or not AlvoAtual.Parent or AlvoAtual.Parent:FindFirstChild("Humanoid").Health <= 0 then
                local menorDist = 500
                for _, player in pairs(Players:GetPlayers()) do
                    if player ~= localPlayer and player.Character then
                        local eRoot = player.Character:FindFirstChild("HumanoidRootPart")
                        local eHum = player.Character:FindFirstChild("Humanoid")
                        if eRoot and eHum and eHum.Health > 0 then
                            local pos, naTela = Cam:WorldToViewportPoint(eRoot.Position)
                            if naTela then
                                local distCenter = (Vector2.new(pos.X, pos.Y) - Vector2.new(Cam.ViewportSize.X/2, Cam.ViewportSize.Y/2)).Magnitude
                                if distCenter < menorDist then
                                    menorDist = distCenter
                                    AlvoAtual = eRoot
                                end
                            end
                        end
                    end
                end
            end

            if AlvoAtual then
                local targetPos = AlvoAtual.Position + Vector3.new(0, 1.5, 0)
                Cam.CFrame = Cam.CFrame:Lerp(CFrame.lookAt(Cam.CFrame.Position, targetPos), ForcaGruda)
            end
        end
    else
        AlvoAtual = nil
    end
end)

-- =======================================================
-- 5. INTERFACE DASHBOARD (TELA DEITADA / MÓVEL / MINIMIZÁVEL)
-- =======================================================
local ScreenGui = Instance.new("ScreenGui", localPlayer.PlayerGui)
ScreenGui.ResetOnSpawn = false

-- JANELA PRINCIPAL (Design Horizontal Otimizado para Landscape)
local MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 210, 0, 240)
MainFrame.Position = UDim2.new(0.05, 0, 0.15, 0) -- Posicionado lateralmente para não atrapalhar a visão na horizontal
MainFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 20) -- Dark Cinza fosco (Sem cores vibrantes)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true

local Corner = Instance.new("UICorner", MainFrame)
Corner.CornerRadius = UDim.new(0, 8)

local Stroke = Instance.new("UIStroke", MainFrame)
Stroke.Color = Color3.fromRGB(38, 38, 42)
Stroke.Thickness = 1

-- CABEÇALHO / TÍTULO COM BOTÃO MINIMIZAR
local Header = Instance.new("Frame", MainFrame)
Header.Size = UDim2.new(1, 0, 0, 32)
Header.BackgroundTransparency = 1

local Title = Instance.new("TextLabel", Header)
Title.Size = UDim2.new(0.7, 0, 1, 0)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.Text = "LINEAR DASH"
Title.Font = Enum.Font.GothamBold
Title.TextSize = 10
Title.TextColor3 = Color3.fromRGB(160, 160, 170)
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.BackgroundTransparency = 1

-- BOTÃO DE MINIMIZAR (_)
local BtnMinimize = Instance.new("TextButton", Header)
BtnMinimize.Size = UDim2.new(0, 24, 0, 24)
BtnMinimize.Position = UDim2.new(1, -30, 0.5, -12)
BtnMinimize.Text = "–"
BtnMinimize.Font = Enum.Font.GothamBold
BtnMinimize.TextSize = 14
BtnMinimize.TextColor3 = Color3.fromRGB(160, 160, 170)
BtnMinimize.BackgroundColor3 = Color3.fromRGB(28, 28, 32)
BtnMinimize.BorderSizePixel = 0
Instance.new("UICorner", BtnMinimize).CornerRadius = UDim.new(0, 4)

local Divider = Instance.new("Frame", MainFrame)
Divider.Size = UDim2.new(1, -20, 0, 1)
Divider.Position = UDim2.new(0, 10, 0, 32)
Divider.BackgroundColor3 = Color3.fromRGB(32, 32, 36)
Divider.BorderSizePixel = 0

-- CONTAINER DOS BOTÕES
local Container = Instance.new("Frame", MainFrame)
Container.Size = UDim2.new(1, -16, 1, -40)
Container.Position = UDim2.new(0, 8, 0, 36)
Container.BackgroundTransparency = 1

local UIList = Instance.new("UIListLayout", Container)
UIList.Padding = UDim.new(0, 4)

-- LÓGICA DE MINIMIZAR / EXPANDIR
local Minimizado = false
BtnMinimize.MouseButton1Click:Connect(function()
    Minimizado = not Minimizado
    if Minimizado then
        Container.Visible = false
        Divider.Visible = false
        MainFrame.Size = UDim2.new(0, 210, 0, 32)
        BtnMinimize.Text = "+"
    else
        MainFrame.Size = UDim2.new(0, 210, 0, 240)
        Container.Visible = true
        Divider.Visible = true
        BtnMinimize.Text = "–"
    end
end)

-- CRIADOR DE BOTÕES TONS FOSCOS / SOBRIOS
local function CriarBotaoLinear(texto, callback)
    local Button = Instance.new("TextButton", Container)
    Button.Size = UDim2.new(1, 0, 0, 28)
    Button.BackgroundColor3 = Color3.fromRGB(25, 25, 28)
    Button.AutoButtonColor = false
    Button.Text = ""
    Button.BorderSizePixel = 0

    local BtnCorner = Instance.new("UICorner", Button)
    BtnCorner.CornerRadius = UDim.new(0, 5)

    local BtnStroke = Instance.new("UIStroke", Button)
    BtnStroke.Color = Color3.fromRGB(35, 35, 40)
    BtnStroke.Thickness = 1

    local Label = Instance.new("TextLabel", Button)
    Label.Size = UDim2.new(0.7, 0, 1, 0)
    Label.Position = UDim2.new(0, 8, 0, 0)
    Label.Text = texto
    Label.Font = Enum.Font.GothamMedium
    Label.TextSize = 10
    Label.TextColor3 = Color3.fromRGB(190, 190, 200)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.BackgroundTransparency = 1

    local StatusDot = Instance.new("Frame", Button)
    StatusDot.Size = UDim2.new(0, 5, 0, 5)
    StatusDot.Position = UDim2.new(1, -14, 0.5, -2.5)
    StatusDot.BackgroundColor3 = Color3.fromRGB(55, 55, 62)
    StatusDot.BorderSizePixel = 0
    Instance.new("UICorner", StatusDot).CornerRadius = UDim.new(1, 0)

    local ativado = false

    Button.MouseButton1Click:Connect(function()
        ativado = not ativado
        if ativado then
            Button.BackgroundColor3 = Color3.fromRGB(35, 38, 45)
            StatusDot.BackgroundColor3 = Color3.fromRGB(0, 120, 215)
            BtnStroke.Color = Color3.fromRGB(50, 60, 75)
        else
            Button.BackgroundColor3 = Color3.fromRGB(25, 25, 28)
            StatusDot.BackgroundColor3 = Color3.fromRGB(55, 55, 62)
            BtnStroke.Color = Color3.fromRGB(35, 35, 40)
        end
        callback(ativado)
    end)
end

-- INICIALIZAÇÃO DOS BOTÕES
CriarBotaoLinear("Fly", function(s)
    FlyAtivado = s
    if s then AtivarVoo() else DesativarVoo() end
end)

CriarBotaoLinear("Noclip", function(s)
    NoclipAtivado = s
end)

CriarBotaoLinear("Anti-Ragdoll", function(s)
    AntiRagdollAtivado = s
end)

CriarBotaoLinear("Aim Bot", function(s)
    LockAtivado = s
    AlvoAtual = nil
end)

CriarBotaoLinear("Shift Lock", function(s)
    ShiftLockAtivado = s
end)

CriarBotaoLinear("ESP Ponto Azul", function(s)
    ToggleESP(s)
end)

-- REGISTRO DE JOGADORES
for _, p in pairs(Players:GetPlayers()) do CriarESP(p) end
Players.PlayerAdded:Connect(CriarESP)
