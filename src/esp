--[[
    IRONMETAL ESP+ UNIVERSAL

    A Matcha-native conversion of the Exunys Universal ESP concept.
    It uses only the Matcha UI binding, Drawing API, WorldToScreen,
    supported Roblox objects, RenderStepped, and polling loops.

    Matcha does not expose the primitives required for the source module's
    metatable hooks, remote config library, Quad renderer, Highlight objects,
    or Instance.new chams. Those features are intentionally not emulated.
]]

-- Clean up an older copy when the script is executed again.
pcall(function()
    if MatchaESP and MatchaESP.Exit then
        MatchaESP:Exit()
    end
end)

-- Remove stale menu registrations left by older builds of this script.
pcall(function()
    UI.RemoveTab("Matcha ESP")
end)
pcall(function()
    UI.RemoveTab("Matcha ESP+")
end)
pcall(function()
    UI.RemoveTab("IRONMETAL ESP+ UNIVERSAL")
end)

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer

local API = {}
local running = true
local targets = {}
local renderConnection = nil
local UI_TAB_NAME = "ESP+"
local UI_PREFIX = "mxesp3_"

local Theme = {
    Visible = Color3.fromRGB(80, 255, 120),
    Occluded = Color3.fromRGB(255, 90, 90),
    Friendly = Color3.fromRGB(90, 190, 255),
    NPC = Color3.fromRGB(255, 175, 70),
    Outline = Color3.fromRGB(0, 0, 0),
    Crosshair = Color3.fromRGB(80, 255, 120),
    Skeleton = Color3.fromRGB(255, 255, 255),
    Health = Color3.fromRGB(80, 255, 120),
    Radar = Color3.fromRGB(30, 30, 38)
}

local function clamp(value, minimum, maximum)
    if value < minimum then
        return minimum
    end
    if value > maximum then
        return maximum
    end
    return value
end

local function uiValue(id, fallback)
    local ok, value = pcall(function()
        return UI.GetValue(UI_PREFIX .. id)
    end)

    if ok and value ~= nil then
        return value
    end
    return fallback
end

local function asBoolean(id, fallback)
    local value = uiValue(id, fallback)
    return value == true
end

local function uiColor(id, fallback)
    local value = uiValue(id, fallback)
    local ok, isColor = pcall(function()
        return typeof(value) == "Color3"
    end)

    if ok and isColor then
        return value
    end
    return fallback
end

local function safeChild(parent, name)
    if not parent then
        return nil
    end

    local ok, child = pcall(function()
        return parent:FindFirstChild(name)
    end)
    return ok and child or nil
end

local function safeIsA(object, className)
    if not object then
        return false
    end

    local ok, result = pcall(function()
        return object:IsA(className)
    end)
    return ok and result or false
end

local function safeParent(object)
    local ok, parent = pcall(function()
        return object.Parent
    end)
    return ok and parent or nil
end

local function project(position)
    local ok, point, onScreen = pcall(function()
        return WorldToScreen(position)
    end)

    if ok and point then
        return point, onScreen == true
    end
    return nil, false
end

local function newDrawing(kind)
    local ok, object = pcall(function()
        return Drawing.new(kind)
    end)

    if not ok or not object then
        return nil
    end

    pcall(function()
        object.Visible = false
    end)
    return object
end

local function setVisible(object, state)
    if object then
        pcall(function()
            object.Visible = state
        end)
    end
end

local function removeDrawing(object)
    if object then
        pcall(function()
            object:Remove()
        end)
    end
end

local function drawLine(object, from, to, color, thickness, transparency)
    if not object then
        return
    end

    pcall(function()
        object.From = from
        object.To = to
        object.Color = color
        object.Thickness = thickness
        object.Transparency = transparency
    end)
    setVisible(object, true)
end

local function drawSquare(object, position, size, filled, color, thickness, transparency)
    if not object then
        return
    end

    pcall(function()
        object.Position = position
        object.Size = size
        object.Filled = filled
        object.Color = color
        object.Transparency = transparency
    end)

    -- Some Matcha builds expose Thickness on Square and some do not.
    pcall(function()
        object.Thickness = thickness
    end)
    setVisible(object, true)
end

local function drawCircle(object, position, radius, filled, color, thickness, transparency)
    if not object then
        return
    end

    pcall(function()
        object.Position = position
        object.Radius = radius
        object.NumSides = 30
        object.Filled = filled
        object.Color = color
        object.Thickness = thickness
        object.Transparency = transparency
    end)
    setVisible(object, true)
end

local function drawText(object, position, text, color, size, outline)
    if not object then
        return
    end

    pcall(function()
        object.Position = position
        object.Text = text
        object.Color = color
        object.Center = true
        object.Outline = outline ~= false
        object.Font = 2
        object.FontSize = size
        object.Transparency = 1
    end)
    setVisible(object, true)
end

local function drawTriangle(object, pointA, pointB, pointC, color, transparency)
    if not object then
        return
    end

    pcall(function()
        object.PointA = pointA
        object.PointB = pointB
        object.PointC = pointC
        object.Color = color
        object.Transparency = transparency
        object.Filled = true
    end)
    setVisible(object, true)
end

local function drawImage(object, position, size, data, transparency)
    if not object or not data then
        return
    end

    pcall(function()
        object.Position = position
        object.Size = size
        object.Data = data
        object.Transparency = transparency
        object.Rounding = 4
    end)
    setVisible(object, true)
end

