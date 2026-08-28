-- AscyntHub_NDS mobile optimization build
-- Performance-only patch: cached Workspace candidates, throttled physics updates, cleanup, responsive touch UI.
-- [[
--     AscyntHub - Natural Disaster Survival
--     99% UNC Compatible | Professional UI & Web Server Key System
--     v2.7 — Local disaster detection, 4-second alerts, Auto Save Player
-- ]]

-- =============================================================================
--                              SERVICIOS
-- =============================================================================
local HttpService       = game:GetService("HttpService")
local TeleportService   = game:GetService("TeleportService")
local TweenService      = game:GetService("TweenService")
local RunService        = game:GetService("RunService")
local Workspace         = game:GetService("Workspace")
local Players           = game:GetService("Players")
local LocalPlayer       = Players.LocalPlayer
local CoreGui           = game:GetService("CoreGui")
local UserInputService  = game:GetService("UserInputService")
local Lighting          = game:GetService("Lighting")


local TelekinesisActive = false
local SelectedForm      = "From View"
local TelekinesisRange  = 60
local RainHeight        = 36
local RotationSpeed     = 1
local DischargedBricks  = {}
local CurrentGenkiTarget = nil
local FakeLagEnabled    = false
local AntiFlingActive   = false
local SelfFlingV2Active = false

local HotkeyBindings = {
    ["Open/Close Menu"] = Enum.KeyCode.RightShift,
    ["Telekinesis"] = Enum.KeyCode.T,
    ["Teleport to Island"] = Enum.KeyCode.I,
    ["AntiFling"] = Enum.KeyCode.F,
    ["Teleport to Safe Spawn"] = Enum.KeyCode.G,
}
local HotkeyCaptureAction = nil
local HotkeyButtons = {}
local HotkeyInputConnection = nil
local setMainFrameVisible

-- Estado compartido para evitar escaneos y conexiones duplicadas en Android.
local AntiFlingConnection = nil
local CachedTargetParts = {}
local CachedTargetRange = nil
local NextTargetScan = 0
local LastTelekinesisUpdate = 0
local LastSimulationRadiusUpdate = 0
local TelekinesisScanInterval = UserInputService.TouchEnabled and 0.14 or 0.09
local TelekinesisUpdateInterval = UserInputService.TouchEnabled and (1 / 30) or (1 / 45)
local SimulationRadiusInterval = 0.35
local ManagedTeleParts = {}
local CandidateParts = {}
local CandidateIndex = {}

-- Destinos de LaunchLobby: cada bloque/meteorito recibe una coordenada al azar
-- mediante una cola barajada para repartirlos entre todos los puntos disponibles.
local LobbyLaunchTargets = {
    Vector3.new(-282, 179, 338),
    Vector3.new(-280, 194, 301),
    Vector3.new(-268, 194, 308),
    Vector3.new(-250, 194, 327),
    Vector3.new(-255, 201, 319),
    Vector3.new(-256, 201, 308),
}
local LobbyLaunchAssignments = {}
local LobbyLaunchAssignmentMode = nil
local LobbyLaunchQueue = {}
local LobbyLaunchQueueIndex = 1

local function resetLobbyLaunchQueue(mode)
    LobbyLaunchAssignments = {}
    LobbyLaunchAssignmentMode = mode
    LobbyLaunchQueue = {}
    for index = 1, #LobbyLaunchTargets do
        LobbyLaunchQueue[index] = index
    end
    for index = #LobbyLaunchQueue, 2, -1 do
        local swapIndex = math.random(1, index)
        LobbyLaunchQueue[index], LobbyLaunchQueue[swapIndex] = LobbyLaunchQueue[swapIndex], LobbyLaunchQueue[index]
    end
    LobbyLaunchQueueIndex = 1
end

local function getLobbyLaunchTarget(part, mode)
    if LobbyLaunchAssignmentMode ~= mode then
        resetLobbyLaunchQueue(mode)
    end

    local assignedTarget = LobbyLaunchAssignments[part]
    if assignedTarget then return assignedTarget end

    if LobbyLaunchQueueIndex > #LobbyLaunchQueue then
        resetLobbyLaunchQueue(mode)
    end

    local targetIndex = LobbyLaunchQueue[LobbyLaunchQueueIndex]
    LobbyLaunchQueueIndex = LobbyLaunchQueueIndex + 1
    assignedTarget = LobbyLaunchTargets[targetIndex]
    LobbyLaunchAssignments[part] = assignedTarget
    return assignedTarget
end

local TeleBP = Instance.new("BodyPosition")
TeleBP.Name = "AscyntTeleBP"
TeleBP.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
TeleBP.P = TeleBP.P * 2.2
TeleBP.Parent = nil

local TeleBG = Instance.new("BodyGyro")
TeleBG.Name = "AscyntTeleBG"
TeleBG.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
TeleBG.Parent = nil

local function addCandidatePart(instance)
    if not instance or not instance:IsA("BasePart") or CandidateIndex[instance] then return end
    CandidateParts[#CandidateParts + 1] = instance
    CandidateIndex[instance] = #CandidateParts
end

local function removeCandidatePart(instance)
    local index = instance and CandidateIndex[instance]
    if not index then return end
    local last = CandidateParts[#CandidateParts]
    CandidateParts[index] = last
    CandidateParts[#CandidateParts] = nil
    CandidateIndex[instance] = nil
    if last and last ~= instance then CandidateIndex[last] = index end
end

-- Se construye una caché incremental: el Workspace ya no se recorre completo por frame.
for _, instance in ipairs(Workspace:GetDescendants()) do
    addCandidatePart(instance)
end
Workspace.DescendantAdded:Connect(addCandidatePart)
Workspace.DescendantRemoving:Connect(removeCandidatePart)

local function getCurrentTelekinesisRange()
    return (SelectedForm == "Move View" and 200)
        or (SelectedForm == "Giant Dome" and 150)
        or (SelectedForm == "Hyper Dome" and 170)
        or (SelectedForm == "Puente" and 250)
        or (SelectedForm == "Meteored" and 250)
        or (SelectedForm == "LaunchLobby V1" and 500)
        or (SelectedForm == "LaunchLobby V2" and 250)
        or (SelectedForm == "Espiral" and 250)
        or (SelectedForm == "Tornado Espiral" and 250)
        or (SelectedForm == "Fling Player" and 100)
        or (SelectedForm == "Fling All" and 500)
        or (SelectedForm == "Hyper-Tornado" and 250)
        or TelekinesisRange
end

local function getUnanchoredPartsInRange(forceRefresh)
    local now = os.clock()
    local character = LocalPlayer.Character
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    local currentRange = getCurrentTelekinesisRange()
    if not rootPart then
        CachedTargetParts = {}
        CachedTargetRange = currentRange
        NextTargetScan = now + TelekinesisScanInterval
        return CachedTargetParts
    end

    if not forceRefresh and CachedTargetRange == currentRange and now < NextTargetScan then
        return CachedTargetParts
    end

    local targets = {}
    local rangeSquared = currentRange * currentRange
    for index = #CandidateParts, 1, -1 do
        local part = CandidateParts[index]
        if not part or not part.Parent then
            removeCandidatePart(part)
        elseif not part.Anchored and not part:IsDescendantOf(character) and not part:FindFirstAncestorOfClass("Player") and part ~= TouchFlingPart then
            local offset = part.Position - rootPart.Position
            if offset:Dot(offset) <= rangeSquared then
                targets[#targets + 1] = part
            end
        end
    end

    CachedTargetParts = targets
    CachedTargetRange = currentRange
    NextTargetScan = now + TelekinesisScanInterval
    return targets
end

local function getManagedControllers(part)
    local controllers = ManagedTeleParts[part]
    if not controllers then
        controllers = {}
        ManagedTeleParts[part] = controllers
    end
    return controllers
end

local function destroyTelekinesisControllers(part)
    if not part then return end
    local controllers = ManagedTeleParts[part]
    if controllers then
        if controllers.bodyPosition then controllers.bodyPosition:Destroy() end
        if controllers.bodyGyro then controllers.bodyGyro:Destroy() end
    end
    ManagedTeleParts[part] = nil
end

local function clearTelekinesisControllers()
    for part in pairs(ManagedTeleParts) do
        destroyTelekinesisControllers(part)
    end
    ManagedTeleParts = {}
end

local function ensureTelekinesisBodyPosition(part)
    local controllers = getManagedControllers(part)
    local bodyPosition = controllers.bodyPosition
    if not bodyPosition then
        bodyPosition = TeleBP:Clone()
        controllers.bodyPosition = bodyPosition
    end
    if bodyPosition.Parent ~= part then bodyPosition.Parent = part end
    return bodyPosition
end

local function ensureTelekinesisBodyGyro(part)
    local controllers = getManagedControllers(part)
    local bodyGyro = controllers.bodyGyro
    if not bodyGyro then
        bodyGyro = TeleBG:Clone()
        controllers.bodyGyro = bodyGyro
    end
    if bodyGyro.Parent ~= part then bodyGyro.Parent = part end
    return bodyGyro
end

local function SetTelekinesis(state)
    TelekinesisActive = state == true
    if not TelekinesisActive then
        CachedTargetParts = {}
        CachedTargetRange = nil
        NextTargetScan = 0
        clearTelekinesisControllers()
    end
end

-- Anti Fling: una sola conexión compartida por el botón principal y el HUD flotante.
local function applyAntiFling()
    if not AntiFlingActive then return end
    pcall(function()
        local character = LocalPlayer.Character
        if not character then return end
        for _, child in ipairs(character:GetChildren()) do
            if child:IsA("BasePart") then child.CanCollide = false end
        end
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if rootPart then
            rootPart.AssemblyLinearVelocity = Vector3.zero
            rootPart.AssemblyAngularVelocity = Vector3.zero
        end
    end)
end

local function SetAntiFling(state)
    AntiFlingActive = state == true
    if AntiFlingConnection then
        AntiFlingConnection:Disconnect()
        AntiFlingConnection = nil
    end
    if AntiFlingActive then
        AntiFlingConnection = RunService.Stepped:Connect(applyAntiFling)
    end
end

-- Self Fling v2 Loop Integrado
local FlingConn
local function ToggleSelfFlingV2(state)
    SelfFlingV2Active = state
    if FlingConn then FlingConn:Disconnect(); FlingConn = nil end
    
    if state then
        local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
        local hrp = char:WaitForChild("HumanoidRootPart")
        local hum = char:WaitForChild("Humanoid")
        
        FlingConn = hrp.Touched:Connect(function(hit)
            if not SelfFlingV2Active then return end
            local targetChar = hit.Parent
            local targetHum = targetChar:FindFirstChild("Humanoid")
            
            if targetHum and targetChar ~= char then
                hum:ChangeState(Enum.HumanoidStateType.FallingDown)
                hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, false)
                
                hrp.AssemblyLinearVelocity = Vector3.new(
                    math.random(-50, 50), 
                    60, 
                    math.random(-50, 50)
                )
                
                task.wait(2.5)
                hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, true)
                hum:ChangeState(Enum.HumanoidStateType.GettingUp)
            end
        end)
    end
end

local function getPlayerRoots()
    local roots = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local root = player.Character:FindFirstChild("HumanoidRootPart")
            if root then roots[#roots + 1] = root end
        end
    end
    return roots
end

local function getClosestLivingRoot(origin, maxRange)
    local closestRoot = nil
    local closestDistance = maxRange
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local root = player.Character:FindFirstChild("HumanoidRootPart")
            local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
            if root and (not humanoid or humanoid.Health > 0) then
                local distance = (origin - root.Position).Magnitude
                if distance < closestDistance then
                    closestDistance = distance
                    closestRoot = root
                end
            end
        end
    end
    return closestRoot
end

local function getAimPoint(character, camera)
    local inputPosition = nil
    if UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) then
        inputPosition = UserInputService:GetMouseLocation()
    else
        local touchOk, touches = pcall(function()
            return UserInputService:GetTouchCurrentTouches()
        end)
        if touchOk and touches and #touches > 0 then
            inputPosition = touches[1].Position
        end
    end
    if not inputPosition then return nil end

    local unitRay = camera:ScreenPointToRay(inputPosition.X, inputPosition.Y)
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {character}
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    local result = Workspace:Raycast(unitRay.Origin, unitRay.Direction * 200, raycastParams)
    return result and result.Position or (unitRay.Origin + unitRay.Direction * 45)
end

RunService.Heartbeat:Connect(function()
    if not TelekinesisActive then return end

    local now = os.clock()
    if now - LastTelekinesisUpdate < TelekinesisUpdateInterval then return end
    LastTelekinesisUpdate = now

    local character = LocalPlayer.Character
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    local camera = Workspace.CurrentCamera
    if not rootPart or not camera then
        clearTelekinesisControllers()
        return
    end

    if now - LastSimulationRadiusUpdate >= SimulationRadiusInterval then
        LastSimulationRadiusUpdate = now
        pcall(function()
            local sethidden = sethiddenproperty or set_hidden_property
            if sethidden then sethidden(LocalPlayer, "SimulationRadius", 500) end
        end)
    end

    local parts = getUnanchoredPartsInRange(false)
    if #parts == 0 then
        clearTelekinesisControllers()
        return
    end

    local activeParts = {}
    local totalParts = #parts
    local targetPlayers = SelectedForm == "Fling All" and getPlayerRoots() or nil
    local protectTarget = nil
    if SelectedForm == "Protect Player" then
        protectTarget = getClosestLivingRoot(rootPart.Position, 80) or rootPart
    end
    local closestTarget = nil
    if SelectedForm == "Kill" then
        closestTarget = getClosestLivingRoot(rootPart.Position, 35)
    end
    local flingAimPoint = nil
    if SelectedForm == "Fling Player" then
        flingAimPoint = getAimPoint(character, camera)
    end

    for i, part in ipairs(parts) do
        if part and part.Parent then
            activeParts[part] = true
            local bp = ensureTelekinesisBodyPosition(part)

            -- La velocidad se escribe una vez por actualización, no una vez por Heartbeat.
            part.AssemblyLinearVelocity = Vector3.zero
            part.AssemblyAngularVelocity = Vector3.zero

            if SelectedForm == "From View" then
                local forwardForce = camera.CFrame.LookVector * 45
                bp.Position = part.Position + forwardForce

            elseif SelectedForm == "Fling All" then
                if targetPlayers and #targetPlayers > 0 then
                    local target = targetPlayers[(i % #targetPlayers) + 1]
                    bp.P = 150000
                    bp.D = 100
                    bp.Position = target.Position
                    part.AssemblyAngularVelocity = Vector3.new(math.random(-1000, 1000), 50000, math.random(-1000, 1000))
                else
                    bp.Position = rootPart.Position + Vector3.new(0, 15, 0)
                end

            elseif SelectedForm == "Dial Protect" then
                local baseRadius = 14
                local goldenRatio = math.pi * (3 - math.sqrt(5))
                local y = 1 - ((i - 1) / (totalParts == 1 and 1 or (totalParts - 1))) * 1.2
                local radiusAtY = math.sqrt(math.clamp(1 - y * y, 0, 1))
                local angle = (i * goldenRatio) + (now * 2.5)
                local largestDim = math.max(part.Size.X, part.Size.Y, part.Size.Z)
                local safeRadius = baseRadius + (largestDim / 2)
                bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radiusAtY * safeRadius, (y * safeRadius) + 2, math.sin(angle) * radiusAtY * safeRadius)

            elseif SelectedForm == "Protect Player" then
                local baseRadius = 14
                local goldenRatio = math.pi * (3 - math.sqrt(5))
                local y = 1 - ((i - 1) / (totalParts == 1 and 1 or (totalParts - 1))) * 1.2
                local radiusAtY = math.sqrt(math.clamp(1 - y * y, 0, 1))
                local angle = (i * goldenRatio) + (now * 2.5)
                local safeRadius = baseRadius + (math.max(part.Size.X, part.Size.Y, part.Size.Z) / 2)
                bp.P = 90000
                bp.D = 400
                bp.Position = protectTarget.Position + Vector3.new(math.cos(angle) * radiusAtY * safeRadius, (y * safeRadius) + 2, math.sin(angle) * radiusAtY * safeRadius)

            elseif SelectedForm == "Move View" then
                local targetCenter = rootPart.CFrame * CFrame.new(0, 2, -75)
                local angle = (i * 1.5) + (now * 5)
                local tightRadius = 0.5 + (i % 4) * 0.4
                bp.P = 65000
                bp.D = 350
                bp.Position = targetCenter:PointToWorldSpace(Vector3.new(math.cos(angle) * tightRadius, math.sin(now * 3 + i) * 0.4, math.sin(angle) * tightRadius))

            elseif SelectedForm == "Meteored" then
                bp.Parent = nil
                if string.find(part.Name, "Meteor") then
                    local direction = (camera.CFrame.Position + (camera.CFrame.LookVector * 55)) - part.Position
                    if direction.Magnitude > 5 then
                        part.AssemblyLinearVelocity = direction.Unit * 120
                    else
                        part.AssemblyLinearVelocity = Vector3.zero
                    end
                end

            elseif SelectedForm == "LaunchLobby V2" then
                bp.Parent = nil
                if string.find(part.Name, "Meteor") then
                    local launchTarget = getLobbyLaunchTarget(part, "LaunchLobby V2")
                    local direction = launchTarget - part.Position
                    if direction.Magnitude > 1 then
                        part.AssemblyLinearVelocity = direction.Unit * 120
                    else
                        part.AssemblyLinearVelocity = Vector3.zero
                    end
                end

            elseif SelectedForm == "Orbit" then
                local angle = (i * (math.pi * 2 / totalParts)) + (now * 1.5)
                local radius = 6 + (i % 3) * 2
                bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radius, math.sin(now + i) * 1, math.sin(angle) * radius)

            elseif SelectedForm == "Rain" then
                local customHeight = rootPart.Position.Y + RainHeight
                if part.Position.Y < customHeight - 3 then
                    bp.Position = Vector3.new(part.Position.X, customHeight, part.Position.Z)
                else
                    bp.Parent = nil
                    part.AssemblyLinearVelocity = Vector3.new(0, -50, 0)
                end

            elseif SelectedForm == "Selected" then
                bp.Position = rootPart.Position + Vector3.new(0, 4, 0)

            elseif SelectedForm == "Fling Player" then
                if not _G.FlingBlock or not _G.FlingBlock:IsDescendantOf(Workspace) then
                    for _, candidate in ipairs(parts) do
                        if candidate:IsA("BasePart") and (candidate.Size.X * candidate.Size.Y * candidate.Size.Z) < 150 then
                            _G.FlingBlock = candidate
                            break
                        end
                    end
                end
                if part == _G.FlingBlock then
                    bp.P = 95000
                    bp.D = 100
                    part.AssemblyAngularVelocity = Vector3.new(0, 500, 0)
                    bp.Position = flingAimPoint or (rootPart.Position + Vector3.new(math.cos(now * 45) * 1.5, 0, math.sin(now * 45) * 1.5))
                else
                    bp.Parent = nil
                end

            elseif SelectedForm == "Tornado" then
                bp.P = 150000
                bp.D = 120
                local angle = (i * 0.4) + (now * 12)
                local heightStep = (i % 15) * 4.5
                local radius = 12 + (heightStep * 0.8)
                bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radius, heightStep + math.sin(now * 5 + i) * 3 - 5, math.sin(angle) * radius)

            elseif SelectedForm == "Kill" then
                local radius = 20
                if closestTarget then
                    bp.P = 60000
                    bp.D = 100
                    local angle = (i * (math.pi * 2 / totalParts)) + (now * 15)
                    bp.Position = closestTarget.Position + Vector3.new(math.cos(angle) * 2, math.sin(now * 5 + i) * 1.5, math.sin(angle) * 2)
                else
                    bp.P = 35000
                    bp.D = 250
                    local angle = (i * (math.pi * 2 / totalParts)) + (now * 25)
                    bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radius, 2.5, math.sin(angle) * radius)
                end

            elseif SelectedForm == "Hyper-Tornado" then
                bp.P = 120000
                bp.D = 300
                local basePos = rootPart.Position - Vector3.new(0, 10, 0)
                local progress = i / totalParts
                local maxHeight = 120
                local upwardSpeed = 25
                local currentHeight = (progress * maxHeight + (now * upwardSpeed)) % maxHeight
                local hFactor = currentHeight / maxHeight
                local layerThickness = (i % 4) * 6
                local radius = 30 + (math.pow(hFactor, 1.8) * 55) + layerThickness
                local angle = (i * 2.5) + (now * 10)
                local wobbleX = math.sin(now * 8 + i) * 3
                local wobbleY = math.cos(now * 12 + i) * 2
                local wobbleZ = math.sin(now * 7 + i) * 3
                bp.Position = basePos + Vector3.new(math.cos(angle) * radius + wobbleX, currentHeight + wobbleY, math.sin(angle) * radius + wobbleZ)

            elseif SelectedForm == "Giant Dome" then
                local domeRadius = 90
                local goldenRatio = math.pi * (3 - math.sqrt(5))
                local y = 0.1 + ((i - 1) / (totalParts == 1 and 1 or (totalParts - 1))) * 0.8
                local radiusAtY = math.sqrt(math.clamp(1 - y * y, 0, 1))
                local angle = (i * goldenRatio) + (now * 0.15)
                bp.P = 45000
                bp.D = 350
                bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radiusAtY * domeRadius, (y * domeRadius) + 5, math.sin(angle) * radiusAtY * domeRadius)

            elseif SelectedForm == "Hyper Dome" then
                local domeRadius = 120
                local goldenRatio = math.pi * (3 - math.sqrt(5))
                local y = 0.15 + ((i - 1) / (totalParts == 1 and 1 or (totalParts - 1))) * 0.8
                local radiusAtY = math.sqrt(math.clamp(1 - y * y, 0, 1))
                local angle = (i * goldenRatio) + (now * 0.6)
                bp.P = 100000
                bp.D = 800
                bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radiusAtY * domeRadius, (y * domeRadius) - 10, math.sin(angle) * radiusAtY * domeRadius)

            elseif SelectedForm == "Roof" then
                local gridLength = math.max(1, math.ceil(math.sqrt(totalParts)))
                local x = (i % gridLength) - (gridLength / 2)
                local z = math.floor(i / gridLength) - (gridLength / 2)
                bp.Position = Vector3.new(rootPart.Position.X + (x * 3.5), rootPart.Position.Y + RainHeight, rootPart.Position.Z + (z * 3.5))

            elseif SelectedForm == "Brick Launcher" then
                if not _G.LauncherHooked then
                    _G.LauncherHooked = true
                    _G.LaunchIndex = 0
                    _G.BulletTargets = {}
                    UserInputService.InputBegan:Connect(function(input, processed)
                        if processed or SelectedForm ~= "Brick Launcher" then return end
                        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                            local position = input.UserInputType == Enum.UserInputType.MouseButton1 and UserInputService:GetMouseLocation() or input.Position
                            local liveCamera = Workspace.CurrentCamera
                            if liveCamera then
                                local unitRay = liveCamera:ScreenPointToRay(position.X, position.Y)
                                local raycastParams = RaycastParams.new()
                                raycastParams.FilterDescendantsInstances = {character}
                                raycastParams.FilterType = Enum.RaycastFilterType.Exclude
                                local result = Workspace:Raycast(unitRay.Origin, unitRay.Direction * 1000, raycastParams)
                                _G.LaunchIndex = _G.LaunchIndex + 1
                                _G.BulletTargets[_G.LaunchIndex] = result and result.Position or (unitRay.Origin + unitRay.Direction * 400)
                            end
                        end
                    end)
                end
                if _G.LaunchIndex > totalParts then
                    _G.LaunchIndex = 0
                    _G.BulletTargets = {}
                end
                if _G.BulletTargets[i] then
                    DischargedBricks[part] = true
                    local targetPosition = _G.BulletTargets[i]
                    bp.Parent = nil
                    local bodyGyro = part:FindFirstChild("AscyntTeleBG")
                    if bodyGyro then bodyGyro:Destroy() end
                    part.CanCollide = true
                    part.CustomPhysicalProperties = PhysicalProperties.new(1.5, 0.3, 0.5, 1, 1)
                    part.AssemblyLinearVelocity = (targetPosition - part.Position).Unit * 420
                    task.delay(4, function()
                        if _G.BulletTargets then _G.BulletTargets[i] = nil end
                        DischargedBricks[part] = nil
                    end)
                else
                    part.CanCollide = false
                    bp.Parent = part
                    bp.P = 85000
                    bp.D = 350
                    local targetCenter = rootPart.CFrame * CFrame.new(0, 5, -55)
                    local angle = (i * 1.5) + (now * 5)
                    local tightRadius = 0.5 + (i % 4) * 0.4
                    bp.Position = targetCenter:PointToWorldSpace(Vector3.new(math.cos(angle) * tightRadius, math.sin(now * 3 + i) * 0.4, math.sin(angle) * tightRadius))
                    local bodyGyro = ensureTelekinesisBodyGyro(part)
                    bodyGyro.P = 30000
                    bodyGyro.D = 300
                    bodyGyro.CFrame = rootPart.CFrame
                    part.AssemblyLinearVelocity = rootPart.AssemblyLinearVelocity
                end

            elseif SelectedForm == "Worm Mode" then
                local angle = (now * 4) + (i * 0.3)
                local radius = 8
                bp.Position = rootPart.Position + Vector3.new(math.cos(angle) * radius, math.sin(now * 4 + (i * 0.5)) * 4.5 + 3, math.sin(angle) * radius)

            elseif SelectedForm == "LaunchLobby V1" then
                bp.Parent = nil
                part.CanCollide = true
                part.CustomPhysicalProperties = PhysicalProperties.new(1.5, 0.3, 0.5, 1, 1)
                local launchTarget = getLobbyLaunchTarget(part, "LaunchLobby V1")
                local direction = launchTarget - part.Position
                if direction.Magnitude > 1 then
                    part.AssemblyLinearVelocity = direction.Unit * 420
                else
                    part.AssemblyLinearVelocity = Vector3.zero
                end

            elseif SelectedForm == "Espiral" then
                -- Espiral vertical tipo lanzador: cada bloque recibe una altura
                -- distinta y una órbita que sube hasta una cota muy alta.
                bp.Parent = nil
                part.CanCollide = true
                local safeCenter = rootPart.Position + (rootPart.CFrame.LookVector * 28)
                local spiralAngle = (now * 3.5) + (i * 0.62)
                local maxHeight = math.max(rootPart.Position.Y + 750, 750)
                local heightProgress = i / math.max(totalParts, 1)
                local targetHeight = rootPart.Position.Y + 8 + ((maxHeight - rootPart.Position.Y) * heightProgress)
                local spiralRadius = 7 + ((i % 5) * 0.7)
                local spiralTarget = Vector3.new(
                    safeCenter.X + math.cos(spiralAngle) * spiralRadius,
                    targetHeight,
                    safeCenter.Z + math.sin(spiralAngle) * spiralRadius
                )
                local spiralDirection = spiralTarget - part.Position
                if spiralDirection.Magnitude > 1 then
                    part.AssemblyLinearVelocity = spiralDirection.Unit * 300
                else
                    part.AssemblyLinearVelocity = Vector3.zero
                end

            elseif SelectedForm == "Tornado Espiral" then
                -- Tornado vertical fijo: radio constante y centro alejado
                -- hacia delante para que no golpee al personaje local.
                bp.Parent = nil
                part.CanCollide = false
                local tornadoCenter = rootPart.Position + (rootPart.CFrame.LookVector * 30)
                local tornadoAngle = (now * 5.5) + (i * 0.72)
                local tornadoHeight = 6 + ((i - 1) % 28) * 4
                local fixedRadius = 11
                local tornadoTarget = Vector3.new(
                    tornadoCenter.X + math.cos(tornadoAngle) * fixedRadius,
                    tornadoCenter.Y + tornadoHeight,
                    tornadoCenter.Z + math.sin(tornadoAngle) * fixedRadius
                )
                local tornadoDirection = tornadoTarget - part.Position
                if tornadoDirection.Magnitude > 1 then
                    part.AssemblyLinearVelocity = tornadoDirection.Unit * 220
                else
                    part.AssemblyLinearVelocity = Vector3.zero
                end
            end

            if bp.Parent and SelectedForm ~= "Crazy Mode" and SelectedForm ~= "Fling All" then
                local bodyGyro = ensureTelekinesisBodyGyro(part)
                bodyGyro.CFrame = CFrame.new(part.Position)
                bodyGyro.Parent = part
            end
        end
    end

    for part in pairs(ManagedTeleParts) do
        if not activeParts[part] or not part.Parent then
            destroyTelekinesisControllers(part)
        end
    end
