-- ServerScriptService/BrainrotNotifier.lua
local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Cria RemoteEvents (ou coloque manualmente no ReplicatedStorage)
local notifyEvent = ReplicatedStorage:FindFirstChild("AdminBrainrotNotify")
local requestTeleportEvent = ReplicatedStorage:FindFirstChild("AdminRequestTeleport")
if not notifyEvent then
    notifyEvent = Instance.new("RemoteEvent")
    notifyEvent.Name = "AdminBrainrotNotify"
    notifyEvent.Parent = ReplicatedStorage
end
if not requestTeleportEvent then
    requestTeleportEvent = Instance.new("RemoteEvent")
    requestTeleportEvent.Name = "AdminRequestTeleport"
    requestTeleportEvent.Parent = ReplicatedStorage
end

-- CONFIG
local THRESHOLD = 10_000_000
local ALLOWED_ADMINS = {
}
local RATE_LIMIT_SECONDS = 300 -- evita spam por jogador (5 minutos)

-- internal rate-limit map: playerUserId -> timestamp when allowed again
local lastNotified = {}

-- Função getPlayerBrainrot: ajuste conforme seu sistema (attributes, leaderstats, datastore...)
local function getPlayerBrainrot(player)
    if player:FindFirstChild("leaderstats") and player.leaderstats:FindFirstChild("Brainrot") then
        return player.leaderstats.Brainrot.Value
    end
    -- exemplo se usar Attributes:
    if player:GetAttribute("Brainrot") then
        return player:GetAttribute("Brainrot")
    end
    return 0
end

local function notifyAdmins(targetPlayer, brainrotValue)
    local now = os.time()
    local uid = targetPlayer.UserId
    local last = lastNotified[uid] or 0
    if now - last < RATE_LIMIT_SECONDS then
        return -- já notificado recentemente
    end
    lastNotified[uid] = now

    local payload = {
        playerName = targetPlayer.Name,
        userId = targetPlayer.UserId,
        brainrot = brainrotValue,
        placeId = game.PlaceId,
        jobId = game.JobId,
        timestamp = now,
    }

    -- Dispara o RemoteEvent para todos admins presentes no jogo
    for _, pl in ipairs(Players:GetPlayers()) do
        if ALLOWED_ADMINS[pl.UserId] then
            notifyEvent:FireClient(pl, payload)
        end
    end

    -- opcional: também logar no servidor (DataStore / arquivo externo) aqui
end

-- hookup: quando jogador entra
Players.PlayerAdded:Connect(function(player)
    wait(1) -- espere leaderstats/attributes carregarem
    local brainrot = getPlayerBrainrot(player)
    if brainrot >= THRESHOLD then
        notifyAdmins(player, brainrot)
    end
    -- se usar Attributes:
    player:GetAttributeChangedSignal("Brainrot"):Connect(function()
        local val = player:GetAttribute("Brainrot") or 0
        if val >= THRESHOLD then
            notifyAdmins(player, val)
        end
    end)
end)

-- Handler: quando um admin pede teleport (validação do servidor)
requestTeleportEvent.OnServerEvent:Connect(function(adminPlayer, request)
    -- request = { placeId = number, jobId = string, targetName = string, timestamp = number }
    if not ALLOWED_ADMINS[adminPlayer.UserId] then
        return -- não autorizado
    end
    -- validações básicas:
    if type(request) ~= "table" then return end
    local placeId = tonumber(request.placeId)
    local jobId = tostring(request.jobId)
    if not placeId or jobId == "" then return end

    -- opcional: checar timestamp / rate-limits aqui
    -- Teleporta somente o admin que pediu
    local success, err = pcall(function()
        TeleportService:TeleportToPlaceInstance(placeId, jobId, adminPlayer)
    end)
    if not success then
        warn("Erro no teleport solicitado por", adminPlayer.Name, err)
        -- opcional: avisar o cliente do erro via RemoteEvent (crie outro RemoteEvent para respostas)
    end
end)
