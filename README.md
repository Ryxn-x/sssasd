local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- CONFIGURAÇÕES DO CÍRCULO DE MIRA
local CIRCLE_RADIUS = 70 -- pixels (menor que antes)
local CIRCLE_COLOR = Color3.fromRGB(255, 0, 0)
local CIRCLE_TRANSPARENCY = 0.4

-- Painel GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PainelAimbotCirculo"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 220, 0, 110)
frame.Position = UDim2.new(0.05, 0, 0.35, 0)
frame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
frame.Active = true
frame.Draggable = true
frame.Parent = screenGui

local titulo = Instance.new("TextLabel")
titulo.Size = UDim2.new(1, 0, 0, 30)
titulo.Position = UDim2.new(0, 0, 0, 0)
titulo.Text = "Aimbot com Circulo"
titulo.TextColor3 = Color3.fromRGB(255, 255, 255)
titulo.BackgroundTransparency = 1
titulo.Font = Enum.Font.SourceSansBold
titulo.TextSize = 20
titulo.Parent = frame

local botao = Instance.new("TextButton")
botao.Size = UDim2.new(0.85, 0, 0, 40)
botao.Position = UDim2.new(0.075, 0, 0, 50)
botao.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
botao.Text = "Ativar Aimbot"
botao.TextColor3 = Color3.fromRGB(255, 255, 255)
botao.Font = Enum.Font.SourceSansBold
botao.TextSize = 18
botao.Parent = frame

-- Circulo de mira CENTRALIZADO e menor
local circle = Instance.new("Frame")
circle.Size = UDim2.new(0, CIRCLE_RADIUS * 2, 0, CIRCLE_RADIUS * 2)
circle.AnchorPoint = Vector2.new(0.5, 0.5)
circle.Position = UDim2.new(0.5, 0, 0.5, 0) -- Centraliza na tela
circle.BackgroundTransparency = CIRCLE_TRANSPARENCY
circle.BackgroundColor3 = CIRCLE_COLOR
circle.BorderSizePixel = 2
circle.BorderColor3 = Color3.fromRGB(255, 255, 255)
circle.Visible = false
circle.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(1,0)
corner.Parent = circle

-- Lógica do Aimbot
local ativo = false
local segurandoDireito = false
local conn = nil
local alvoAtual = nil
local alvoMortoCon = nil

-- Função para converter posição 3D para 2D na tela
local function worldToScreen(pos)
    local camera = workspace.CurrentCamera
    local screenPos, onScreen = camera:WorldToViewportPoint(pos)
    return Vector2.new(screenPos.X, screenPos.Y), onScreen
end

-- Agora o centro é sempre o centro da tela!
local function estaDentroDoCirculo(cabeca)
    local pos2D, visivel = worldToScreen(cabeca.Position)
    if not visivel then return false end
    local viewport = workspace.CurrentCamera.ViewportSize
    local centro = Vector2.new(viewport.X / 2, viewport.Y / 2)
    local dist = (pos2D - centro).Magnitude
    return dist <= CIRCLE_RADIUS
end

-- Retorna a cabeça mais próxima do centro do círculo
local function getAlvoDentroDoCirculo()
    local menorDistancia = math.huge
    local alvo = nil
    local viewport = workspace.CurrentCamera.ViewportSize
    local centro = Vector2.new(viewport.X / 2, viewport.Y / 2)
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local cabeca = player.Character:FindFirstChild("Head")
            local humanoid = player.Character:FindFirstChild("Humanoid")
            if cabeca and humanoid and humanoid.Health > 0 then
                local pos2D, visivel = worldToScreen(cabeca.Position)
                if visivel then
                    local dist = (pos2D - centro).Magnitude
                    if dist <= CIRCLE_RADIUS and dist < menorDistancia then
                        menorDistancia = dist
                        alvo = cabeca
                    end
                end
            end
        end
    end
    return alvo
end

local function aimbotLoop()
    if not ativo or not segurandoDireito then return end
    local camera = workspace.CurrentCamera
    local alvo = getAlvoDentroDoCirculo()
    if alvo and alvo.Parent:FindFirstChild("Humanoid") and alvo.Parent.Humanoid.Health > 0 then
        camera.CFrame = CFrame.new(camera.CFrame.Position, alvo.Position)
        if alvoAtual ~= alvo then
            if alvoMortoCon then
                alvoMortoCon:Disconnect()
                alvoMortoCon = nil
            end
            alvoAtual = alvo
            alvoMortoCon = alvo.Parent.Humanoid.Died:Connect(function()
                alvoAtual = nil
                alvoMortoCon = nil
            end)
        end
    else
        alvoAtual = nil
        if alvoMortoCon then
            alvoMortoCon:Disconnect()
            alvoMortoCon = nil
        end
    end
end

botao.MouseButton1Click:Connect(function()
    ativo = not ativo
    circle.Visible = ativo
    if ativo then
        botao.Text = "Desativar Aimbot"
        botao.BackgroundColor3 = Color3.fromRGB(0, 200, 0)
        if not conn then
            conn = RunService.RenderStepped:Connect(aimbotLoop)
        end
    else
        botao.Text = "Ativar Aimbot"
        botao.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
        if conn then conn:Disconnect() conn = nil end
        segurandoDireito = false
        alvoAtual = nil
        if alvoMortoCon then alvoMortoCon:Disconnect() alvoMortoCon = nil end
    end
end)

-- Controla segurar/soltar botão direito do mouse
UserInputService.InputBegan:Connect(function(input, processed)
    if not processed and input.UserInputType == Enum.UserInputType.MouseButton2 and ativo then
        segurandoDireito = true
    end
end)

UserInputService.InputEnded:Connect(function(input, processed)
    if not processed and input.UserInputType == Enum.UserInputType.MouseButton2 and ativo then
        segurandoDireito = false
        alvoAtual = nil
        if alvoMortoCon then alvoMortoCon:Disconnect() alvoMortoCon = nil end
    end
end)

-- Garante que o painel reapareça após respawn
LocalPlayer.CharacterAdded:Connect(function()
    screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end)