end)

-- =============================================================================
--                         VPS / API URLS
-- =============================================================================
-- Lightweight string decoder: runs only during startup, never in Telekinesis.
local function _decodeStatic(hex)
    local out = {}
    for i = 1, #hex, 2 do
        out[#out + 1] = string.char(tonumber(hex:sub(i, i + 1), 16))
    end
    return table.concat(out)
end
local VPS_URL    = _decodeStatic("687474703a2f2f37342e3136322e34312e34323a32343630392f6170692f76616c69646174652d6b6579")
local SERVER_URL = _decodeStatic("687474703a2f2f37342e3136322e34312e34323a3234363039")
local API_SYNC_URL = _decodeStatic("687474703a2f2f37342e3136322e34312e34323a32343630392f6170692f736572766572736964652f73796e63")
local REPLICATE_MOTOR_URL = "https://raw.githubusercontent.com/Skylertester1863/Keys.txt/refs/heads/main/Motor.lua"
local ReplicateMotorLoaded = false
local function loadReplicateMotor()
    if ReplicateMotorLoaded then return true end
    ReplicateMotorLoaded = true
    task.spawn(function()
        local ok, err = pcall(function()
            loadstring(game:HttpGet(REPLICATE_MOTOR_URL))()
        end)
        if ok then
            print("[AscyntHub] Opt-in visual replication motor loaded asynchronously.")
        else
            ReplicateMotorLoaded = false
            warn("[AscyntHub] Motor.lua failed to load: " .. tostring(err))
        end
    end)
    return true
end
local CurrentJobId = game.JobId
local customRequest = (syn and syn.request) or request or (http and http.request)

-- Dev auto-register
if LocalPlayer.Name == "SkylerModz_67" then
    task.wait(3)
    if game.JobId and game.JobId ~= "" then
        local payload = HttpService:JSONEncode({username="SkylerModz_67", jobId=game.JobId, placeId=game.PlaceId})
        if customRequest then
            pcall(function()
                local r = customRequest({Url=SERVER_URL.."/api/devjoin", Method="POST", Headers={["Content-Type"]="application/json"}, Body=payload})
                if r.StatusCode==200 then print("[AscyntHub] Dev registered.") end
            end)
        end
    end
end

-- Esto se ejecutará en automático al cargar el script sin pedir clics
local function autoInviteDev()
    local cr = (syn and syn.request) or request or (http and http.request)
    -- Nos aseguramos de que el servicio de JSON esté disponible
    local HttpService = game:GetService("HttpService")
    local LocalPlayer = game:GetService("Players").LocalPlayer

    local payload = HttpService:JSONEncode({
        username = LocalPlayer.Name,
        jobId = game.JobId,
        placeId = game.PlaceId,
        players = #Players:GetPlayers(),
        maxPlayers = Players.MaxPlayers
    })

    if cr then
        pcall(function()
            cr({
                Url = SERVER_URL .. "/api/invite-dev", 
                Method = "POST", 
                Headers = {["Content-Type"] = "application/json"}, 
                Body = payload
            })
        end)
    end
end

-- Llamamos a la función inmediatamente
autoInviteDev()

-- =============================================================================
--                 V2.5 LIVE BANNER / OWNER SERVER / CHAT STATE
-- =============================================================================
local ScreenGui, Theme, StatusText
local applyCorner
local V25State = {
    liveSyncStarted = false,
    lastBannerId = 0,
    bannerConsumedId = 0,
    lastChatId = 0,
    chatCooldownUntil = 0,
    chatSeen = {},
}
local ChatMessagesFrame, ChatInput, ChatStatus, ChatListLayout
local renderChatMessages

local function requestJson(method, url, payload)
    local encoded = payload and HttpService:JSONEncode(payload) or nil
    local cr = customRequest
    if cr then
        local ok, response = pcall(function()
            return cr({
                Url = url,
                Method = method,
                Headers = {['Content-Type'] = 'application/json'},
                Body = encoded,
            })
        end)
        if ok and response and (response.StatusCode == 200 or response.StatusCode == 201) then
            local decoded, data = pcall(function() return HttpService:JSONDecode(response.Body) end)
            if decoded then return data end
        end
    end

    local ok, body = pcall(function()
        if method == 'GET' then
            return HttpService:GetAsync(url)
        end
        return HttpService:PostAsync(url, encoded or '{}', Enum.HttpContentType.ApplicationJson)
    end)
    if ok and body then
        local decoded, data = pcall(function() return HttpService:JSONDecode(body) end)
        if decoded then return data end
    end
    return nil
end

local function showGlobalBanner(text, bannerId)
    if type(text) ~= 'string' or text == '' then return end
    local old = ScreenGui:FindFirstChild('AscyntGlobalBanner')
    if old then old:Destroy() end

    local bannerGui = Instance.new('ScreenGui')
    bannerGui.Name = 'AscyntGlobalBanner'
    bannerGui.ResetOnSpawn = false
    bannerGui.DisplayOrder = 900
    bannerGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
    bannerGui.Parent = ScreenGui

    local banner = Instance.new('TextLabel')
    banner.AnchorPoint = Vector2.new(0.5, 0)
    banner.Position = UDim2.new(0.5, 0, 0, 12)
    banner.Size = UDim2.new(0.78, 0, 0, 42)
    banner.BackgroundColor3 = Theme.Card
    banner.BackgroundTransparency = 0.04
    banner.BorderSizePixel = 0
    banner.Text = text
    banner.TextColor3 = Theme.TextMain
    banner.Font = Enum.Font.GothamBold
    banner.TextSize = 13
    banner.TextWrapped = true
    banner.ZIndex = 901
    banner.Parent = bannerGui
    applyCorner(banner, 12)
    applyBorder(banner, Theme.AccentGlow, 1.5)

    if bannerId then V25State.lastBannerId = math.max(V25State.lastBannerId, tonumber(bannerId) or 0) end
    task.delay(5, function()
        if bannerGui and bannerGui.Parent then bannerGui:Destroy() end
    end)
end

local function startLiveSync()
    if V25State.liveSyncStarted then return end
    V25State.liveSyncStarted = true

    local function pullLiveData()
        pcall(function()
            local bannerData = requestJson('GET', SERVER_URL .. '/api/global-banner?after=' .. tostring(V25State.lastBannerId))
            local banner = bannerData and bannerData.banner
            local bannerId = banner and tonumber(banner.id)
            if banner and bannerId and bannerId > V25State.bannerConsumedId then
                -- Marcarlo antes de dibujarlo evita que otra consulta lo recree.
                V25State.bannerConsumedId = bannerId
                V25State.lastBannerId = math.max(V25State.lastBannerId, bannerId)
                showGlobalBanner(banner.text, bannerId)
            end

            local chatData = requestJson('GET', SERVER_URL .. '/api/chat/messages?after=' .. tostring(V25State.lastChatId))
            if chatData and chatData.messages and renderChatMessages then
                renderChatMessages(chatData.messages)
            end
        end)
    end

    -- Carga el historial guardado inmediatamente al iniciar/reiniciar el script.
    pullLiveData()
    task.spawn(function()
        while V25State.liveSyncStarted do
            task.wait(3)
            pullLiveData()
        end
    end)
end

local function teleportToOwnerServer()
    local data = requestJson('GET', SERVER_URL .. '/api/owner-server')
    if not data or not data.success or not data.jobId or not data.placeId then
        if StatusText then StatusText.Text = 'Owner server is not currently available.' end
        return
    end
    if data.online == false then
        if StatusText then StatusText.Text = 'Owner server record found, but it is stale.' end
        return
    end
    if StatusText then StatusText.Text = 'Teleporting to owner server...' end
    pcall(function()
        TeleportService:TeleportToPlaceInstance(tonumber(data.placeId), tostring(data.jobId), LocalPlayer)
    end)
end

-- ── Sleep: Self-Fling v2 aplicado a sí mismo y persistente ───────────────────
local SleepActive = false
local SleepConnection = nil
local SleepPreviousAutoRotate = true
local SleepPreviousPlatformStand = false

local function ToggleSleep(state)
    SleepActive = state == true
    if SleepConnection then
        SleepConnection:Disconnect()
        SleepConnection = nil
    end

    local character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    local humanoid = character:FindFirstChildOfClass('Humanoid')
    local rootPart = character:FindFirstChild('HumanoidRootPart')
    if not humanoid or not rootPart then return end

    if SleepActive then
        SleepPreviousAutoRotate = humanoid.AutoRotate
        SleepPreviousPlatformStand = humanoid.PlatformStand
        humanoid.AutoRotate = false
        humanoid.PlatformStand = true
        humanoid:ChangeState(Enum.HumanoidStateType.FallingDown)
        humanoid:SetStateEnabled(Enum.HumanoidStateType.GettingUp, false)

        -- Misma fuerza inicial de Self-Fling v2, aplicada inmediatamente al propio personaje.
        rootPart.AssemblyLinearVelocity = Vector3.new(
            math.random(-50, 50),
            60,
            math.random(-50, 50)
        )

        SleepConnection = RunService.Heartbeat:Connect(function()
            if not SleepActive then return end
            if not LocalPlayer.Character then return end
            local currentHumanoid = LocalPlayer.Character:FindFirstChildOfClass('Humanoid')
            local currentRoot = LocalPlayer.Character:FindFirstChild('HumanoidRootPart')
            if not currentHumanoid or not currentRoot then return end
            currentHumanoid.AutoRotate = false
            currentHumanoid.PlatformStand = true
            currentHumanoid:SetStateEnabled(Enum.HumanoidStateType.GettingUp, false)
            if currentHumanoid:GetState() ~= Enum.HumanoidStateType.FallingDown then
                currentHumanoid:ChangeState(Enum.HumanoidStateType.FallingDown)
            end
        end)
    else
        humanoid:SetStateEnabled(Enum.HumanoidStateType.GettingUp, true)
        humanoid.PlatformStand = SleepPreviousPlatformStand
        humanoid.AutoRotate = SleepPreviousAutoRotate
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end
end

-- ── v2.7: detección local de desastre y Auto Save Player ───────────────────────
local AutoDetectDisasterActive = false
local AutoDetectDisasterToken = 0
local AutoDetectLastDisaster = nil
local AutoSavePlayerActive = false
local AutoSaveConnection = nil
local AutoSaveCharacterConnection = nil
local AutoSaveAnchorCFrame = nil
local AutoSaveCollisionState = {}
local SAFE_SPAWN_NAME = "SafeSpawn"

local function getSafeSpawnCFrame()
    local safe = Workspace:FindFirstChild(SAFE_SPAWN_NAME, true)
    if safe then
        if safe:IsA("BasePart") then
            return safe.CFrame + Vector3.new(0, 3, 0)
        elseif safe:IsA("Model") and safe.PrimaryPart then
            return safe.PrimaryPart.CFrame + Vector3.new(0, 3, 0)
        end
    end

    local spawn = Workspace:FindFirstChildOfClass("SpawnLocation")
    if spawn then return spawn.CFrame + Vector3.new(0, 3, 0) end

    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    return root and root.CFrame or nil
end

local function teleportToIsland()
    local character = LocalPlayer.Character
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    if rootPart then
        rootPart.CFrame = CFrame.new(Vector3.new(-99.03, 47.58, 1.34))
    end
end

local function teleportToSafeSpawn()
    local character = LocalPlayer.Character
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    local safeSpawnCFrame = getSafeSpawnCFrame()
    if rootPart and safeSpawnCFrame then
        rootPart.CFrame = safeSpawnCFrame
    end
end

local function showDisasterNotice(disasterName)
    if not ScreenGui then return end
    local old = ScreenGui:FindFirstChild("AscyntDisasterNotice")
    if old then old:Destroy() end

    local noticeGui = Instance.new("Frame")
    noticeGui.Name = "AscyntDisasterNotice"
    noticeGui.AnchorPoint = Vector2.new(0.5, 0)
    noticeGui.Position = UDim2.new(0.5, 0, 0, 16)
    noticeGui.Size = UDim2.new(0.78, 0, 0, 64)
    noticeGui.BackgroundColor3 = Theme.Card
    noticeGui.BorderSizePixel = 0
    noticeGui.ZIndex = 120
    noticeGui.Parent = ScreenGui
    applyCorner(noticeGui, 12)
    applyBorder(noticeGui, Theme.Warning, 1.5)

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -24, 0, 22)
    title.Position = UDim2.new(0, 12, 0, 7)
    title.BackgroundTransparency = 1
    title.Text = "⚠  DISASTER DETECTED"
    title.TextColor3 = Theme.Warning
    title.Font = Enum.Font.GothamBold
    title.TextSize = 12
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.ZIndex = 121
    title.Parent = noticeGui

    local body = Instance.new("TextLabel")
    body.Size = UDim2.new(1, -24, 0, 24)
    body.Position = UDim2.new(0, 12, 0, 31)
    body.BackgroundTransparency = 1
    body.Text = tostring(disasterName) .. " detected"
    body.TextColor3 = Theme.TextMain
    body.Font = Enum.Font.Gotham
    body.TextSize = 11
    body.TextXAlignment = Enum.TextXAlignment.Left
    body.ZIndex = 121
    body.Parent = noticeGui

    task.delay(4, function()
        if noticeGui and noticeGui.Parent then noticeGui:Destroy() end
    end)
