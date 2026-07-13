-- --- lightning_farm.lua ---
-- Auto-farm Lightning Fruit | Blox Fruits
-- Requires: Synapse X / KRNL / Fluxus

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

-- ═══════════════════════════════════════
--           CONFIG
-- ═══════════════════════════════════════
local CONFIG = {
    FruitName = "Lightning",
    TeleportToMob = true,
    AutoAttack = true,
    AutoCollectFruit = true,
    AttackRange = 40,
    LoopDelay = 0.1,
    MobArea = Vector3.new(0, 0, 0), -- غير الكوردينات حسب المنطقة
}

-- ═══════════════════════════════════════
--           UTILITY FUNCTIONS
-- ═══════════════════════════════════════
local function getClosestMob()
    local closest = nil
    local minDist = math.huge

    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("Model") and obj:FindFirstChild("Humanoid") then
            local mob = obj:FindFirstChild("HumanoidRootPart")
            if mob and obj:FindFirstChild("Humanoid").Health > 0 then
                local dist = (rootPart.Position - mob.Position).Magnitude
                if dist < minDist then
                    minDist = dist
                    closest = obj
                end
            end
        end
    end

    return closest, minDist
end

local function teleportTo(position)
    rootPart.CFrame = CFrame.new(position + Vector3.new(0, 3, 0))
end

local function attackMob(mob)
    local mobRoot = mob:FindFirstChild("HumanoidRootPart")
    if not mobRoot then return end

    teleportTo(mobRoot.Position + Vector3.new(5, 0, 0))

    -- Simulate attack via tool activation
    local tool = character:FindFirstChildOfClass("Tool")
    if tool and tool:FindFirstChild("RemoteEvent") then
        tool.RemoteEvent:FireServer()
    end

    -- Fallback: use equipped skills
    local args = {
        [1] = mob,
        [2] = mobRoot.Position
    }

    pcall(function()
        game:GetService("ReplicatedStorage"):FindFirstChild("REvent"):FireServer(unpack(args))
    end)
end

local function collectFruit()
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj.Name:find(CONFIG.FruitName) or
           (obj:IsA("Model") and obj.Name:find("Fruit")) then
            local part = obj:FindFirstChildOfClass("BasePart")
            if part then
                local dist = (rootPart.Position - part.Position).Magnitude
                if dist < 60 then
                    teleportTo(part.Position)
                    task.wait(0.3)

                    -- Touch trigger for pickup
                    local touchEvent = obj:FindFirstChild("Collect") or
                                       obj:FindFirstChild("Touch")
                    if touchEvent then
                        pcall(function() touchEvent:FireServer() end)
                    end
                end
            end
        end
    end
end

local function antiLag()
    -- تقليل rendering لرفع الأداء
    settings().Rendering.QualityLevel = 1
    game:GetService("Lighting").GlobalShadows = false
    game:GetService("Lighting").FogEnd = 9e9

    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("ParticleEmitter") or
           obj:IsA("Trail") or
           obj:IsA("Beam") then
            obj.Enabled = false
        end
    end
end

-- ═══════════════════════════════════════
--           RESPAWN HANDLER
-- ═══════════════════════════════════════
player.CharacterAdded:Connect(function(newChar)
    character = newChar
    humanoid = newChar:WaitForChild("Humanoid")
    rootPart = newChar:WaitForChild("HumanoidRootPart")
    task.wait(2)
    antiLag()
end)

-- ═══════════════════════════════════════
--           MAIN LOOP
-- ═══════════════════════════════════════
antiLag()
print("[Lightning Farm] Started — Anti-lag active")

while task.wait(CONFIG.LoopDelay) do
    if not character or not humanoid or humanoid.Health <= 0 then
        task.wait(1)
        continue
    end

    -- اصطياد الأعداء
    if CONFIG.AutoAttack then
        local mob, dist = getClosestMob()
        if mob and dist <= CONFIG.AttackRange then
            attackMob(mob)
        elseif mob and CONFIG.TeleportToMob then
            local mobRoot = mob:FindFirstChild("HumanoidRootPart")
            if mobRoot then
                teleportTo(mobRoot.Position)
            end
        end
    end

    -- جمع الفاكهة
    if CONFIG.AutoCollectFruit then
        collectFruit()
    end
end