local function createVisuals()
    local visuals = {
        All = {},
        BoxFill = nil,
        BoxGlow = nil,
        BoxOutline = nil,
        Box = nil,
        Corner = {},
        Box3D = {},
        HealthBackground = nil,
        HealthFill = nil,
        HealthText = nil,
        TracerOutline = nil,
        Tracer = nil,
        HeadOutline = nil,
        Head = nil,
        HeadGlow = nil,
        LabelBackground = nil,
        LabelGlow = nil,
        Label = nil,
        Skeleton = {},
        Chams = {},
        ChamsHead = nil,
        OOFArrow = nil,
        RadarDot = nil,
        Profile = nil
    }

    local function add(kind)
        local object = newDrawing(kind)
        if object then
            visuals.All[#visuals.All + 1] = object
        end
        return object
    end

    visuals.BoxFill = add("Square")
    visuals.BoxGlow = add("Square")
    visuals.BoxOutline = add("Square")
    visuals.Box = add("Square")
    visuals.HealthBackground = add("Square")
    visuals.HealthFill = add("Square")
    visuals.HealthText = add("Text")
    visuals.TracerOutline = add("Line")
    visuals.Tracer = add("Line")
    visuals.HeadOutline = add("Circle")
    visuals.Head = add("Circle")
    visuals.HeadGlow = add("Circle")
    visuals.LabelBackground = add("Square")
    visuals.LabelGlow = add("Text")
    visuals.Label = add("Text")
    visuals.ChamsHead = add("Circle")
    visuals.OOFArrow = add("Triangle")
    visuals.RadarDot = add("Circle")
    visuals.Profile = add("Image")

    for index = 1, 8 do
        visuals.Corner[index] = add("Line")
    end

    for index = 1, 16 do
        visuals.Skeleton[index] = add("Line")
        visuals.Chams[index] = add("Line")
    end

    for index = 1, 12 do
        visuals.Box3D[index] = add("Line")
    end

    return visuals
end

local function hideVisuals(visuals)
    if not visuals then
        return
    end

    for _, object in ipairs(visuals.All) do
        setVisible(object, false)
    end
end

local function destroyVisuals(visuals)
    if not visuals then
        return
    end

    for _, object in ipairs(visuals.All) do
        removeDrawing(object)
    end
    visuals.All = {}
end

local function addTarget(object, kind, pseudoName, renderDistance, manual)
    if not object or targets[object] then
        return nil
    end

    local entry = {
        Object = object,
        Kind = kind,
        Name = pseudoName,
        RenderDistance = renderDistance,
        Manual = manual == true,
        Visibility = {
            LastCheck = 0,
            Visible = true
        },
        Profile = {
            State = "idle",
            Data = nil,
            RetryAt = 0
        },
        Visuals = createVisuals()
    }

    targets[object] = entry
    return entry
end

local function removeTarget(object)
    local entry = targets[object]
    if not entry then
        return false
    end

    destroyVisuals(entry.Visuals)
    targets[object] = nil
    return true
end

local function playerList()
    local ok, list = pcall(function()
        return Players:GetPlayers()
    end)
    return ok and list or {}
end

local function isPlayerCharacter(model)
    for _, player in ipairs(playerList()) do
        local ok, character = pcall(function()
            return player.Character
        end)
        if ok and character == model then
            return true
        end
    end
    return false
end

local function sameTeam(player)
    if not player or not LocalPlayer then
        return false
    end

    local playerNeutral = false
    local localNeutral = false
    pcall(function()
        playerNeutral = player.Neutral == true
    end)
    pcall(function()
        localNeutral = LocalPlayer.Neutral == true
    end)

    -- Neutral players are never considered teammates.
    if playerNeutral or localNeutral then
        return false
    end

    local playerTeam = nil
    local localTeam = nil
    pcall(function()
        playerTeam = player.Team
    end)
    pcall(function()
        localTeam = LocalPlayer.Team
    end)

    if playerTeam ~= nil and localTeam ~= nil then
        return playerTeam == localTeam
    end

    local ok, result = pcall(function()
        return player.TeamColor == LocalPlayer.TeamColor
    end)
    return ok and result == true
end

local function belongsTo(instance, ancestor)
    if not instance or not ancestor then
        return false
    end

    local current = instance
    for _ = 1, 16 do
        if current == ancestor then
            return true
        end

        current = safeParent(current)
        if not current then
            return false
        end
    end
    return false
end

local function targetIsVisible(entry, resolved, cameraPosition, targetPosition)
    if not asBoolean("matcha_visibility_check", true) then
        return true
    end

    local now = tick()
    local interval = uiValue("matcha_visibility_rate", 100) / 1000
    local cache = entry.Visibility

    if now - cache.LastCheck < interval then
        return cache.Visible
    end
    cache.LastCheck = now

    local localCharacter = nil
    pcall(function()
        localCharacter = LocalPlayer.Character
    end)

    local origin = cameraPosition
    for _ = 1, 4 do
        local direction = targetPosition - origin
        local ok, result = pcall(function()
            return Workspace:Raycast(origin, direction)
        end)

        -- If a Matcha build cannot raycast, fail open so ESP does not disappear.
        if not ok or not result then
            cache.Visible = true
            return true
        end

        local hit = nil
        local hitPosition = nil
        pcall(function()
            hit = result.Instance
            hitPosition = result.Position
        end)

        if not hit then
            cache.Visible = true
            return true
        end

        local visible = hit == resolved.Head or hit == resolved.Root
        if not visible and entry.Kind ~= "Part" then
            visible = belongsTo(hit, resolved.Model)
        end

        if visible then
            cache.Visible = true
            return true
        end

        -- The documented Matcha surface does not expose RaycastParams creation.
        -- If the camera ray hits the local character, step beyond it and retry.
        if localCharacter and belongsTo(hit, localCharacter) and hitPosition then
            local remaining = targetPosition - hitPosition
            local magnitude = remaining.Magnitude
            if magnitude <= 0.01 then
                cache.Visible = true
                return true
            end
            origin = hitPosition + (remaining * (3 / magnitude))
        else
            cache.Visible = false
            return false
        end
    end

    cache.Visible = false
    return false
end

local function resolveEntry(entry)
    local object = entry.Object
    local model = nil
    local root = nil
    local head = nil
    local humanoid = nil
    local displayName = entry.Name
    local username = entry.Name
    local friendly = false

    if entry.Kind == "Player" then
        local ok, character = pcall(function()
            return object.Character
        end)
        model = ok and character or nil
        if not model then
            return nil
        end

        root = safeChild(model, "HumanoidRootPart") or safeChild(model, "Torso")
        head = safeChild(model, "Head") or root
        humanoid = safeChild(model, "Humanoid")
        friendly = sameTeam(object)

        pcall(function()
            username = object.Name
        end)

        if not displayName then
            pcall(function()
                displayName = object.DisplayName
            end)
            if not displayName or displayName == "" then
                pcall(function()
                    displayName = object.Name
                end)
            end
        end
    elseif entry.Kind == "NPC" or entry.Kind == "Model" then
        model = object
        root = safeChild(model, "HumanoidRootPart") or safeChild(model, "Torso") or safeChild(model, "UpperTorso")
        head = safeChild(model, "Head") or root
        humanoid = safeChild(model, "Humanoid")

        if not displayName then
            pcall(function()
                displayName = model.Name
            end)
        end
        username = displayName
    else
        root = object
        head = object
        model = safeParent(object)

        if not displayName then
            pcall(function()
                displayName = object.Name
            end)
        end
        username = displayName
    end

    if not root or not head then
        return nil
    end

    return {
        Model = model,
        Root = root,
        Head = head,
        Humanoid = humanoid,
        Name = displayName or "Object",
        DisplayName = displayName or "Object",
        Username = username or displayName or "Object",
        Friendly = friendly,
        IsSelf = object == LocalPlayer
    }
end

local function getPosition(part)
    local ok, position = pcall(function()
        return part.Position
    end)
    return ok and position or nil
end

local function getHealth(humanoid)
    if not humanoid then
        return nil, nil, nil
    end

    local ok, health, maximum = pcall(function()
        return humanoid.Health, humanoid.MaxHealth
    end)

    if not ok or not maximum or maximum <= 0 then
        return nil, nil, nil
    end

    return health, maximum, clamp(health / maximum, 0, 1)
end

local function healthColor(ratio)
    return Color3.fromRGB(
        math.floor(255 * (1 - ratio)),
        math.floor(255 * ratio),
        0
    )
end

local function renderCorners(lines, x, y, width, height, color)
    local length = clamp(math.floor(math.min(width, height) * 0.25), 3, 20)
    local left = x
    local right = x + width
    local top = y
    local bottom = y + height

    drawLine(lines[1], Vector2.new(left, top), Vector2.new(left + length, top), color, 2, 1)
    drawLine(lines[2], Vector2.new(left, top), Vector2.new(left, top + length), color, 2, 1)
    drawLine(lines[3], Vector2.new(right, top), Vector2.new(right - length, top), color, 2, 1)
    drawLine(lines[4], Vector2.new(right, top), Vector2.new(right, top + length), color, 2, 1)
    drawLine(lines[5], Vector2.new(left, bottom), Vector2.new(left + length, bottom), color, 2, 1)
    drawLine(lines[6], Vector2.new(left, bottom), Vector2.new(left, bottom - length), color, 2, 1)
    drawLine(lines[7], Vector2.new(right, bottom), Vector2.new(right - length, bottom), color, 2, 1)
    drawLine(lines[8], Vector2.new(right, bottom), Vector2.new(right, bottom - length), color, 2, 1)
end

local R15Segments = {
    {"Head", "UpperTorso"},
    {"UpperTorso", "LowerTorso"},
    {"UpperTorso", "LeftUpperArm"},
    {"LeftUpperArm", "LeftLowerArm"},
    {"LeftLowerArm", "LeftHand"},
    {"UpperTorso", "RightUpperArm"},
    {"RightUpperArm", "RightLowerArm"},
    {"RightLowerArm", "RightHand"},
    {"LowerTorso", "LeftUpperLeg"},
    {"LeftUpperLeg", "LeftLowerLeg"},
    {"LeftLowerLeg", "LeftFoot"},
    {"LowerTorso", "RightUpperLeg"},
    {"RightUpperLeg", "RightLowerLeg"},
    {"RightLowerLeg", "RightFoot"}
}

local R6Segments = {
    {"Head", "Torso"},
    {"Torso", "Left Arm"},
    {"Torso", "Right Arm"},
    {"Torso", "Left Leg"},
    {"Torso", "Right Leg"}
}

local function renderRigLines(lines, model, color, thickness, transparency)
    if not model then
        return
    end

    local isR15 = safeChild(model, "UpperTorso") ~= nil
    local segments = isR15 and R15Segments or R6Segments

    for index, pair in ipairs(segments) do
        local first = safeChild(model, pair[1])
        local second = safeChild(model, pair[2])
        local firstPosition = getPosition(first)
        local secondPosition = getPosition(second)

        if firstPosition and secondPosition then
            local a, aVisible = project(firstPosition)
            local b, bVisible = project(secondPosition)
            if aVisible and bVisible then
                drawLine(lines[index], a, b, color, thickness, transparency)
            end
        end
    end
end

local function renderSkeleton(entry, model, color)
    renderRigLines(entry.Visuals.Skeleton, model, color, 1, 1)
end

local function renderChams(entry, resolved, color)
    if not resolved.Model then
        return
    end

    local filled = asBoolean("matcha_chams_filled", false)
    local outlineMode = uiValue("matcha_chams_mode", 0) == 1
    local thickness = filled and 9 or (outlineMode and 2 or 4)
    local transparency = filled and 0.55 or 0.9
    renderRigLines(entry.Visuals.Chams, resolved.Model, color, thickness, transparency)

    local headPosition = getPosition(resolved.Head)
    if headPosition then
        local point, onScreen = project(headPosition)
        if point and onScreen then
            drawCircle(entry.Visuals.ChamsHead, point, filled and 8 or 5, filled, color, thickness, transparency)
        end
    end
end

local function getCFrame(part)
    local ok, value = pcall(function()
        return part.CFrame
    end)
    return ok and value or nil
end

local function render3DBox(entry, root, color, distance)
    local rootCFrame = getCFrame(root)
    if not rootCFrame then
        return
    end

    local scale = clamp(1 - (distance / 10000), 0.65, 1)
    local halfWidth = 2 * scale
    local halfHeight = 3 * scale
    local halfDepth = 1.25 * scale
    local offsets = {
        Vector3.new(-halfWidth, halfHeight, -halfDepth),
        Vector3.new(halfWidth, halfHeight, -halfDepth),
        Vector3.new(halfWidth, -halfHeight, -halfDepth),
        Vector3.new(-halfWidth, -halfHeight, -halfDepth),
        Vector3.new(-halfWidth, halfHeight, halfDepth),
        Vector3.new(halfWidth, halfHeight, halfDepth),
        Vector3.new(halfWidth, -halfHeight, halfDepth),
        Vector3.new(-halfWidth, -halfHeight, halfDepth)
    }
    local edges = {
        {1, 2}, {2, 3}, {3, 4}, {4, 1},
        {5, 6}, {6, 7}, {7, 8}, {8, 5},
        {1, 5}, {2, 6}, {3, 7}, {4, 8}
    }
    local points = {}
    local visible = {}

    for index, offset in ipairs(offsets) do
        local worldPoint = rootCFrame * offset
        points[index], visible[index] = project(worldPoint)
    end

    for index, edge in ipairs(edges) do
        if points[edge[1]] and points[edge[2]] and visible[edge[1]] and visible[edge[2]] then
            drawLine(entry.Visuals.Box3D[index], points[edge[1]], points[edge[2]], color, 1, 1)
        end
    end
end

local function getEquippedItem(model)
    if not model then
        return nil
    end

    local ok, children = pcall(function()
        return model:GetChildren()
    end)
    if not ok then
        return nil
    end

    for _, child in ipairs(children) do
        if safeIsA(child, "Tool") then
            local name = nil
            pcall(function()
                name = child.Name
            end)
            if name and name ~= "" then
                return name
            end
        end
    end
    return nil
end

local function loadProfile(entry)
    if entry.Kind ~= "Player" or not asBoolean("matcha_profile_picture", false) then
        return
    end

    local profile = entry.Profile
    if profile.State == "loading" or profile.State == "ready" then
        return
    end
    if profile.State == "failed" and tick() < profile.RetryAt then
        return
    end

    local userId = nil
    pcall(function()
        userId = entry.Object.UserId
    end)
    if not userId then
        return
    end

    profile.State = "loading"
    task.spawn(function()
        task.wait(0)
        local ok, data = pcall(function()
            local metadata = httpget(
                "https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds="
                .. tostring(userId)
                .. "&size=150x150&format=Png&isCircular=false"
            )
            local decoded = HttpService:JSONDecode(metadata)
            local first = decoded and decoded.data and decoded.data[1]
            if not first or not first.imageUrl then
                error("thumbnail unavailable")
            end
            return httpget(first.imageUrl)
        end)

        if ok and data and #data > 0 then
            profile.Data = data
            profile.State = "ready"
        else
            profile.State = "failed"
            profile.RetryAt = tick() + 30
        end
    end)
end

local function renderOffscreenArrow(entry, cameraCFrame, cameraPosition, viewport, targetPosition, color)
    if not asBoolean("matcha_oof_arrow", false) then
        return
    end

    color = uiColor("matcha_oof_color", color)
    local relative = targetPosition - cameraPosition
    local horizontal = relative:Dot(cameraCFrame.RightVector)
    local vertical = relative:Dot(cameraCFrame.UpVector)
    local forward = relative:Dot(cameraCFrame.LookVector)

    if forward < 0 then
        horizontal = -horizontal
        vertical = -vertical
    end

    local magnitude = math.sqrt((horizontal * horizontal) + (vertical * vertical))
    if magnitude < 0.001 then
        horizontal = 0
        vertical = -1
        magnitude = 1
    end

    local directionX = horizontal / magnitude
    local directionY = -vertical / magnitude
    local radius = uiValue("matcha_oof_radius", 42) / 100
    local center = Vector2.new(viewport.X / 2, viewport.Y / 2)
    local edgeX = center.X + directionX * viewport.X * radius
    local edgeY = center.Y + directionY * viewport.Y * radius
    local size = uiValue("matcha_oof_size", 12)
    local perpendicularX = -directionY
    local perpendicularY = directionX
    local tip = Vector2.new(edgeX + directionX * size, edgeY + directionY * size)
    local base = Vector2.new(edgeX - directionX * size * 0.55, edgeY - directionY * size * 0.55)
    local left = Vector2.new(base.X + perpendicularX * size * 0.65, base.Y + perpendicularY * size * 0.65)
    local right = Vector2.new(base.X - perpendicularX * size * 0.65, base.Y - perpendicularY * size * 0.65)

    drawTriangle(entry.Visuals.OOFArrow, tip, left, right, color, 1)
end

local function renderRadarDot(entry, cameraCFrame, cameraPosition, viewport, targetPosition, color)
    if not asBoolean("matcha_radar", false) then
        return
    end

    local radius = uiValue("matcha_radar_size", 90) / 2
    local range = uiValue("matcha_radar_range", 500)
    local center = Vector2.new(24 + radius, viewport.Y - 24 - radius)
    local relative = targetPosition - cameraPosition
    local right = relative:Dot(cameraCFrame.RightVector)
    local forward = relative:Dot(cameraCFrame.LookVector)
    local scale = radius / math.max(range, 1)
    local x = clamp(right * scale, -radius + 4, radius - 4)
    local y = clamp(-forward * scale, -radius + 4, radius - 4)

    drawCircle(entry.Visuals.RadarDot, Vector2.new(center.X + x, center.Y + y), 3, true, color, 1, 1)
end

local function cameraInformation()
    local camera = nil
    pcall(function()
        camera = Workspace.CurrentCamera
    end)
    if not camera then
        return nil, nil, nil
    end

    local ok, position, viewport, cameraCFrame = pcall(function()
        return camera.CFrame.Position, camera.ViewportSize, camera.CFrame
    end)
    if not ok then
        return nil, nil, nil
    end
    return camera, position, viewport, cameraCFrame
end

local function renderEntry(entry, cameraPosition, viewport, cameraCFrame)
    hideVisuals(entry.Visuals)

    if not asBoolean("matcha_esp_enabled", true) then
        return
    end

    local isSelf = entry.Object == LocalPlayer
    if isSelf and not asBoolean("matcha_self_esp", false) then
        return
    end
    if entry.Kind == "Player" and not isSelf and not asBoolean("matcha_players", true) then
        return
    end
    if entry.Kind == "NPC" and not asBoolean("matcha_npcs", false) then
        return
    end

    local resolved = resolveEntry(entry)
    if not resolved then
        return
    end

    if entry.Kind == "Player" and not resolved.IsSelf and asBoolean("matcha_team_check", false) and resolved.Friendly then
        return
    end

    local rootPosition = getPosition(resolved.Root)
    local headPosition = getPosition(resolved.Head)
    if not rootPosition or not headPosition then
        return
    end

    local health, maximumHealth, healthRatio = getHealth(resolved.Humanoid)
    if resolved.Humanoid and asBoolean("matcha_alive_check", true) and (not health or health <= 0) then
        return
    end

    local okDistance, distance = pcall(function()
        return (cameraPosition - rootPosition).Magnitude
    end)
    if not okDistance then
        return
    end

    local maximumDistance = entry.RenderDistance or uiValue("matcha_max_distance", 3000)
    if distance > maximumDistance then
        return
    end

    local visible = targetIsVisible(entry, resolved, cameraPosition, headPosition)
    if not visible and asBoolean("matcha_hide_occluded", false) then
        return
    end

    local color = visible
        and uiColor("matcha_visible_color", Theme.Visible)
        or uiColor("matcha_occluded_color", Theme.Occluded)

    if entry.Kind == "NPC" then
        color = uiColor("matcha_npc_color", Theme.NPC)
    elseif resolved.Friendly and asBoolean("matcha_team_based_color", true) then
        color = uiColor("matcha_team_color", Theme.Friendly)
    end
    local teamColorOverride = resolved.Friendly and asBoolean("matcha_team_based_color", true)

    renderRadarDot(entry, cameraCFrame, cameraPosition, viewport, rootPosition, color)

    local top, topVisible = project(headPosition + Vector3.new(0, 0.65, 0))
    local bottom, bottomVisible = project(rootPosition - Vector3.new(0, 3, 0))
    local rootScreen, rootVisible = project(rootPosition)

    if not top or not bottom or not rootScreen or not (topVisible and bottomVisible and rootVisible) then
        renderOffscreenArrow(entry, cameraCFrame, cameraPosition, viewport, rootPosition, color)
        return
    end

    local height = math.max(math.abs(bottom.Y - top.Y), 8)
    local width = math.max(height * 0.55, 5)

    local sizingType = uiValue("matcha_sizing_type", 0)
    if sizingType == 1 then
        height = clamp(1200 / math.max(distance, 1), 18, 240)
        width = height * 0.55
    elseif sizingType == 2 then
        height = uiValue("matcha_fixed_size", 80)
        width = height * 0.55
    end

    local x = rootScreen.X - (width / 2)
    local y = sizingType == 0 and math.min(top.Y, bottom.Y) or (rootScreen.Y - height / 2)
    local outlines = asBoolean("matcha_outline", true)
    local glow = asBoolean("matcha_glow", false)

    if asBoolean("matcha_boxes", true) then
        local style = uiValue("matcha_box_style", 0)
        local boxColor = color
        if not teamColorOverride then
            boxColor = visible
                and uiColor("matcha_box_visible_color", color)
                or uiColor("matcha_box_occluded_color", color)
        end

        if asBoolean("matcha_box_fill", false) then
            local fillOpacity = uiValue("matcha_box_fill_opacity", 15) / 100
            drawSquare(
                entry.Visuals.BoxFill,
                Vector2.new(x, y),
                Vector2.new(width, height),
                true,
                boxColor,
                1,
                fillOpacity
            )
        end

        if glow and style ~= 2 then
            drawSquare(
                entry.Visuals.BoxGlow,
                Vector2.new(x - 2, y - 2),
                Vector2.new(width + 4, height + 4),
                false,
                boxColor,
                5,
                0.35
            )
        end

        if style == 1 then
            renderCorners(entry.Visuals.Corner, x, y, width, height, boxColor)
        elseif style == 2 then
            render3DBox(entry, resolved.Root, boxColor, distance)
        else
            if outlines then
                drawSquare(
                    entry.Visuals.BoxOutline,
                    Vector2.new(x - 1, y - 1),
                    Vector2.new(width + 2, height + 2),
                    false,
                    Theme.Outline,
                    3,
                    1
                )
            end
            drawSquare(
                entry.Visuals.Box,
                Vector2.new(x, y),
                Vector2.new(width, height),
                false,
                boxColor,
                1,
                1
            )
        end
    end

    if healthRatio then
        local onRight = uiValue("matcha_health_position", 0) == 1
        local barX = onRight and (x + width + 4) or (x - 7)
        local fillHeight = math.max(height * healthRatio, 1)
        local barColor = asBoolean("matcha_health_based", true)
            and healthColor(healthRatio)
            or uiColor("matcha_health_color", Theme.Health)

        if asBoolean("matcha_health", true) then
            drawSquare(
                entry.Visuals.HealthBackground,
                Vector2.new(barX - 1, y - 1),
                Vector2.new(5, height + 2),
                true,
                Theme.Outline,
                1,
                1
            )
            drawSquare(
                entry.Visuals.HealthFill,
                Vector2.new(barX, y + height - fillHeight),
                Vector2.new(3, fillHeight),
                true,
                barColor,
                1,
                1
            )
        end

        if asBoolean("matcha_show_health_text", true) then
            local healthString = tostring(math.floor(health + 0.5)) .. " HP"
            local textPosition = uiValue("matcha_health_text_position", 0)
            local point = Vector2.new(barX + 2, y + height / 2 - 6)
            if textPosition == 1 then
                point = Vector2.new(rootScreen.X, y - 26)
            elseif textPosition == 2 then
                point = Vector2.new(rootScreen.X, y + height + 4)
            end
            drawText(entry.Visuals.HealthText, point, healthString, barColor, 11, outlines)
        end
    end

    if asBoolean("matcha_tracers", true) then
        local origin = nil
        local tracerOrigin = uiValue("matcha_tracer_origin", 0)
        if tracerOrigin == 1 then
            origin = Vector2.new(viewport.X / 2, viewport.Y / 2)
        elseif tracerOrigin == 2 then
            origin = Vector2.new(viewport.X / 2, 2)
        else
            origin = Vector2.new(viewport.X / 2, viewport.Y - 2)
        end

        local tracerColor = color
        if not teamColorOverride then
            tracerColor = visible
                and uiColor("matcha_tracer_visible_color", color)
                or uiColor("matcha_tracer_occluded_color", color)
        end
        if outlines then
            drawLine(entry.Visuals.TracerOutline, origin, rootScreen, Theme.Outline, glow and 5 or 3, glow and 0.45 or 1)
        end
        drawLine(entry.Visuals.Tracer, origin, rootScreen, tracerColor, 1, 1)
    end

    if asBoolean("matcha_head_dot", false) then
        local headScreen, headVisible = project(headPosition)
        if headScreen and headVisible then
            local radius = clamp(width * 0.13, 2, 10)
            local headColor = teamColorOverride and color or uiColor("matcha_head_color", color)
            if asBoolean("matcha_head_glow", false) then
                drawCircle(entry.Visuals.HeadGlow, headScreen, radius + 3, false, headColor, 5, 0.35)
            end
            if outlines then
                drawCircle(entry.Visuals.HeadOutline, headScreen, radius + 1, false, Theme.Outline, 3, 1)
            end
            drawCircle(entry.Visuals.Head, headScreen, radius, false, headColor, 1, 1)
        end
    end

    if asBoolean("matcha_skeleton", false) then
        renderSkeleton(
            entry,
            resolved.Model,
            teamColorOverride and color or uiColor("matcha_skeleton_color", color)
        )
    end

    if asBoolean("matcha_chams", false) then
        local chamsColor = color
        if not teamColorOverride then
            chamsColor = visible
                and uiColor("matcha_chams_visible_color", color)
                or uiColor("matcha_chams_occluded_color", color)
        end
        if uiValue("matcha_chams_rendering", 0) == 1 then
            chamsColor = Color3.fromHSV((tick() % 3) / 3, 0.8, 1)
        end
        renderChams(entry, resolved, chamsColor)
    end

    if asBoolean("matcha_text", true) then
        local labels = {}

        if asBoolean("matcha_show_name", true) then
            local nameType = uiValue("matcha_name_type", 0)
            if nameType == 1 then
                labels[#labels + 1] = tostring(resolved.Username)
            elseif nameType == 2 and resolved.DisplayName ~= resolved.Username then
                labels[#labels + 1] = tostring(resolved.DisplayName) .. " (@" .. tostring(resolved.Username) .. ")"
            else
                labels[#labels + 1] = tostring(resolved.DisplayName)
            end
        end
        if asBoolean("matcha_show_distance", true) then
            labels[#labels + 1] = tostring(math.floor(distance + 0.5)) .. "m"
        end
        if asBoolean("matcha_equipped_item", false) then
            local item = getEquippedItem(resolved.Model)
            if item then
                labels[#labels + 1] = item
            end
        end

        if #labels > 0 then
            local fontSize = clamp(math.floor(18 - (distance / 250)), 10, 16)
            local label = table.concat(labels, " | ")
            local labelColor = teamColorOverride and color or uiColor("matcha_text_color", color)
            if asBoolean("matcha_text_gradient", false) then
                labelColor = Color3.fromHSV((tick() % 3) / 3, 0.8, 1)
            end
            local labelPosition = Vector2.new(rootScreen.X, y - fontSize - 2)

            if asBoolean("matcha_text_background", false) then
                local estimatedWidth = math.max(#label * fontSize * 0.52, 20)
                drawSquare(
                    entry.Visuals.LabelBackground,
                    Vector2.new(labelPosition.X - estimatedWidth / 2 - 3, labelPosition.Y - 2),
                    Vector2.new(estimatedWidth + 6, fontSize + 5),
                    true,
                    Theme.Outline,
                    1,
                    0.65
                )
            end

            if glow then
                drawText(entry.Visuals.LabelGlow, labelPosition, label, labelColor, fontSize + 2, false)
                pcall(function()
                    entry.Visuals.LabelGlow.Transparency = 0.3
                end)
            end
            drawText(
                entry.Visuals.Label,
                labelPosition,
                label,
                labelColor,
                fontSize,
                outlines
            )
        end
    end

    if entry.Kind == "Player" and asBoolean("matcha_profile_picture", false) then
        loadProfile(entry)
        if entry.Profile.State == "ready" and entry.Profile.Data then
            local imageSize = clamp(height * 0.25, 24, 42)
            drawImage(
                entry.Visuals.Profile,
                Vector2.new(x + width + 9, y),
                Vector2.new(imageSize, imageSize),
                entry.Profile.Data,
                1
            )
        end
    end
end

local crosshair = {
    All = {},
    Outline = {},
    Main = {},
    DotOutline = nil,
    Dot = nil
}

local function addCrosshairDrawing(kind)
    local object = newDrawing(kind)
    if object then
        crosshair.All[#crosshair.All + 1] = object
    end
    return object
end

for index = 1, 4 do
    crosshair.Outline[index] = addCrosshairDrawing("Line")
    crosshair.Main[index] = addCrosshairDrawing("Line")
end
crosshair.DotOutline = addCrosshairDrawing("Circle")
crosshair.Dot = addCrosshairDrawing("Circle")

local function hideCrosshair()
    for _, object in ipairs(crosshair.All) do
        setVisible(object, false)
    end
end

local function renderCrosshair(viewport)
    hideCrosshair()
    if not asBoolean("matcha_crosshair", true) then
        return
    end

    local center = Vector2.new(viewport.X / 2, viewport.Y / 2)
    local size = uiValue("matcha_crosshair_size", 10)
    local gap = uiValue("matcha_crosshair_gap", 5)
    if asBoolean("matcha_crosshair_pulse", false) then
        local pulseSpeed = uiValue("matcha_crosshair_pulse_speed", 5)
        gap = gap + ((math.sin(tick() * pulseSpeed) + 1) * 2)
    end

    local color = uiColor("matcha_crosshair_color", Theme.Crosshair)
    local outlineColor = uiColor("matcha_crosshair_outline_color", Theme.Outline)
    local angle = math.rad(uiValue("matcha_crosshair_rotation", 0))
    if asBoolean("matcha_crosshair_rotate", false) then
        angle = angle + tick() * uiValue("matcha_crosshair_rotation_speed", 60) * math.pi / 180
    end

    local function rotatePoint(x, y)
        local cosine = math.cos(angle)
        local sine = math.sin(angle)
        return Vector2.new(
            center.X + x * cosine - y * sine,
            center.Y + x * sine + y * cosine
        )
    end

    local segments = {
        {rotatePoint(-gap - size, 0), rotatePoint(-gap, 0)},
        {rotatePoint(gap, 0), rotatePoint(gap + size, 0)},
        {rotatePoint(0, -gap - size), rotatePoint(0, -gap)},
        {rotatePoint(0, gap), rotatePoint(0, gap + size)}
    }

    for index, segment in ipairs(segments) do
        local hiddenByTStyle = index == 3 and asBoolean("matcha_crosshair_tstyle", false)
        if not hiddenByTStyle then
            if asBoolean("matcha_crosshair_outline", true) then
                drawLine(crosshair.Outline[index], segment[1], segment[2], outlineColor, 3, 1)
            end
            drawLine(crosshair.Main[index], segment[1], segment[2], color, 1, 1)
        end
    end

    if asBoolean("matcha_crosshair_dot", true) then
        if asBoolean("matcha_crosshair_outline", true) then
            drawCircle(crosshair.DotOutline, center, 3, true, outlineColor, 1, 1)
        end
        drawCircle(crosshair.Dot, center, 2, true, color, 1, 1)
    end
end

local radar = {
    All = {},
    Background = nil,
    Border = nil,
    Horizontal = nil,
    Vertical = nil,
    Center = nil
}

local function addRadarDrawing(kind)
    local object = newDrawing(kind)
    if object then
        radar.All[#radar.All + 1] = object
    end
    return object
end

radar.Background = addRadarDrawing("Circle")
radar.Border = addRadarDrawing("Circle")
radar.Horizontal = addRadarDrawing("Line")
radar.Vertical = addRadarDrawing("Line")
radar.Center = addRadarDrawing("Circle")

local function hideRadar()
    for _, object in ipairs(radar.All) do
        setVisible(object, false)
    end
end

local function renderRadar(viewport)
    hideRadar()
    if not asBoolean("matcha_radar", false) then
        return
    end

    local size = uiValue("matcha_radar_size", 90)
    local radius = size / 2
    local center = Vector2.new(24 + radius, viewport.Y - 24 - radius)
    local foreground = uiColor("matcha_radar_color", Color3.fromRGB(180, 180, 195))

    drawCircle(radar.Background, center, radius, true, Theme.Radar, 1, 0.7)
    drawCircle(radar.Border, center, radius, false, foreground, 1, 1)
    drawLine(
        radar.Horizontal,
        Vector2.new(center.X - radius, center.Y),
        Vector2.new(center.X + radius, center.Y),
        foreground,
        1,
        0.45
    )
    drawLine(
        radar.Vertical,
        Vector2.new(center.X, center.Y - radius),
        Vector2.new(center.X, center.Y + radius),
        foreground,
        1,
        0.45
    )
    drawCircle(radar.Center, center, 3, true, Theme.Crosshair, 1, 1)
end

local uiFailures = {}

local function reportUIFailure(key, message)
    if uiFailures[key] then
        return
    end

    uiFailures[key] = true
    warn("IRONMETAL ESP+ UNIVERSAL UI > " .. key .. " failed: " .. tostring(message))
end

local function safeSection(tab, key, name, side, pages, maxHeight)
    local section = nil
    local ok, message = pcall(function()
        if pages then
            section = tab:Section(name, side, pages, maxHeight)
        else
            section = tab:Section(name, side)
        end
    end)

    if not ok then
        reportUIFailure("section." .. key, message)
    end
    return section
end

local function safeWidget(section, method, ...)
    if not section then
        return nil
    end

    local arguments = {...}
    local key = method .. "." .. tostring(arguments[1])

    -- Widget IDs are versioned to avoid collisions with stale Matcha UI state.
    if method == "ColorPicker2" then
        arguments[1] = UI_PREFIX .. tostring(arguments[1])
        arguments[3] = UI_PREFIX .. tostring(arguments[3])
    elseif method ~= "Text" and method ~= "Tip" and method ~= "Spacing" and method ~= "Button" then
        arguments[1] = UI_PREFIX .. tostring(arguments[1])
    end

    local widget = nil
    local ok, message = pcall(function()
        local callback = section[method]
        widget = callback(section, unpack(arguments))
    end)

    if not ok then
        reportUIFailure(key, message)
    end
    return widget
end

local function sectionPage(section)
    local page = 0
    pcall(function()
        page = section.page
    end)
    return page or 0
end


UI.AddTab(UI_TAB_NAME, function(tab)
    local left = safeSection(
        tab,
        "esp_left",
        "ESP",
        "Left",
        {"Main", "Box", "Text", "Indicators"},
        520
    )
    local leftPage = sectionPage(left)

    if leftPage == 0 then
        safeWidget(left, "Toggle", "matcha_esp_enabled", "Enabled", true)
        safeWidget(left, "Keybind", "matcha_esp_key", 0, "toggle")
        safeWidget(left, "Toggle", "matcha_players", "Players", true)
        safeWidget(left, "Toggle", "matcha_npcs", "NPCs", false)
        safeWidget(left, "ColorPicker", "matcha_npc_color", 1, 0.69, 0.27, 1)
        safeWidget(left, "Toggle", "matcha_alive_check", "Alive Check", true)
        safeWidget(left, "Toggle", "matcha_team_check", "Team Check", false)
        safeWidget(left, "Toggle", "matcha_visibility_check", "Visible Check", true)
        safeWidget(
            left,
            "ColorPicker2",
            "matcha_visible_color",
            {0.31, 1, 0.47, 1},
            "matcha_occluded_color",
            {1, 0.35, 0.35, 1}
        )
        safeWidget(left, "Toggle", "matcha_hide_occluded", "Show Only Visible", false)
        safeWidget(left, "Toggle", "matcha_team_based_color", "Team Based Color", true)
        safeWidget(left, "ColorPicker", "matcha_team_color", 0.35, 0.75, 1, 1)
        safeWidget(left, "Toggle", "matcha_outline", "Outline", true)
        safeWidget(left, "Toggle", "matcha_glow", "Glow", false)
        safeWidget(left, "Toggle", "matcha_self_esp", "Self ESP", false)
        safeWidget(left, "Combo", "matcha_sizing_type", "Sizing Type", {"Bounding", "Distance", "Fixed"}, 0)
        safeWidget(left, "SliderInt", "matcha_fixed_size", "Fixed Size", 20, 220, 80)
        safeWidget(left, "SliderInt", "matcha_max_distance", "Render Distance", 100, 10000, 3000)
        safeWidget(left, "SliderInt", "matcha_visibility_rate", "Visible Check Rate", 25, 500, 100)
    elseif leftPage == 1 then
        safeWidget(left, "Toggle", "matcha_boxes", "Enabled", true)
        safeWidget(
            left,
            "ColorPicker2",
            "matcha_box_visible_color",
            {1, 0.45, 0.75, 1},
            "matcha_box_occluded_color",
            {0.35, 0.35, 0.4, 1}
        )
        safeWidget(left, "Toggle", "matcha_box_fill", "Fill Box", false)
        safeWidget(left, "SliderInt", "matcha_box_fill_opacity", "Fill Opacity", 0, 100, 15)
        safeWidget(left, "Combo", "matcha_box_style", "Box Type", {"2D", "Corner", "3D"}, 0)
    elseif leftPage == 2 then
        safeWidget(left, "Toggle", "matcha_text", "Text Enabled", true)
        safeWidget(left, "ColorPicker", "matcha_text_color", 0.9, 0.7, 0.82, 1)
        safeWidget(left, "Toggle", "matcha_show_name", "Name", true)
        safeWidget(left, "Combo", "matcha_name_type", "Name Type", {"Display Name", "Username", "Both"}, 0)
        safeWidget(left, "Toggle", "matcha_show_distance", "Distance", true)
        safeWidget(left, "Toggle", "matcha_equipped_item", "Equipped Item", false)
        safeWidget(left, "Toggle", "matcha_text_gradient", "Text Gradient (Animated)", false)
        safeWidget(left, "Toggle", "matcha_text_background", "Text Background", false)
    else
        safeWidget(left, "Toggle", "matcha_skeleton", "Skeleton", false)
        safeWidget(left, "ColorPicker", "matcha_skeleton_color", 0.9, 0.25, 0.55, 1)
        safeWidget(left, "Toggle", "matcha_head_dot", "Head Dot", false)
        safeWidget(left, "ColorPicker", "matcha_head_color", 1, 1, 1, 1)
        safeWidget(left, "Toggle", "matcha_head_glow", "Head Dot Glow", false)
        safeWidget(left, "Toggle", "matcha_profile_picture", "Profile Picture", false)
    end

    local right = safeSection(
        tab,
        "esp_right",
        "Features",
        "Right",
        {"Health", "Chams", "Tracer", "Radar", "Crosshair"},
        520
    )
    local rightPage = sectionPage(right)

    if rightPage == 0 then
        safeWidget(right, "Toggle", "matcha_health", "Health Bar", true)
        safeWidget(right, "ColorPicker", "matcha_health_color", 0.31, 1, 0.47, 1)
        safeWidget(right, "Toggle", "matcha_health_based", "Health Based", true)
        safeWidget(right, "Toggle", "matcha_show_health_text", "Health Text", true)
        safeWidget(
            right,
            "Combo",
            "matcha_health_text_position",
            "Text Position",
            {"On Healthbar", "Above", "Below"},
            0
        )
        safeWidget(right, "Combo", "matcha_health_position", "Bar Position", {"Left", "Right"}, 0)
    elseif rightPage == 1 then
        safeWidget(right, "Toggle", "matcha_chams", "Enabled", false)
        safeWidget(
            right,
            "ColorPicker2",
            "matcha_chams_visible_color",
            {0.75, 0.75, 0.75, 1},
            "matcha_chams_occluded_color",
            {0.65, 0.3, 0.5, 1}
        )
        safeWidget(right, "Toggle", "matcha_chams_filled", "Filled", false)
        safeWidget(right, "Combo", "matcha_chams_mode", "Mode", {"Default", "Outline"}, 0)
        safeWidget(right, "Combo", "matcha_chams_rendering", "Rendering Type", {"Static", "Rainbow"}, 0)
        safeWidget(right, "Tip", "Projected Drawing chams; not engine Highlights")
    elseif rightPage == 2 then
        safeWidget(right, "Toggle", "matcha_tracers", "Enabled", true)
        safeWidget(
            right,
            "ColorPicker2",
            "matcha_tracer_visible_color",
            {1, 1, 1, 1},
            "matcha_tracer_occluded_color",
            {1, 0.35, 0.35, 1}
        )
        safeWidget(right, "Combo", "matcha_tracer_origin", "Origin", {"Bottom", "Center", "Top"}, 0)
    elseif rightPage == 3 then
        safeWidget(right, "Toggle", "matcha_oof_arrow", "Out-of-FOV Arrow", false)
        safeWidget(right, "ColorPicker", "matcha_oof_color", 1, 0.5, 0.75, 1)
        safeWidget(right, "SliderInt", "matcha_oof_size", "Arrow Size", 5, 30, 12)
        safeWidget(right, "SliderInt", "matcha_oof_radius", "Arrow Radius", 20, 48, 42)
        safeWidget(right, "Toggle", "matcha_radar", "Radar", false)
        safeWidget(right, "ColorPicker", "matcha_radar_color", 0.7, 0.7, 0.8, 1)
        safeWidget(right, "SliderInt", "matcha_radar_size", "Radar Size", 60, 220, 90)
        safeWidget(right, "SliderInt", "matcha_radar_range", "Radar Range", 50, 3000, 500)
    else
        safeWidget(right, "Toggle", "matcha_crosshair", "Enabled", true)
        safeWidget(right, "ColorPicker", "matcha_crosshair_color", 0.31, 1, 0.47, 1)
        safeWidget(right, "Toggle", "matcha_crosshair_outline", "Outline", true)
        safeWidget(right, "ColorPicker", "matcha_crosshair_outline_color", 0, 0, 0, 1)
        safeWidget(right, "Toggle", "matcha_crosshair_dot", "Center Dot", true)
        safeWidget(right, "Toggle", "matcha_crosshair_tstyle", "T Style", false)
        safeWidget(right, "Toggle", "matcha_crosshair_rotate", "Auto Rotate", false)
        safeWidget(right, "Toggle", "matcha_crosshair_pulse", "Pulse Gap", false)
        safeWidget(right, "SliderInt", "matcha_crosshair_size", "Size", 2, 30, 10)
        safeWidget(right, "SliderInt", "matcha_crosshair_gap", "Gap", 0, 20, 5)
        safeWidget(right, "SliderInt", "matcha_crosshair_rotation", "Rotation", 0, 360, 0)
        safeWidget(right, "SliderInt", "matcha_crosshair_rotation_speed", "Rotation Speed", 1, 360, 60)
        safeWidget(right, "SliderInt", "matcha_crosshair_pulse_speed", "Pulse Speed", 1, 20, 5)
    end
end)

local function syncPlayers()
    local active = {}
    local includeSelf = asBoolean("matcha_self_esp", false)

    for _, player in ipairs(playerList()) do
        if player ~= LocalPlayer or includeSelf then
            active[player] = true
            if not targets[player] then
                addTarget(player, "Player", nil, nil, false)
            end
        end
    end

    local stale = {}
    for object, entry in pairs(targets) do
        if entry.Kind == "Player" and not entry.Manual and not active[object] then
            stale[#stale + 1] = object
        end
    end
    for _, object in ipairs(stale) do
        removeTarget(object)
    end
end

local function syncNPCs()
    if not asBoolean("matcha_npcs", false) then
        return
    end

    local ok, descendants = pcall(function()
        return Workspace:GetDescendants()
    end)
    if not ok then
        return
    end

    local seen = {}
    local count = 0
    local maximumNPCs = 64

    for _, object in ipairs(descendants) do
        if count >= maximumNPCs then
            break
        end

        if safeIsA(object, "Humanoid") then
            local model = safeParent(object)
            if model and not seen[model] and not isPlayerCharacter(model) then
                local root = safeChild(model, "HumanoidRootPart") or safeChild(model, "Torso") or safeChild(model, "UpperTorso")
                local head = safeChild(model, "Head")

                if root and head then
                    seen[model] = true
                    count = count + 1
                    if not targets[model] then
                        addTarget(model, "NPC", nil, nil, false)
                    end
                end
            end
        end
    end

    local stale = {}
    for object, entry in pairs(targets) do
        if entry.Kind == "NPC" and not entry.Manual and not seen[object] then
            stale[#stale + 1] = object
        end
    end
    for _, object in ipairs(stale) do
        removeTarget(object)
    end
end

task.spawn(function()
    while running do
        pcall(syncPlayers)
        task.wait(1)
    end
end)

task.spawn(function()
    while running do
        pcall(syncNPCs)
        task.wait(2)
    end
end)

renderConnection = RunService.RenderStepped:Connect(function()
    if not running then
        return
    end

    local _, cameraPosition, viewport, cameraCFrame = cameraInformation()
    if not cameraPosition or not viewport or not cameraCFrame then
        return
    end

    pcall(renderRadar, viewport)
    for _, entry in pairs(targets) do
        pcall(renderEntry, entry, cameraPosition, viewport, cameraCFrame)
    end
    pcall(renderCrosshair, viewport)
end)

-- Public helpers retained in a Matcha-safe form. Matcha drops top-level return
-- values, so the API is intentionally published as the global MatchaESP.
function API:WrapObject(object, pseudoName, renderDistance)
    if not object then
        return nil
    end

    local kind = "Part"
    if safeIsA(object, "Player") then
        kind = "Player"
    elseif safeIsA(object, "Model") then
        kind = "Model"
    end

    return addTarget(object, kind, pseudoName, renderDistance, true)
end

function API:UnwrapObject(object)
    return removeTarget(object)
end

function API:GetEntry(object)
    return targets[object]
end

function API:Exit()
    if not running then
        return true
    end

    running = false

    pcall(function()
        UI.RemoveTab(UI_TAB_NAME)
    end)

    if renderConnection then
        pcall(function()
            renderConnection:Disconnect()
        end)
        renderConnection = nil
    end

    local remove = {}
    for object, _ in pairs(targets) do
        remove[#remove + 1] = object
    end
    for _, object in ipairs(remove) do
        removeTarget(object)
    end

    for _, object in ipairs(crosshair.All) do
        removeDrawing(object)
    end
    crosshair.All = {}

    for _, object in ipairs(radar.All) do
        removeDrawing(object)
    end
    radar.All = {}

    return true
end

API.Theme = Theme
API.Targets = targets
MatchaESP = API

pcall(function()
    notify("Loaded", "IRONMETAL ESP+ UNIVERSAL", 4)
end)