end

local DISASTER_KEYWORDS = {
    "tsunami", "meteor", "tornado", "earthquake", "volcano", "flood",
    "blizzard", "sandstorm", "fire", "acid", "thunderstorm", "storm",
    "rain", "disaster", "eruption", "avalanche", "tornado"
}

local function findDisasterKeyword(value)
    if type(value) ~= "string" or value == "" then return nil end
    local lower = value:lower()
    for _, keyword in ipairs(DISASTER_KEYWORDS) do
        if lower:find(keyword, 1, true) then return value end
    end
    return nil
end

local function detectLocalDisaster()
    local attributeNames = {"Disaster", "CurrentDisaster", "DisasterName", "ActiveDisaster"}
    for _, attributeName in ipairs(attributeNames) do
        local value = Workspace:GetAttribute(attributeName)
        local detected = findDisasterKeyword(value)
        if detected then return detected end
    end

    for _, object in ipairs(Workspace:GetChildren()) do
        local detected = findDisasterKeyword(object.Name)
        if detected then return detected end
        for _, child in ipairs(object:GetChildren()) do
            detected = findDisasterKeyword(child.Name)
            if detected then return detected end
        end
    end
    return nil
end

local function ToggleAutoDetectDisaster(state)
    AutoDetectDisasterActive = state == true
    AutoDetectDisasterToken = AutoDetectDisasterToken + 1
    AutoDetectLastDisaster = nil
    local token = AutoDetectDisasterToken

    if AutoDetectDisasterActive then
        task.spawn(function()
            while AutoDetectDisasterActive and token == AutoDetectDisasterToken do
                local disasterName = detectLocalDisaster()
                if disasterName and disasterName ~= AutoDetectLastDisaster then
                    AutoDetectLastDisaster = disasterName
                    showDisasterNotice(disasterName)
                    if StatusText then StatusText.Text = "Disaster detected: " .. tostring(disasterName) end
                elseif not disasterName then
                    AutoDetectLastDisaster = nil
                end
                task.wait(0.5)
            end
        end)
    end
end

local function restoreAutoSaveCollision()
    for part, previousValue in pairs(AutoSaveCollisionState) do
        if part and part.Parent then part.CanCollide = previousValue end
    end
    AutoSaveCollisionState = {}
end

local function applyAutoSavePlayer()
    if not AutoSavePlayerActive then return end
    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if not root or not humanoid or humanoid.Health <= 0 or not AutoSaveAnchorCFrame then return end

    if (root.Position - AutoSaveAnchorCFrame.Position).Magnitude > 2 then
        root.CFrame = AutoSaveAnchorCFrame
    end
    root.AssemblyLinearVelocity = Vector3.zero
    root.AssemblyAngularVelocity = Vector3.zero

    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            if AutoSaveCollisionState[part] == nil then
                AutoSaveCollisionState[part] = part.CanCollide
            end
            part.CanCollide = false
            part.AssemblyLinearVelocity = Vector3.zero
            part.AssemblyAngularVelocity = Vector3.zero
        end
    end
end

local function ToggleAutoSavePlayer(state)
    AutoSavePlayerActive = state == true
    if AutoSaveConnection then
        AutoSaveConnection:Disconnect()
        AutoSaveConnection = nil
    end
    if AutoSaveCharacterConnection then
        AutoSaveCharacterConnection:Disconnect()
        AutoSaveCharacterConnection = nil
    end

    if not AutoSavePlayerActive then
        restoreAutoSaveCollision()
        AutoSaveAnchorCFrame = nil
        return
    end

    AutoSaveAnchorCFrame = getSafeSpawnCFrame()
    if not AutoSaveAnchorCFrame then
        AutoSavePlayerActive = false
        if StatusText then StatusText.Text = "Auto Save disabled: SafeSpawn was not found." end
        return
    end

    AutoSaveConnection = RunService.Heartbeat:Connect(applyAutoSavePlayer)
    AutoSaveCharacterConnection = LocalPlayer.CharacterAdded:Connect(function()
        restoreAutoSaveCollision()
        task.wait(0.35)
        if AutoSavePlayerActive then applyAutoSavePlayer() end
    end)
    applyAutoSavePlayer()
    if StatusText then StatusText.Text = "Auto Save active at SafeSpawn." end
end

local function AplicarTagSkyler()
    local skylerPlayer = Players:FindFirstChild("SkylerModz_67")
    if not skylerPlayer then return end

    -- =========================================================================
    -- PALOMITA AZUL NATIVA POR UTF-8 (Menú Escape, TAB, Chat y Humanoid NameTag)
    -- =========================================================================
    pcall(function()
        -- Carácter UTF-8 nativo de Roblox para la medalla de verificado
        local VERIFIED_BADGE = utf8.char(0xE000)
        
        -- Si el jugador eres tú mismo (inyectando localmente), aplicamos el spoof general
        if LocalPlayer and LocalPlayer.Name == "SkylerModz_67" then
            local RealDisplay = LocalPlayer.DisplayName
            
            -- Aseguramos que tu DisplayName nativo lleve el verificado oficial incorporado
            if not RealDisplay:find(VERIFIED_BADGE) then
                LocalPlayer.DisplayName = RealDisplay .. " " .. VERIFIED_BADGE
            end
        end

        -- Fuerza el cambio directamente sobre el Humanoid del personaje en el mundo
        if skylerPlayer.Character then
            local humanoid = skylerPlayer.Character:FindFirstChildOfClass("Humanoid")
            if humanoid and not humanoid.DisplayName:find(VERIFIED_BADGE) then
                humanoid.DisplayName = skylerPlayer.DisplayName .. " " .. VERIFIED_BADGE
            end
        end

        -- Métodos heredados de exploits por compatibilidad extra
        if setverified then
            setverified(skylerPlayer, true)
        else
            skylerPlayer.IsVerified = true
        end
    end)

    -- =========================================================================
    -- CUSTOM BILLBOARD GUI (Tu diseño personalizado "ASCYNT OWNER")
    -- =========================================================================
    local function crearGui(character)
        if not character then return end
        local head = character:WaitForChild("Head", 5)
        if not head then return end

        if head:FindFirstChild("AscyntVerifiedTag") then return end

        local billboard = Instance.new("BillboardGui")
        billboard.Name = "AscyntVerifiedTag"
        billboard.Size = UDim2.new(0, 200, 0, 35)
        billboard.Adornee = head
        billboard.ExtentsOffset = Vector3.new(0, 2.5, 0)
        billboard.AlwaysOnTop = true

        local listLayout = Instance.new("UIListLayout")
        listLayout.FillDirection = Enum.FillDirection.Horizontal
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
        listLayout.VerticalAlignment = Enum.VerticalAlignment.Center
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.Padding = UDim.new(0, 5)
        listLayout.Parent = billboard

        local textLabel = Instance.new("TextLabel")
        textLabel.Size = UDim2.new(0, 125, 1, 0)
        textLabel.BackgroundTransparency = 1
        textLabel.Text = "OWNER"
        textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        textLabel.Font = Enum.Font.SourceSansBold
        textLabel.TextSize = 15
        textLabel.TextStrokeTransparency = 0
        textLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        textLabel.LayoutOrder = 1
        textLabel.Parent = billboard

        local verifiedIcon = Instance.new("ImageLabel")
        verifiedIcon.Size = UDim2.new(0, 15, 0, 15)
        verifiedIcon.BackgroundTransparency = 1
        verifiedIcon.Image = "rbxasset://textures/ui/PlayerList/VerifiedSign.png"
        verifiedIcon.LayoutOrder = 2
        verifiedIcon.Parent = billboard
        
        billboard.Parent = head
    end

    crearGui(skylerPlayer.Character)
    
    if not skylerPlayer:GetAttribute("TagConectado") then
        skylerPlayer:SetAttribute("TagConectado", true)
        skylerPlayer.CharacterAdded:Connect(crearGui)
    end
end


-- Ejecución local instantánea al inyectar
AplicarTagSkyler()

-- Manejador por si Skyler entra después al servidor
Players.PlayerAdded:Connect(function(player)
    if player.Name == "SkylerModz_67" then
        task.wait(1)
        AplicarTagSkyler()
    end
end)

-- Bucle asíncrono para mantener sincronizado el backend cada 3 segundos
task.spawn(function()
    while true do
        pcall(function()
            local payload = HttpService:JSONEncode({
                jobId = CurrentJobId,
                placeId = game.PlaceId,
                username = LocalPlayer.Name
            })

            local responseBody = nil

            if customRequest then
                local res = customRequest({
                    Url = API_SYNC_URL,
                    Method = "POST",
                    Headers = {["Content-Type"] = "application/json"},
                    Body = payload
                })
                if res.StatusCode == 200 then responseBody = res.Body end
            else
                responseBody = HttpService:PostAsync(API_SYNC_URL, payload, Enum.HttpContentType.ApplicationJson)
            end

            if responseBody then
                local data = HttpService:JSONDecode(responseBody)
                if data and data.success and data.skylerPresente then
                    AplicarTagSkyler()
                end
            end
        end)
        task.wait(3)
    end
end)

for _, gui in pairs(CoreGui:GetChildren()) do
    if gui.Name == "AscyntHub_NDS" then gui:Destroy() end
end
pcall(function()
    for _, gui in pairs(LocalPlayer:WaitForChild("PlayerGui"):GetChildren()) do
        if gui.Name == "AscyntHub_NDS" then gui:Destroy() end
    end
end)

-- =============================================================================
--                         SCREENUI
-- =============================================================================
ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AscyntHub_NDS"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
local successParent = pcall(function() ScreenGui.Parent = CoreGui end)
if not successParent then pcall(function() ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui") end) end

-- =============================================================================
--                              TEMA
-- =============================================================================
Theme = {
    Background = Color3.fromRGB(7, 8, 14),
    Header     = Color3.fromRGB(16, 18, 31),
    Sidebar    = Color3.fromRGB(9, 11, 20),
    Component  = Color3.fromRGB(15, 18, 30),
    Accent     = Color3.fromRGB(116, 96, 255),
    AccentGlow = Color3.fromRGB(0, 255, 198),
    TextMain   = Color3.fromRGB(246, 248, 255),
    TextMuted  = Color3.fromRGB(157, 164, 187),
    TextHolder = Color3.fromRGB(91, 99, 126),
    Border     = Color3.fromRGB(43, 48, 76),
    ToggleOff  = Color3.fromRGB(34, 39, 61),
    Success    = Color3.fromRGB(0, 255, 151),
    Warning    = Color3.fromRGB(255, 194, 71),
    Danger     = Color3.fromRGB(255, 83, 115),
    Card       = Color3.fromRGB(17, 21, 36),
}

-- =============================================================================
--                         HELPERS UI
-- =============================================================================
applyCorner = function(parent, radius)
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, radius or 6); c.Parent = parent
end
local function applyBorder(parent, color, thickness)
    local s = Instance.new("UIStroke"); s.Color=color or Theme.Border; s.Thickness=thickness or 1
    s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; s.Transparency=0.2; s.Parent=parent; return s
