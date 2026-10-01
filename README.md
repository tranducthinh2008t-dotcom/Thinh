local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local VirtualUser = game:GetService("VirtualUser")
local GuiService = game:GetService("GuiService")
local TeleportService = game:GetService("TeleportService")
local lp = Players.LocalPlayer

-- ===== CẤU HÌNH =====
local FPS_CAP = 12
local DISABLE_3D = true        -- tự bật lại 3D nếu cap không ăn
local SHOW_FPS = true          -- test xong đổi thành false
local MUTE_SOUND = true
local LIGHTEN_MAP = true
local AUTO_REJOIN = true
local LISTEN_SECONDS = 60

task.wait(math.random() * 8)   -- lệch giờ giữa các acc

-- ===== KHÓA FPS =====
local function applyCap() pcall(function() setfpscap(FPS_CAP) end) end
applyCap()
task.delay(15, applyCap)
task.spawn(function()
    while task.wait(30) do applyCap() end
end)

pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
Lighting.GlobalShadows = false
Lighting.FogEnd = 9e9

-- ===== TẮT 3D (tự kiểm tra) =====
if DISABLE_3D then
    pcall(function() RunService:Set3dRenderingEnabled(false) end)
    task.delay(20, function()
        local n, t0 = 0, os.clock()
        local c = RunService.Heartbeat:Connect(function() n += 1 end)
        task.wait(3)
        c:Disconnect()
        local fps = n / (os.clock() - t0)
        if fps > FPS_CAP + 10 then
            pcall(function() RunService:Set3dRenderingEnabled(true) end)
            warn("Cap không ăn khi tắt 3D (" .. math.floor(fps) .. " FPS), đã bật lại 3D")
        end
    end)
end

-- ===== ANTI AFK =====
lp.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)
task.spawn(function()
    while task.wait(300) do
        pcall(function()
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end)
    end
end)

-- ===== AUTO REJOIN (chống lặp) =====
if AUTO_REJOIN then
    local rejoining = false
    GuiService.ErrorMessageChanged:Connect(function()
        if rejoining then return end
        rejoining = true
        task.wait(5)
        pcall(function() TeleportService:Teleport(game.PlaceId, lp) end)
        task.wait(30)
        rejoining = false
    end)
end

-- ===== MUTE =====
if MUTE_SOUND then
    pcall(function() UserSettings():GetService("UserGameSettings").MasterVolume = 0 end)
end

-- ===== DỌN MAP (quét 1 lần) =====
local EFFECTS = {"ParticleEmitter","Trail","Beam","Smoke","Fire","Sparkles",
    "PointLight","SpotLight","SurfaceLight","PostEffect","Atmosphere","Highlight","Explosion"}

local function handle(v)
    for _, c in ipairs(EFFECTS) do
        if v:IsA(c) then v:Destroy() return end
    end
    if MUTE_SOUND and v:IsA("Sound") then
        v.Volume = 0
    elseif LIGHTEN_MAP then
        if v:IsA("Decal") or v:IsA("Texture") then
            if not (lp.Character and v:IsDescendantOf(lp.Character)) then v:Destroy() end
        elseif v:IsA("BasePart") and not v:IsA("Terrain") then
            v.Material = Enum.Material.SmoothPlastic
            v.Reflectance = 0
            v.CastShadow = false
        end
    end
end

for _, v in ipairs(Lighting:GetDescendants()) do pcall(handle, v) end
local n = 0
for _, v in ipairs(workspace:GetDescendants()) do
    pcall(handle, v)
    n += 1
    if n % 300 == 0 then task.wait() end
end
local conn = workspace.DescendantAdded:Connect(function(v)
    task.defer(pcall, handle, v)
end)
task.delay(LISTEN_SECONDS, function() conn:Disconnect() end)

-- ===== NHÃN FPS =====
if SHOW_FPS then
    local root = (gethui and gethui()) or game:GetService("CoreGui")
    local gui = Instance.new("ScreenGui")
    gui.Name = "HudGui"
    gui.ResetOnSpawn = false
    pcall(function() gui.Parent = root end)
    if not gui.Parent then gui.Parent = lp:WaitForChild("PlayerGui") end
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(0, 70, 0, 18)
    l.Position = UDim2.new(0, 4, 0, 4)
    l.BackgroundColor3 = Color3.new(0, 0, 0)
    l.BackgroundTransparency = 0.4
    l.TextColor3 = Color3.new(0, 1, 0)
    l.TextSize = 12
    l.Text = "FPS: ..."
    l.Parent = gui
    local f, t0 = 0, os.clock()
    RunService.Heartbeat:Connect(function()
        f += 1
        if os.clock() - t0 >= 1 then
            l.Text = "FPS: " .. f
            f, t0 = 0, os.clock()
        end
    end)
end

print("Treo bản nhẹ | cap " .. FPS_CAP .. " | 3D tắt: " .. tostring(DISABLE_3D))
