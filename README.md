-- Strong Aim-Assist (LocalScript)
-- للاستخدام داخل لعبتك الخاصة فقط
-- مميزات: FOV, Aim strength, Prediction (lead), smoothing, team check, tool check, toggle

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- ========== CONFIG ==========
local ENABLED_BY_DEFAULT = true
local TOGGLE_KEY = Enum.KeyCode.K
local AIM_HOLD_BUTTON = Enum.UserInputType.MouseButton2 -- يمين الفأرة
local AIM_STRENGTH = 0.85      -- 0..1  (1 = أقوى تأثير، 0 = لا تأثير)
local AIM_SMOOTH = 0.18        -- 0..1  (0 = فوري, 1 = لا تغيير). أصغر => أسرع التتبع
local AIM_FOV = 60             -- درجات FOV
local MAX_TARGET_DISTANCE = 250 -- أقصى مسافة (متر) للبحث عن الأهداف
local PREDICTION_FACTOR = 0.22 -- مضروب لسرعة الهدف لتقدير موضعه (جرب تغييره)
local TEAM_SAFE = true         -- لو لعبتك فيها Teams، لا يستهدف نفس الفريق
local REQUIRE_TOOL = false     -- لو تريد تقييد العمل فقط أثناء حمل سلاح
local ALLOWED_TOOL_NAMES = nil -- أو جدول بالأسماء {"Rifle", "Pistol"}, nil = أي Tool مقبول
-- ============================

local aimEnabled = ENABLED_BY_DEFAULT
local rightHeld = false

-- Helper: team check (قابل للتعديل بحسب نظام لعبتك)
local Teams = game:GetService("Teams")
local function isEnemy(player)
    if not player or not player.Character then return false end
    if player == localPlayer then return false end
    if TEAM_SAFE then
        if player.Team ~= localPlayer.Team then
            return true
        else
            return false
        end
    end
    return true -- لو TEAM_SAFE false اعتبره عدو
end

-- Helper: requires holding tool?
local function isHoldingAllowedTool()
    if not REQUIRE_TOOL then return true end
    local char = localPlayer.Character
    if not char then return false end
    local tool = char:FindFirstChildOfClass("Tool")
    if not tool then return false end
    if ALLOWED_TOOL_NAMES == nil then return true end
    for _, name in ipairs(ALLOWED_TOOL_NAMES) do
        if tool.Name == name then return true end
    end
    return false
end

-- Get candidate targets: returns table of {player, part, predictedPos, angle}
local function getCandidates()
    local results = {}
    local camCFrame = camera.CFrame
    local camPos = camCFrame.Position
    local forward = camCFrame.LookVector

    for _, player in pairs(Players:GetPlayers()) do
        if isEnemy(player) and player.Character and player.Character.PrimaryPart then
            local root = player.Character.PrimaryPart
            local dist = (root.Position - camPos).Magnitude
            if dist <= MAX_TARGET_DISTANCE then
                -- choose aim point: Head if exists else HumanoidRootPart
                local targetPart = player.Character:FindFirstChild("Head") or player.Character.PrimaryPart
                if targetPart then
                    -- Predict position using velocity (simple linear prediction)
                    local vel = Vector3.new(0,0,0)
                    if targetPart:IsA("BasePart") then
                        vel = targetPart.Velocity
                    else
                        local hrp = player.Character:FindFirstChild("HumanoidRootPart")
                        if hrp then vel = hrp.Velocity end
                    end
                    local lead = vel * PREDICTION_FACTOR
                    local predictedPos = targetPart.Position + lead

                    local dir = (predictedPos - camPos).Unit
                    local dot = math.clamp(forward:Dot(dir), -1, 1)
                    local angle = math.deg(math.acos(dot))

                    -- Raycast visibility check (ignore local character)
                    local rayParams = RaycastParams.new()
                    rayParams.FilterDescendantsInstances = {localPlayer.Character}
                    rayParams.FilterType = Enum.RaycastFilterType.Blacklist
                    local r = workspace:Raycast(camPos, predictedPos - camPos, rayParams)
                    local blocked = false
                    if r and r.Instance then
                        if not r.Instance:IsDescendantOf(player.Character) then
                            blocked = true
                        end
                    end

                    if not blocked and angle <= (AIM_FOV/2) then
                        table.insert(results, {
                            player = player,
                            part = targetPart,
                            predictedPos = predictedPos,
                            angle = angle,
                            distance = dist
                        })
                    end
                end
            end
        end
    end

    return results
end

-- choose best target by closest angle then distance
local function chooseBestTarget(candidates)
    if #candidates == 0 then return nil end
    table.sort(candidates, function(a,b)
        if a.angle == b.angle then
            return a.distance < b.distance
        end
        return a.angle < b.angle
    end)
    return candidates[1]
end

-- Input handling
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == AIM_HOLD_BUTTON then
        rightHeld = true
    elseif input.KeyCode == TOGGLE_KEY then
        aimEnabled = not aimEnabled
        -- optional: show a message to player using StarterGui:SetCore or a Billboard
        -- game.StarterGui:SetCore("SendNotification", {Title="Aim Assist", Text = aimEnabled and "Enabled" or "Disabled", Duration = 2})
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == AIM_HOLD_BUTTON then
        rightHeld = false
    end
end)

-- Main loop
RunService.RenderStepped:Connect(function(dt)
    -- only run if enabled + right held + holding tool (if required)
    if not aimEnabled or not rightHeld then return end
    if not isHoldingAllowedTool() then return end

    local candidates = getCandidates()
    local best = chooseBestTarget(candidates)
    if not best then return end

    -- compute desired camera CFrame looking to predictedPos
    local camPos = camera.CFrame.Position
    local desired = CFrame.new(camPos, best.predictedPos)

    -- compute interpolation factor: mix between current and desired based on AIM_STRENGTH and AIM_SMOOTH
    -- stronger AIM_STRENGTH => camera moves closer to desired; AIM_SMOOTH controls speed
    -- finalLerp roughly = 1 - (1 - AIM_STRENGTH) * (1 - AIM_SMOOTH)  -- adjust to feel good
    local finalLerp = math.clamp(AIM_STRENGTH, 0, 1)
    -- apply smoothing over time:
    local smoothFactor = 1 - math.exp(-AIM_SMOOTH * (dt * 60)) -- framerate-independent feel
    local lerpAmount = smoothFactor * finalLerp

    camera.CFrame = camera.CFrame:Lerp(desired, lerpAmount)
end)

-- Cleanup: optional on character respawn remove things (none created here)
print("Strong Aim-Assist script loaded (for use in your own game).")