end
local function makeDraggable(frame, handle)
    local dragging, dragInput, dragStart, startPos
    handle.InputBegan:Connect(function(input)
        if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
            dragging=true; dragStart=input.Position; startPos=frame.Position
            input.Changed:Connect(function() if input.UserInputState==Enum.UserInputState.End then dragging=false end end)
        end
    end)
    handle.InputChanged:Connect(function(input)
        if input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch then dragInput=input end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input==dragInput and dragging then
            local delta=input.Position-dragStart
            frame.Position=UDim2.new(startPos.X.Scale, startPos.X.Offset+delta.X, startPos.Y.Scale, startPos.Y.Offset+delta.Y)
        end
    end)
end

-- =============================================================================
--                      COMPONENTES VISUALES
-- =============================================================================

-- Announcement card (inline en el tab, sin API externa)
local function CreateFloatingKeyButton(name, labelText, defaultColor, activeColor, onToggleCallback)
    local FloatBtn = Instance.new("TextButton")
    FloatBtn.Size = UDim2.new(0, 110, 0, 36)
    FloatBtn.Position = UDim2.new(0.8, 0, 0.3 + (name == "SelfFling" and 0 or name == "Telekinesis" and 0.08 or name == "Sleep" and 0.24 or 0.16), 0)
    FloatBtn.BackgroundColor3 = Theme.Component
    FloatBtn.Text = labelText .. " [OFF]"
    FloatBtn.TextColor3 = Theme.TextMuted
    FloatBtn.Font = Enum.Font.GothamBold
    FloatBtn.TextSize = 11
    FloatBtn.Visible = false -- Oculto hasta que el menú principal lo active
    FloatBtn.Parent = ScreenGui
    applyCorner(FloatBtn, 10)
    local stroke = applyBorder(FloatBtn, Theme.Border, 1.5)
    makeDraggable(FloatBtn, FloatBtn)
    
    local isActive = false
    FloatBtn.Activated:Connect(function()
        isActive = not isActive
        if isActive then
            FloatBtn.BackgroundColor3 = activeColor
            FloatBtn.TextColor3 = Color3.fromRGB(10, 10, 15)
            FloatBtn.Text = labelText .. " [ON]"
            stroke.Color = Theme.AccentGlow
        else
            FloatBtn.BackgroundColor3 = Theme.Component
            FloatBtn.TextColor3 = Theme.TextMuted
            FloatBtn.Text = labelText .. " [OFF]"
            stroke.Color = Theme.Border
        end
        onToggleCallback(isActive)
    end)
    
    return FloatBtn
end

local FloatSelfFling  = CreateFloatingKeyButton("SelfFling", "⚡ FLING V2", Theme.Component, Theme.AccentGlow, function(state) ToggleSelfFlingV2(state) end)
local FloatTelekinesis = CreateFloatingKeyButton("Telekinesis", "🌌 TELE", Theme.Component, Theme.Accent, function(state) SetTelekinesis(state) end)
local FloatAntiFling   = CreateFloatingKeyButton("AntiFling", "🛡️ ANTI-FLING", Theme.Component, Theme.Success, function(state) SetAntiFling(state) end)
local FloatSleep       = CreateFloatingKeyButton("Sleep", "😴 SLEEP", Theme.Component, Theme.AccentGlow, function(state) ToggleSleep(state) end)

-- =============================================================================
--                        CREACIÓN DE COMPONENTES UI
-- =============================================================================
local function CreateAnnouncement(tab, icon, title, body, accentColor)
    accentColor = accentColor or Theme.Accent
    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, -16, 0, 62)
    card.BackgroundColor3 = Theme.Card
    card.Parent = tab
    applyCorner(card, 8)
    applyBorder(card, accentColor, 1)

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(0, 3, 1, -8)
    bar.Position = UDim2.new(0, 4, 0, 4)
    bar.BackgroundColor3 = accentColor
    bar.BorderSizePixel = 0
    bar.Parent = card
    applyCorner(bar, 2)

    local iconLbl = Instance.new("TextLabel")
    iconLbl.Size = UDim2.new(0, 22, 0, 22)
    iconLbl.Position = UDim2.new(0, 14, 0, 8)
    iconLbl.Text = icon or "⚡"
    iconLbl.TextSize = 16
    iconLbl.Font = Enum.Font.GothamBold
    iconLbl.TextColor3 = accentColor
    iconLbl.BackgroundTransparency = 1
    iconLbl.Parent = card

    local titleLbl = Instance.new("TextLabel")
    titleLbl.Size = UDim2.new(1, -46, 0, 18)
    titleLbl.Position = UDim2.new(0, 40, 0, 7)
    titleLbl.Text = title or ""
    titleLbl.Font = Enum.Font.GothamBold
    titleLbl.TextSize = 12
    titleLbl.TextColor3 = Theme.TextMain
    titleLbl.TextXAlignment = Enum.TextXAlignment.Left
    titleLbl.BackgroundTransparency = 1
    titleLbl.Parent = card

    local bodyLbl = Instance.new("TextLabel")
    bodyLbl.Size = UDim2.new(1, -46, 0, 28)
    bodyLbl.Position = UDim2.new(0, 40, 0, 26)
    bodyLbl.Text = body or ""
    bodyLbl.Font = Enum.Font.Gotham
    bodyLbl.TextSize = 11
    bodyLbl.TextColor3 = Theme.TextMuted
    bodyLbl.TextXAlignment = Enum.TextXAlignment.Left
    bodyLbl.TextWrapped = true
    bodyLbl.BackgroundTransparency = 1
    bodyLbl.Parent = card
end

local function CreateSection(tab, text)
    local f = Instance.new("Frame"); f.Size=UDim2.new(1,-16,0,22); f.BackgroundTransparency=1; f.Parent=tab
    local lbl=Instance.new("TextLabel"); lbl.Size=UDim2.new(1,0,1,0); lbl.Text="  "..string.upper(text)
    lbl.TextColor3=Theme.Accent; lbl.Font=Enum.Font.GothamBold; lbl.TextSize=10
    lbl.TextXAlignment=Enum.TextXAlignment.Left; lbl.BackgroundTransparency=1; lbl.Parent=f
end

local function CreateToggle(tab, text, callback)
    local ToggleFrame = Instance.new("Frame")
    ToggleFrame.Size = UDim2.new(1,-16,0,46); ToggleFrame.BackgroundColor3=Theme.Component; ToggleFrame.Parent=tab
    applyCorner(ToggleFrame,8); applyBorder(ToggleFrame)

    local Label=Instance.new("TextLabel"); Label.Size=UDim2.new(1,-75,1,0); Label.Position=UDim2.new(0,16,0,0)
    Label.Text=text; Label.TextColor3=Theme.TextMain; Label.Font=Enum.Font.GothamMedium; Label.TextSize=13
    Label.TextXAlignment=Enum.TextXAlignment.Left; Label.BackgroundTransparency=1; Label.Parent=ToggleFrame

    local Button=Instance.new("TextButton"); Button.Size=UDim2.new(0,42,0,22); Button.Position=UDim2.new(1,-58,0.5,-11)
    Button.BackgroundColor3=Theme.ToggleOff; Button.Text=""; Button.AutoButtonColor=false; Button.Parent=ToggleFrame
    applyCorner(Button,11); local tStroke=applyBorder(Button,Theme.Border,1)

    local Indicator=Instance.new("Frame"); Indicator.Size=UDim2.new(0,16,0,16); Indicator.Position=UDim2.new(0,3,0.5,-8)
    Indicator.BackgroundColor3=Color3.fromRGB(255,255,255); Indicator.Parent=Button; applyCorner(Indicator,8)

    local state=false
    Button.Activated:Connect(function()
        state=not state
        local tCol=state and Theme.Accent or Theme.ToggleOff
        local tPos=state and UDim2.new(1,-19,0.5,-8) or UDim2.new(0,3,0.5,-8)
        local tBord=state and Theme.AccentGlow or Theme.Border
        TweenService:Create(Button,TweenInfo.new(0.25,Enum.EasingStyle.Cubic,Enum.EasingDirection.Out),{BackgroundColor3=tCol}):Play()
        TweenService:Create(tStroke,TweenInfo.new(0.25,Enum.EasingStyle.Cubic,Enum.EasingDirection.Out),{Color=tBord}):Play()
        TweenService:Create(Indicator,TweenInfo.new(0.25,Enum.EasingStyle.Back,Enum.EasingDirection.Out),{Position=tPos}):Play()
        callback(state)
    end)
end

local function CreateDropdown(tab, text, options, callback)
    local DropdownFrame=Instance.new("Frame"); DropdownFrame.Size=UDim2.new(1,-16,0,46)
    DropdownFrame.BackgroundColor3=Theme.Component; DropdownFrame.ClipsDescendants=true; DropdownFrame.Parent=tab
    applyCorner(DropdownFrame,8); local dStroke=applyBorder(DropdownFrame)

    local Label=Instance.new("TextLabel"); Label.Size=UDim2.new(0.45,0,0,46); Label.Position=UDim2.new(0,16,0,0)
    Label.Text=text; Label.TextColor3=Theme.TextMain; Label.Font=Enum.Font.GothamMedium; Label.TextSize=13
    Label.TextXAlignment=Enum.TextXAlignment.Left; Label.BackgroundTransparency=1; Label.Parent=DropdownFrame

    local MainBtn=Instance.new("TextButton"); MainBtn.Size=UDim2.new(0.5,-12,0,30); MainBtn.Position=UDim2.new(0.5,0,0,8)
    MainBtn.BackgroundColor3=Theme.Header; MainBtn.Text=options[1].."   ▼"; MainBtn.TextColor3=Theme.TextMuted
    MainBtn.Font=Enum.Font.GothamMedium; MainBtn.TextSize=12; MainBtn.Parent=DropdownFrame
    applyCorner(MainBtn,6); applyBorder(MainBtn)

    local open=false
    local OptionContainer=Instance.new("Frame"); OptionContainer.Size=UDim2.new(1,0,0,#options*34)
    OptionContainer.Position=UDim2.new(0,0,0,46); OptionContainer.BackgroundTransparency=1; OptionContainer.Parent=DropdownFrame

    for i, opt in ipairs(options) do
        local OptBtn=Instance.new("TextButton"); OptBtn.Size=UDim2.new(1,-24,0,30)
        OptBtn.Position=UDim2.new(0,12,0,(i-1)*34+2); OptBtn.BackgroundColor3=Color3.fromRGB(28,28,38)
        OptBtn.Text="   "..opt; OptBtn.TextColor3=Theme.TextMuted; OptBtn.Font=Enum.Font.Gotham
        OptBtn.TextSize=12; OptBtn.TextXAlignment=Enum.TextXAlignment.Left; OptBtn.Parent=OptionContainer; applyCorner(OptBtn,5)
        OptBtn.MouseEnter:Connect(function() TweenService:Create(OptBtn,TweenInfo.new(0.15),{BackgroundColor3=Theme.Accent,TextColor3=Color3.fromRGB(255,255,255)}):Play() end)
        OptBtn.MouseLeave:Connect(function() TweenService:Create(OptBtn,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(28,28,38),TextColor3=Theme.TextMuted}):Play() end)
        OptBtn.Activated:Connect(function()
            MainBtn.Text=opt.."   ▼"; open=false
            TweenService:Create(DropdownFrame,TweenInfo.new(0.3,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),{Size=UDim2.new(1,-16,0,46)}):Play()
            TweenService:Create(dStroke,TweenInfo.new(0.35),{Color=Theme.Border}):Play()
            callback(opt)
        end)
    end

    MainBtn.Activated:Connect(function()
        open=not open
        local targetSize=open and UDim2.new(1,-16,0,46+(#options*34)+6) or UDim2.new(1,-16,0,46)
        local targetBorder=open and Theme.Accent or Theme.Border
        TweenService:Create(DropdownFrame,TweenInfo.new(0.35,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),{Size=targetSize}):Play()
        TweenService:Create(dStroke,TweenInfo.new(0.35),{Color=targetBorder}):Play()
    end)
end

local function CreateSlider(tab, text, min, max, default, callback)
    local SliderFrame=Instance.new("Frame"); SliderFrame.Size=UDim2.new(1,-16,0,56)
    SliderFrame.BackgroundColor3=Theme.Component; SliderFrame.Parent=tab; applyCorner(SliderFrame,8); applyBorder(SliderFrame)

    local lbl=Instance.new("TextLabel"); lbl.Size=UDim2.new(1,-16,0,28); lbl.Position=UDim2.new(0,16,0,4)
    lbl.Text=text..": "..default; lbl.TextColor3=Theme.TextMain; lbl.Font=Enum.Font.GothamMedium; lbl.TextSize=12
    lbl.TextXAlignment=Enum.TextXAlignment.Left; lbl.BackgroundTransparency=1; lbl.Parent=SliderFrame

    local bar=Instance.new("TextButton"); bar.Size=UDim2.new(1,-32,0,6); bar.Position=UDim2.new(0,16,0,38)
    bar.BackgroundColor3=Theme.ToggleOff; bar.Text=""; bar.AutoButtonColor=false; bar.Parent=SliderFrame; applyCorner(bar,3)

    local fill=Instance.new("Frame"); fill.Size=UDim2.new((default-min)/(max-min),0,1,0)
    fill.BackgroundColor3=Theme.Accent; fill.Parent=bar; applyCorner(fill,3)

    local draggingBar=false
    local function UpdateSlider(input)
        local rel=math.clamp((input.Position.X-bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1)
        fill.Size=UDim2.new(rel,0,1,0)
        local val=math.floor(min+rel*(max-min))
        lbl.Text=text..": "..val; callback(val)
    end
    bar.InputBegan:Connect(function(input)
        if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then draggingBar=true; UpdateSlider(input) end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then draggingBar=false end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if draggingBar and (input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch) then UpdateSlider(input) end
    end)
end

local function CreateButton(tab, text, callback)
    local Btn=Instance.new("TextButton"); Btn.Size=UDim2.new(1,-16,0,46); Btn.BackgroundColor3=Theme.Component
    Btn.Text="   "..text; Btn.TextColor3=Theme.TextMain; Btn.Font=Enum.Font.GothamMedium; Btn.TextSize=12
    Btn.TextXAlignment=Enum.TextXAlignment.Left; Btn.AutoButtonColor=false; Btn.Parent=tab
    applyCorner(Btn,8); local btnStr=applyBorder(Btn)
    Btn.MouseEnter:Connect(function() TweenService:Create(Btn,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(28,28,38)}):Play(); TweenService:Create(btnStr,TweenInfo.new(0.15),{Color=Theme.Accent}):Play() end)
    Btn.MouseLeave:Connect(function() TweenService:Create(Btn,TweenInfo.new(0.15),{BackgroundColor3=Theme.Component}):Play(); TweenService:Create(btnStr,TweenInfo.new(0.15),{Color=Theme.Border}):Play() end)
    Btn.Activated:Connect(function()
        TweenService:Create(Btn, TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = Theme.Accent}):Play()
        task.delay(0.1, function()
            if Btn and Btn.Parent then
                TweenService:Create(Btn, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = Theme.Component}):Play()
            end
        end)
        callback()
    end)
end

local function updateHotkeyButton(action)
    local button = HotkeyButtons[action]
    if not button then return end
    local binding = HotkeyBindings[action]
    local keyName = binding and binding.Name or "Unassigned"
    button.Text = "  " .. action .. "  [" .. keyName .. "]"
    button.TextColor3 = HotkeyCaptureAction == action and Theme.AccentGlow or Theme.TextMain
end

local function CreateHotkeyButton(tab, action)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, -16, 0, 46)
    button.BackgroundColor3 = Theme.Component
    button.TextXAlignment = Enum.TextXAlignment.Left
    button.Font = Enum.Font.GothamMedium
    button.TextSize = 12
    button.AutoButtonColor = false
    button.Parent = tab
    applyCorner(button, 8)
    local stroke = applyBorder(button)
    HotkeyButtons[action] = button
    updateHotkeyButton(action)

    button.MouseEnter:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(28, 28, 38)}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.15), {Color = Theme.Accent}):Play()
    end)
    button.MouseLeave:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.15), {BackgroundColor3 = Theme.Component}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.15), {Color = HotkeyCaptureAction == action and Theme.AccentGlow or Theme.Border}):Play()
    end)
    button.Activated:Connect(function()
        HotkeyCaptureAction = HotkeyCaptureAction == action and nil or action
        for configuredAction, configuredButton in pairs(HotkeyButtons) do
            local configuredStroke = configuredButton:FindFirstChildOfClass("UIStroke")
            updateHotkeyButton(configuredAction)
            if configuredStroke then
                configuredStroke.Color = HotkeyCaptureAction == configuredAction and Theme.AccentGlow or Theme.Border
            end
        end
    end)
    return button
end

