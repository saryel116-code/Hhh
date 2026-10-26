--[[
    SCRIPT: Reroll Manager (BLOQUEIO TOTAL) - Defesa do Reino dos Doces
    FUNÇÃO: Botão de Toggle (Liga/Desliga) + Bloqueio de rede + Desconexão Rápida.
]]

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")
local LocalPlayer = Players.LocalPlayer

-- Variáveis de Estado
local isSaveActive = false
local savedJobId = game.JobId
local originalNamecall = nil

-- ==========================================
-- 1. CRIAÇÃO DA INTERFACE (UI) - LIMPA E ATUALIZADA
-- ==========================================

-- Destrói a UI antiga se ela ainda estiver na tela
local oldGui = CoreGui:FindFirstChild("RerollManagerUI")
if oldGui then oldGui:Destroy() end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "RerollManagerUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui
ScreenGui.DisplayOrder = 999

-- Painel Principal
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 280, 0, 280)
MainFrame.Position = UDim2.new(0.5, -140, 0.3, 0) -- Centralizado na tela
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 8)
MainCorner.Parent = MainFrame

-- Título
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, 0, 0, 35)
TitleLabel.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
TitleLabel.BorderSizePixel = 0
TitleLabel.Text = "REROLL CONTROL (DOCES)"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextSize = 14
TitleLabel.Parent = MainFrame

local TitleCorner = Instance.new("UICorner")
TitleCorner.CornerRadius = UDim.new(0, 8)
TitleCorner.Parent = TitleLabel

-- BOTÃO 1: Toggle (Ligar/Desligar)
local ToggleSaveBtn = Instance.new("TextButton")
ToggleSaveBtn.Size = UDim2.new(1, -20, 0, 45)
ToggleSaveBtn.Position = UDim2.new(0, 10, 0, 45)
ToggleSaveBtn.BackgroundColor3 = Color3.fromRGB(80, 40, 40) -- Vermelho (Desligado)
ToggleSaveBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleSaveBtn.Text = "🔴 Proteção: DESLIGADA"
ToggleSaveBtn.Font = Enum.Font.GothamBold
ToggleSaveBtn.TextSize = 14
ToggleSaveBtn.Parent = MainFrame

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(0, 6)
ToggleCorner.Parent = ToggleSaveBtn

-- BOTÃO 2: Desconexão Rápida (Kick)
local KickBtn = Instance.new("TextButton")
KickBtn.Size = UDim2.new(1, -20, 0, 45)
KickBtn.Position = UDim2.new(0, 10, 0, 100)
KickBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
KickBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
KickBtn.Text = "🚪 Desconectar Agora (Kick)"
KickBtn.Font = Enum.Font.GothamBold
KickBtn.TextSize = 14
KickBtn.Parent = MainFrame

local KickCorner = Instance.new("UICorner")
KickCorner.CornerRadius = UDim.new(0, 6)
KickCorner.Parent = KickBtn

-- BOTÃO 3: Relogar (Rejoin)
local RejoinBtn = Instance.new("TextButton")
RejoinBtn.Size = UDim2.new(1, -20, 0, 45)
RejoinBtn.Position = UDim2.new(0, 10, 0, 155)
RejoinBtn.BackgroundColor3 = Color3.fromRGB(40, 80, 150)
RejoinBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
RejoinBtn.Text = "🔄 Relogar no Servidor"
RejoinBtn.Font = Enum.Font.GothamBold
RejoinBtn.TextSize = 14
RejoinBtn.Parent = MainFrame

local RejoinCorner = Instance.new("UICorner")
RejoinCorner.CornerRadius = UDim.new(0, 6)
RejoinCorner.Parent = RejoinBtn

-- Status Label
local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -20, 0, 50)
StatusLabel.Position = UDim2.new(0, 10, 0, 210)
StatusLabel.BackgroundTransparency = 1
StatusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
StatusLabel.Text = "Status: Aguardando ação..."
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextSize = 12
StatusLabel.TextWrapped = true
StatusLabel.Parent = MainFrame

-- ==========================================
-- 2. LÓGICA DE PROTEÇÃO (O VERDADEIRO ROLLBACK)
-- ==========================================

local function updateStatus(text, color)
    StatusLabel.Text = "Status: " .. text
    StatusLabel.TextColor3 = color or Color3.fromRGB(200, 200, 200)
end

ToggleSaveBtn.MouseButton1Click:Connect(function()
    if not isSaveActive then
        -- LIGAR PROTEÇÃO
        isSaveActive = true
        ToggleSaveBtn.Text = "🟢 Proteção: LIGADA (Anti-Gasto)"
        ToggleSaveBtn.BackgroundColor3 = Color3.fromRGB(40, 150, 40) -- Verde
        updateStatus("Proteção LIGADA. Pode rolar! Se não gostar, aperte Desconectar.", Color3.fromRGB(100, 255, 100))
        
        -- O HOOK: Intercepta a comunicação do jogo
        if not originalNamecall then
            originalNamecall = hookmetamethod(game, "__namecall", function(self, ...)
                local method = getnamecallmethod()
                
                -- Se a proteção estiver ligada e o jogo tentar enviar dados de Reroll/Gasto
                if isSaveActive and method == "FireServer" then
                    local remoteName = string.lower(tostring(self.Name))
                    local className = tostring(self.ClassName)
                    
                    -- Bloqueia TUDO que pareça ser uma ação de gastar itens
                    if remoteName:find("roll") or remoteName:find("trait") or remoteName:find("gem") or remoteName:find("crystal") or remoteName:find("item") or remoteName:find("use") then
                        print("[PROTEÇÃO] Bloqueado envio para o servidor: " .. remoteName)
                        return nil -- Impede o envio (O servidor não sabe que você gastou)
                    end
                end
                
                return originalNamecall(self, ...)
            end)
        end
    else
        -- DESLIGAR PROTEÇÃO
        isSaveActive = false
        ToggleSaveBtn.Text = "🔴 Proteção: DESLIGADA"
        ToggleSaveBtn.BackgroundColor3 = Color3.fromRGB(80, 40, 40) -- Vermelho
        updateStatus("Proteção DESLIGADA. O jogo está normal.", Color3.fromRGB(255, 100, 100))
    end
end)

-- ==========================================
-- 3. LÓGICA DE DESCONEXÃO E RELOGAR
-- ==========================================

KickBtn.MouseButton1Click:Connect(function()
    updateStatus("Forçando desconexão...", Color3.fromRGB(255, 200, 0))
    task.wait(0.2)
    
    -- O Kick fecha o jogo bruscamente. Se o servidor não recebeu a mensagem
    -- de gasto (por causa do bloqueio acima), o cristal não será perdido.
    pcall(function()
        LocalPlayer:Kick("Desconexão Segura Ativada!")
    end)
end)

RejoinBtn.MouseButton1Click:Connect(function()
    updateStatus("Reconectando...", Color3.fromRGB(255, 200, 0))
    pcall(function()
        if savedJobId and savedJobId ~= "" then
            TeleportService:TeleportToPlaceInstance(game.PlaceId, savedJobId, LocalPlayer)
        else
            TeleportService:Teleport(game.PlaceId, LocalPlayer)
        end
    end)
end)

updateStatus("Pronto. Ative a Proteção antes de rolar.", Color3.fromRGB(200, 200, 200))
