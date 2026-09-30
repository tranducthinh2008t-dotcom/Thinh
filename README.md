local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local VirtualUser = game:GetService("VirtualUser")
local GuiService = game:GetService("GuiService")
local TeleportService = game:GetService("TeleportService")
local lp = Players.LocalPlayer

-- ===== CẤU HÌNH =====
local FPS_CAP = 15
local DISABLE_3D = true      -- false = giữ 3D để cap FPS ăn chắc hơn
local SHOW_FPS = true
local SHOW_FLOOR_HUD = true
local HIDE_STUFF = true
local HIDE_KEYWORDS = {"house", "home", "building", "nha", "shop", "tower"}
local AUTO_REJOIN = true
local LISTEN_SECONDS = 30

-- ===== KHÓA FPS =====
local function applyCap()
    pcall(function() setfpscap(FPS_CAP) end)
end
applyCap()
task.delay(15, applyCap)
task.spawn(function()
    while task.wait(10) do applyCap() end
end)
pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
pcall(function() RunService:Set3dRenderingEnabled(not DISABLE_3D) end)

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

-- ===== AUTO REJOIN =====
if AUTO_REJOIN then
    GuiService.ErrorMessageChanged:Connect(function()
        task.wait(3)
        pcall(function() TeleportService:Teleport(game.PlaceId, lp) end)
    end)
end

-- ===== XÓA HIỆU ỨNG + ẨN NHÀ/CON =====
Lighting.GlobalShadows = false
Lighting.FogEnd = 9e9

local EFFECTS = {"ParticleEmitter","Trail","Beam","Smoke","Fire","Sparkles",
    "PointLight","SpotLight","SurfaceLight","PostEffect","Atmosphere","Highlight","Explosion"}

local function isEffect(v)
    for _, c in ipairs(EFFECTS) do
        if v:IsA(c) then return true end
    end
    return false
end

local function shouldHideModel(m)
    if m:FindFirstChildOfClass("Humanoid") and m ~= lp.Character then return true end
    local n = m.Name:lower()
    for _, k in ipairs(HIDE_KEYWORDS) do
        if n:find(k, 1, true) then return true end
    end
    return false
end

local function handle(v)
    local char = lp.Character
    if isEffect(v) then
        v:Destroy()
    elseif HIDE_STUFF and not DISABLE_3D then
        if v:IsA("BillboardGui") then
            if not (char and v:IsDescendantOf(char)) then v.Enabled = false end
        elseif v:IsA("BasePart") and not v:IsA("Terrain") then
            if char and v:IsDescendantOf(char) then return end
            local m = v:FindFirstAncestorOfClass("Model")
            while m do
                if shouldHideModel(m) then
                    v.LocalTransparencyModifier = 1
                    return
                end
                m = m:FindFirstAncestorOfClass("Model")
            end
        end
    end
end

for _, v in ipairs(Lighting:GetDescendants()) do pcall(handle, v) end
for _, v in ipairs(workspace:GetDescendants()) do pcall(handle, v) end
if LISTEN_SECONDS > 0 then
    local conn = workspace.DescendantAdded:Connect(function(v)
        task.defer(pcall, handle, v)
    end)
    task.delay(LISTEN_SECONDS, function() conn:Disconnect() end)
end

-- ===== GUI CHUNG =====
local guiRoot = (gethui and gethui()) or game:GetService("CoreGui")
pcall(function()
    local old = guiRoot:FindFirstChild("HudGui")
    if old then old:Destroy() end
end)
local gui = Instance.new("ScreenGui")
gui.Name = "HudGui"
gui.ResetOnSpawn = false
pcall(function() gui.Parent = guiRoot end)
if not gui.Parent then gui.Parent = lp:WaitForChild("PlayerGui") end

local function makeLabel(pos, color, text)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(0, 80, 0, 18)
    l.Position = pos
    l.BackgroundColor3 = Color3.new(0, 0, 0)
    l.BackgroundTransparency = 0.4
    l.TextColor3 = color
    l.TextSize = 12
    l.Font = Enum.Font.GothamBold
    l.Text = text
    l.Parent = gui
    return l
end

-- ===== HIỆN FPS =====
if SHOW_FPS then
    local fpsLabel = makeLabel(UDim2.new(0, 4, 0, 4), Color3.new(0, 1, 0), "FPS: ...")
    local frames, t0 = 0, os.clock()
    RunService.Heartbeat:Connect(function()
        frames += 1
        local now = os.clock()
        if now - t0 >= 1 then
            fpsLabel.Text = "FPS: " .. frames
            frames, t0 = 0, now
        end
    end)
end

-- ===== NHÃN SỐ TẦNG =====
if SHOW_FLOOR_HUD then
    local label = makeLabel(UDim2.new(1, -90, 0, 4), Color3.new(1, 1, 1), "...")
    local bound = {}
    local function match(t)
        return t.Text:match("^T[aầ]ng%s*%d+") or t.Text:match("^Floor%s*%d+")
    end
    local function bind(t)
        if bound[t] or not t:IsA("TextLabel") or t:IsDescendantOf(gui) then return end
        if not match(t) then return end
        bound[t] = true
        label.Text = t.Text
        t:GetPropertyChangedSignal("Text"):Connect(function()
            if match(t) then label.Text = t.Text end
        end)
    end
    for _, d in ipairs(workspace:GetDescendants()) do pcall(bind, d) end
    for _, d in ipairs(lp.PlayerGui:GetDescendants()) do pcall(bind, d) end
    local c1 = workspace.DescendantAdded:Connect(function(d) task.defer(pcall, bind, d) end)
    local c2 = lp.PlayerGui.DescendantAdded:Connect(function(d) task.defer(pcall, bind, d) end)
    task.delay(60, function() c1:Disconnect() c2:Disconnect() end)
end

print("Đã bật | FPS cap " .. FPS_CAP .. " | 3D tắt: " .. tostring(DISABLE_3D))