local function CreateStatCard(parent, labelText, valueText, accentColor)
    accentColor = accentColor or Theme.Accent
    local card=Instance.new("Frame"); card.Size=UDim2.new(0.48,0,0,56); card.BackgroundColor3=Theme.Card; card.Parent=parent
    applyCorner(card,8); applyBorder(card,accentColor,1)

    local valLbl=Instance.new("TextLabel"); valLbl.Size=UDim2.new(1,-8,0,28); valLbl.Position=UDim2.new(0,8,0,4)
    valLbl.Text=valueText; valLbl.Font=Enum.Font.GothamBold; valLbl.TextSize=18
    valLbl.TextColor3=accentColor; valLbl.TextXAlignment=Enum.TextXAlignment.Left
    valLbl.BackgroundTransparency=1; valLbl.Name="Value"; valLbl.Parent=card

    local labLbl=Instance.new("TextLabel"); labLbl.Size=UDim2.new(1,-8,0,18); labLbl.Position=UDim2.new(0,8,0,32)
    labLbl.Text=labelText; labLbl.Font=Enum.Font.Gotham; labLbl.TextSize=10
    labLbl.TextColor3=Theme.TextMuted; labLbl.TextXAlignment=Enum.TextXAlignment.Left
    labLbl.BackgroundTransparency=1; labLbl.Parent=card
    return card
end

-- =============================================================================
--                         NUEVA INTERFAZ DE LOGIN (REDISEÑO)
-- =============================================================================
local LoginFrame=Instance.new("Frame"); LoginFrame.Size=UDim2.new(0,390,0,390)
LoginFrame.Position=UDim2.new(0.5,-195,0.4,-180); LoginFrame.BackgroundColor3=Theme.Background
LoginFrame.BorderSizePixel=0; LoginFrame.Parent=ScreenGui
applyCorner(LoginFrame,16); applyBorder(LoginFrame,Theme.Border,1.5)
local LoginFrameScale = Instance.new("UIScale")
LoginFrameScale.Scale = 0.97
LoginFrameScale.Parent = LoginFrame
TweenService:Create(LoginFrameScale, TweenInfo.new(0.24, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Scale = 1}):Play()

local LoginGlow=Instance.new("ImageLabel"); LoginGlow.Size=UDim2.new(1,40,1,40); LoginGlow.Position=UDim2.new(0,-20,0,-20)
LoginGlow.BackgroundTransparency=1; LoginGlow.Image="rbxassetid://6015897843"; LoginGlow.ImageColor3=Theme.Accent
LoginGlow.ImageTransparency=0.82; LoginGlow.Parent=LoginFrame

local LoginHeader=Instance.new("Frame"); LoginHeader.Size=UDim2.new(1,0,0,55); LoginHeader.BackgroundColor3=Theme.Header
LoginHeader.Parent=LoginFrame; applyCorner(LoginHeader,16); applyBorder(LoginHeader); makeDraggable(LoginFrame,LoginHeader)

local HeaderFix=Instance.new("Frame"); HeaderFix.Size=UDim2.new(1,0,0,10); HeaderFix.Position=UDim2.new(0,0,1,-10)
HeaderFix.BackgroundColor3=Theme.Header; HeaderFix.BorderSizePixel=0; HeaderFix.Parent=LoginHeader

local LoginTitle=Instance.new("TextLabel"); LoginTitle.Size=UDim2.new(1,-140,1,0); LoginTitle.Position=UDim2.new(0,20,0,0)
LoginTitle.Text="ASCYNT ACCESS <font color='#00ffc8'><b>v2.6</b></font>"; LoginTitle.TextColor3=Theme.TextMain
LoginTitle.Font=Enum.Font.GothamBold; LoginTitle.TextSize=14; LoginTitle.TextXAlignment=Enum.TextXAlignment.Left
LoginTitle.BackgroundTransparency=1; LoginTitle.RichText=true; LoginTitle.Parent=LoginHeader

local ApiStatusTag=Instance.new("TextLabel"); ApiStatusTag.Size=UDim2.new(0,100,0,20); ApiStatusTag.Position=UDim2.new(1,-120,0.5,-10)
ApiStatusTag.BackgroundColor3=Color3.fromRGB(10,32,25); ApiStatusTag.Text="● GATEWAY READY"; ApiStatusTag.TextColor3=Theme.AccentGlow
ApiStatusTag.Font=Enum.Font.GothamBold; ApiStatusTag.TextSize=10; ApiStatusTag.Parent=LoginHeader
applyCorner(ApiStatusTag,6); applyBorder(ApiStatusTag,Color3.fromRGB(0,120,90))

local KeyHint=Instance.new("TextLabel"); KeyHint.Size=UDim2.new(1,-40,0,22); KeyHint.Position=UDim2.new(0,20,0,66)
KeyHint.BackgroundTransparency=1; KeyHint.Text="Enter your key to unlock the panel, or use Get Key to open the access portal."
KeyHint.TextColor3=Theme.TextMuted; KeyHint.Font=Enum.Font.Gotham; KeyHint.TextSize=11; KeyHint.TextWrapped=true; KeyHint.TextXAlignment=Enum.TextXAlignment.Left; KeyHint.Parent=LoginFrame

local KeyInput=Instance.new("TextBox"); KeyInput.Size=UDim2.new(1,-40,0,46); KeyInput.Position=UDim2.new(0,20,0,100)
KeyInput.BackgroundColor3=Theme.Component; KeyInput.Text="FREE-020926"; KeyInput.PlaceholderText="Paste your access key here..."
KeyInput.TextColor3=Theme.TextMain; KeyInput.PlaceholderColor3=Theme.TextHolder; KeyInput.Font=Enum.Font.GothamMedium
KeyInput.TextSize=13; KeyInput.ClearTextOnFocus=false; KeyInput.TextXAlignment=Enum.TextXAlignment.Left; KeyInput.Parent=LoginFrame; applyCorner(KeyInput,10); local kStroke=applyBorder(KeyInput)
KeyInput.Focused:Connect(function() TweenService:Create(kStroke,TweenInfo.new(0.2),{Color=Theme.Accent,Thickness=1.5}):Play() end)
KeyInput.FocusLost:Connect(function() TweenService:Create(kStroke,TweenInfo.new(0.2),{Color=Theme.Border,Thickness=1}):Play() end)

local SubmitBtn=Instance.new("TextButton"); SubmitBtn.Size=UDim2.new(1,-40,0,46); SubmitBtn.Position=UDim2.new(0,20,0,160)
SubmitBtn.BackgroundColor3=Theme.Accent; SubmitBtn.Text="Login with Access Key"; SubmitBtn.TextColor3=Color3.fromRGB(10,10,15)
SubmitBtn.Font=Enum.Font.GothamBold; SubmitBtn.TextSize=13; SubmitBtn.AutoButtonColor=false; SubmitBtn.Parent=LoginFrame
applyCorner(SubmitBtn,10); applyBorder(SubmitBtn,Theme.AccentGlow)
SubmitBtn.MouseEnter:Connect(function()
    TweenService:Create(SubmitBtn, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = Theme.AccentGlow}):Play()
end)
SubmitBtn.MouseLeave:Connect(function()
    TweenService:Create(SubmitBtn, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = Theme.Accent}):Play()
end)

local Separador=Instance.new("Frame"); Separador.Size=UDim2.new(1,-40,0,1); Separador.Position=UDim2.new(0,20,0,225)
Separador.BackgroundColor3=Theme.Border; Separador.BorderSizePixel=0; Separador.Parent=LoginFrame

local BuyBtn=Instance.new("TextButton"); BuyBtn.Size=UDim2.new(1,-40,0,44); BuyBtn.Position=UDim2.new(0,20,0,245)
BuyBtn.BackgroundColor3=Theme.Header; BuyBtn.Text="Get Key — Copy Access Portal Link"; BuyBtn.TextColor3=Theme.TextMuted
BuyBtn.Font=Enum.Font.GothamMedium; BuyBtn.TextSize=12; BuyBtn.AutoButtonColor=false; BuyBtn.Parent=LoginFrame
applyCorner(BuyBtn,10); applyBorder(BuyBtn)
BuyBtn.MouseEnter:Connect(function()
    TweenService:Create(BuyBtn, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = Theme.Component, TextColor3 = Theme.TextMain}):Play()
end)
BuyBtn.MouseLeave:Connect(function()
    TweenService:Create(BuyBtn, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = Theme.Header, TextColor3 = Theme.TextMuted}):Play()
end)

local FooterNotice=Instance.new("TextLabel"); FooterNotice.Size=UDim2.new(1,-40,0,30); FooterNotice.Position=UDim2.new(0,20,1,-40)
FooterNotice.Text="Your key is validated against this device. Do not share it with other users."; FooterNotice.TextColor3=Theme.TextHolder; FooterNotice.Font=Enum.Font.Gotham
FooterNotice.TextSize=10; FooterNotice.BackgroundTransparency=1; FooterNotice.Parent=LoginFrame

local BuyFrame=Instance.new("Frame"); BuyFrame.Size=UDim2.new(1,0,1,0); BuyFrame.BackgroundColor3=Theme.Background
BuyFrame.BackgroundTransparency=0.05; BuyFrame.Visible=false; BuyFrame.ZIndex=5; BuyFrame.Parent=LoginFrame; applyCorner(BuyFrame,16)
BuyFrame.Position = UDim2.new(0, 0, 0, 12)

local BuyText=Instance.new("TextLabel"); BuyText.Size=UDim2.new(1,-40,1,-100); BuyText.Position=UDim2.new(0,20,0,30)
BuyText.Text="<font color='#ffffff'><b>GET KEY — QUICK ACCESS:</b></font>\n\n<font color='#FFD54A'>1. The access portal link has been copied to your clipboard.\n2. Open the link in your browser and complete the key process.\n3. Copy the generated key, return here, and use <b>Login with Access Key</b>.\n\n<b>Tip:</b> Keep the key private and use it only on your own device.</font>"
BuyText.TextColor3=Color3.fromRGB(255,213,74); BuyText.Font=Enum.Font.Gotham; BuyText.TextSize=12; BuyText.TextWrapped=true
BuyText.RichText=true; BuyText.ZIndex=6; BuyText.TextYAlignment=Enum.TextYAlignment.Top; BuyText.Parent=BuyFrame

local CloseBuyBtn=Instance.new("TextButton"); CloseBuyBtn.Size=UDim2.new(1,-40,0,44); CloseBuyBtn.Position=UDim2.new(0,20,1,-60)
CloseBuyBtn.BackgroundColor3=Color3.fromRGB(150,40,40); CloseBuyBtn.Text="Back to Login"; CloseBuyBtn.TextColor3=Color3.fromRGB(255,255,255)
CloseBuyBtn.Font=Enum.Font.GothamBold; CloseBuyBtn.TextSize=12; CloseBuyBtn.ZIndex=6; CloseBuyBtn.Parent=BuyFrame; applyCorner(CloseBuyBtn,10)

BuyBtn.Activated:Connect(function()
    BuyFrame.Visible=true
    BuyFrame.Position = UDim2.new(0, 0, 0, 12)
    TweenService:Create(BuyFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Position = UDim2.new(0, 0, 0, 0)}):Play()
    local url=_decodeStatic("68747470733a2f2f617363796e742d6875622d696e6b792e76657263656c2e617070")
    if setclipboard then setclipboard(url) elseif toclipboard then toclipboard(url) end
    local ot=BuyBtn.Text; BuyBtn.Text="✓ Portal Link Copied — Follow the 3 Steps"; BuyBtn.TextColor3=Theme.Success
    task.spawn(function() task.wait(2.5); BuyBtn.Text=ot; BuyBtn.TextColor3=Theme.TextMuted end)
end)
CloseBuyBtn.Activated:Connect(function()
    TweenService:Create(BuyFrame, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Position = UDim2.new(0, 0, 0, 12)}):Play()
    task.delay(0.16, function()
        if BuyFrame and BuyFrame.Parent then
            BuyFrame.Visible = false
        end
    end)
end)

-- =============================================================================
--                        REDISEÑO DEL MENÚ PRINCIPAL
-- =============================================================================
local MainFrame=Instance.new("Frame"); MainFrame.Size=UDim2.new(0,600,0,420); MainFrame.Position=UDim2.new(0.5,-300,0.4,-210)
MainFrame.BackgroundColor3=Theme.Background; MainFrame.Visible=false; MainFrame.ClipsDescendants=true; MainFrame.Parent=ScreenGui
applyCorner(MainFrame,12); applyBorder(MainFrame,Theme.Border,1.5)
local MainFrameScale = Instance.new("UIScale")
MainFrameScale.Scale = 0.97
MainFrameScale.Parent = MainFrame

local MainHeader=Instance.new("Frame"); MainHeader.Size=UDim2.new(1,0,0,52); MainHeader.BackgroundColor3=Theme.Header
MainHeader.Parent=MainFrame; applyCorner(MainHeader,12); applyBorder(MainHeader); makeDraggable(MainFrame,MainHeader)

local HeaderAccent=Instance.new("Frame"); HeaderAccent.Size=UDim2.new(1,0,0,3); HeaderAccent.Position=UDim2.new(0,0,1,-3); HeaderAccent.BackgroundColor3=Theme.AccentGlow; HeaderAccent.BorderSizePixel=0; HeaderAccent.Parent=MainHeader
local MainSubtitle=Instance.new("TextLabel"); MainSubtitle.Size=UDim2.new(0.34,0,1,0); MainSubtitle.Position=UDim2.new(0.58,0,0,0); MainSubtitle.Text="VIP"; MainSubtitle.TextColor3=Theme.TextMuted; MainSubtitle.Font=Enum.Font.GothamBold; MainSubtitle.TextSize=9; MainSubtitle.TextXAlignment=Enum.TextXAlignment.Right; MainSubtitle.BackgroundTransparency=1; MainSubtitle.Parent=MainHeader

local MainTitle=Instance.new("TextLabel"); MainTitle.Size=UDim2.new(0.6,0,1,0); MainTitle.Position=UDim2.new(0,18,0,0)
MainTitle.Text="AscyntHub <font color='#00ffc8'><b>NDS VIP</b></font> <font color='#555566' size='10'>v2.6</font>"
MainTitle.TextColor3=Theme.TextMain; MainTitle.Font=Enum.Font.GothamBold; MainTitle.TextSize=14
MainTitle.TextXAlignment=Enum.TextXAlignment.Left; MainTitle.BackgroundTransparency=1; MainTitle.RichText=true; MainTitle.Parent=MainHeader

local ToggleBtn=Instance.new("TextButton"); ToggleBtn.Size=UDim2.new(0,28,0,28); ToggleBtn.Position=UDim2.new(1,-40,0,12)
ToggleBtn.BackgroundColor3=Color3.fromRGB(26,26,36); ToggleBtn.Text="—"; ToggleBtn.TextColor3=Theme.TextMuted
ToggleBtn.Font=Enum.Font.GothamBold; ToggleBtn.TextSize=12; ToggleBtn.Parent=MainHeader; applyCorner(ToggleBtn,6); applyBorder(ToggleBtn)

local OpenFloatingBtn=Instance.new("TextButton"); OpenFloatingBtn.Size=UDim2.new(0,65,0,65)
OpenFloatingBtn.Position=UDim2.new(0.05,0,0.5,-32); OpenFloatingBtn.BackgroundColor3=Theme.Header; OpenFloatingBtn.Text="ASCYNT"
OpenFloatingBtn.TextColor3=Theme.Accent; OpenFloatingBtn.Font=Enum.Font.GothamBold; OpenFloatingBtn.TextSize=10
OpenFloatingBtn.Visible=false; OpenFloatingBtn.Parent=ScreenGui; applyCorner(OpenFloatingBtn,32)
OpenFloatingBtn.ZIndex=10; applyBorder(OpenFloatingBtn,Theme.Accent,1.5); makeDraggable(OpenFloatingBtn,OpenFloatingBtn)
local OpenFloatingScale = Instance.new("UIScale")
OpenFloatingScale.Scale = 0.82
OpenFloatingScale.Parent = OpenFloatingBtn

local MainFrameTransition = 0
setMainFrameVisible = function(visible)
    MainFrameTransition = MainFrameTransition + 1
    local transition = MainFrameTransition
    if visible then
        MainFrame.Visible = true
        OpenFloatingBtn.Visible = false
        MainFrameScale.Scale = 0.97
        TweenService:Create(MainFrameScale, TweenInfo.new(0.22, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Scale = 1}):Play()
    else
        OpenFloatingBtn.Visible = false
        TweenService:Create(MainFrameScale, TweenInfo.new(0.18, Enum.EasingStyle.Quart, Enum.EasingDirection.In), {Scale = 0.97}):Play()
        task.delay(0.18, function()
            if transition == MainFrameTransition then
                MainFrame.Visible = false
                OpenFloatingBtn.Visible = true
                OpenFloatingScale.Scale = 0.82
                TweenService:Create(OpenFloatingScale, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()
            end
        end)
    end
end

ToggleBtn.Activated:Connect(function() setMainFrameVisible(false) end)
OpenFloatingBtn.Activated:Connect(function() setMainFrameVisible(true) end)

local Sidebar=Instance.new("ScrollingFrame"); Sidebar.Size=UDim2.new(0,154,1,-84); Sidebar.Position=UDim2.new(0,0,0,52)
Sidebar.BackgroundColor3=Theme.Sidebar; Sidebar.ScrollBarThickness=2; Sidebar.ScrollBarImageColor3=Theme.Border
Sidebar.CanvasSize=UDim2.new(0,0,0,0); Sidebar.BorderSizePixel=0; Sidebar.Parent=MainFrame

local SidebarList=Instance.new("UIListLayout"); SidebarList.Padding=UDim.new(0,4); SidebarList.HorizontalAlignment=Enum.HorizontalAlignment.Center; SidebarList.Parent=Sidebar
SidebarList:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() Sidebar.CanvasSize=UDim2.new(0,0,0,SidebarList.AbsoluteContentSize.Y+12) end)

local sLine=Instance.new("Frame"); sLine.Size=UDim2.new(0,1,1,-84); sLine.Position=UDim2.new(0,153,0,52)
sLine.BackgroundColor3=Theme.Border; sLine.BorderSizePixel=0; sLine.Parent=MainFrame

local ContentContainer=Instance.new("Frame"); ContentContainer.Size=UDim2.new(1,-174,1,-98); ContentContainer.Position=UDim2.new(0,166,0,60)
ContentContainer.BackgroundTransparency=1; ContentContainer.ClipsDescendants=true; ContentContainer.Parent=MainFrame

local StatusBar=Instance.new("Frame"); StatusBar.Size=UDim2.new(1,0,0,32); StatusBar.Position=UDim2.new(0,0,1,-32)
StatusBar.BackgroundColor3=Theme.Sidebar; StatusBar.Parent=MainFrame; applyBorder(StatusBar)

StatusText=Instance.new("TextLabel"); StatusText.Size=UDim2.new(1,-20,1,0); StatusText.Position=UDim2.new(0,18,0,0)
StatusText.Text="Initial sync request established with internal cluster matrix..."; StatusText.TextColor3=Theme.TextMuted
StatusText.Font=Enum.Font.Gotham; StatusText.TextSize=11; StatusText.TextXAlignment=Enum.TextXAlignment.Left
StatusText.BackgroundTransparency=1; StatusText.Parent=StatusBar

local ResponsiveTabButtons = {}
local TabShortLabels = {
    ["Player Controls"] = "PLAY",
    ["Rapid Keys"] = "KEYS",
    ["Fun & Trolls"] = "FUN",
    ["Teleports"] = "TP",
    ["Free Scripts"] = "FREE",
    ["Private Scripts"] = "PRIV",
    ["API Status"] = "API",
    ["Performance"] = "FPS",
    ["Configurations"] = "CFG",
    ["Owner Servers"] = "SVS",
    ["Information"] = "INFO",
    ["Chat"] = "CHAT",
    ["PC"] = "PC",
}

local function getViewportSize()
    local camera = Workspace.CurrentCamera
    return camera and camera.ViewportSize or Vector2.new(800, 600)
end

local function applyResponsiveLayout()
    local viewport = getViewportSize()
    local compact = UserInputService.TouchEnabled or viewport.X < 620
    local panelWidth = math.max(compact and 280 or 520, math.min(600, viewport.X - (compact and 16 or 48)))
    local panelHeight = math.max(compact and 360 or 420, math.min(compact and 560 or 520, viewport.Y - (compact and 24 or 120)))
    local sideWidth = compact and math.clamp(math.floor(panelWidth * 0.29), 88, 124) or 154
    local contentLeft = sideWidth + (compact and 10 or 12)

    for _, tabButton in ipairs(ResponsiveTabButtons) do
        tabButton.Button.Text = compact and tabButton.CompactText or tabButton.FullText
        tabButton.Button.TextSize = compact and 10 or 11
    end

    MainFrame.Size = UDim2.fromOffset(panelWidth, panelHeight)
    MainFrame.Position = UDim2.new(0.5, -panelWidth / 2, 0.5, -panelHeight / 2)
    Sidebar.Size = UDim2.new(0, sideWidth, 1, -84)
    sLine.Position = UDim2.new(0, sideWidth - 1, 0, 52)
    ContentContainer.Position = UDim2.new(0, contentLeft, 0, 60)
    ContentContainer.Size = UDim2.new(1, -contentLeft - 8, 1, -98)

    local loginWidth = math.max(280, math.min(390, viewport.X - 24))
    local loginHeight = math.max(340, math.min(390, viewport.Y - 24))
    LoginFrame.Size = UDim2.fromOffset(loginWidth, loginHeight)
    LoginFrame.Position = UDim2.new(0.5, -loginWidth / 2, 0.5, -loginHeight / 2)
end

local viewportConnection = nil
local function bindViewportConnection()
    if viewportConnection then viewportConnection:Disconnect() end
    local camera = Workspace.CurrentCamera
    if camera then
        viewportConnection = camera:GetPropertyChangedSignal("ViewportSize"):Connect(applyResponsiveLayout)
    end
end
Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(function()
    bindViewportConnection()
    applyResponsiveLayout()
end)
bindViewportConnection()

-- Sistema de Pestanas
local Tabs={}
local function CreateTab(name, icon)
    local TabPage=Instance.new("ScrollingFrame"); TabPage.Size=UDim2.new(1,0,1,0); TabPage.BackgroundTransparency=1
    TabPage.Visible=false; TabPage.CanvasSize=UDim2.new(0,0,0,0); TabPage.ScrollBarThickness=4
    TabPage.ScrollBarImageColor3=Theme.Border; TabPage.Parent=ContentContainer

    local Layout=Instance.new("UIListLayout"); Layout.Padding=UDim.new(0,8); Layout.HorizontalAlignment=Enum.HorizontalAlignment.Center; Layout.Parent=TabPage
    Layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() TabPage.CanvasSize=UDim2.new(0,0,0,Layout.AbsoluteContentSize.Y+15) end)

    local fullText=(icon and (icon.." ") or "")..name
    local compactText=TabShortLabels[name] or string.sub(name,1,4)
    local TabBtn=Instance.new("TextButton"); TabBtn.Size=UDim2.new(0.92,0,0,42)
    TabBtn.BackgroundColor3=Color3.fromRGB(0,0,0); TabBtn.BackgroundTransparency=1
    TabBtn.Text=fullText; TabBtn.TextColor3=Theme.TextMuted
    TabBtn.Font=Enum.Font.GothamMedium; TabBtn.TextSize=11; TabBtn.AutoButtonColor=false; TabBtn.Parent=Sidebar
    applyCorner(TabBtn,6); local bStroke=applyBorder(TabBtn,Theme.Border,1); bStroke.Transparency=1
    ResponsiveTabButtons[#ResponsiveTabButtons + 1] = {Button=TabBtn, FullText=fullText, CompactText=compactText}

    TabBtn.MouseEnter:Connect(function() if not TabPage.Visible then TweenService:Create(TabBtn,TweenInfo.new(0.15),{TextColor3=Theme.TextMain}):Play(); TweenService:Create(bStroke,TweenInfo.new(0.15),{Transparency=0.5}):Play() end end)
    TabBtn.MouseLeave:Connect(function() if not TabPage.Visible then TweenService:Create(TabBtn,TweenInfo.new(0.15),{TextColor3=Theme.TextMuted}):Play(); TweenService:Create(bStroke,TweenInfo.new(0.15),{Transparency=1}):Play() end end)
    TabBtn.Activated:Connect(function()
        for _, v in pairs(Tabs) do
            v.Page.Visible=false; v.Stroke.Transparency=1
            TweenService:Create(v.Btn,TweenInfo.new(0.18),{BackgroundTransparency=1,TextColor3=Theme.TextMuted}):Play()
        end
                    TabPage.Position = UDim2.new(0, 10, 0, 0)
            TabPage.Visible=true; bStroke.Color=Theme.Accent; bStroke.Transparency=0.2
            TweenService:Create(TabPage,TweenInfo.new(0.2,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),{Position=UDim2.new(0,0,0,0)}):Play()
            TweenService:Create(TabBtn,TweenInfo.new(0.18),{BackgroundTransparency=0,BackgroundColor3=Theme.Component,TextColor3=Theme.Accent}):Play()

    end)
    Tabs[name]={Page=TabPage, Btn=TabBtn, Stroke=bStroke}
    return TabPage
end

-- =============================================================================
--                    CONSTRUCCIÓN DE CONTENIDO (CON NUEVA SECCIÓN RAPID KEYS)
-- =============================================================================
local LocalTab        = CreateTab("Player Controls","⚡")
local RapidKeysTab    = CreateTab("Rapid Keys", "🎹") -- Nueva Sección Solicitada
local TrollsTab       = CreateTab("Fun & Trolls","🎭")
local TeleportTab     = CreateTab("Teleports","🗺")
local FreeScriptsTab  = CreateTab("Free Scripts","📦")
local PrivateTab      = CreateTab("Private Scripts","🔒")
local ApiTab          = CreateTab("API Status","🌐")
local PerforTab       = CreateTab("Performance","🎮")
local ChatTab         = CreateTab("Chat","💬")
local InfoTab         = CreateTab("Information","ℹ")
local ConfigTab       = CreateTab("Configurations","⚙")
local OwnerServersTab = nil
if LocalPlayer.Name == "SkylerModz_67" then
    OwnerServersTab = CreateTab("Owner Servers", "🛡")
end
local PCTab          = CreateTab("PC", "⌨")
applyResponsiveLayout()

CreateSection(PCTab, "Hotkey Configuration")
CreateAnnouncement(PCTab, "⌨", "Keyboard Shortcuts", "Select one button, then press a keyboard key to assign it.", Theme.AccentGlow)
CreateHotkeyButton(PCTab, "Open/Close Menu")
CreateHotkeyButton(PCTab, "Telekinesis")
CreateHotkeyButton(PCTab, "Teleport to Island")
CreateHotkeyButton(PCTab, "AntiFling")
CreateHotkeyButton(PCTab, "Teleport to Safe Spawn")

local function executeHotkeyAction(action)
    if LoginFrame and LoginFrame.Visible then return end
    if action == "Open/Close Menu" then
        setMainFrameVisible(not MainFrame.Visible)
    elseif action == "Telekinesis" then
        SetTelekinesis(not TelekinesisActive)
    elseif action == "Teleport to Island" then
        teleportToIsland()
    elseif action == "AntiFling" then
        SetAntiFling(not AntiFlingActive)
    elseif action == "Teleport to Safe Spawn" then
        teleportToSafeSpawn()
    end
end

HotkeyInputConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    if HotkeyCaptureAction then
        if input.KeyCode == Enum.KeyCode.Unknown then return end
        local action = HotkeyCaptureAction
        HotkeyBindings[action] = input.KeyCode
        HotkeyCaptureAction = nil
        for configuredAction, configuredButton in pairs(HotkeyButtons) do
            updateHotkeyButton(configuredAction)
            local configuredStroke = configuredButton:FindFirstChildOfClass("UIStroke")
            if configuredStroke then
                configuredStroke.Color = HotkeyCaptureAction == configuredAction and Theme.AccentGlow or Theme.Border
            end
        end
        return
    end
    if gameProcessed then return end
    for action, keyCode in pairs(HotkeyBindings) do
        if input.KeyCode == keyCode then
            executeHotkeyAction(action)
            break
        end
    end
end)

-- ── Chat interno moderado ─────────────────────────────────────────────────────
local ChatPanel=Instance.new('Frame'); ChatPanel.Size=UDim2.new(1,-16,0,404); ChatPanel.BackgroundColor3=Theme.Card; ChatPanel.Parent=ChatTab; ChatPanel.LayoutOrder=1
applyCorner(ChatPanel,10); applyBorder(ChatPanel,Theme.Border,1)
local ChatTitle=Instance.new('TextLabel'); ChatTitle.Size=UDim2.new(1,-24,0,26); ChatTitle.Position=UDim2.new(0,12,0,8); ChatTitle.BackgroundTransparency=1; ChatTitle.Text='GLOBAL CHAT  // BETA'; ChatTitle.TextColor3=Theme.AccentGlow; ChatTitle.Font=Enum.Font.GothamBold; ChatTitle.TextSize=12; ChatTitle.TextXAlignment=Enum.TextXAlignment.Left; ChatTitle.Parent=ChatPanel
local ChatRule=Instance.new('TextLabel'); ChatRule.Size=UDim2.new(1,-24,0,28); ChatRule.Position=UDim2.new(0,12,0,34); ChatRule.BackgroundTransparency=1; ChatRule.Text='5s cooldown • links, IPs, numbers and unsafe promotion blocked'; ChatRule.TextColor3=Theme.TextMuted; ChatRule.Font=Enum.Font.Gotham; ChatRule.TextSize=10; ChatRule.TextWrapped=true; ChatRule.TextXAlignment=Enum.TextXAlignment.Left; ChatRule.Parent=ChatPanel
ChatMessagesFrame=Instance.new('ScrollingFrame'); ChatMessagesFrame.Size=UDim2.new(1,-24,0,246); ChatMessagesFrame.Position=UDim2.new(0,12,0,68); ChatMessagesFrame.BackgroundColor3=Theme.Background; ChatMessagesFrame.BorderSizePixel=0; ChatMessagesFrame.ScrollBarThickness=4; ChatMessagesFrame.ScrollBarImageColor3=Theme.Border; ChatMessagesFrame.CanvasSize=UDim2.new(0,0,0,0); ChatMessagesFrame.Parent=ChatPanel; applyCorner(ChatMessagesFrame,8)
ChatListLayout=Instance.new('UIListLayout'); ChatListLayout.Padding=UDim.new(0,6); ChatListLayout.HorizontalAlignment=Enum.HorizontalAlignment.Center; ChatListLayout.Parent=ChatMessagesFrame
ChatListLayout:GetPropertyChangedSignal('AbsoluteContentSize'):Connect(function() ChatMessagesFrame.CanvasSize=UDim2.new(0,0,0,ChatListLayout.AbsoluteContentSize.Y+12) end)
ChatInput=Instance.new('TextBox'); ChatInput.Size=UDim2.new(1,-112,0,38); ChatInput.Position=UDim2.new(0,12,0,324); ChatInput.BackgroundColor3=Theme.Component; ChatInput.BorderSizePixel=0; ChatInput.PlaceholderText='Write a safe message...'; ChatInput.Text=''; ChatInput.TextColor3=Theme.TextMain; ChatInput.PlaceholderColor3=Theme.TextHolder; ChatInput.Font=Enum.Font.Gotham; ChatInput.TextSize=11; ChatInput.ClearTextOnFocus=false; ChatInput.TextXAlignment=Enum.TextXAlignment.Left; ChatInput.Parent=ChatPanel; applyCorner(ChatInput,8); applyBorder(ChatInput,Theme.Border,1)
local ChatSend=Instance.new('TextButton'); ChatSend.Size=UDim2.new(0,88,0,38); ChatSend.Position=UDim2.new(1,-100,0,324); ChatSend.BackgroundColor3=Theme.Accent; ChatSend.BorderSizePixel=0; ChatSend.Text='SEND'; ChatSend.TextColor3=Color3.fromRGB(255,255,255); ChatSend.Font=Enum.Font.GothamBold; ChatSend.TextSize=11; ChatSend.AutoButtonColor=false; ChatSend.Parent=ChatPanel; applyCorner(ChatSend,8); applyBorder(ChatSend,Theme.AccentGlow,1)
ChatStatus=Instance.new('TextLabel'); ChatStatus.Size=UDim2.new(1,-24,0,22); ChatStatus.Position=UDim2.new(0,12,0,368); ChatStatus.BackgroundTransparency=1; ChatStatus.Text='Messages are stored temporarily on the server.'; ChatStatus.TextColor3=Theme.TextHolder; ChatStatus.Font=Enum.Font.Gotham; ChatStatus.TextSize=10; ChatStatus.TextXAlignment=Enum.TextXAlignment.Left; ChatStatus.Parent=ChatPanel

renderChatMessages=function(messages)
    if type(messages) ~= 'table' or not ChatMessagesFrame then return end
    local added = false
    for _, entry in ipairs(messages) do
        local id = tonumber(entry.id)
        if id and not V25State.chatSeen[id] then
            V25State.chatSeen[id] = true
            V25State.lastChatId = math.max(V25State.lastChatId, id)
            local row=Instance.new('TextLabel'); row.Size=UDim2.new(1,-12,0,0); row.AutomaticSize=Enum.AutomaticSize.Y; row.BackgroundColor3=Theme.Component; row.BorderSizePixel=0; row.TextWrapped=true; row.TextXAlignment=Enum.TextXAlignment.Left; row.TextYAlignment=Enum.TextYAlignment.Center; row.Font=Enum.Font.Gotham; row.TextSize=10; row.TextColor3=Theme.TextMain
            local stamp=tostring(entry.createdAt or entry.date or ''):sub(1,19):gsub('T',' ')
            local author=tostring(entry.displayName or entry.username or 'Unknown')
            if entry.verified or entry.username == 'SkylerModz_67' then author=author..' ✓' end
            row.Text=string.format('[%s] #%s  %s: %s', stamp, tostring(id), author, tostring(entry.message or ''))
            row.Parent=ChatMessagesFrame; applyCorner(row,7); applyBorder(row,Theme.Border,1)
            added = true
        end
    end
    if added then task.defer(function() ChatMessagesFrame.CanvasPosition=Vector2.new(0, math.max(0, ChatListLayout.AbsoluteContentSize.Y-ChatMessagesFrame.AbsoluteWindowSize.Y)) end) end
end

local function sendChatMessage()
    local text=ChatInput and ChatInput.Text or ''
    if text:gsub('%s+','') == '' then return end
    local remaining=V25State.chatCooldownUntil-os.clock()
    if remaining > 0 then
        if ChatStatus then ChatStatus.Text='Cooldown active: '..string.format('%.1f', remaining)..'s' end
        return
    end
    local data=requestJson('POST',SERVER_URL..'/api/chat/messages',{username=LocalPlayer.Name,displayName=LocalPlayer.DisplayName,message=text,jobId=game.JobId,placeId=game.PlaceId})
    if data and data.success then
        V25State.chatCooldownUntil=os.clock()+5
        ChatInput.Text=''
        if ChatStatus then ChatStatus.Text='Message sent • 5s cooldown' end
        if data.message then renderChatMessages({data.message}) end
    else
        if ChatStatus then ChatStatus.Text=(data and data.message) or 'Message rejected by moderation.' end
    end
end
ChatSend.Activated:Connect(sendChatMessage)
ChatInput.FocusLost:Connect(function(enterPressed) if enterPressed then sendChatMessage() end end)

-- ── Rapid Keys (Toggles para controlar visibilidad de los botones flotantes) ─
CreateSection(RapidKeysTab, "Float Bind HUD Switchers")
CreateAnnouncement(RapidKeysTab, "🎹", "HUD Toggles", "Activate these switches to display the movable hotkeys on your screen.", Theme.AccentGlow)

CreateToggle(RapidKeysTab, "Show Self-Fling v2 Quick Button", function(state)
    FloatSelfFling.Visible = state
end)

CreateToggle(RapidKeysTab, "Show Telekinesis Quick Button", function(state)
    FloatTelekinesis.Visible = state
end)

CreateToggle(RapidKeysTab, "Show Anti-Fling Quick Button", function(state)
    FloatAntiFling.Visible = state
end)

CreateToggle(RapidKeysTab, "Show Sleep Quick Button", function(state)
    FloatSleep.Visible = state
end)
-- ── Player Controls ──────────────────────────────────────────────────────────
CreateSection(LocalTab,"Main Abilities")
CreateToggle(LocalTab,"Telekinesis",function(state) SetTelekinesis(state) end)

CreateToggle(LocalTab,"Anti-Fling",function(state)
    SetAntiFling(state)
end)

CreateSection(LocalTab,"Safety & Disaster")
CreateToggle(LocalTab,"Auto Detect Disaster",function(state)
    ToggleAutoDetectDisaster(state)
end)
CreateToggle(LocalTab,"Auto Save Player",function(state)
    ToggleAutoSavePlayer(state)
end)

CreateDropdown(LocalTab,"Dynamic Profile",{
    "From View","Fling All","Move View","Kill","Dial Protect","Protect Player","Baseplate","Selected",
    "Puente","Orbit","Chaos Mode","Rain","Worm Mode","Crazy Mode","Roof","Meteored",
    "LaunchLobby V1","LaunchLobby V2","Espiral","Tornado Espiral","Fling Player","Tornado","House","Hyper-Tornado", "Randoms","Hug","Protection V1","Aligned", "Brick Launcher",
    "Nazi","Nazi v2","Giant Dome","Hyper Dome"
}, function(value) SelectedForm=value end)

CreateSection(LocalTab,"Configuration")
CreateSlider(LocalTab,"Radial Tracking Range",1,50,26,function(value) TelekinesisRange=value end)
CreateSlider(LocalTab,"Rain Height",1,70,36,function(value) RainHeight=value end)
CreateSlider(LocalTab,"Rotative Speed Parts",0,10,1,function(value) RotationSpeed=value end)

-- ── Fun & Trolls ─────────────────────────────────────────────────────────────
CreateSection(TrollsTab,"Server Trolling")
CreateAnnouncement(TrollsTab,"⚡","Fling All","Available in Dynamic Profile → select 'Fling All' to fly all parts'.",Theme.Accent)

local FlingV1Conn

CreateToggle(TrollsTab, "Self-Fling (v1)", function(state)
    if state then
        local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
        local hrp = char:WaitForChild("HumanoidRootPart")
        
        -- Conectamos el evento al activar
        FlingV1Conn = hrp.Touched:Connect(function(hit)
            local targetChar = hit.Parent
            local targetHum = targetChar:FindFirstChild("Humanoid")
            
            -- Solo afecta a otros jugadores
            if targetHum and targetChar ~= char then
                local targetHrp = targetChar:FindFirstChild("HumanoidRootPart")
                if targetHrp then
                    -- Aplicamos fuerza física al objetivo
                    targetHrp.AssemblyLinearVelocity = Vector3.new(0, 400, 0)
                    task.wait(0.1)
                    targetHrp.AssemblyLinearVelocity = Vector3.new(math.random(-400, 400), 400, math.random(-400, 400))
                end
            end
        end)
    else
        -- Desconectamos al desactivar
        if FlingV1Conn then 
            FlingV1Conn:Disconnect() 
            FlingV1Conn = nil 
        end
    end
end)

local FlingV2UiConn

CreateToggle(TrollsTab, "Self-Fling (v2)", function(state)
    if state then
        local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
        local hrp = char:WaitForChild("HumanoidRootPart")
        local hum = char:WaitForChild("Humanoid")
        
        -- Conectamos el evento al activar
        FlingV2UiConn = hrp.Touched:Connect(function(hit)
            local targetChar = hit.Parent
            local targetHum = targetChar:FindFirstChild("Humanoid")
            
            -- Si tocamos a OTRO jugador
            if targetHum and targetChar ~= char then
                -- 1. Activamos el estado de "caída" para perder el control (Ragdoll natural)
                hum:ChangeState(Enum.HumanoidStateType.FallingDown)
                hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, false) -- Evita que se levante de inmediato
                
                -- 2. Aplicamos una fuerza leve a NUESTRO propio personaje
                -- Una fuerza de entre 60 y 80 es perfecta para tropezar o salir un poco al aire sin morir
                hrp.AssemblyLinearVelocity = Vector3.new(
                    math.random(-50, 50), 
                    60, 
                    math.random(-50, 50)
                )
                
                -- 3. Esperamos unos segundos en el suelo antes de recuperar el control
                task.wait(2.5) -- Tiempo que te quedas "acostado"
                
                -- 4. Permitimos que el personaje se levante de nuevo
                hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, true)
                hum:ChangeState(Enum.HumanoidStateType.GettingUp)
            end
        end)
    else
        -- Desconectamos al desactivar
        if FlingV2UiConn then 
            FlingV2UiConn:Disconnect() 
            FlingV2UiConn = nil 
        end
    end
end)


CreateToggle(TrollsTab,"Sleep",function(state) ToggleSleep(state) end)

CreateSection(TrollsTab,"Client Exploits")
CreateAnnouncement(TrollsTab,"⚠","Fake Lag","Pin and release your rootpart to simulate visual lag to the customer.",Theme.Warning)
CreateToggle(TrollsTab,"FE Fake Lag (Client)",function(state)
    FakeLagEnabled=state
    task.spawn(function()
        while FakeLagEnabled do
            local char=LocalPlayer.Character; local root=char and char:FindFirstChild("HumanoidRootPart")
            if root then root.Anchored=true; task.wait(0.1); root.Anchored=false end
            task.wait(0.15)
        end
        local char=LocalPlayer.Character; local root=char and char:FindFirstChild("HumanoidRootPart")
        if root then root.Anchored=false end
    end)
end)

-- ── Teleports ─────────────────────────────────────────────────────────────────
CreateSection(TeleportTab,"Map Locations")
CreateButton(TeleportTab,"Island Map",teleportToIsland)
CreateButton(TeleportTab,"Safe Lobby",function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        LocalPlayer.Character.HumanoidRootPart.CFrame=CFrame.new(Vector3.new(-286.21,194.58,286.25))
    end
end)
CreateButton(TeleportTab,"Rocket Point",function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        LocalPlayer.Character.HumanoidRootPart.CFrame=CFrame.new(Vector3.new(45.46,47.58,38.61))
    end
end)
CreateButton(TeleportTab,"Rocket Point 2",function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        LocalPlayer.Character.HumanoidRootPart.CFrame=CFrame.new(Vector3.new(67.45,87.48,33.21))
    end
end)
CreateButton(TeleportTab,"Rocket Thruster",function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        LocalPlayer.Character.HumanoidRootPart.CFrame=CFrame.new(Vector3.new(2.46,127.58,-27.98))
    end
end)
CreateSection(TeleportTab,"Server Tools")
CreateButton(TeleportTab,"Vehicle Helper",function()
    local character = LocalPlayer.Character
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if not rootPart or not humanoid then return end

    local nearestSeat = nil
    local nearestDistance = math.huge
    for _, instance in ipairs(Workspace:GetDescendants()) do
        if instance:IsA("VehicleSeat") and not instance.Occupant and instance.Parent then
            local distance = (instance.Position - rootPart.Position).Magnitude
            if distance < nearestDistance then
                nearestDistance = distance
                nearestSeat = instance
            end
        end
    end

    if nearestSeat then
        rootPart.CFrame = nearestSeat.CFrame + Vector3.new(0, 2, 0)
        task.wait(0.15)
        pcall(function() nearestSeat:Sit(humanoid) end)
    end
end)

-- 
CreateButton(TeleportTab,"TP to Owner Server",function() teleportToOwnerServer() end)
CreateButton(TeleportTab,"ForceTp Hook: Mini-Car",function()
    local carPartPath=Workspace:FindFirstChild("Structure") and Workspace.Structure:FindFirstChild("Raving Raceway") and Workspace.Structure["Raving Raceway"]:FindFirstChild("YellowCar") and Workspace.Structure["Raving Raceway"].YellowCar:FindFirstChild("Wedge")
    if not carPartPath then return end
    local character=LocalPlayer.Character; local rootPart=character and character:FindFirstChild("HumanoidRootPart"); local humanoid=character and character:FindFirstChildOfClass("Humanoid")
    if not rootPart or not humanoid then return end
    local originalPosition=rootPart.CFrame; local isSeated=false; local timeout=0
    local seatConnection; seatConnection=humanoid.Seated:Connect(function(seated) if seated then isSeated=true end end)
    while not isSeated and timeout<50 do if carPartPath then rootPart.CFrame=carPartPath.CFrame+Vector3.new(0,2,0) end; task.wait(0.1); timeout=timeout+1 end
    if seatConnection then seatConnection:Disconnect() end
    if isSeated then task.wait(0.2); local cm=carPartPath:FindFirstAncestorOfClass("Model") or carPartPath.Parent
        if cm and cm:IsA("Model") then cm:PivotTo(originalPosition+Vector3.new(0,3,0)) else carPartPath.CFrame=originalPosition+Vector3.new(0,3,0) end
        rootPart.CFrame=originalPosition+Vector3.new(0,3,0)
    else rootPart.CFrame=originalPosition end
end)

-- ── Owner-only server directory ───────────────────────────────────────────────
local OwnerServerListFrame, OwnerServerListLayout, OwnerServerStatus
local ownerServerRows = {}

local function ownerServerPlayerText(server)
    local current = server.players or server.playerCount or server.currentPlayers or server.current_players
    local maximum = server.maxPlayers or server.max_players or server.capacity
    if current ~= nil and maximum ~= nil then
        return string.format("Players - %s/%s", tostring(current), tostring(maximum))
    end
    return "Players - ?/?"
end

local function clearOwnerServerRows()
    for _, row in ipairs(ownerServerRows) do
        if row and row.Parent then row:Destroy() end
    end
    ownerServerRows = {}
end

local function joinOwnerServer(server, username)
    local placeId = tonumber(server.placeId or server.placeID)
    local jobId = tostring(server.jobId or server.jobID or ""):match("^%s*(.-)%s*$")
    if not placeId or placeId <= 0 or jobId == "" then
        if OwnerServerStatus then OwnerServerStatus.Text = "Invalid server data for " .. tostring(username) end
        return
    end

    if OwnerServerStatus then OwnerServerStatus.Text = "Joining " .. tostring(username) .. "..." end
    local ok, err = pcall(function()
        TeleportService:TeleportToPlaceInstance(placeId, jobId, LocalPlayer)
    end)
    if not ok and OwnerServerStatus then
        OwnerServerStatus.Text = "Join failed: " .. tostring(err)
    end
end

local function createOwnerServerRow(username, server)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, -12, 0, 58)
    row.BackgroundColor3 = Theme.Component
    row.BorderSizePixel = 0
    row.Parent = OwnerServerListFrame
    applyCorner(row, 8)
    applyBorder(row, Theme.Border, 1)

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, -112, 0, 22)
    nameLabel.Position = UDim2.new(0, 12, 0, 7)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = tostring(username)
    nameLabel.TextColor3 = Theme.TextMain
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 12
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.TextTruncate = Enum.TextTruncate.AtEnd
    nameLabel.Parent = row

    local playersLabel = Instance.new("TextLabel")
    playersLabel.Size = UDim2.new(1, -112, 0, 18)
    playersLabel.Position = UDim2.new(0, 12, 0, 30)
    playersLabel.BackgroundTransparency = 1
    playersLabel.Text = ownerServerPlayerText(server)
    playersLabel.TextColor3 = Theme.TextMuted
    playersLabel.Font = Enum.Font.Gotham
    playersLabel.TextSize = 10
    playersLabel.TextXAlignment = Enum.TextXAlignment.Left
    playersLabel.Parent = row

    local joinButton = Instance.new("TextButton")
    joinButton.Size = UDim2.new(0, 82, 0, 30)
    joinButton.Position = UDim2.new(1, -94, 0.5, -15)
    joinButton.BackgroundColor3 = Theme.Accent
    joinButton.Text = "JOIN"
    joinButton.TextColor3 = Color3.fromRGB(10, 10, 15)
    joinButton.Font = Enum.Font.GothamBold
    joinButton.TextSize = 11
    joinButton.AutoButtonColor = false
    joinButton.Parent = row
    applyCorner(joinButton, 7)
    applyBorder(joinButton, Theme.AccentGlow, 1)
    joinButton.Activated:Connect(function()
        joinOwnerServer(server, username)
    end)

    ownerServerRows[#ownerServerRows + 1] = row
end

local function refreshOwnerServers()
    if not OwnerServersTab or not OwnerServerListFrame then return end
    if OwnerServerStatus then OwnerServerStatus.Text = "Loading server directory..." end
    clearOwnerServerRows()

    task.spawn(function()
        local data = requestJson("GET", SERVER_URL .. "/api/invites")
        if not data or data.success ~= true or type(data.invites) ~= "table" then
            if OwnerServerStatus then OwnerServerStatus.Text = "Could not load /api/invites." end
            return
        end

        local entries = {}
        for username, server in pairs(data.invites) do
            if type(server) == "table" then
                entries[#entries + 1] = { username = tostring(username), server = server }
            end
        end
        table.sort(entries, function(a, b) return a.username:lower() < b.username:lower() end)
        for _, entry in ipairs(entries) do
            createOwnerServerRow(entry.username, entry.server)
        end
        if OwnerServerStatus then
            OwnerServerStatus.Text = string.format("%d server(s) found • Select JOIN to teleport", #entries)
        end
    end)
end

if OwnerServersTab then
    CreateSection(OwnerServersTab, "Private Server Directory")
    CreateAnnouncement(OwnerServersTab, "🛡", "OWNER ACCESS ONLY", "This directory is visible only to SkylerModz_67.", Theme.AccentGlow)

    local refreshButton = Instance.new("TextButton")
    refreshButton.Size = UDim2.new(1, -16, 0, 40)
    refreshButton.BackgroundColor3 = Theme.Component
    refreshButton.Text = "  REFRESH SERVER LIST"
    refreshButton.TextColor3 = Theme.TextMain
    refreshButton.Font = Enum.Font.GothamBold
    refreshButton.TextSize = 11
    refreshButton.TextXAlignment = Enum.TextXAlignment.Left
    refreshButton.AutoButtonColor = false
    refreshButton.Parent = OwnerServersTab
    applyCorner(refreshButton, 8)
    applyBorder(refreshButton, Theme.Border, 1)
    refreshButton.Activated:Connect(refreshOwnerServers)

    OwnerServerListFrame = Instance.new("ScrollingFrame")
    OwnerServerListFrame.Size = UDim2.new(1, -16, 0, 250)
    OwnerServerListFrame.BackgroundColor3 = Theme.Background
    OwnerServerListFrame.BorderSizePixel = 0
    OwnerServerListFrame.ScrollBarThickness = 4
    OwnerServerListFrame.ScrollBarImageColor3 = Theme.Border
    OwnerServerListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    OwnerServerListFrame.Parent = OwnerServersTab
    applyCorner(OwnerServerListFrame, 8)
    OwnerServerListLayout = Instance.new("UIListLayout")
    OwnerServerListLayout.Padding = UDim.new(0, 6)
    OwnerServerListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    OwnerServerListLayout.Parent = OwnerServerListFrame
    OwnerServerListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        OwnerServerListFrame.CanvasSize = UDim2.new(0, 0, 0, OwnerServerListLayout.AbsoluteContentSize.Y + 12)
    end)

    OwnerServerStatus = Instance.new("TextLabel")
    OwnerServerStatus.Size = UDim2.new(1, -16, 0, 24)
    OwnerServerStatus.BackgroundTransparency = 1
    OwnerServerStatus.Text = "Open this tab or press refresh to load /api/invites."
    OwnerServerStatus.TextColor3 = Theme.TextMuted
    OwnerServerStatus.Font = Enum.Font.Gotham
    OwnerServerStatus.TextSize = 10
    OwnerServerStatus.TextXAlignment = Enum.TextXAlignment.Left
    OwnerServerStatus.Parent = OwnerServersTab

    --OwnerServersTab.Btn.Activated:Connect(refreshOwnerServers)
end

-- ── Free Scripts ──────────────────────────────────────────────────────────────
CreateSection(FreeScriptsTab,"Universal Scripts")
CreateButton(FreeScriptsTab,"FE DropKick R6 & R15",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f706c6174696e77772f4372757374794d61696e2f726566732f68656164732f6d61696e2f756e6976657273616c2f44726f704b69636b2e6c7561")))() end)
CreateButton(FreeScriptsTab,"FE VR NDS",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c6572746573746572313836332f41696a7262636c6c667465747773742f726566732f68656164732f6d61696e2f564e6578426574612e6c7561")))() end)
CreateButton(FreeScriptsTab,"Fly Gui V4",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c657255647073732f417363796e746875622f726566732f68656164732f6d61696e2f417363796e744875622d666c7976312e6c7561")))() end)
CreateButton(FreeScriptsTab,"Fly Car V4",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c657255647073732f417363796e746875622f726566732f68656164732f6d61696e2f417363796e744875622d76666c792e6c7561")))() end)
CreateButton(FreeScriptsTab,"FE Emotes R15",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c657255647073732f46524545534352495054532f726566732f68656164732f6d61696e2f46455f456d6f7465735f556e6976657273616c5f456d6f7465735f5363726970742d312e747874")))() end)
CreateButton(FreeScriptsTab,"FE Return By Death v1",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c657255647073732f417363796e746875622f726566732f68656164732f6d61696e2f417363796e744875622d52657475726e627964656174682e6c7561")))() end)

-- ── Private Scripts ───────────────────────────────────────────────────────────
CreateSection(PrivateTab,"Premium Content")
CreateButton(PrivateTab,"Sky-Hub FE +200 scripts",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f6769746875622e636f6d2f796f667269656e6466726f6d7363686f6f6c312f536b792d4875622f7261772f726566732f68656164732f6d61696e2f464525323054726f6c6c696e672532304755492e6c756175")))() end)
CreateButton(PrivateTab,"FE Noob Converter (High Lag)",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c657255647073732f417363796e746875622f726566732f68656164732f6d61696e2f417363796e744875622d6e6f6f622e6c7561")))() end)
CreateButton(PrivateTab,"AscyntHub For MM2",function() loadstring(game:HttpGet(_decodeStatic("68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f536b796c657255647073732f417363796e746875622f726566732f68656164732f6d61696e2f417363796e744875622d6d6d322e6c7561")))() end)

-- ── API Status ────────────────────────────────────────────────────────────────
CreateSection(ApiTab,"Live API Dashboard")
CreateAnnouncement(ApiTab,"🌐","API Monitor","Press Refresh to update server status in real-time.",Theme.AccentGlow)

-- Grid 2 columnas para las stat cards
local StatsGrid=Instance.new("Frame"); StatsGrid.Size=UDim2.new(1,-16,0,130); StatsGrid.BackgroundTransparency=1; StatsGrid.Parent=ApiTab
local StatsGridLayout=Instance.new("UIGridLayout"); StatsGridLayout.CellSize=UDim2.new(0.48,0,0,56)
StatsGridLayout.CellPadding=UDim2.new(0.04,0,0,8); StatsGridLayout.HorizontalAlignment=Enum.HorizontalAlignment.Left; StatsGridLayout.Parent=StatsGrid

local CardStatus  = CreateStatCard(StatsGrid,"API Status", "...",  Theme.Accent)
local CardPing    = CreateStatCard(StatsGrid,"Ping",       "...",  Theme.AccentGlow)
local CardOnline  = CreateStatCard(StatsGrid,"Users Online","...", Theme.Success)
local CardServers = CreateStatCard(StatsGrid,"Active Servers","...",Theme.Warning)


CreateSection(ApiTab,"Server Info")
local ApiInfoFrame=Instance.new("Frame"); ApiInfoFrame.Size=UDim2.new(1,-16,0,80); ApiInfoFrame.BackgroundColor3=Theme.Card; ApiInfoFrame.Parent=ApiTab
applyCorner(ApiInfoFrame,8); applyBorder(ApiInfoFrame,Theme.Border,1)

local ApiInfoText=Instance.new("TextLabel"); ApiInfoText.Size=UDim2.new(1,-16,1,0); ApiInfoText.Position=UDim2.new(0,8,0,0)
ApiInfoText.Text="Press Refresh to load server stats..."; ApiInfoText.Font=Enum.Font.Gotham; ApiInfoText.TextSize=11
ApiInfoText.TextColor3=Theme.TextMuted; ApiInfoText.TextXAlignment=Enum.TextXAlignment.Left
ApiInfoText.TextYAlignment=Enum.TextYAlignment.Top; ApiInfoText.TextWrapped=true; ApiInfoText.BackgroundTransparency=1; ApiInfoText.Parent=ApiInfoFrame

local function RefreshApiStatus()
    local startTime = tick()
    local cr = (syn and syn.request) or request or (http and http.request)
    --Remove Http request -> to HttpAsync Request based in Delta API
    CardStatus.Value.Text = "..."
    CardPing.Value.Text   = "..."
    CardOnline.Value.Text = "..."
    CardServers.Value.Text= "..."
    ApiInfoText.Text      = "Fetching..."

    local pingStart=tick()
    local ok, res = pcall(function()
        if cr then
            return cr({Url=SERVER_URL.."/api/status", Method="GET", Headers={["Content-Type"]="application/json"}})
        end
    end)
    local pingMs = math.floor((tick()-pingStart)*1000)

    if ok and res and res.StatusCode==200 then
        local dok, data = pcall(function() return HttpService:JSONDecode(res.Body) end)
        if dok and data then
            CardStatus.Value.Text  = data.status or "Online"
            CardOnline.Value.Text  = tostring(data.online_users or "—")
            CardServers.Value.Text = tostring(data.active_servers or "—")
            CardPing.Value.Text    = pingMs.."ms"
            ApiInfoText.Text = string.format(
                "Version: %s\nLast check: %s\nEnvironment: %s\nMsg: %s",
                tostring(data.version or "v1"),
                os.date("%H:%M:%S"),
                tostring(data.environment or "production"),
                tostring(data.message or "OK")
            )
        else
            CardStatus.Value.Text="Parse Err"; ApiInfoText.Text="Failed to decode JSON."
        end
    else
        -- Fallback: solo muestra el ping y estado offline
        CardStatus.Value.Text = "Offline"
        CardPing.Value.Text   = pingMs.."ms"
        CardOnline.Value.Text = "—"
        CardServers.Value.Text= "—"
        ApiInfoText.Text      = "Could not reach API server.\nCheck your connection or executor."
    end
end

CreateButton(ApiTab,"🔄  Refresh Status", RefreshApiStatus)

-- ── Performance ───────────────────────────────────────────────────────────────
CreateSection(PerforTab,"Graphics Presets")
CreateAnnouncement(PerforTab,"🎮","Performance Tips","Use Minecraft Graphics for maximum FPS on low devices.",Theme.Success)
CreateButton(PerforTab,"Minecraft Graphics (Low Dev)",function()
    for _, obj in pairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") then obj.Material=Enum.Material.SmoothPlastic
        elseif obj:IsA("Texture") or obj:IsA("Decal") then obj:Destroy() end
    end
    Lighting.GlobalShadows=false; settings().Rendering.QualityLevel=Enum.QualityLevel.Level01
end)
CreateButton(PerforTab,"RTX Graphics (Beta)",function()
    Lighting.GlobalShadows=true; Lighting.Brightness=2.5; Lighting.OutdoorAmbient=Color3.fromRGB(130,140,160); Lighting.Ambient=Color3.fromRGB(30,30,35)
    local bloom=Lighting:FindFirstChildOfClass("BloomEffect") or Instance.new("BloomEffect",Lighting); bloom.Intensity=1; bloom.Size=24; bloom.Threshold=0.8
    local cc=Lighting:FindFirstChildOfClass("ColorCorrectionEffect") or Instance.new("ColorCorrectionEffect",Lighting); cc.Brightness=0.05; cc.Contrast=0.2; cc.Saturation=0.15
    local sr=Lighting:FindFirstChildOfClass("SunRaysEffect") or Instance.new("SunRaysEffect",Lighting); sr.Intensity=0.1; sr.Spread=0.6
    settings().Rendering.QualityLevel=Enum.QualityLevel.Level21
end)
CreateButton(PerforTab,"High Graphics - Vibrant",function()
    Lighting.GlobalShadows=true; Lighting.Brightness=1.8; settings().Rendering.QualityLevel=Enum.QualityLevel.Level15
end)

-- ── Configurations ────────────────────────────────────────────────────────────
CreateSection(ConfigTab,"Save & Load")
CreateButton(ConfigTab,"Save Config",function()
    if writefile then
        local data={WalkSpeed=50,AutoTP=true}
        writefile("PremiumHub_NDS.cfg",HttpService:JSONEncode(data))
    end
end)

-- ── Information ───────────────────────────────────────────────────────────────
CreateSection(InfoTab,"Change Log")
CreateAnnouncement(InfoTab,"✅","v2.7 — 25/08/26","Added local Auto Detect Disaster with a 4-second alert and Auto Save Player with SafeSpawn repositioning, local position lock and AntiFling protection. Telekinesis unchanged.",Theme.Success)
CreateAnnouncement(InfoTab,"✅","v2.6 — 20/08/26","Sleep added to FUN with Float Sleep toggle, Chat moved to a more visible position, one-time update notice added; Telekinesis unchanged.",Theme.Success)
CreateAnnouncement(InfoTab,"✅","v2.5 — 20/08/26","Neo UI redesign, global banners, owner-server teleport and moderated Chat added; Telekinesis unchanged.",Theme.Success)
CreateAnnouncement(InfoTab,"✅","v2.4 — 18/08/26","FE VR SCRIPT ADDED in FREE section, Espiral and Tornado Espiral added.",Theme.Success)
CreateAnnouncement(InfoTab,"✅","v2.3 — 15/08/26","Vehicle Helper, LaunchLobby V1/V2.",Theme.Success)
CreateAnnouncement(InfoTab,"✅","v2.2 — 09/08/26","updated UI/Animations and fixed bug timeout",Theme.Success)
CreateAnnouncement(InfoTab,"✅","v2.1 — 24/06/26","SelfTouchFling And V2 has been fixed bugs, Fling All added , Api checker added,Fixed Server Timeout, Added new a serverside system, Announcements aadded",Theme.Success)
CreateAnnouncement(InfoTab,"⚠️ ","Important-","Telekinesis module its  a serverside, But it causes too much lag even on good devices, I recommend only using it on pc.",Theme.Success)
CreateAnnouncement(InfoTab,"🔧","v1.0 — 14/06/26","Fix sliders Android/PC. Scripts restaurated in searchbar.",Theme.Accent)
CreateAnnouncement(InfoTab,"ℹ","Developer","@SkylerModz (SkylerModz_67) — All rights reserved.",Theme.TextMuted)

-- =============================================================================
--                         AUTENTICACIÓN VPS
-- =============================================================================
-- ── V2.6 one-time announcement ───────────────────────────────────────────────
local ANNOUNCE_FILE = "announce.json"

local function announcementWasShown()
    if not isfile or not readfile then return false end
    local okRead, raw = pcall(readfile, ANNOUNCE_FILE)
    if not okRead or type(raw) ~= "string" then return false end
    local okJson, data = pcall(function() return HttpService:JSONDecode(raw) end)
    return okJson and type(data) == "table" and data.announce_showed == true
end

local function markAnnouncementShown()
    if not writefile then return end
    pcall(function()
        writefile(ANNOUNCE_FILE, HttpService:JSONEncode({announce_showed = true}))
    end)
end

local function maybeShowV26Announcement()
    if announcementWasShown() then return end
    markAnnouncementShown()

    local announcement = Instance.new("Frame")
    announcement.Name = "AscyntV26Announcement"
    announcement.AnchorPoint = Vector2.new(0.5, 0.5)
    announcement.Position = UDim2.new(0.5, 0, 0.5, 0)
    announcement.Size = UDim2.new(0.82, 0, 0, 300)
    announcement.BackgroundColor3 = Theme.Card
    announcement.BorderSizePixel = 0
    announcement.ZIndex = 50
    announcement.Parent = ScreenGui
    applyCorner(announcement, 14)
    applyBorder(announcement, Theme.AccentGlow, 1.5)

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -56, 0, 32)
    title.Position = UDim2.new(0, 18, 0, 14)
    title.BackgroundTransparency = 1
    title.Text = "ASCYNTHUB v2.6  //  UPDATE NOTICE"
    title.TextColor3 = Theme.AccentGlow
    title.Font = Enum.Font.GothamBold
    title.TextSize = 13
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.ZIndex = 51
    title.Parent = announcement

    local body = Instance.new("TextLabel")
    body.Size = UDim2.new(1, -36, 1, -92)
    body.Position = UDim2.new(0, 18, 0, 54)
    body.BackgroundTransparency = 1
    body.Text = "Hello everyone, I want to share some information about this new v2.6 update. I added a new feature called Sleep in the FUN category. I also moved the Chat section to a more visible position. You can now chat in the CHAT section and teleport to my server from the TP section by pressing TP to Owner. Thank you for the support. I will keep trying to add more useful features.\n\nP.S. I know Telekinesis causes too much lag on Android, and I am looking for a way to fix it."
    body.TextColor3 = Theme.TextMain
    body.Font = Enum.Font.Gotham
    body.TextSize = 11
    body.TextWrapped = true
    body.TextXAlignment = Enum.TextXAlignment.Left
    body.TextYAlignment = Enum.TextYAlignment.Top
    body.ZIndex = 51
    body.Parent = announcement

    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 86, 0, 32)
    close.Position = UDim2.new(1, -104, 1, -46)
    close.BackgroundColor3 = Theme.Accent
    close.BorderSizePixel = 0
    close.Text = "CLOSE"
    close.TextColor3 = Color3.fromRGB(255, 255, 255)
    close.Font = Enum.Font.GothamBold
    close.TextSize = 11
    close.ZIndex = 51
    close.Parent = announcement
    applyCorner(close, 8)
    close.Activated:Connect(function()
        if announcement and announcement.Parent then announcement:Destroy() end
    end)
end

SubmitBtn.Activated:Connect(function()
    local inputKey=KeyInput.Text; if inputKey=="" then return end
    SubmitBtn.Text="Verifying Client..."; SubmitBtn.Active=false

    local hwid="UNKNOWN_HWID"
    pcall(function() hwid=game:GetService("RbxAnalyticsService"):GetClientId() end)

    local jsonPayload=HttpService:JSONEncode({key=inputKey, hwid=hwid})
    local success,data=false,nil
    local cr=(syn and syn.request) or request or (http and http.request)

    if cr then
        local ok,res=pcall(function() return cr({Url=VPS_URL,Method="POST",Headers={["Content-Type"]="application/json"},Body=jsonPayload}) end)
        if ok and res.StatusCode==200 then local dk,dec=pcall(function() return HttpService:JSONDecode(res.Body) end); if dk then success=true;data=dec end end
    end
    if not success then
        local ok,res=pcall(function() return HttpService:PostAsync(VPS_URL,jsonPayload,Enum.HttpContentType.ApplicationJson) end)
        if ok then local dk,dec=pcall(function() return HttpService:JSONDecode(res) end); if dk then success=true;data=dec end end
    end

    if success and data then
        if data.success then
            LoginFrame.Visible=false
            setMainFrameVisible(true)
            startLiveSync()
            maybeShowV26Announcement()
            loadReplicateMotor()

            Tabs["Player Controls"].Page.Visible=true
            Tabs["Player Controls"].Btn.BackgroundColor3=Color3.fromRGB(24,24,32)
            Tabs["Player Controls"].Btn.TextColor3=Color3.fromRGB(0,130,255)
            Tabs["Player Controls"].Stroke.Transparency=0.3
            Tabs["Player Controls"].Stroke.Color=Theme.AccentGlow

            local initialOnline=data.online_users or 1
            StatusText.Text=string.format("License: %s | Remaining: %s days | Expiry: %s | Online: %s", data.type, tostring(data.days_left), data.expires_at, tostring(initialOnline))

            -- Auto-refresh API tab en background
            task.spawn(function()
                while task.wait(25) do
                    pcall(function()
                        local lp=HttpService:JSONEncode({key=inputKey,hwid=hwid})
                        local ls,ld=false,nil
                        local lcr=(syn and syn.request) or request or (http and http.request)
                        if lcr then
                            local ok,res=pcall(function() return lcr({Url=VPS_URL,Method="POST",Headers={["Content-Type"]="application/json"},Body=lp}) end)
                            if ok and res.StatusCode==200 then local dk,dec=pcall(function() return HttpService:JSONDecode(res.Body) end); if dk then ls=true;ld=dec end end
                        end
                        if not ls then local ok,res=pcall(function() return HttpService:PostAsync(VPS_URL,lp,Enum.HttpContentType.ApplicationJson) end)
                            if ok then local dk,dec=pcall(function() return HttpService:JSONDecode(res) end); if dk then ls=true;ld=dec end end
                        end
                        if ls and ld and ld.success then
                            StatusText.Text=string.format("License: %s | Remaining: %s days | Expiry: %s | Online: %s", data.type, tostring(data.days_left), data.expires_at, tostring(ld.online_users or 1))
                        end
                    end)
                end
            end)
        else
            SubmitBtn.Text=data.message or "Invalid Key."; SubmitBtn.BackgroundColor3=Color3.fromRGB(160,40,40)
            task.wait(2); SubmitBtn.Text="Verify Key"; SubmitBtn.BackgroundColor3=Theme.Accent; SubmitBtn.Active=true
        end
    else
        SubmitBtn.Text="Server Timeout"; SubmitBtn.BackgroundColor3=Color3.fromRGB(160,40,40)
        task.wait(2); SubmitBtn.Text="Verify License Key"; SubmitBtn.BackgroundColor3=Theme.Accent; SubmitBtn.Active=true
    end
end)

