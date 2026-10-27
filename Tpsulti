-- Valkyrie.ui | Fivelyss systems | red VEF shell | no key
-- Configs tab + full workspace save/load (reach, sliders, GK, macros, anti-ban, tools)
-- FIXED: SetCircleReachEnabled was mid-function spliced (syntax break). Restored clean.
-- TUNED: GK helper stronger on rockets (faster move/boost, tighter lateral stick, still smoothed)
-- TUNED: Keep+Save GK arm window 1.3s
-- TUNED: GK ~10x stronger on rockets · hard stop on slow balls (no drift)
-- NEW: Mobile button transparency (0 = solid · 1 = fully invisible, still works)
-- NEW: Circle Reach axis scales (RadiusX / RadiusY / RadiusZ ellipsoid)

local Players, RunService, UserInputService, Lighting, Workspace,
      TeleportService, CoreGui, StarterGui, Stats, Debris, VirtualUser,
      HttpService, LogService, LocalPlayer

do
    Players           = game:GetService("Players")
    RunService        = game:GetService("RunService")
    UserInputService  = game:GetService("UserInputService")
    Lighting          = game:GetService("Lighting")
    Workspace         = game:GetService("Workspace")
    TeleportService   = game:GetService("TeleportService")
    CoreGui           = game:GetService("CoreGui")
    StarterGui        = game:GetService("StarterGui")
    Stats             = game:GetService("Stats")
    Debris            = game:GetService("Debris")
    VirtualUser       = game:GetService("VirtualUser")
    HttpService       = game:GetService("HttpService")
    LogService        = game:GetService("LogService")
    LocalPlayer       = Players.LocalPlayer
end

local BASE_WALKSPEED, SPRINT_WALKSPEED, BALL_NAMES, BALL_SET

do
    BASE_WALKSPEED = 22
    SPRINT_WALKSPEED = 16
    BALL_NAMES = {
        "TPS", "MPS", "MPSBall", "StreetBall",
        "PSoccerBall", "SoccerBall", "Ball", "Football",
        "MatchBall", "fakeBaIlExpIoiter", "fakeBall", "ballPart",
        "CSF", "VEF", "IFF", "Futbol", "Pelota"
    }
    BALL_SET = {}
    for _, n in ipairs(BALL_NAMES) do BALL_SET[n] = true end
end

local State, Cache, AntiBan, R_HELD, TrackedBalls, ForceLocalScore

do
    State = { SavedPos = nil }
    Cache = {
        Character = nil, Humanoid = nil, Root = nil,
        MainBall = nil, LastBallScan = 0,
        BallList = {}, LastBallListScan = 0,
    }
    AntiBan = { Enabled = true }
    R_HELD = false
    TrackedBalls = {}
    ForceLocalScore = { Green = 0, Blue = 0, Enabled = false }
end

---------------------------------------------------------------------------
-- CONFIG SYSTEM  (workspace + ValkyrieConfigs folder)
---------------------------------------------------------------------------
local CONFIG_DIR  = "ValkyrieConfigs"
local CONFIG_FILE = "tps_ulti.json"
local CONFIG_FLAT = "valkyrie_tps_ulti.json"   -- always also written to executor workspace root
local ConfigStore = {}
local ConfigBindings = {}
local PendingConfig = nil
local LastLoadedName = CONFIG_FILE

local function JsonEncode(tbl)
    local ok, res = pcall(function() return HttpService:JSONEncode(tbl) end)
    if ok then return res end
    return "{}"
end

local function JsonDecode(str)
    local ok, res = pcall(function() return HttpService:JSONDecode(str) end)
    if ok and type(res) == "table" then return res end
    return {}
end

local function EnsureConfigDir()
    if not makefolder then return end
    pcall(function()
        if isfolder then
            if not isfolder(CONFIG_DIR) then makefolder(CONFIG_DIR) end
        else
            makefolder(CONFIG_DIR)
        end
    end)
end

local function udim2ToTable(u)
    if typeof(u) ~= "UDim2" then return nil end
    return { u.X.Scale, u.X.Offset, u.Y.Scale, u.Y.Offset }
end

local function tableToUDim2(t)
    if type(t) ~= "table" then return nil end
    local a, b, c, d = t[1], t[2], t[3], t[4]
    if a == nil then return nil end
    return UDim2.new(a or 0, b or 0, c or 0, d or 0)
end

local function RegisterConfig(key, ctl, default, apply)
    ConfigStore[key] = ConfigStore[key] ~= nil and ConfigStore[key] or default
    ConfigBindings[key] = function(v)
        ConfigStore[key] = v
        if apply then pcall(apply, v) end
        if ctl and ctl.Set then pcall(function() ctl:Set(v) end) end
    end
end

-- forward decls used by Collect / Apply (filled later)
local CC, reachSettings, GKHelper, StartGKHelper, StopGKHelper
local CreateAllLimbHitboxes, RemoveAllLimbHitboxes, UpdateLimbVisualizer, LimbBodyParts
local NRC, ReactKiller, SkyboxCfg, ApplySkybox, RemoveCustomSky, ApplyLighting
local ccSyncSizes, ccSyncOpacity, ccSyncLock, ccSyncVisible
local CircleReach, SetCircleReachEnabled, UpdateCircleReachVisual
local UIControls = {}
local SyncUIFromState
local SetNRCEnabled, SetReactKillerEnabled, SetReactBreakEnabled
local Notify
local ReactBreak

local function CollectMobilePositions()
    local positions = {}
    pcall(function()
        if CC and type(CC.positions) == "table" then
            for id, pos in pairs(CC.positions) do
                local t = udim2ToTable(pos)
                if t then positions[id] = t end
            end
        end
    end)
    pcall(function()
        local roots = {}
        if gethui then pcall(function() table.insert(roots, gethui()) end) end
        table.insert(roots, CoreGui)
        pcall(function() table.insert(roots, LocalPlayer:FindFirstChild("PlayerGui")) end)
        for _, root in ipairs(roots) do
            if root then
                local g = root:FindFirstChild("Fivelyss_MobileGK")
                if g then
                    for _, child in ipairs(g:GetChildren()) do
                        if child:IsA("GuiObject") and child.Position then
                            local t = udim2ToTable(child.Position)
                            if t then positions[child.Name] = t end
                        end
                    end
                end
            end
        end
    end)
    return positions
end

local function ApplyMobilePositions(positions)
    if type(positions) ~= "table" then return end
    pcall(function()
        if CC then
            CC.positions = CC.positions or {}
            for id, t in pairs(positions) do
                local u = tableToUDim2(t)
                if u then CC.positions[id] = u end
            end
        end
    end)
    pcall(function()
        local roots = {}
        if gethui then pcall(function() table.insert(roots, gethui()) end) end
        table.insert(roots, CoreGui)
        pcall(function() table.insert(roots, LocalPlayer:FindFirstChild("PlayerGui")) end)
        for _, root in ipairs(roots) do
            if root then
                local g = root:FindFirstChild("Fivelyss_MobileGK")
                if g then
                    for id, t in pairs(positions) do
                        local btn = g:FindFirstChild(id)
                        local u = tableToUDim2(t)
                        if btn and btn:IsA("GuiObject") and u then
                            btn.Position = u
                        end
                    end
                end
            end
        end
    end)
end

local function CollectConfigSnapshot()
    local data = {
        version = 5,
        placeId = game.PlaceId,
        savedAt = os.date("%Y-%m-%d %H:%M:%S"),
        mobile = {
            positions = CollectMobilePositions(),
            btn = nil,
            keepToSave = nil,
            mobEnabled = nil,
            gkHudVisible = nil,
        },
        reach = nil,
        gk = nil,
        antiban = nil,
        nrc = nil,
        reactKiller = nil,
        circleReach = nil,
        sky = nil,
        extra = {},
    }

    pcall(function()
        if CC then
            data.mobile.btn = {
                size = CC.btn and CC.btn.size,
                transparency = CC.btn and CC.btn.transparency,
                locked = CC.btn and CC.btn.locked,
                visible = CC.btn and CC.btn.visible,
            }
            data.mobile.keepToSave = CC.keepToSave
            data.mobile.mobEnabled = CC.mobEnabled
            data.mobile.gkHudVisible = CC.gkHudVisible
        end
    end)

    pcall(function()
        if reachSettings then
            data.reach = {}
            for k, v in pairs(reachSettings) do
                if type(v) ~= "function" then data.reach[k] = v end
            end
        end
    end)

    pcall(function()
        if GKHelper then
            data.gk = { Enabled = GKHelper.Enabled == true }
        end
    end)

    pcall(function()
        data.antiban = { Enabled = AntiBan.Enabled == true }
    end)

    pcall(function()
        if NRC then
            data.nrc = {
                Enabled = NRC.Enabled == true,
                Magnitude = NRC.Magnitude,
                Distance = NRC.Distance,
            }
        end
    end)

    pcall(function()
        if ReactKiller then
            data.reactKiller = {
                Enabled = ReactKiller.Enabled == true,
                Fires = ReactKiller.Fires,
                Distance = ReactKiller.Distance,
            }
        end
    end)

    pcall(function()
        if ReactBreak then
            data.reactBreak = {
                Enabled = ReactBreak.Enabled == true,
                Strength = ReactBreak.Strength,
                Distance = ReactBreak.Distance,
            }
        end
    end)

    pcall(function()
        if CircleReach then
            data.circleReach = {
                Enabled = CircleReach.Enabled == true,
                Radius = CircleReach.Radius,
                RadiusX = CircleReach.RadiusX,
                RadiusY = CircleReach.RadiusY,
                RadiusZ = CircleReach.RadiusZ,
                Visible = CircleReach.Visible == true,
                Transparency = CircleReach.Transparency,
                Mode = CircleReach.Mode or "Both",
            }
        end
    end)

    pcall(function()
        if SkyboxCfg then
            data.sky = {
                Enabled = SkyboxCfg.Enabled == true,
                Current = SkyboxCfg.Current,
            }
        end
    end)

    for k, v in pairs(ConfigStore) do
        data.extra[k] = v
    end

    return data
end

local function ApplyConfigSnapshot(data)
    if type(data) ~= "table" then return end

    -- mobile positions + button style
    if data.mobile then
        if data.mobile.positions then
            ApplyMobilePositions(data.mobile.positions)
        end
        pcall(function()
            if CC and data.mobile.btn then
                local b = data.mobile.btn
                if b.size and CC.btn then CC.btn.size = b.size end
                if b.transparency ~= nil and CC.btn then CC.btn.transparency = b.transparency end
                if b.locked ~= nil and CC.btn then CC.btn.locked = b.locked end
                if b.visible ~= nil and CC.btn then CC.btn.visible = b.visible end
                if ccSyncSizes then ccSyncSizes() end
                if ccSyncOpacity then ccSyncOpacity() end
                if ccSyncLock then ccSyncLock() end
                if ccSyncVisible then ccSyncVisible() end
            end
            if CC and data.mobile.keepToSave ~= nil then CC.keepToSave = data.mobile.keepToSave end
            if CC and data.mobile.mobEnabled ~= nil then CC.mobEnabled = data.mobile.mobEnabled end
            if CC and data.mobile.gkHudVisible ~= nil then
                CC.gkHudVisible = data.mobile.gkHudVisible
                if getgenv().Valkyrie_SetGkHud then
                    getgenv().Valkyrie_SetGkHud(CC.gkHudVisible)
                elseif ccSyncVisible then
                    ccSyncVisible()
                end
            end
        end)
    end

    -- REACH — write table, rebuild hitboxes (now + short deferred pass)
    pcall(function()
        if data.reach and reachSettings then
            for k, v in pairs(data.reach) do
                if type(v) ~= "function" then
                    reachSettings[k] = v
                end
            end
            local function rebuild()
                if RemoveAllLimbHitboxes then pcall(RemoveAllLimbHitboxes) end
                if CreateAllLimbHitboxes then pcall(CreateAllLimbHitboxes) end
                if LimbBodyParts and UpdateLimbVisualizer then
                    for _, part in ipairs(LimbBodyParts) do pcall(UpdateLimbVisualizer, part) end
                end
            end
            rebuild()
            task.defer(function()
                task.wait(0.4)
                rebuild()
                if SyncUIFromState then pcall(SyncUIFromState) end
            end)
        end
    end)

    -- GK
    pcall(function()
        if data.gk and StartGKHelper and StopGKHelper and GKHelper then
            if data.gk.Enabled then
                if not GKHelper.Enabled then StartGKHelper() end
            else
                if GKHelper.Enabled then StopGKHelper() end
            end
        end
    end)

    -- AntiBan
    pcall(function()
        if data.antiban then
            AntiBan.Enabled = data.antiban.Enabled == true
        end
    end)

    -- NRC — numbers + actually arm/disarm
    pcall(function()
        if data.nrc and NRC then
            NRC.Magnitude = data.nrc.Magnitude or NRC.Magnitude
            NRC.Distance  = data.nrc.Distance  or NRC.Distance
            if data.nrc.Enabled ~= nil and SetNRCEnabled then
                SetNRCEnabled(data.nrc.Enabled == true, true)
            elseif data.nrc.Enabled ~= nil then
                NRC.Enabled = data.nrc.Enabled == true
            end
        end
    end)

    -- React Killer
    pcall(function()
        if data.reactKiller and ReactKiller then
            ReactKiller.Fires    = data.reactKiller.Fires or ReactKiller.Fires
            ReactKiller.Distance = data.reactKiller.Distance or ReactKiller.Distance
            if data.reactKiller.Enabled ~= nil and SetReactKillerEnabled then
                SetReactKillerEnabled(data.reactKiller.Enabled == true, true)
            elseif data.reactKiller.Enabled ~= nil then
                ReactKiller.Enabled = data.reactKiller.Enabled == true
            end
        end
    end)

    -- React Break
    pcall(function()
        if data.reactBreak and ReactBreak then
            ReactBreak.Strength = data.reactBreak.Strength or ReactBreak.Strength
            ReactBreak.Distance = data.reactBreak.Distance or ReactBreak.Distance
            if data.reactBreak.Enabled ~= nil and SetReactBreakEnabled then
                SetReactBreakEnabled(data.reactBreak.Enabled == true, true)
            elseif data.reactBreak.Enabled ~= nil then
                ReactBreak.Enabled = data.reactBreak.Enabled == true
            end
        end
    end)

    -- Circle Reach
    pcall(function()
        if data.circleReach and CircleReach then
            if data.circleReach.Radius ~= nil then
                CircleReach.Radius = tonumber(data.circleReach.Radius) or CircleReach.Radius
            end
            if data.circleReach.RadiusX ~= nil then
                CircleReach.RadiusX = tonumber(data.circleReach.RadiusX) or CircleReach.RadiusX
            end
            if data.circleReach.RadiusY ~= nil then
                CircleReach.RadiusY = tonumber(data.circleReach.RadiusY) or CircleReach.RadiusY
            end
            if data.circleReach.RadiusZ ~= nil then
                CircleReach.RadiusZ = tonumber(data.circleReach.RadiusZ) or CircleReach.RadiusZ
            end
            -- legacy: if only Radius present, seed all axes
            if data.circleReach.Radius ~= nil and data.circleReach.RadiusX == nil then
                local r = tonumber(data.circleReach.Radius) or 12
                CircleReach.RadiusX = r
                CircleReach.RadiusY = r
                CircleReach.RadiusZ = r
            end
            if data.circleReach.Transparency ~= nil then
                CircleReach.Transparency = tonumber(data.circleReach.Transparency) or CircleReach.Transparency
            end
            if data.circleReach.Visible ~= nil then
                CircleReach.Visible = data.circleReach.Visible == true
            end
            if data.circleReach.Mode ~= nil then
                local m = tostring(data.circleReach.Mode)
                if m == "Arms" or m == "Legs" or m == "Both" then
                    CircleReach.Mode = m
                end
            end
            if data.circleReach.Enabled ~= nil and SetCircleReachEnabled then
                SetCircleReachEnabled(data.circleReach.Enabled == true, true)
            elseif data.circleReach.Enabled ~= nil then
                CircleReach.Enabled = data.circleReach.Enabled == true
            end
            if UpdateCircleReachVisual then pcall(UpdateCircleReachVisual) end
        end
    end)

    -- Sky
    pcall(function()
        if data.sky and SkyboxCfg then
            SkyboxCfg.Current = data.sky.Current or SkyboxCfg.Current
            SkyboxCfg.Enabled = data.sky.Enabled == true
            if SkyboxCfg.Enabled and ApplySkybox then
                ApplySkybox(SkyboxCfg.Current)
            elseif not SkyboxCfg.Enabled and RemoveCustomSky then
                RemoveCustomSky()
            end
        end
    end)

    if type(data.extra) == "table" then
        for k, v in pairs(data.extra) do
            if ConfigBindings[k] then
                pcall(ConfigBindings[k], v)
            else
                ConfigStore[k] = v
            end
        end
    end

    -- legacy flat format
    if data.mobilePositions then ApplyMobilePositions(data.mobilePositions) end

    -- push everything into the UI toggles/sliders so the menu matches live state
    if SyncUIFromState then
        pcall(SyncUIFromState)
    end
end

local function SaveConfigToFile(name)
    EnsureConfigDir()
    if not writefile then return false, "writefile not supported" end
    local fileName = (name and name ~= "") and name or CONFIG_FILE
    if not fileName:lower():find("%.json$") then fileName = fileName .. ".json" end
    local snapshot = CollectConfigSnapshot()
    ConfigStore._lastSave = snapshot
    local encoded = JsonEncode(snapshot)
    local path = CONFIG_DIR .. "/" .. fileName
    local ok, err = pcall(function()
        writefile(path, encoded)
        -- always also dump flat copy into executor workspace root
        pcall(function() writefile(CONFIG_FLAT, encoded) end)
        -- also write the exact name into workspace root for easy find
        pcall(function() writefile(fileName, encoded) end)
    end)
    if ok then LastLoadedName = fileName end
    return ok, ok and path or tostring(err)
end

local function LoadConfigFromFile(name)
    EnsureConfigDir()
    if not (readfile and isfile) then return false, "readfile not supported" end
    local fileName = (name and name ~= "") and name or CONFIG_FILE
    if not fileName:lower():find("%.json$") then fileName = fileName .. ".json" end
    local path = CONFIG_DIR .. "/" .. fileName

    local function tryRead(p)
        if isfile and not isfile(p) then return nil end
        local ok, c = pcall(function() return readfile(p) end)
        if ok and type(c) == "string" and #c > 2 then return c end
        return nil
    end

    local content = tryRead(path) or tryRead(CONFIG_FLAT) or tryRead(fileName)
    if not content then
        PendingConfig = nil
        return false, "missing file"
    end
    local decoded = JsonDecode(content)
    if next(decoded) == nil then return false, "bad json" end
    PendingConfig = decoded
    ApplyConfigSnapshot(decoded)
    LastLoadedName = fileName
    return true, path
end

local function ListConfigFiles()
    local out = {}
    if not listfiles then return out end
    EnsureConfigDir()
    local ok, files = pcall(function() return listfiles(CONFIG_DIR) end)
    if ok and type(files) == "table" then
        for _, f in ipairs(files) do
            local name = tostring(f):match("([^/\\]+)$") or tostring(f)
            if name:lower():find("%.json$") then
                table.insert(out, name)
            end
        end
    end
    -- also check workspace root flat
    pcall(function()
        if isfile and isfile(CONFIG_FLAT) then
            local already = false
            for _, n in ipairs(out) do if n == CONFIG_FLAT then already = true break end end
            if not already then table.insert(out, CONFIG_FLAT) end
        end
    end)
    return out
end

local function DeleteConfigFile(name)
    if not delfile then return false, "delfile not supported" end
    local fileName = (name and name ~= "") and name or CONFIG_FILE
    if not fileName:lower():find("%.json$") then fileName = fileName .. ".json" end
    local path = CONFIG_DIR .. "/" .. fileName
    local ok = pcall(function()
        if isfile and isfile(path) then delfile(path) end
        if isfile and isfile(fileName) then delfile(fileName) end
    end)
    return ok, path
end

getgenv().Valkyrie_Config = {
    Store   = ConfigStore,
    Save    = SaveConfigToFile,
    Load    = LoadConfigFromFile,
    List    = ListConfigFiles,
    Delete  = DeleteConfigFile,
    Collect = CollectConfigSnapshot,
    Apply   = ApplyConfigSnapshot,
}
getgenv().Fivelyss_Config = getgenv().Valkyrie_Config

---------------------------------------------------------------------------
-- ANTI-BAN / ANTI-KICK
---------------------------------------------------------------------------
do
    pcall(function()
        local mt = getrawmetatable(game)
        if not mt then return end
        setreadonly(mt, false)
        local oldNC = mt.__namecall
        mt.__namecall = newcclosure(function(self, ...)
            if not AntiBan.Enabled then return oldNC(self, ...) end
            local method = getnamecallmethod()
            if method == "Kick" and self == LocalPlayer then return nil end
            if method == "FireServer" or method == "InvokeServer" then
                local ok, name = pcall(function() return self.Name end)
                if ok and type(name) == "string" then
                    local n = name:lower()
                    if n:find("anticheat") or n:find("detect") or n:find("ban")
                       or n:find("flag") or n:find("report") or n:find("vote")
                       or n:find("kick") then
                        return nil
                    end
                end
            end
            return oldNC(self, ...)
        end)
        setreadonly(mt, true)
    end)

    pcall(function()
        local Destroyed = setmetatable({}, {__mode = "k"})
        local Keywords = {"Banned", "Walkspeed", "Ban", "AntiCheat"}
        local function ContainsBad(text)
            for _, k in ipairs(Keywords) do
                if text:find(k, 1, true) then return true end
            end
            return false
        end
        local function IsAntiCheat(scr)
            if typeof(scr) ~= "Instance" or not scr:IsA("LuaSourceContainer") then return false end
            local ok, src = pcall(function() return scr.Source end)
            return ok and typeof(src) == "string" and ContainsBad(src)
        end
        for _, d in ipairs(game:GetDescendants()) do
            if (d:IsA("LocalScript") or d:IsA("ModuleScript") or
                (d:IsA("Script") and d.RunContext == Enum.RunContext.Client)) and
                not Destroyed[d] and IsAntiCheat(d) then
                pcall(function() d:Destroy() end)
                Destroyed[d] = true
            end
        end
        game.DescendantAdded:Connect(function(d)
            if (d:IsA("LocalScript") or d:IsA("ModuleScript") or d:IsA("Script")) and IsAntiCheat(d) then
                pcall(function() d:Destroy() end)
                Destroyed[d] = true
            end
        end)
    end)
end

---------------------------------------------------------------------------
-- CHARACTER / BALL HELPERS
---------------------------------------------------------------------------
local GetCharacter, GetHumanoid, GetRoot, GetFE, ResetVelocity, IsBall
local FindAllBalls, GetMainBall, GetPracticeBalls, GetBallList

do
    GetCharacter = function() return Cache.Character or LocalPlayer.Character end
    GetHumanoid = function()
        if Cache.Humanoid and Cache.Humanoid.Parent then return Cache.Humanoid end
        local c = GetCharacter()
        if c then Cache.Humanoid = c:FindFirstChildOfClass("Humanoid") end
        return Cache.Humanoid
    end
    GetRoot = function()
        if Cache.Root and Cache.Root.Parent then return Cache.Root end
        local c = GetCharacter()
        if c then Cache.Root = c:FindFirstChild("HumanoidRootPart") end
        return Cache.Root
    end
    GetFE = function() return Workspace:FindFirstChild("FE") end
    ResetVelocity = function()
        local root = GetRoot()
        if root then
            root.AssemblyLinearVelocity = Vector3.zero
            root.AssemblyAngularVelocity = Vector3.zero
        end
    end
    IsBall = function(inst)
        return inst and inst:IsA("BasePart") and BALL_SET[inst.Name] == true
    end

    FindAllBalls = function()
        TrackedBalls = {}
        local tps = Workspace:FindFirstChild("TPSSystem")
        if tps then
            local tpsBall = tps:FindFirstChild("TPS")
            if tpsBall and tpsBall:IsA("BasePart") then table.insert(TrackedBalls, tpsBall) end
        end
        local mps = Workspace:FindFirstChild("MPSSystem") or Workspace:FindFirstChild("MPS")
        if mps then
            for _, child in ipairs(mps:GetChildren()) do
                if child:IsA("BasePart") and (child.Name == "MPSBall" or child.Name == "Ball" or child.Name == "MPS") then
                    table.insert(TrackedBalls, child)
                end
            end
        end
        local practice = Workspace:FindFirstChild("Practice")
        if practice then
            for _, child in ipairs(practice:GetChildren()) do
                if child:IsA("BasePart") and (child.Name == "PSoccerBall" or child.Name == "Ball") then
                    table.insert(TrackedBalls, child)
                end
            end
        end
        for _, n in ipairs(BALL_NAMES) do
            local b = Workspace:FindFirstChild(n)
            if b and b:IsA("BasePart") and not table.find(TrackedBalls, b) then
                table.insert(TrackedBalls, b)
            end
        end
    end

    GetMainBall = function()
        local now = os.clock()
        if Cache.MainBall and Cache.MainBall.Parent and (now - Cache.LastBallScan) < 0.5 then
            return Cache.MainBall
        end
        local tps = Workspace:FindFirstChild("TPSSystem")
        if tps then
            local b = tps:FindFirstChild("TPS")
            if b then Cache.MainBall = b Cache.LastBallScan = now return b end
        end
        for _, name in ipairs(BALL_NAMES) do
            local ball = Workspace:FindFirstChild(name)
            if ball and ball:IsA("BasePart") then
                Cache.MainBall = ball Cache.LastBallScan = now return ball
            end
        end
        Cache.MainBall = nil
        return nil
    end

    GetPracticeBalls = function()
        local list = {}
        local practice = Workspace:FindFirstChild("Practice")
        if practice then
            for _, child in ipairs(practice:GetChildren()) do
                if child:IsA("BasePart") and (child.Name == "PSoccerBall" or child.Name == "Ball") then
                    table.insert(list, child)
                end
            end
        end
        return list
    end

    GetBallList = function()
        local now = os.clock()
        if (now - Cache.LastBallListScan) < 0.5 and #Cache.BallList > 0 then
            local valid = true
            for _, b in ipairs(Cache.BallList) do
                if not b.Parent then valid = false break end
            end
            if valid then return Cache.BallList end
        end
        local balls, seen = {}, {}
        local tps = Workspace:FindFirstChild("TPSSystem")
        if tps then
            local tpsBall = tps:FindFirstChild("TPS")
            if tpsBall and tpsBall:IsA("BasePart") then
                table.insert(balls, tpsBall) seen[tpsBall] = true
            end
        end
        local practice = Workspace:FindFirstChild("Practice")
        if practice then
            for _, child in ipairs(practice:GetChildren()) do
                if child:IsA("BasePart") and (child.Name == "PSoccerBall" or child.Name == "Ball") then
                    if not seen[child] then table.insert(balls, child) seen[child] = true end
                end
            end
        end
        for _, n in ipairs(BALL_NAMES) do
            local b = Workspace:FindFirstChild(n)
            if b and b:IsA("BasePart") and not seen[b] then
                table.insert(balls, b) seen[b] = true
            end
        end
        Cache.BallList = balls
        Cache.LastBallListScan = now
        return balls
    end
end

do
    LocalPlayer.CharacterAdded:Connect(function(c)
        Cache.Character = c
        Cache.Humanoid = c:WaitForChild("Humanoid", 5)
        Cache.Root = c:WaitForChild("HumanoidRootPart", 5)
    end)
    LocalPlayer.CharacterRemoving:Connect(function()
        Cache.Character = nil; Cache.Humanoid = nil; Cache.Root = nil
    end)
    if LocalPlayer.Character then
        Cache.Character = LocalPlayer.Character
        Cache.Humanoid = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        Cache.Root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    end

    Workspace.DescendantAdded:Connect(function(c)
        if IsBall(c) and not table.find(TrackedBalls, c) then
            table.insert(TrackedBalls, c)
        end
    end)
    FindAllBalls()

    task.spawn(function()
        task.wait(1)
        local hum = GetHumanoid()
        if hum then hum.WalkSpeed = BASE_WALKSPEED end
    end)

    RunService.Heartbeat:Connect(function()
        if not Cache.Humanoid then return end
        local target = R_HELD and SPRINT_WALKSPEED or BASE_WALKSPEED
        if Cache.Humanoid.WalkSpeed ~= target then Cache.Humanoid.WalkSpeed = target end
    end)

    UserInputService.InputBegan:Connect(function(Input, Processed)
        if Processed then return end
        if Input.KeyCode == Enum.KeyCode.R then
            R_HELD = true
            if UserInputService:IsKeyDown(Enum.KeyCode.X) then
                task.spawn(function()
                    local root = GetRoot() local ball = GetMainBall()
                    if root and ball then
                        local dir = (ball.Position - root.Position).Unit
                        root.AssemblyLinearVelocity = Vector3.new(dir.X * 80, root.AssemblyLinearVelocity.Y, dir.Z * 80)
                    end
                end)
            end
        end
        if Input.KeyCode == Enum.KeyCode.F and UserInputService:IsKeyDown(Enum.KeyCode.X) then
            task.spawn(function()
                local root = GetRoot() local ball = GetMainBall()
                if root and ball then ball.AssemblyLinearVelocity = (root.Position - ball.Position).Unit * 100 end
            end)
        end
    end)

    UserInputService.InputEnded:Connect(function(Input, Processed)
        if Processed then return end
        if Input.KeyCode == Enum.KeyCode.R then R_HELD = false end
    end)
end

---------------------------------------------------------------------------
-- LIMB REACH
---------------------------------------------------------------------------
local LimbHitboxes, LimbVisualizers, LimbActiveTouches,
      SafeClear, GetReachDistanceForPart, CreateLimbHitbox,
      GetVisTransparencyFor, RefreshVisTransparency,
      FindBallPartLimb, CheckLimbBallCollisions

do
    LimbHitboxes = {}
    LimbVisualizers = {}
    LimbActiveTouches = {}
    LimbBodyParts = { "Head", "Right Arm", "Left Arm", "Right Leg", "Left Leg" }

    reachSettings = {
        HeadEnabled = false, HeadVisualizer = false, HeadVisTransparency = 0.7,
        ReachX_Head = 1, ReachY_Head = 1, ReachZ_Head = 1,
        RightArmEnabled = false, RightArmVisualizer = false, ArmVisTransparency = 0.7,
        LeftArmEnabled = false, LeftArmVisualizer = false,
        ReachX_Arm = 1, ReachY_Arm = 1, ReachZ_Arm = 1,
        RightLegEnabled = false, RightLegVisualizer = false, LegVisTransparency = 0.7,
        LeftLegEnabled = false, LeftLegVisualizer = false,
        ReachX_Leg = 1, ReachY_Leg = 1, ReachZ_Leg = 1, ReachDistance = 50,
    }

    SafeClear = function(tbl)
        if not tbl then return end
        if table.clear then table.clear(tbl) else for k in pairs(tbl) do tbl[k] = nil end end
    end

    GetReachDistanceForPart = function(partName)
        if partName == "Head" then
            if not reachSettings.HeadEnabled then return 0 end
            return reachSettings.ReachDistance * ((reachSettings.ReachX_Head + reachSettings.ReachY_Head + reachSettings.ReachZ_Head) / 3)
        elseif partName == "Right Arm" or partName == "Left Arm" then
            local side = partName:match("(%w+)%s")
            if not reachSettings[side .. "ArmEnabled"] then return 0 end
            return reachSettings.ReachDistance * ((reachSettings.ReachX_Arm + reachSettings.ReachY_Arm + reachSettings.ReachZ_Arm) / 3)
        elseif partName == "Right Leg" or partName == "Left Leg" then
            local side = partName:match("(%w+)%s")
            if not reachSettings[side .. "LegEnabled"] then return 0 end
            return reachSettings.ReachDistance * ((reachSettings.ReachX_Leg + reachSettings.ReachY_Leg + reachSettings.ReachZ_Leg) / 3)
        end
        return 0
    end

    CreateLimbHitbox = function(bodyPart, partName)
        local hb = Instance.new("Part")
        hb.Name = partName .. "_Hitbox"
        hb.Anchored = false
        hb.CanCollide = false
        hb.Transparency = 1
        hb.Massless = true
        local isArm = (partName == "Right Arm" or partName == "Left Arm")
        if partName == "Head" then
            hb.Size = Vector3.new(bodyPart.Size.X * reachSettings.ReachX_Head * 2, bodyPart.Size.Y * reachSettings.ReachY_Head * 2, bodyPart.Size.Z * reachSettings.ReachZ_Head * 2)
        elseif isArm then
            -- slightly larger arm volume + optional hard collide when circle reach is on (stops GK phase-through)
            local mul = 2.15
            if CircleReach and CircleReach.Enabled then
                mul = 2.45
                hb.CanCollide = true
            end
            hb.Size = Vector3.new(bodyPart.Size.X * reachSettings.ReachX_Arm * mul, bodyPart.Size.Y * reachSettings.ReachY_Arm * mul, bodyPart.Size.Z * reachSettings.ReachZ_Arm * mul)
        else
            hb.Size = Vector3.new(bodyPart.Size.X * reachSettings.ReachX_Leg * 2, bodyPart.Size.Y * reachSettings.ReachY_Leg * 2, bodyPart.Size.Z * reachSettings.ReachZ_Leg * 2)
        end
        hb.CFrame = bodyPart.CFrame
        hb.Parent = bodyPart
        local w = Instance.new("WeldConstraint")
        w.Part0 = bodyPart
        w.Part1 = hb
        w.Parent = hb
        return hb
    end

    GetVisTransparencyFor = function(partName)
        if partName == "Head" then return reachSettings.HeadVisTransparency
        elseif partName == "Right Arm" or partName == "Left Arm" then return reachSettings.ArmVisTransparency
        else return reachSettings.LegVisTransparency end
    end

    UpdateLimbVisualizer = function(partName)
        local char = Cache.Character
        if not char then return end
        local bodyPart = char:FindFirstChild(partName)
        if not bodyPart then return end
        if GetReachDistanceForPart(partName) > 0 then
            if LimbHitboxes[partName] then LimbHitboxes[partName]:Destroy() end
            LimbHitboxes[partName] = CreateLimbHitbox(bodyPart, partName)
        else
            if LimbHitboxes[partName] then LimbHitboxes[partName]:Destroy() LimbHitboxes[partName] = nil end
        end
        local visOn = false
        if partName == "Head" then visOn = reachSettings.HeadVisualizer
        elseif partName == "Right Arm" then visOn = reachSettings.RightArmVisualizer
        elseif partName == "Left Arm" then visOn = reachSettings.LeftArmVisualizer
        elseif partName == "Right Leg" then visOn = reachSettings.RightLegVisualizer
        elseif partName == "Left Leg" then visOn = reachSettings.LeftLegVisualizer end
        if LimbVisualizers[partName] then LimbVisualizers[partName]:Destroy() LimbVisualizers[partName] = nil end
        if visOn and LimbHitboxes[partName] then
            local hb = LimbHitboxes[partName]
            local vis = Instance.new("Part")
            vis.Name = partName .. "_Visualizer"
            vis.Size = hb.Size
            vis.CFrame = hb.CFrame
            vis.Anchored = false
            vis.CanCollide = false
            vis.Transparency = GetVisTransparencyFor(partName)
            vis.Color = Color3.fromRGB(0, 255, 100)
            vis.Material = Enum.Material.Neon
            vis.Massless = true
            vis.Parent = hb
            local w = Instance.new("WeldConstraint")
            w.Part0 = hb
            w.Part1 = vis
            w.Parent = vis
            LimbVisualizers[partName] = vis
        end
    end

    RefreshVisTransparency = function()
        for partName, v in pairs(LimbVisualizers) do
            if v and v.Parent then v.Transparency = GetVisTransparencyFor(partName) end
        end
    end

    FindBallPartLimb = function(part)
        if part:IsA("BasePart") and BALL_SET[part.Name] then return part end
        local cur = part
        while cur and cur ~= Workspace do
            if cur:IsA("BasePart") and BALL_SET[cur.Name] then return cur end
            cur = cur.Parent
        end
        return nil
    end

    CreateAllLimbHitboxes = function()
        local char = Cache.Character
        if not char then return end
        for _, partName in ipairs(LimbBodyParts) do
            local bodyPart = char:FindFirstChild(partName)
            if bodyPart and GetReachDistanceForPart(partName) > 0 then
                if LimbHitboxes[partName] then LimbHitboxes[partName]:Destroy() end
                LimbHitboxes[partName] = CreateLimbHitbox(bodyPart, partName)
                UpdateLimbVisualizer(partName)
            end
        end
    end

    RemoveAllLimbHitboxes = function()
        for _, h in pairs(LimbHitboxes) do if h then h:Destroy() end end
        for _, v in pairs(LimbVisualizers) do if v then v:Destroy() end end
        SafeClear(LimbHitboxes)
        SafeClear(LimbVisualizers)
        SafeClear(LimbActiveTouches)
    end

    CheckLimbBallCollisions = function()
        local char = Cache.Character
        if not char or not char.Parent then return end
        local now = tick()
        local newTouches = {}
        for partName, hb in pairs(LimbHitboxes) do
            if hb and hb.Parent then
                local bodyPart = char:FindFirstChild(partName)
                if bodyPart then
                    local op = OverlapParams.new()
                    op.FilterDescendantsInstances = { char }
                    op.FilterType = Enum.RaycastFilterType.Exclude
                    local partsInBox = Workspace:GetPartBoundsInBox(hb.CFrame, hb.Size, op)
                    for _, part in ipairs(partsInBox) do
                        local ballPart = FindBallPartLimb(part)
                        if ballPart then
                            local key = partName .. ":" .. ballPart:GetFullName()
                            newTouches[key] = true
                            if not LimbActiveTouches[key] or (now - LimbActiveTouches[key]) > 0.08 then
                                LimbActiveTouches[key] = now
                                pcall(function()
                                    firetouchinterest(ballPart, bodyPart, 0)
                                    task.wait()
                                    firetouchinterest(ballPart, bodyPart, 1)
                                end)
                                -- anti phase-through (GK + arm reach + circle reach): push ball out of hitbox
                                if (partName == "Right Arm" or partName == "Left Arm") and CircleReach and CircleReach.Enabled then
                                    pcall(function()
                                        local dir = (ballPart.Position - hb.Position)
                                        if dir.Magnitude < 0.05 then dir = bodyPart.CFrame.LookVector end
                                        dir = dir.Unit
                                        local speed = math.max(ballPart.AssemblyLinearVelocity.Magnitude, 22)
                                        ballPart.AssemblyLinearVelocity = dir * speed
                                        ballPart.AssemblyAngularVelocity = Vector3.zero
                                    end)
                                end
                            end
                        end
                    end
                end
            end
        end
        for key, t in pairs(LimbActiveTouches) do
            if not newTouches[key] and (now - t) > 0.1 then LimbActiveTouches[key] = nil end
        end
    end

    local lastTick = 0
    RunService.Heartbeat:Connect(function()
        local now = tick()
        if now - lastTick >= 0.1 then
            lastTick = now
            pcall(CheckLimbBallCollisions)
        end
    end)

    getgenv().FIVELYSS_LimbRespawnToken = 0
    LocalPlayer.CharacterAdded:Connect(function()
        getgenv().FIVELYSS_LimbRespawnToken = (getgenv().FIVELYSS_LimbRespawnToken or 0) + 1
        local myToken = getgenv().FIVELYSS_LimbRespawnToken
        RemoveAllLimbHitboxes()
        task.wait(2)
        if getgenv().FIVELYSS_LimbRespawnToken ~= myToken then return end
        if not Cache.Character or not Cache.Character.Parent then return end
        pcall(CreateAllLimbHitboxes)
    end)
end

---------------------------------------------------------------------------
-- CIRCLE REACH  (body ellipsoid · arms / legs / both · firetouchinterest)
-- RadiusX / RadiusY / RadiusZ = independent axes (Y-up world)
---------------------------------------------------------------------------
do
    local ARMS = { "Right Arm", "Left Arm" }
    local LEGS = { "Right Leg", "Left Leg" }

    CircleReach = {
        Enabled = false,
        Radius = 12,      -- legacy master (kept for old configs)
        RadiusX = 12,     -- left-right
        RadiusY = 12,     -- up-down
        RadiusZ = 12,     -- forward-back
        Visible = true,
        Transparency = 0.65,
        Mode = "Both", -- "Arms" | "Legs" | "Both"
        Visual = nil,
        Conn = nil,
        LastTouch = {},
    }

    local function activeLimbs()
        local m = CircleReach.Mode or "Both"
        if m == "Arms" then return ARMS end
        if m == "Legs" then return LEGS end
        local out = {}
        for _, n in ipairs(ARMS) do table.insert(out, n) end
        for _, n in ipairs(LEGS) do table.insert(out, n) end
        return out
    end

    local function getRadii()
        local rx = math.max(tonumber(CircleReach.RadiusX) or CircleReach.Radius or 12, 0.5)
        local ry = math.max(tonumber(CircleReach.RadiusY) or CircleReach.Radius or 12, 0.5)
        local rz = math.max(tonumber(CircleReach.RadiusZ) or CircleReach.Radius or 12, 0.5)
        return rx, ry, rz
    end

    UpdateCircleReachVisual = function()
        local root = GetRoot()
        if not CircleReach.Enabled or not CircleReach.Visible or not root then
            if CircleReach.Visual then
                pcall(function() CircleReach.Visual:Destroy() end)
                CircleReach.Visual = nil
            end
            return
        end
        local rx, ry, rz = getRadii()
        local vis = CircleReach.Visual
        if not vis or not vis.Parent then
            vis = Instance.new("Part")
            vis.Name = "Valkyrie_CircleReach"
            vis.Shape = Enum.PartType.Ball
            vis.Anchored = true
            vis.CanCollide = false
            vis.CanQuery = false
            vis.CanTouch = false
            vis.Massless = true
            vis.Material = Enum.Material.ForceField
            vis.Color = Color3.fromRGB(255, 80, 120)
            vis.CastShadow = false
            vis.Parent = Workspace
            CircleReach.Visual = vis
        end
        -- Ball shape scales into an ellipsoid when Size axes differ
        vis.Size = Vector3.new(rx * 2, ry * 2, rz * 2)
        vis.Transparency = CircleReach.Transparency or 0.65
        vis.CFrame = root.CFrame
    end

    local function destroyVisual()
        if CircleReach.Visual then
            pcall(function() CircleReach.Visual:Destroy() end)
            CircleReach.Visual = nil
        end
    end

    local function touchBall(limb, ball)
        if not firetouchinterest then return end
        pcall(function()
            firetouchinterest(limb, ball, 0)
            firetouchinterest(limb, ball, 1)
        end)
    end

    local function circleTick()
        if not CircleReach.Enabled then return end
        local char = Cache.Character or GetCharacter()
        local root = Cache.Root or GetRoot()
        if not char or not root then return end

        UpdateCircleReachVisual()

        local rx, ry, rz = getRadii()
        local limbs = activeLimbs()
        local balls = GetBallList()
        if #balls == 0 then
            local main = GetMainBall()
            if main then balls = { main } end
        end

        local now = tick()
        local rootPos = root.Position
        for i = 1, #balls do
            local ball = balls[i]
            if ball and ball.Parent and ball:IsA("BasePart") then
                local offset = ball.Position - rootPos
                -- ellipsoid test: (dx/rx)^2 + (dy/ry)^2 + (dz/rz)^2 <= 1
                local nx = offset.X / rx
                local ny = offset.Y / ry
                local nz = offset.Z / rz
                if (nx * nx + ny * ny + nz * nz) <= 1 then
                    for _, limbName in ipairs(limbs) do
                        local limb = char:FindFirstChild(limbName)
                        if limb and limb:IsA("BasePart") then
                            local key = limbName .. ":" .. tostring(ball)
                            local last = CircleReach.LastTouch[key] or 0
                            if (now - last) >= 0.08 then
                                CircleReach.LastTouch[key] = now
                                touchBall(limb, ball)
                            end
                        end
                    end
                end
            end
        end
    end

    -- FIXED: clean SetCircleReachEnabled
    SetCircleReachEnabled = function(v, silent)
        CircleReach.Enabled = v and true or false
        if CircleReach.Conn then
            pcall(function() CircleReach.Conn:Disconnect() end)
            CircleReach.Conn = nil
        end
        if CircleReach.Enabled then
            CircleReach.Conn = RunService.Heartbeat:Connect(function()
                pcall(circleTick)
            end)
            UpdateCircleReachVisual()
            if not silent and Notify then Notify("Circle Reach ON", 2) end
        else
            destroyVisual()
            CircleReach.LastTouch = {}
            if not silent and Notify then Notify("Circle Reach OFF", 2) end
        end
        if UIControls.CircleEnabled then
            pcall(function() UIControls.CircleEnabled:Set(CircleReach.Enabled) end)
        end
    end

    -- rebuild limb hitboxes when circle reach state changes (prevents intermittent arm phase-through as GK)
    do
        local _prevSetCircle = SetCircleReachEnabled
        SetCircleReachEnabled = function(v, silent)
            _prevSetCircle(v, silent)
            task.defer(function()
                if RemoveAllLimbHitboxes then pcall(RemoveAllLimbHitboxes) end
                if CreateAllLimbHitboxes then pcall(CreateAllLimbHitboxes) end
            end)
        end
    end

    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(1)
        if CircleReach.Enabled then
            UpdateCircleReachVisual()
        end
    end)

    LocalPlayer.CharacterRemoving:Connect(function()
        destroyVisual()
    end)
end

---------------------------------------------------------------------------
-- MOBILE GK BUTTONS + MACROS
---------------------------------------------------------------------------
local ccRunCombo
local CC_DEFAULT_POS, ccRefs

do
    CC = {
        mobEnabled = true,
        keepToSave = 0.08,
        gkHudVisible = true,
        btn = { size = 88, transparency = 0.12, locked = false, visible = true },
        positions = {},
    }
    CC_DEFAULT_POS = { ccKS = UDim2.new(0.62, 0, 0.66, 0), ccJKS = UDim2.new(0.76, 0, 0.66, 0) }
    ccRefs = {}

    local function ccFindTool(name)
        local bp = LocalPlayer:FindFirstChild("Backpack")
        local ch = LocalPlayer.Character
        return (bp and bp:FindFirstChild(name)) or (ch and ch:FindFirstChild(name))
    end
    local function ccGkTool()
        local tc = LocalPlayer.TeamColor
        if tc == BrickColor.new("Camo") then return ccFindTool("ToolGGK") end
        if tc == BrickColor.new("Really blue") then return ccFindTool("ToolBGK") end
        return ccFindTool("ToolGGK") or ccFindTool("ToolBGK")
    end
    local function ccGetModule(tool, which)
        if not tool then return nil end
        local m = tool:FindFirstChild(which)
        if m then return m end
        for _, c in ipairs(tool:GetChildren()) do if c.Name:lower() == which:lower() then return c end end
        return nil
    end
    local function ccKeep1()
        local mod = ccGetModule(ccGkTool(), "Keep1Module")
        if not mod then return false end
        task.spawn(function() pcall(function() require(mod).Keep1() end) end)
        return true
    end
    local function ccSave()
        local mod = ccGetModule(ccGkTool(), "SaveModule")
        if not mod then return false end
        task.spawn(function() pcall(function() require(mod).Save() end) end)
        return true
    end
    local function ccJump()
        local RS = game:GetService("ReplicatedStorage")
        local mgr = RS:FindFirstChild("Managers")
        local bm = mgr and mgr:FindFirstChild("BManager")
        local ani = bm and bm:FindFirstChild("Animate")
        local jmp = ani and ani:FindFirstChild("jump")
        if jmp then
            local ok = pcall(function()
                local mod = require(jmp)
                if type(mod) == "function" then mod()
                elseif type(mod) == "table" and type(mod.Play) == "function" then mod.Play()
                elseif type(mod) == "table" and type(mod.Jump) == "function" then mod.Jump() end
            end)
            if ok then return true end
        end
        local char = LocalPlayer.Character
        local h = char and char:FindFirstChildOfClass("Humanoid")
        if h and h.Parent and h.Health > 0 then
            pcall(function()
                h:SetStateEnabled(Enum.HumanoidStateType.Jumping, true)
                h:SetStateEnabled(Enum.HumanoidStateType.Freefall, true)
                h.Jump = true
                h:ChangeState(Enum.HumanoidStateType.Jumping)
            end)
            return true
        end
        return false
    end
    local function macro_KeepSave()
        -- INSTANT GK arm the moment the button is hit (no wait)
        if not GKHelper.Enabled then
            StartGKHelper()
        end
        task.delay(1.3, function()
            if GKHelper.Enabled then
                StopGKHelper()
            end
        end)
        -- keep + save still runs
        ccKeep1()
        task.wait(CC.keepToSave)
        ccSave()
    end
    local function macro_JumpKeepSave() ccJump() task.wait(CC.keepToSave) ccKeep1() task.wait(CC.keepToSave) ccSave() end
    local CC_MACROS = { KeepSave = macro_KeepSave, JumpKeepSave = macro_JumpKeepSave }
    local CC_BTNS = { { id = "ccKS", label = "K1\n+SAVE", combo = "KeepSave" }, { id = "ccJKS", label = "JUMP\nK1+SV", combo = "JumpKeepSave" } }

    local ccRoot = CoreGui
    pcall(function() local ok, hui = pcall(gethui) if ok and hui then ccRoot = hui end end)
    local ccOld = ccRoot:FindFirstChild("Fivelyss_MobileGK")
    if ccOld then ccOld:Destroy() end
    local ccGui = Instance.new("ScreenGui")
    ccGui.Name = "Fivelyss_MobileGK"
    ccGui.ResetOnSpawn = false
    ccGui.DisplayOrder = 2147483646
    ccGui.IgnoreGuiInset = true
    ccGui.Enabled = true
    ccGui.Parent = ccRoot

    local ccBusy = false
    ccRunCombo = function(id)
        local fn = CC_MACROS[id]
        if not fn or ccBusy then return end
        ccBusy = true
        task.spawn(function() pcall(fn) ccBusy = false end)
    end

    local function ccAddButton(def)
        local pos = CC.positions[def.id] or CC_DEFAULT_POS[def.id] or UDim2.new(0.5, 0, 0.5, 0)
        CC.positions[def.id] = pos
        local btn = Instance.new("TextButton")
        btn.Name = def.id
        btn.Size = UDim2.new(0, CC.btn.size, 0, CC.btn.size)
        btn.Position = pos
        btn.BackgroundColor3 = Color3.fromRGB(22, 12, 20)
        btn.BackgroundTransparency = CC.btn.transparency
        btn.TextColor3 = Color3.fromRGB(255, 105, 180)
        btn.TextTransparency = CC.btn.transparency
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 14
        btn.Text = def.label or def.id
        btn.BorderSizePixel = 0
        btn.Active = true
        btn.Visible = true
        btn.AutoButtonColor = false
        -- stay clickable even at transparency 1
        btn.Parent = ccGui
        local corner = Instance.new("UICorner") corner.CornerRadius = UDim.new(0, 10) corner.Parent = btn
        local stroke = Instance.new("UIStroke")
        stroke.Color = CC.btn.locked and Color3.fromRGB(255, 105, 180) or Color3.fromRGB(255, 200, 60)
        stroke.Thickness = 1.4
        stroke.Transparency = math.clamp(0.35 + CC.btn.transparency, 0, 1)
        stroke.Parent = btn
        local dragging, dragStart, startPos, fired = false, nil, nil, false
        btn.InputBegan:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
            if input.UserInputState and input.UserInputState ~= Enum.UserInputState.Begin then return end
            if CC.btn.locked then
                if CC.mobEnabled then ccRunCombo(def.combo) end
                fired = true
                return
            end
            dragging = true
            dragStart = input.Position
            startPos = btn.Position
            fired = false
        end)
        btn.InputChanged:Connect(function(input)
            if not dragging then return end
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                local d = input.Position - dragStart
                btn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
            end
        end)
        btn.InputEnded:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
            if dragging then
                dragging = false
                CC.positions[def.id] = btn.Position
                local movedDist = (input.Position - (dragStart or input.Position)).Magnitude
                if movedDist >= 6 then
                    pcall(function() SaveConfigToFile(CONFIG_FILE) end)
                elseif not fired and CC.mobEnabled then
                    ccRunCombo(def.combo)
                end
            end
            fired = false
        end)
        ccRefs[def.id] = { btn = btn, stroke = stroke }
    end
    for _, def in ipairs(CC_BTNS) do ccAddButton(def) end
    task.defer(function()
        CC.btn.visible = true
        ccGui.Enabled = true
        for _, r in pairs(ccRefs) do
            if r.btn then r.btn.Visible = true end
        end
        pcall(function()
            if PendingConfig and PendingConfig.mobile and PendingConfig.mobile.positions then
                ApplyMobilePositions(PendingConfig.mobile.positions)
            else
                LoadConfigFromFile(CONFIG_FILE)
            end
        end)
    end)

    ccSyncSizes = function() for _, r in pairs(ccRefs) do r.btn.Size = UDim2.new(0, CC.btn.size, 0, CC.btn.size) end end
    ccSyncOpacity = function()
        local t = math.clamp(CC.btn.transparency or 0, 0, 1)
        for _, r in pairs(ccRefs) do
            r.btn.BackgroundTransparency = t
            r.btn.TextTransparency = t
            if r.stroke then r.stroke.Transparency = math.clamp(0.35 + t, 0, 1) end
        end
        -- also push opacity to GK helper + eye so the whole mobile set fades together
        pcall(function()
            local gk = ccGui:FindFirstChild("ccGKHelper")
            if gk then
                gk.BackgroundTransparency = t
                gk.TextTransparency = t
                local st = gk:FindFirstChildOfClass("UIStroke")
                if st then st.Transparency = math.clamp(0.35 + t, 0, 1) end
            end
            local eye = ccGui:FindFirstChild("ccEyeToggle")
            if eye then
                eye.BackgroundTransparency = math.clamp(t * 0.9, 0, 0.95)
                eye.TextTransparency = t
                local st = eye:FindFirstChildOfClass("UIStroke")
                if st then st.Transparency = math.clamp(0.3 + t, 0, 1) end
            end
        end)
    end
    ccSyncLock = function()
        for _, r in pairs(ccRefs) do r.stroke.Color = CC.btn.locked and Color3.fromRGB(255, 105, 180) or Color3.fromRGB(255, 200, 60) end
        -- GK helper button respects the same lock
        pcall(function()
            local roots = {}
            if gethui then pcall(function() table.insert(roots, gethui()) end) end
            table.insert(roots, CoreGui)
            for _, root in ipairs(roots) do
                local host = root and root:FindFirstChild("Fivelyss_MobileGK")
                local gk = host and host:FindFirstChild("ccGKHelper")
                if gk then
                    local st = gk:FindFirstChildOfClass("UIStroke")
                    if st then
                        st.Color = CC.btn.locked and Color3.fromRGB(255, 105, 180) or Color3.fromRGB(255, 200, 60)
                    end
                end
            end
        end)
    end

    local function setGkHudVisible(on)
        CC.gkHudVisible = on and true or false
        local show = CC.gkHudVisible and CC.btn.visible
        for _, r in pairs(ccRefs) do
            if r.btn then r.btn.Visible = show end
        end
        local gk = ccGui:FindFirstChild("ccGKHelper")
        if gk then gk.Visible = show end
        local eye = ccGui:FindFirstChild("ccEyeToggle")
        if eye then
            eye.Text = CC.gkHudVisible and "👁" or "👁‍🗨"
            eye.TextColor3 = CC.gkHudVisible and Color3.fromRGB(255, 200, 220) or Color3.fromRGB(120, 90, 100)
            eye.BackgroundColor3 = CC.gkHudVisible and Color3.fromRGB(40, 12, 18) or Color3.fromRGB(18, 8, 10)
        end
    end

    ccSyncVisible = function()
        ccGui.Enabled = true
        setGkHudVisible(CC.gkHudVisible)
    end
    getgenv().Valkyrie_SetGkHud = setGkHudVisible

    do
        local eyeOld = ccGui:FindFirstChild("ccEyeToggle")
        if eyeOld then eyeOld:Destroy() end
        local eye = Instance.new("TextButton")
        eye.Name = "ccEyeToggle"
        eye.Size = UDim2.new(0, 52, 0, 52)
        eye.Position = CC.positions["ccEyeToggle"] or UDim2.new(0.02, 0, 0.42, 0)
        eye.BackgroundColor3 = Color3.fromRGB(40, 12, 18)
        eye.BackgroundTransparency = 0.1
        eye.Text = "👁"
        eye.TextSize = 22
        eye.Font = Enum.Font.GothamBold
        eye.TextColor3 = Color3.fromRGB(255, 200, 220)
        eye.BorderSizePixel = 0
        eye.AutoButtonColor = false
        eye.Active = true
        eye.Parent = ccGui
        Instance.new("UICorner", eye).CornerRadius = UDim.new(1, 0)
        local eyeStroke = Instance.new("UIStroke")
        eyeStroke.Color = Color3.fromRGB(220, 50, 70)
        eyeStroke.Thickness = 1.4
        eyeStroke.Parent = eye

        local dragging, dragStart, startPos, moved = false, nil, nil, false
        eye.InputBegan:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1
                and input.UserInputType ~= Enum.UserInputType.Touch then return end
            dragging = true
            moved = false
            dragStart = input.Position
            startPos = eye.Position
        end)
        eye.InputChanged:Connect(function(input)
            if not dragging then return end
            if input.UserInputType == Enum.UserInputType.MouseMovement
                or input.UserInputType == Enum.UserInputType.Touch then
                local d = input.Position - dragStart
                if d.Magnitude > 6 then moved = true end
                eye.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + d.X,
                    startPos.Y.Scale, startPos.Y.Offset + d.Y
                )
            end
        end)
        eye.InputEnded:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1
                and input.UserInputType ~= Enum.UserInputType.Touch then return end
            if not dragging then return end
            dragging = false
            if moved then
                CC.positions["ccEyeToggle"] = eye.Position
                pcall(function() SaveConfigToFile(CONFIG_FILE) end)
                return
            end
            setGkHudVisible(not CC.gkHudVisible)
            pcall(function()
                ConfigStore.gkHudVisible = CC.gkHudVisible
                SaveConfigToFile(CONFIG_FILE)
            end)
            if Notify then
                Notify(CC.gkHudVisible and "GK buttons ON" or "GK buttons OFF", 2)
            end
        end)
        setGkHudVisible(CC.gkHudVisible ~= false)
    end
end

---------------------------------------------------------------------------
-- GK HELPER
---------------------------------------------------------------------------
local GK_CONFIG
local CleanupGKHelper

do
    GK_CONFIG = {
        GOAL_WIDTH = 42,
        PITCH_GROUND_Y = 5.5,
        BALL_SPEED_THRESHOLD = 4.5,   -- ignore slow floaters harder
        FAST_SHOT_SPEED = 38,
        BLAST_SHOT_SPEED = 58,
        PREDICTION_MAX_T = 8.0,
        TRIGGER_T = 6.5,
        EARLY_T = 8.0,
        MOVE_SPEED = 180,
        EARLY_SPEED = 150,
        FAST_MOVE_SPEED = 240,
        BLAST_MOVE_SPEED = 320,
        ORIGINAL_SPEED = 20,
        GK_BASE_OFFSET = 1.2,
        MAX_COMMIT = 62,
        GRAVITY_FAST = 18,
        GRAVITY_SLOW = 30,
        AIR_FOLLOW_HEIGHT = 3.5,
        AIR_TRACK_LERP = 0.97,
        HIGH_BALL_Y = 8.5,
        HIGH_LERP = 0.985,
        AIR_MIN_TTG = 0.05,
        RETURN_FADE_TIME = 0.08,
        BALL_REFRESH_INTERVAL = 0.02,
        MIN_MOVE_DIST = 0.06,
        TOWARD_VEL = 0.25,
        FAR_PRE_RANGE = 180,
        PENALTY_DEPTH = 62,
        PENALTY_HALF_WIDTH = 30,
        MAX_RETREAT = 18,
        RETREAT_HIGH_STRAIGHT = 5.5,
        RETREAT_LOW_STRAIGHT = 1.8,
        FORWARD_BLEND = 0.58,
        LAT_SMOOTH = 0.72,
        LAT_SMOOTH_FAST = 0.94,
        CLOSE_LEAD = 0.78,
        BOOST_MAX = 340,
        BOOST_MAX_BLAST = 480,
    }

    GKHelper = {
        Enabled = false,
        Conn = nil,
        PredCircle = nil,
        InnerRing = nil,
        IsMoving = false,
        OriginalSpeed = 16,
        GoalPart = nil,
        GoalAxis = nil,
        GoalValue = nil,
        GoalSign = nil,
        LastTarget = nil,
        LastTargetAge = 0,
        AirMode = false,
        CachedBall = nil,
        LastBallRefresh = 0,
        SmoothLat = nil,
    }

    local GK_ZONE_DEPTH = 75
    local GK_ZONE_WIDTH = 48

    local function getGoalObjects()
        return Workspace:FindFirstChild("BlueGoal"), Workspace:FindFirstChild("GreenGoal")
    end

    local function extractGoalInfo(goalObj, hrpPos)
        if not goalObj then return nil end
        local pos
        if goalObj:IsA("Model") then
            local ok, pivot = pcall(function() return goalObj:GetPivot() end)
            if ok and pivot then pos = pivot.Position end
        elseif goalObj:IsA("BasePart") then
            pos = goalObj.Position
        end
        if not pos then
            local p = goalObj:FindFirstChildWhichIsA("BasePart", true)
            if p then pos = p.Position end
        end
        if not pos then return nil end
        local useZ = math.abs(pos.Z) >= math.abs(pos.X)
        local value = useZ and pos.Z or pos.X
        local sign
        if hrpPos then
            local hAxis = useZ and hrpPos.Z or hrpPos.X
            sign = (hAxis < value) and 1 or -1
        else
            sign = (value >= 0) and 1 or -1
        end
        return {
            Part = goalObj,
            Axis = useZ and "Z" or "X",
            Value = value,
            Sign = sign,
            Pos = pos,
        }
    end

    local function isInFrontOfGoal(hrp, info)
        if not hrp or not info then return false end
        local p = hrp.Position
        local axisDist, latDist
        if info.Axis == "Z" then
            axisDist = math.abs(p.Z - info.Value)
            latDist = math.min(math.abs(p.X), math.abs(p.X - (info.Pos and info.Pos.X or 0)))
        else
            axisDist = math.abs(p.X - info.Value)
            latDist = math.min(math.abs(p.Z), math.abs(p.Z - (info.Pos and info.Pos.Z or 0)))
        end
        return axisDist <= GK_ZONE_DEPTH and latDist <= GK_ZONE_WIDTH
    end

    local function pickOurGoal(hrp)
        local blue, green = getGoalObjects()
        local hp = hrp and hrp.Position
        local blueInfo = extractGoalInfo(blue, hp)
        local greenInfo = extractGoalInfo(green, hp)
        if not blueInfo and not greenInfo then return nil end

        if blueInfo and greenInfo then
            local hAxis = (blueInfo.Axis == "Z") and hp.Z or hp.X
            local dBlue = math.abs(hAxis - blueInfo.Value)
            local dGreen = math.abs(hAxis - greenInfo.Value)
            local closer = (dBlue <= dGreen) and blueInfo or greenInfo
            local closerDist = math.min(dBlue, dGreen)
            if closerDist <= GK_ZONE_DEPTH then
                GKHelper.GoalPart = closer.Part
                return closer
            end
            if closerDist <= GK_ZONE_DEPTH * 1.6 then
                GKHelper.GoalPart = closer.Part
                return closer
            end
            return nil
        end

        local only = blueInfo or greenInfo
        if only and isInFrontOfGoal(hrp, only) then
            GKHelper.GoalPart = only.Part
            return only
        end
        return nil
    end

    local function isBallTowardOurGoal(ballPos, ballVel, info)
        if not info or not info.Pos then return false end
        local goalPos = info.Pos
        local toGoal = goalPos - ballPos
        local dist = toGoal.Magnitude
        if dist < 2 then return false end
        local closeSpeed = ballVel:Dot(toGoal.Unit)
        local pAxis = (info.Axis == "Z") and ballPos.Z or ballPos.X
        local vAxis = (info.Axis == "Z") and ballVel.Z or ballVel.X
        local d = info.Value - pAxis
        if info.Sign > 0 and d < -3 then return false end
        if info.Sign < 0 and d > 3 then return false end
        if closeSpeed >= 0.35 then return true end
        if info.Sign > 0 and vAxis >= 0.25 and d > 0 then return true end
        if info.Sign < 0 and vAxis <= -0.25 and d < 0 then return true end
        local high = ballPos.Y >= (GK_CONFIG.HIGH_BALL_Y or 9) or ballVel.Y > 1.2
        if high and dist < (GK_CONFIG.FAR_PRE_RANGE or 140) and closeSpeed > -1 then
            if info.Sign > 0 and d > 0 then return true end
            if info.Sign < 0 and d < 0 then return true end
        end
        return false
    end

    -- only true when ball is inside the penalty box (past the 18-yard line toward goal)
    local function isBallInPenaltyArea(ballPos, info)
        if not info or not ballPos then return false end
        local depth = GK_CONFIG.PENALTY_DEPTH or 55
        local halfW = GK_CONFIG.PENALTY_HALF_WIDTH or 26
        local pAxis = (info.Axis == "Z") and ballPos.Z or ballPos.X
        local lat = (info.Axis == "Z") and ballPos.X or ballPos.Z
        local goalLat = info.Pos and ((info.Axis == "Z") and info.Pos.X or info.Pos.Z) or 0
        local axisDist = math.abs(pAxis - info.Value)
        -- must be on the pitch side of the goal line, not behind the net
        if info.Sign > 0 and pAxis > info.Value + 1 then return false end
        if info.Sign < 0 and pAxis < info.Value - 1 then return false end
        if axisDist > depth then return false end
        if math.abs(lat - goalLat) > halfW then return false end
        return true
    end

    local function predict(ballPos, ballVel, goalValue, dirSign, axis, ballSpeed)
        local vAxis = (axis == "Z") and ballVel.Z or ballVel.X
        local pAxis = (axis == "Z") and ballPos.Z or ballPos.X
        if (dirSign > 0 and vAxis <= 0.08) or (dirSign < 0 and vAxis >= -0.08) then return nil, nil end
        local d = goalValue - pAxis
        if (dirSign > 0 and d <= 0) or (dirSign < 0 and d >= 0) then return nil, nil end
        local t = d / vAxis
        if t <= 0 or t > GK_CONFIG.PREDICTION_MAX_T then return nil, nil end
        local high = ballPos.Y >= GK_CONFIG.HIGH_BALL_Y or ballVel.Y > 1.5
        local g = high and GK_CONFIG.GRAVITY_SLOW
            or ((ballSpeed > 55) and GK_CONFIG.GRAVITY_FAST or GK_CONFIG.GRAVITY_SLOW)
        local py = math.max(ballPos.Y + ballVel.Y * t - 0.5 * g * t * t, GK_CONFIG.PITCH_GROUND_Y)
        local px, pz
        if axis == "Z" then
            px = ballPos.X + ballVel.X * t
            pz = goalValue
        else
            px = goalValue
            pz = ballPos.Z + ballVel.Z * t
        end
        return Vector3.new(px, py, pz), t
    end

    local function resolveDepth(cur, ballAxis, sign, latAbs, isHigh)
        local base = GKHelper.GoalValue - sign * GK_CONFIG.GK_BASE_OFFSET
        local maxRet = GK_CONFIG.MAX_RETREAT or 14
        local fwd = GK_CONFIG.FORWARD_BLEND or 0.45
        local depth = cur
        local ballInFront = (sign > 0 and ballAxis < cur - 0.5) or (sign < 0 and ballAxis > cur + 0.5)
        local ballBehind = (sign > 0 and ballAxis > cur + 0.5) or (sign < 0 and ballAxis < cur - 0.5)
        local straight = latAbs < 7

        if straight and ballInFront then
            depth = cur + (ballAxis - cur) * (fwd * 0.55)
            local ret = isHigh and (GK_CONFIG.RETREAT_HIGH_STRAIGHT or 4.5) or (GK_CONFIG.RETREAT_LOW_STRAIGHT or 1.2)
            if sign > 0 then
                depth = math.min(depth + ret * 0.35, cur + ret)
            else
                depth = math.max(depth - ret * 0.35, cur - ret)
            end
        elseif straight and not ballInFront then
            local ret = isHigh and (GK_CONFIG.RETREAT_HIGH_STRAIGHT or 4.5) or (GK_CONFIG.RETREAT_LOW_STRAIGHT or 1.2)
            if sign > 0 then depth = cur + ret else depth = cur - ret end
        elseif ballInFront and latAbs < 14 then
            depth = cur + (ballAxis - cur) * (fwd * 0.4)
        elseif ballBehind or latAbs > 10 then
            local pull = (latAbs > 16) and 0.5 or 0.3
            depth = cur + (base - cur) * pull
        end

        if sign > 0 then
            local maxBack = math.min(base, cur + maxRet)
            depth = math.clamp(depth, cur - 18, maxBack)
        else
            local maxBack = math.max(base, cur - maxRet)
            depth = math.clamp(depth, maxBack, cur + 18)
        end
        return depth
    end

    local function resolveLateral(ballPos, vel, pred, ttg, axis)
        local half = GK_CONFIG.GOAL_WIDTH * 0.5
        local live = (axis == "Z") and ballPos.X or ballPos.Z
        local vLat = (axis == "Z") and vel.X or vel.Z
        local predLat = pred and ((axis == "Z") and pred.X or pred.Z) or live
        local absVLat = math.abs(vLat)
        local leadT = GK_CONFIG.CLOSE_LEAD or 0.78
        if ttg then leadT = math.clamp(ttg * 0.85, 0.22, 1.25) end
        if absVLat >= 40 then
            leadT = math.clamp(leadT + 0.55, 0.45, 1.55)
        elseif absVLat >= 22 then
            leadT = math.clamp(leadT + 0.3, 0.35, 1.3)
        end
        local lead = live + vLat * leadT
        local close = (ttg and ttg < 1.4) or false
        local raw
        if close then
            raw = lead * 0.72 + live * 0.12 + predLat * 0.16
        else
            local blend = 0.58
            if ttg and ttg < 2.4 then blend = 0.72 end
            if ttg and ttg < 1.6 then blend = 0.86 end
            if absVLat >= 30 then
                blend = math.min(blend + 0.18, 0.96)
            end
            raw = live * (1 - blend) + predLat * blend
            raw = raw * 0.55 + lead * 0.45
        end
        raw = math.clamp(raw, -half, half)
        local ballSpeed = vel.Magnitude
        local sm = GK_CONFIG.LAT_SMOOTH or 0.72
        if ballSpeed >= (GK_CONFIG.BLAST_SHOT_SPEED or 58) then
            sm = 0.99
        elseif ballSpeed >= (GK_CONFIG.FAST_SHOT_SPEED or 38) then
            sm = GK_CONFIG.LAT_SMOOTH_FAST or 0.94
        end
        if absVLat >= 45 then
            sm = math.max(sm, 0.995)
        elseif absVLat >= 28 then
            sm = math.max(sm, 0.97)
        end
        if GKHelper.SmoothLat == nil then
            GKHelper.SmoothLat = raw
        else
            local delta = raw - GKHelper.SmoothLat
            -- slow / low-lat: almost no movement (kills drift toward floaters)
            if ballSpeed < (GK_CONFIG.FAST_SHOT_SPEED or 38) and absVLat < 18 then
                GKHelper.SmoothLat = GKHelper.SmoothLat + delta * math.min(sm * 0.35, 0.4)
            elseif math.abs(delta) < 2.0 and ballSpeed < (GK_CONFIG.FAST_SHOT_SPEED or 38) then
                GKHelper.SmoothLat = GKHelper.SmoothLat + delta * sm
            else
                GKHelper.SmoothLat = GKHelper.SmoothLat + delta * math.min(sm + 0.12, 0.995)
            end
        end
        return math.clamp(GKHelper.SmoothLat, -half, half)
    end

    local function getNearestBall(hrpPos)
        local now = os.clock()
        if GKHelper.CachedBall and GKHelper.CachedBall.Parent and (now - GKHelper.LastBallRefresh) < GK_CONFIG.BALL_REFRESH_INTERVAL then
            return GKHelper.CachedBall
        end
        local best, bestD = nil, 200 * 200
        local list = GetBallList()
        if #list == 0 and #TrackedBalls > 0 then list = TrackedBalls end
        if #list == 0 then
            local main = GetMainBall()
            if main then list = { main } end
        end
        for i = 1, #list do
            local inst = list[i]
            if inst and inst.Parent and inst:IsA("BasePart") then
                local p = inst.Position
                local dx = p.X - hrpPos.X
                local dy = p.Y - hrpPos.Y
                local dz = p.Z - hrpPos.Z
                local d = dx*dx + dy*dy + dz*dz
                if d < bestD then bestD = d; best = inst end
            end
        end
        GKHelper.CachedBall = best
        GKHelper.LastBallRefresh = now
        return best
    end

    CleanupGKHelper = function()
        if GKHelper.Conn then
            pcall(function() GKHelper.Conn:Disconnect() end)
            GKHelper.Conn = nil
        end
        if GKHelper.PredCircle then pcall(function() GKHelper.PredCircle:Destroy() end) GKHelper.PredCircle = nil end
        if GKHelper.InnerRing  then pcall(function() GKHelper.InnerRing:Destroy()  end) GKHelper.InnerRing  = nil end
        GKHelper.IsMoving = false
        GKHelper.AirMode = false
        GKHelper.LastTarget = nil
        GKHelper.CachedBall = nil
        local hum = GetHumanoid()
        if hum then
            hum.WalkSpeed = GKHelper.OriginalSpeed
            pcall(function() hum:Move(Vector3.zero) end)
        end
    end

    StartGKHelper = function()
        if GKHelper.Enabled then return end
        CleanupGKHelper()
        local hrp = GetRoot()
        if not hrp then Notify("GK Helper: no character", 3) return end
        local blue, green = getGoalObjects()
        if not blue and not green then
            Notify("GK Helper: no BlueGoal or GreenGoal found", 4)
            return
        end
        local info = pickOurGoal(hrp)
        if info then
            GKHelper.GoalAxis = info.Axis
            GKHelper.GoalValue = info.Value
            GKHelper.GoalSign = info.Sign
        end
        local hum = GetHumanoid()
        if hum then GKHelper.OriginalSpeed = hum.WalkSpeed end
        local parent = Workspace:FindFirstChild("Debris") or Workspace

        local ring = Instance.new("Part")
        ring.Name = "GK_PredictionCircle"
        ring.Anchored = true
        ring.CanCollide = false
        ring.Shape = Enum.PartType.Cylinder
        ring.Size = Vector3.new(0.15, 5, 5)
        ring.Color = Color3.fromRGB(0, 255, 80)
        ring.Material = Enum.Material.Neon
        ring.Transparency = 0.3
        ring.CFrame = CFrame.new(0, -100, 0)
        ring.Parent = parent
        GKHelper.PredCircle = ring

        local inner = Instance.new("Part")
        inner.Name = "GK_InnerRing"
        inner.Anchored = true
        inner.CanCollide = false
        inner.Shape = Enum.PartType.Cylinder
        inner.Size = Vector3.new(0.2, 2.5, 2.5)
        inner.Color = Color3.fromRGB(50, 255, 50)
        inner.Material = Enum.Material.Neon
        inner.Transparency = 0.1
        inner.CFrame = CFrame.new(0, -100, 0)
        inner.Parent = parent
        GKHelper.InnerRing = inner

        GKHelper.Enabled = true
        local idleReturnAccum = 0

        GKHelper.Conn = RunService.Heartbeat:Connect(function(dt)
            if not GKHelper.Enabled then return end
            local hum = Cache.Humanoid
            local hrp = Cache.Root
            if not hum or not hrp or not hum.Parent then return end

            local info2 = pickOurGoal(hrp)
            if not info2 then
                if GKHelper.IsMoving then
                    hum.WalkSpeed = GKHelper.OriginalSpeed
                    pcall(function() hum:Move(Vector3.zero) end)
                    GKHelper.IsMoving = false
                    GKHelper.LastTarget = nil
                end
                if GKHelper.PredCircle then GKHelper.PredCircle.CFrame = CFrame.new(0, -100, 0) end
                if GKHelper.InnerRing  then GKHelper.InnerRing.CFrame  = CFrame.new(0, -100, 0) end
                return
            end
            GKHelper.GoalAxis = info2.Axis
            GKHelper.GoalValue = info2.Value
            GKHelper.GoalSign = info2.Sign
            GKHelper.GoalPart = info2.Part

            local ball = getNearestBall(hrp.Position)

            if not ball then
                idleReturnAccum = idleReturnAccum + dt
                if GKHelper.LastTarget and idleReturnAccum > GKHelper.LastTargetAge + GK_CONFIG.RETURN_FADE_TIME then
                    if GKHelper.IsMoving then
                        hum.WalkSpeed = GKHelper.OriginalSpeed
                        pcall(function() hum:Move(Vector3.zero) end)
                        GKHelper.IsMoving = false
                        GKHelper.LastTarget = nil
                    end
                end
                if GKHelper.PredCircle then GKHelper.PredCircle.CFrame = CFrame.new(0, -100, 0) end
                if GKHelper.InnerRing  then GKHelper.InnerRing.CFrame  = CFrame.new(0, -100, 0) end
                return
            end

            local vel = ball.AssemblyLinearVelocity
            local speed = vel.Magnitude
            local ballPos = ball.Position

            -- hard gate: ignore everything still outside / in front of the penalty box
            if not isBallInPenaltyArea(ballPos, info2) then
                if GKHelper.IsMoving then
                    GKHelper.LastTargetAge = (GKHelper.LastTargetAge or 0) + dt
                    if GKHelper.LastTargetAge > GK_CONFIG.RETURN_FADE_TIME then
                        hum.WalkSpeed = GKHelper.OriginalSpeed
                        pcall(function() hum:Move(Vector3.zero) end)
                        GKHelper.IsMoving = false
                        GKHelper.LastTarget = nil
                    end
                end
                GKHelper.SmoothLat = nil
                if GKHelper.PredCircle then GKHelper.PredCircle.CFrame = CFrame.new(0, -100, 0) end
                if GKHelper.InnerRing  then GKHelper.InnerRing.CFrame  = CFrame.new(0, -100, 0) end
                return
            end

                        -- slow ball: full stop. never drift toward / away from a floater
            if speed < (GK_CONFIG.BALL_SPEED_THRESHOLD or 4.5) then
                if GKHelper.IsMoving then
                    hum.WalkSpeed = GKHelper.OriginalSpeed
                    pcall(function() hum:Move(Vector3.zero) end)
                    pcall(function()
                        local v = hrp.AssemblyLinearVelocity
                        hrp.AssemblyLinearVelocity = Vector3.new(0, v.Y, 0)
                    end)
                    GKHelper.IsMoving = false
                    GKHelper.LastTarget = nil
                end
                GKHelper.SmoothLat = nil
                if GKHelper.PredCircle then GKHelper.PredCircle.CFrame = CFrame.new(0, -100, 0) end
                if GKHelper.InnerRing  then GKHelper.InnerRing.CFrame  = CFrame.new(0, -100, 0) end
                return
            end

            local pred, ttg = predict(ballPos, vel, GKHelper.GoalValue, GKHelper.GoalSign, GKHelper.GoalAxis, speed)
            local toward = isBallTowardOurGoal(ballPos, vel, info2)

            if not toward and not (pred and ttg and ttg < GK_CONFIG.EARLY_T) then
                if GKHelper.IsMoving then
                    GKHelper.LastTargetAge = GKHelper.LastTargetAge + dt
                    if GKHelper.LastTargetAge > GK_CONFIG.RETURN_FADE_TIME then
                        hum.WalkSpeed = GKHelper.OriginalSpeed
                        pcall(function() hum:Move(Vector3.zero) end)
                        GKHelper.IsMoving = false
                    end
                end
                if GKHelper.PredCircle then GKHelper.PredCircle.CFrame = CFrame.new(0, -100, 0) end
                if GKHelper.InnerRing  then GKHelper.InnerRing.CFrame  = CFrame.new(0, -100, 0) end
                return
            end

            local heightAboveGK = ballPos.Y - hrp.Position.Y
            local inAir = (heightAboveGK > GK_CONFIG.AIR_FOLLOW_HEIGHT)
                or (vel.Y > 1.5)
                or (ballPos.Y > GK_CONFIG.HIGH_BALL_Y)
            GKHelper.AirMode = inAir
            local isHigh = inAir
                or (pred and pred.Y >= GK_CONFIG.HIGH_BALL_Y)
                or (ballPos.Y >= GK_CONFIG.HIGH_BALL_Y)

            local targetX, targetZ, doMove, useFullSpeed
            useFullSpeed = true

            if inAir then
                local lat = resolveLateral(ballPos, vel, pred, ttg, GKHelper.GoalAxis)
                if GKHelper.GoalAxis == "Z" then
                    targetX = lat
                    local latAbs = math.abs(lat - hrp.Position.X)
                    targetZ = resolveDepth(hrp.Position.Z, ballPos.Z, GKHelper.GoalSign, latAbs, true)
                else
                    targetZ = lat
                    local latAbs = math.abs(lat - hrp.Position.Z)
                    targetX = resolveDepth(hrp.Position.X, ballPos.X, GKHelper.GoalSign, latAbs, true)
                end
                doMove = true
                useFullSpeed = true
                local ringX = pred and pred.X or ballPos.X
                local ringZ = pred and pred.Z or ballPos.Z
                if GKHelper.PredCircle then
                    GKHelper.PredCircle.CFrame = CFrame.new(ringX, GK_CONFIG.PITCH_GROUND_Y + 0.2, ringZ) * CFrame.Angles(0, 0, math.rad(90))
                end
                if GKHelper.InnerRing then
                    GKHelper.InnerRing.CFrame = CFrame.new(ringX, GK_CONFIG.PITCH_GROUND_Y + 0.25, ringZ) * CFrame.Angles(0, 0, math.rad(90))
                end
            elseif pred and ttg then
                local lat = resolveLateral(ballPos, vel, pred, ttg, GKHelper.GoalAxis)
                if GKHelper.GoalAxis == "Z" then
                    targetX = lat
                    local latAbs = math.abs(lat - hrp.Position.X)
                    targetZ = resolveDepth(hrp.Position.Z, ballPos.Z, GKHelper.GoalSign, latAbs, isHigh)
                else
                    targetZ = lat
                    local latAbs = math.abs(lat - hrp.Position.Z)
                    targetX = resolveDepth(hrp.Position.X, ballPos.X, GKHelper.GoalSign, latAbs, isHigh)
                end
                if ttg < GK_CONFIG.TRIGGER_T then
                    doMove = true
                    useFullSpeed = true
                elseif ttg < GK_CONFIG.EARLY_T then
                    doMove = true
                    useFullSpeed = true
                else
                    doMove = false
                end
                if doMove then
                    if GKHelper.PredCircle then
                        GKHelper.PredCircle.CFrame = CFrame.new(pred.X, GK_CONFIG.PITCH_GROUND_Y + 0.2, pred.Z) * CFrame.Angles(0, 0, math.rad(90))
                    end
                    if GKHelper.InnerRing then
                        GKHelper.InnerRing.CFrame = CFrame.new(pred.X, GK_CONFIG.PITCH_GROUND_Y + 0.25, pred.Z) * CFrame.Angles(0, 0, math.rad(90))
                    end
                end
            else
                local axisVal = (GKHelper.GoalAxis == "Z") and ballPos.Z or ballPos.X
                local axisVel = (GKHelper.GoalAxis == "Z") and vel.Z or vel.X
                local tv = GK_CONFIG.TOWARD_VEL or 0.45
                local towardGate = (GKHelper.GoalSign > 0 and axisVel > tv) or (GKHelper.GoalSign < 0 and axisVel < -tv)
                local distToGoal = math.abs(GKHelper.GoalValue - axisVal)
                local far = GK_CONFIG.FAR_PRE_RANGE or 140
                if (towardGate or inAir or isHigh) and distToGoal < far and speed >= GK_CONFIG.BALL_SPEED_THRESHOLD then
                    local lat = resolveLateral(ballPos, vel, pred, ttg, GKHelper.GoalAxis)
                    if GKHelper.GoalAxis == "Z" then
                        targetX = lat
                        local latAbs = math.abs(lat - hrp.Position.X)
                        targetZ = resolveDepth(hrp.Position.Z, ballPos.Z, GKHelper.GoalSign, latAbs, isHigh)
                    else
                        targetZ = lat
                        local latAbs = math.abs(lat - hrp.Position.Z)
                        targetX = resolveDepth(hrp.Position.X, ballPos.X, GKHelper.GoalSign, latAbs, isHigh)
                    end
                    doMove = true
                    useFullSpeed = true
                else
                    doMove = false
                end
            end

            if doMove then
                idleReturnAccum = 0
                local target = Vector3.new(targetX, hrp.Position.Y, targetZ)
                GKHelper.LastTarget = target
                GKHelper.LastTargetAge = 0
                local fastShot = speed >= (GK_CONFIG.FAST_SHOT_SPEED or 55)
                local blastShot = speed >= (GK_CONFIG.BLAST_SHOT_SPEED or 85)

                if inAir or isHigh or fastShot then
                    local curPos = hrp.Position
                    local lerpAmt = isHigh and GK_CONFIG.HIGH_LERP or GK_CONFIG.AIR_TRACK_LERP
                    if blastShot then lerpAmt = 0.995
                    elseif fastShot then lerpAmt = math.max(lerpAmt, 0.985) end
                    local lerpX = curPos.X + (target.X - curPos.X) * lerpAmt
                    local lerpZ = curPos.Z + (target.Z - curPos.Z) * lerpAmt
                    target = Vector3.new(lerpX, curPos.Y, lerpZ)
                end

                local dxT = target.X - hrp.Position.X
                local dzT = target.Z - hrp.Position.Z
                local flatDist = math.sqrt(dxT*dxT + dzT*dzT)
                local commitExtra = blastShot and 40 or (fastShot and 32 or 22)
                if flatDist > GK_CONFIG.MIN_MOVE_DIST and flatDist < GK_CONFIG.MAX_COMMIT + commitExtra then
                    local spd = useFullSpeed and GK_CONFIG.MOVE_SPEED or GK_CONFIG.EARLY_SPEED
                    if isHigh or inAir then spd = math.max(spd, GK_CONFIG.MOVE_SPEED) end
                    if fastShot then spd = math.max(spd, GK_CONFIG.FAST_MOVE_SPEED or 240) end
                    if blastShot then spd = math.max(spd, GK_CONFIG.BLAST_MOVE_SPEED or 320) end
                    if hum.WalkSpeed ~= spd then hum.WalkSpeed = spd end
                    hum:MoveTo(target)

                    local urgent = (ttg and ttg < 2.2)
                        or (fastShot and ttg and ttg < 3.0)
                        or (blastShot)
                        or (inAir and flatDist > 0.6)
                        or (isHigh and flatDist > 0.5)
                        or (fastShot and flatDist > 0.35)
                    if urgent and flatDist > 0.1 then
                        local dir = Vector3.new(dxT, 0, dzT)
                        local mag = dir.Magnitude
                        if mag > 0.04 then
                            dir = dir / mag
                            local boostCap = blastShot and (GK_CONFIG.BOOST_MAX_BLAST or 480) or (fastShot and (GK_CONFIG.BOOST_MAX or 340) or 200)
                            local vLatNow = (GKHelper.GoalAxis == "Z") and vel.X or vel.Z
                            local absVLatNow = math.abs(vLatNow or 0)
                            if absVLatNow >= 40 then
                                boostCap = math.max(boostCap, 520)
                            elseif absVLatNow >= 25 then
                                boostCap = math.max(boostCap, 400)
                            end
                            local boost
                            if blastShot then
                                boost = math.clamp(flatDist / math.max(ttg or 0.14, 0.04), 140, boostCap)
                            elseif ttg and ttg < 0.35 then
                                boost = math.clamp(flatDist / math.max(ttg, 0.04), 120, boostCap)
                            elseif ttg and ttg < 0.75 then
                                boost = math.clamp(flatDist / math.max(ttg, 0.06), 95, boostCap * 0.95)
                            elseif fastShot or absVLatNow >= 28 then
                                boost = math.clamp(100 + flatDist * 5.5, 110, boostCap * 0.97)
                            elseif inAir or isHigh then
                                boost = math.clamp(70 + flatDist * 3.6, 70, 180)
                            else
                                boost = math.clamp(55 + flatDist * 2.4, 55, 140)
                            end
                            local curY = hrp.AssemblyLinearVelocity.Y
                            hrp.AssemblyLinearVelocity = Vector3.new(dir.X * boost, curY, dir.Z * boost)
                        end
                    end
                    GKHelper.IsMoving = true
                else
                    if GKHelper.IsMoving then
                        hum.WalkSpeed = GKHelper.OriginalSpeed
                        pcall(function() hum:Move(Vector3.zero) end)
                        GKHelper.IsMoving = false
                    end
                end
            else
                GKHelper.LastTargetAge = GKHelper.LastTargetAge + dt
                if GKHelper.IsMoving and GKHelper.LastTargetAge > GK_CONFIG.RETURN_FADE_TIME then
                    hum.WalkSpeed = GKHelper.OriginalSpeed
                    pcall(function() hum:Move(Vector3.zero) end)
                    GKHelper.IsMoving = false
                end
                if GKHelper.PredCircle then GKHelper.PredCircle.CFrame = CFrame.new(0, -100, 0) end
                if GKHelper.InnerRing  then GKHelper.InnerRing.CFrame  = CFrame.new(0, -100, 0) end
            end
        end)
    end

    StopGKHelper = function()
        GKHelper.Enabled = false
        GKHelper.GoalPart = nil
        GKHelper.GoalAxis = nil
        GKHelper.GoalValue = nil
        GKHelper.GoalSign = nil
        CleanupGKHelper()
    end

    -- Mobile GK toggle button
    do
        local parentGui = nil
        pcall(function()
            local ok, hui = pcall(gethui)
            if ok and hui then parentGui = hui end
        end)
        if not parentGui then parentGui = CoreGui end
        local host = parentGui:FindFirstChild("Fivelyss_MobileGK")
        if not host then
            host = Instance.new("ScreenGui")
            host.Name = "Fivelyss_MobileGK"
            host.ResetOnSpawn = false
            host.IgnoreGuiInset = true
            host.DisplayOrder = 2147483646
            host.Parent = parentGui
        end
        host.Enabled = true

        local old = host:FindFirstChild("ccGKHelper")
        if old then old:Destroy() end

        local btn = Instance.new("TextButton")
        btn.Name = "ccGKHelper"
        btn.Size = UDim2.new(0, 88, 0, 88)
        btn.Position = UDim2.new(0.48, 0, 0.66, 0)
        btn.BackgroundColor3 = Color3.fromRGB(22, 12, 20)
        btn.BackgroundTransparency = 0.12
        btn.TextColor3 = Color3.fromRGB(255, 105, 180)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 13
        btn.Text = "GK\nOFF"
        btn.BorderSizePixel = 0
        btn.AutoButtonColor = false
        btn.Active = true
        btn.Parent = host
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
        local stroke = Instance.new("UIStroke")
        stroke.Color = Color3.fromRGB(120, 120, 120)
        stroke.Thickness = 1.6
        stroke.Parent = btn

        local function refreshBtn()
            if GKHelper.Enabled then
                btn.Text = "GK\nON"
                btn.BackgroundColor3 = Color3.fromRGB(40, 18, 32)
                btn.TextColor3 = Color3.fromRGB(120, 255, 160)
            else
                btn.Text = "GK\nOFF"
                btn.BackgroundColor3 = Color3.fromRGB(22, 12, 20)
                btn.TextColor3 = Color3.fromRGB(255, 105, 180)
            end
            -- lock color: pink when locked, yellow/green when free
            if CC and CC.btn and CC.btn.locked then
                stroke.Color = Color3.fromRGB(255, 105, 180)
            else
                stroke.Color = GKHelper.Enabled and Color3.fromRGB(80, 255, 120) or Color3.fromRGB(255, 200, 60)
            end
            -- respect global mobile transparency
            if CC and CC.btn and CC.btn.transparency then
                local t = math.clamp(CC.btn.transparency, 0, 1)
                btn.BackgroundTransparency = t
                btn.TextTransparency = t
                stroke.Transparency = math.clamp(0.35 + t, 0, 1)
            end
        end
        refreshBtn()

        local dragging, dragStart, startPos, moved, fired = false, nil, nil, false, false
        btn.InputBegan:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1
                and input.UserInputType ~= Enum.UserInputType.Touch then return end
            -- locked = no drag, just toggle on release
            if CC and CC.btn and CC.btn.locked then
                if GKHelper.Enabled then
                    StopGKHelper()
                    if Notify then Notify("GK Helper OFF", 2) end
                else
                    StartGKHelper()
                    if Notify then Notify("GK Helper ON", 2) end
                end
                refreshBtn()
                fired = true
                return
            end
            dragging = true
            moved = false
            fired = false
            dragStart = input.Position
            startPos = btn.Position
        end)
        btn.InputChanged:Connect(function(input)
            if not dragging then return end
            if input.UserInputType == Enum.UserInputType.MouseMovement
                or input.UserInputType == Enum.UserInputType.Touch then
                local d = input.Position - dragStart
                if d.Magnitude > 6 then moved = true end
                btn.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + d.X,
                    startPos.Y.Scale, startPos.Y.Offset + d.Y
                )
            end
        end)
        btn.InputEnded:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1
                and input.UserInputType ~= Enum.UserInputType.Touch then return end
            if fired then fired = false return end
            if not dragging then return end
            dragging = false
            if moved then
                if CC then
                    CC.positions = CC.positions or {}
                    CC.positions["ccGKHelper"] = btn.Position
                end
                pcall(function() SaveConfigToFile(CONFIG_FILE) end)
                return
            end
            if GKHelper.Enabled then
                StopGKHelper()
                if Notify then Notify("GK Helper OFF", 2) end
            else
                StartGKHelper()
                if Notify then Notify("GK Helper ON", 2) end
            end
            refreshBtn()
        end)

        if CC and CC.positions and CC.positions["ccGKHelper"] then
            btn.Position = CC.positions["ccGKHelper"]
        elseif PendingConfig and PendingConfig.mobile and PendingConfig.mobile.positions and PendingConfig.mobile.positions["ccGKHelper"] then
            local u = tableToUDim2(PendingConfig.mobile.positions["ccGKHelper"])
            if u then btn.Position = u end
        end

        task.spawn(function()
            while btn and btn.Parent do
                refreshBtn()
                task.wait(0.4)
            end
        end)

        getgenv().Fivelyss_ToggleGKHelper = function()
            if GKHelper.Enabled then StopGKHelper() else StartGKHelper() end
            refreshBtn()
        end
    end
end

---------------------------------------------------------------------------
-- UI SHELL
---------------------------------------------------------------------------
local TweenService = game:GetService("TweenService")

local P = {
	bg = Color3.fromRGB(12,4,6), panel = Color3.fromRGB(18,6,8), rail = Color3.fromRGB(14,5,7),
	top = Color3.fromRGB(22,8,10), card = Color3.fromRGB(28,10,12), cardHov = Color3.fromRGB(42,14,16),
	line = Color3.fromRGB(90,25,30), off = Color3.fromRGB(50,18,22), accent = Color3.fromRGB(220,40,50),
	accentD = Color3.fromRGB(140,20,28), glow = Color3.fromRGB(255,120,130),
	hot = Color3.fromRGB(255,240,242), text = Color3.fromRGB(250,235,235), dim = Color3.fromRGB(170,120,125),
	faint = Color3.fromRGB(110,70,75), good = Color3.fromRGB(90,230,160), warn = Color3.fromRGB(255,190,90),
	bad = Color3.fromRGB(255,70,80),
}

local function font(name, weight)
	local ok, f = pcall(function() return Font.new("rbxasset://fonts/families/"..name..".json", weight) end)
	if ok then return f end
	return Font.fromEnum(Enum.Font.Gotham)
end
local F_R = font("BuilderSans", Enum.FontWeight.Regular)
local F_M = font("BuilderSans", Enum.FontWeight.Medium)
local F_S = font("BuilderSans", Enum.FontWeight.SemiBold)
local F_B = font("BuilderSans", Enum.FontWeight.Bold)
local F_D = font("Michroma", Enum.FontWeight.Regular)
local F_MO = font("RobotoMono", Enum.FontWeight.Regular)

local function tw(obj, t, props, style, dir)
	local ti = TweenInfo.new(t, style or Enum.EasingStyle.Quart, dir or Enum.EasingDirection.Out)
	local twn = TweenService:Create(obj, ti, props); twn:Play(); return twn
end
local function corner(o, r) local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, r); c.Parent = o; return c end
local function stroke(o, col, th, tr)
	local s = Instance.new("UIStroke"); s.Color = col; s.Thickness = th or 1; s.Transparency = tr or 0
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border; s.Parent = o; return s
end
local function fr(parent, z)
	local f = Instance.new("Frame"); f.BackgroundTransparency = 1; f.BorderSizePixel = 0; f.ZIndex = z or 2; f.Parent = parent; return f
end
local function bx(parent, col, z)
	local f = fr(parent, z); f.BackgroundColor3 = col; f.BackgroundTransparency = 0; return f
end
local function tx(parent, text, face, size, col)
	local t = Instance.new("TextLabel"); t.BackgroundTransparency = 1; t.Text = text; t.FontFace = face
	t.TextSize = size; t.TextColor3 = col; t.TextXAlignment = Enum.TextXAlignment.Left
	t.TextYAlignment = Enum.TextYAlignment.Center; t.Parent = parent; return t
end
local function sheen(parent, z)
	local s = Instance.new("Frame"); s.BackgroundColor3 = Color3.new(1,1,1); s.BackgroundTransparency = 0.94
	s.BorderSizePixel = 0; s.Size = UDim2.fromScale(1,1); s.ZIndex = z or (parent.ZIndex+1); s.Parent = parent
	local g = Instance.new("UIGradient"); g.Rotation = 115
	g.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0,0.5), NumberSequenceKeypoint.new(0.4,0.9), NumberSequenceKeypoint.new(1,1)
	}); g.Parent = s; return s
end
local function grad(o, a, b, rot)
	local g = Instance.new("UIGradient"); g.Color = ColorSequence.new(a, b); g.Rotation = rot or 90; g.Parent = o; return g
end
local function glow(inst, col, layers, step, rad, start)
	layers = layers or 2; step = step or 2; rad = rad or 10; start = start or 0.55
	local out = {}
	for i = 1, layers do
		local g = Instance.new("Frame")
		g.AnchorPoint = Vector2.new(0.5, 0.5); g.Position = UDim2.fromScale(0.5, 0.5)
		g.Size = UDim2.new(1, i*step*2, 1, i*step*2); g.BackgroundTransparency = 1; g.BorderSizePixel = 0
		g.ZIndex = (inst.ZIndex or 2) - 1; g.Parent = inst
		corner(g, rad + i*step); out[i] = stroke(g, col, 1.2, start + i*0.14)
	end
	return out
end
local MARK_IMG = "rbxassetid://1904867353"
local function drawMark(parent, size, z)
	local h = fr(parent, z); h.Size = UDim2.fromOffset(size, size)
	local ring = fr(h, z); ring.AnchorPoint = Vector2.new(0.5,0.5); ring.Position = UDim2.fromScale(0.5,0.5)
	ring.Size = UDim2.fromOffset(size, size); corner(ring, size); stroke(ring, P.accent, 1.4, 0.2)
	local img = Instance.new("ImageLabel"); img.BackgroundTransparency = 1; img.Size = UDim2.fromScale(0.86,0.86)
	img.AnchorPoint = Vector2.new(0.5,0.5); img.Position = UDim2.fromScale(0.5,0.5); img.Image = MARK_IMG
	img.ImageColor3 = P.accent; img.ScaleType = Enum.ScaleType.Fit; img.ZIndex = z+1; img.Parent = h
	local core = bx(h, P.accent, z+3); core.AnchorPoint = Vector2.new(0.5,0.5); core.Position = UDim2.fromScale(0.5,0.5)
	core.Size = UDim2.fromOffset(size*0.12, size*0.12); core.BackgroundTransparency = 0.3; corner(core, size)
	local pulse = fr(h, z); pulse.AnchorPoint = Vector2.new(0.5,0.5); pulse.Position = UDim2.fromScale(0.5,0.5)
	local ps = stroke(pulse, P.glow, 1.1, 1); corner(pulse, size)
	local t0 = 0
	local conn = RunService.RenderStepped:Connect(function(dt)
		t0 += dt; local f = (t0*0.55)%1
		pulse.Size = UDim2.fromOffset(size*(0.28+f*0.8), size*(0.28+f*0.8))
		ps.Transparency = 0.3+f*0.7
		img.Rotation = math.sin(t0*0.7)*5
		local s = size*0.16 + math.sin(t0*2.2)*size*0.03; core.Size = UDim2.fromOffset(s,s)
	end)
	h.Destroying:Connect(function() pcall(function() conn:Disconnect() end) end)
	return h
end

local function mount(name)
	local sg = Instance.new("ScreenGui"); sg.Name = name; sg.ResetOnSpawn = false; sg.IgnoreGuiInset = true
	sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; sg.DisplayOrder = 100000
	local ok = pcall(function()
		local h = gethui and gethui(); if h then sg.Parent = h else sg.Parent = CoreGui end
	end)
	if not ok or not sg.Parent then sg.Parent = LocalPlayer:WaitForChild("PlayerGui") end
	return sg
end

local function drag(handle, target)
	local dragging, startMouse, startPos, activeInput
	handle.Active = true
	handle.Selectable = true
	pcall(function() handle.ZIndex = math.max(handle.ZIndex or 1, 6) end)
	handle.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			activeInput = i
			startMouse = i.Position
			startPos = target.AbsolutePosition
		end
	end)
	handle.InputEnded:Connect(function(i)
		if i == activeInput or i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = false
			activeInput = nil
		end
	end)
	UserInputService.InputEnded:Connect(function(i)
		if i == activeInput or (dragging and (i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch)) then
			dragging = false
			activeInput = nil
		end
	end)
	UserInputService.InputChanged:Connect(function(i)
		if not dragging then return end
		if i.UserInputType ~= Enum.UserInputType.MouseMovement and i.UserInputType ~= Enum.UserInputType.Touch then return end
		local delta = i.Position - startMouse
		local parentAbs = target.Parent and target.Parent.AbsolutePosition or Vector2.zero
		target.Position = UDim2.fromOffset(startPos.X + delta.X - parentAbs.X, startPos.Y + delta.Y - parentAbs.Y)
		target.AnchorPoint = Vector2.new(0, 0)
	end)
end

local function enableEdgeWindowDrag(target, margin)
	margin = margin or 36
	local dragging, startMouse, startPos
	local function nearEdge(pos)
		local ap, asz = target.AbsolutePosition, target.AbsoluteSize
		if pos.X < ap.X or pos.Y < ap.Y or pos.X > ap.X + asz.X or pos.Y > ap.Y + asz.Y then return false end
		local l = pos.X - ap.X
		local r = ap.X + asz.X - pos.X
		local t = pos.Y - ap.Y
		local b = ap.Y + asz.Y - pos.Y
		return l <= margin or r <= margin or t <= margin or b <= margin
	end
	UserInputService.InputBegan:Connect(function(i)
		if i.UserInputType ~= Enum.UserInputType.MouseButton1 and i.UserInputType ~= Enum.UserInputType.Touch then return end
		if not target.Visible or not target.Parent then return end
		if not nearEdge(i.Position) then return end
		dragging = true
		startMouse = i.Position
		startPos = target.AbsolutePosition
	end)
	UserInputService.InputChanged:Connect(function(i)
		if not dragging then return end
		if i.UserInputType ~= Enum.UserInputType.MouseMovement and i.UserInputType ~= Enum.UserInputType.Touch then return end
		local delta = i.Position - startMouse
		local parentAbs = target.Parent and target.Parent.AbsolutePosition or Vector2.zero
		target.Position = UDim2.fromOffset(startPos.X + delta.X - parentAbs.X, startPos.Y + delta.Y - parentAbs.Y)
		target.AnchorPoint = Vector2.new(0, 0)
	end)
	UserInputService.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end)
end

for _, root in ipairs({ (gethui and gethui()) or nil, CoreGui, LocalPlayer:FindFirstChild("PlayerGui") }) do
	if root then
		pcall(function()
			for _, g in ipairs(root:GetChildren()) do
				if g:IsA("ScreenGui") then
					local n = g.Name
					if n == "Fivelyss_MobileGK" then
						-- keep
					elseif n:match("^Valkyrie_Boot") or n:match("^Valkyrie_ui$") or n:find("Obsidian") or n:match("^Valkyrie_%d") then
						g:Destroy()
					end
				end
			end
		end)
	end
end

local W, H, SIDE, TOP = 720, 500, 168, 48
local function isTouch() return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled end
local function windowScale()
	local cam = Workspace.CurrentCamera; local vp = cam and cam.ViewportSize
	if not vp or vp.Y <= 0 then return 1 end
	if isTouch() then
		local short = math.min(vp.X, vp.Y)
		if short < 500 then return math.clamp(math.min((vp.Y*0.88)/H, (vp.X*0.96)/W), 0.42, 0.85) end
		return math.clamp(math.min((vp.Y*0.76)/H, (vp.X*0.82)/W), 0.74, 1.15)
	end
	return math.clamp(math.min((vp.Y*0.55)/H, (vp.X*0.42)/W), 0.9, 1.35)
end

local gui = mount("Valkyrie_ui")
local main = bx(gui, P.panel, 2)
main.BackgroundTransparency = 0.08; main.Name = "Main"; main.Size = UDim2.fromOffset(W, H)
main.AnchorPoint = Vector2.new(0.5, 0.5); main.Position = UDim2.fromScale(0.5, 0.5)
main.Visible = false; main.ClipsDescendants = true; main.Active = true
corner(main, 18); local mainStroke = stroke(main, P.accent, 1.5, 0.35); sheen(main, 3)

local us = Instance.new("UIScale"); us.Scale = windowScale(); us.Parent = main
pcall(function()
	Workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(function()
		us.Scale = windowScale()
	end)
end)

local top = bx(main, P.top, 4); top.BackgroundTransparency = 0.05; top.Size = UDim2.new(1,0,0,TOP)
top.Active = true; top.Selectable = true
drag(top, main); sheen(top, 5)
local hair = bx(top, P.line, 5); hair.Size = UDim2.new(1,0,0,1); hair.Position = UDim2.new(0,0,1,-1)
grad(hair, P.accent, P.line, 0)
local mark = drawMark(top, 26, 6); mark.Position = UDim2.fromOffset(12,11)
local title = tx(top, "Valkyrie.ui", F_D, 14, P.text); title.Size = UDim2.fromOffset(200,16); title.Position = UDim2.fromOffset(48,10); title.ZIndex = 6
local sub = tx(top, "FIVELYSS  ·  TPS / MPS", F_MO, 8, P.faint); sub.Size = UDim2.fromOffset(220,11); sub.Position = UDim2.fromOffset(48,27); sub.ZIndex = 6

local show, hide, doMinimize
local minimized = false
local function chromeBtn(x, label, col, fn)
	local b = Instance.new("TextButton"); b.Size = UDim2.fromOffset(28,28); b.Position = UDim2.new(1,x,0.5,-14)
	b.BackgroundColor3 = P.card; b.BackgroundTransparency = 1; b.AutoButtonColor = false; b.Text = label
	b.FontFace = F_S; b.TextSize = 12; b.TextColor3 = col; b.BorderSizePixel = 0; b.ZIndex = 7; b.Parent = top
	corner(b, 10); local st = stroke(b, col, 1, 0.7)
	b.MouseEnter:Connect(function() tw(b,0.12,{BackgroundTransparency=0}); tw(st,0.12,{Transparency=0.2}) end)
	b.MouseLeave:Connect(function() tw(b,0.12,{BackgroundTransparency=1}); tw(st,0.12,{Transparency=0.7}) end)
	b.MouseButton1Click:Connect(function() fn() end)
	return b
end

doMinimize = function(force)
	if force ~= nil then minimized = force else minimized = not minimized end
	local k = us.Scale; local shift = ((H-TOP)/2)*k; local p = main.Position
	if minimized then
		tw(main, 0.28, {Size=UDim2.fromOffset(W,TOP), Position=UDim2.new(p.X.Scale,p.X.Offset,p.Y.Scale,p.Y.Offset-shift)}, Enum.EasingStyle.Quint)
	else
		tw(main, 0.28, {Size=UDim2.fromOffset(W,H), Position=UDim2.new(p.X.Scale,p.X.Offset,p.Y.Scale,p.Y.Offset+shift)}, Enum.EasingStyle.Quint)
	end
end

local ORB = isTouch() and 56 or 48
local orb = Instance.new("TextButton"); orb.Name = "Orb"; orb.Size = UDim2.fromOffset(ORB,ORB)
orb.Position = UDim2.new(0,16,0.55,0); orb.BackgroundColor3 = P.bg; orb.AutoButtonColor = false; orb.Text = ""
orb.BorderSizePixel = 0; orb.Visible = false; orb.ZIndex = 30; orb.Parent = gui
corner(orb, ORB/2); stroke(orb, P.accent, 1.4, 0.2); glow(orb, P.accent, 2, 3, ORB/2, 0.42)
local om = drawMark(orb, ORB*0.58, 31); om.AnchorPoint = Vector2.new(0.5,0.5); om.Position = UDim2.fromScale(0.5,0.5)

local mobileBar = bx(gui, P.top, 28); mobileBar.AnchorPoint = Vector2.new(0.5,0)
mobileBar.Position = UDim2.new(0.5,0,0,12); mobileBar.Size = UDim2.fromOffset(230,42); mobileBar.ZIndex = 28
mobileBar.Visible = isTouch(); corner(mobileBar,21); stroke(mobileBar, P.accent,1.3,0.25); glow(mobileBar, P.accent,2,2,21,0.48); sheen(mobileBar,29)
mobileBar.Active = true
drag(mobileBar, mobileBar)

show = function()
	if minimized then doMinimize(false) end
	orb.Visible = false; main.Visible = true
	local t = windowScale(); us.Scale = t*0.9
	main.Position = UDim2.new(main.Position.X.Scale, main.Position.X.Offset, main.Position.Y.Scale, main.Position.Y.Offset+12)
	tw(us, 0.35, {Scale=t}, Enum.EasingStyle.Back)
	tw(main, 0.35, {Position=UDim2.new(main.Position.X.Scale,main.Position.X.Offset,main.Position.Y.Scale,main.Position.Y.Offset-12)}, Enum.EasingStyle.Quint)
end
hide = function()
	main.Visible = false; minimized = false; main.Size = UDim2.fromOffset(W,H)
	orb.Visible = true; orb.Size = UDim2.fromOffset(0,0)
	tw(orb, 0.3, {Size=UDim2.fromOffset(ORB,ORB)}, Enum.EasingStyle.Back)
end

chromeBtn(-34, "x", P.bad, function() hide() end)
chromeBtn(-66, "-", P.dim, function() doMinimize() end)

local function mBtn(label, x, col, fn)
	local b = Instance.new("TextButton"); b.Size = UDim2.fromOffset(66,30); b.Position = UDim2.fromOffset(x,6)
	b.BackgroundColor3 = P.card; b.AutoButtonColor = false; b.Text = label; b.FontFace = F_S; b.TextSize = 11
	b.TextColor3 = col; b.BorderSizePixel = 0; b.ZIndex = 29; b.Parent = mobileBar; corner(b,12)
	local st = stroke(b, col, 1, 0.5)
	b.MouseEnter:Connect(function() tw(b,0.1,{BackgroundColor3=P.cardHov}); tw(st,0.1,{Transparency=0.15}) end)
	b.MouseLeave:Connect(function() tw(b,0.1,{BackgroundColor3=P.card}); tw(st,0.1,{Transparency=0.5}) end)
	b.MouseButton1Click:Connect(function() fn() end)
	return b
end
mBtn("Open", 12, P.good, function() show() end)
mBtn("Half", 82, P.glow, function() if not main.Visible then show() end; task.defer(function() doMinimize(true) end) end)
mBtn("Close", 152, P.bad, function() hide() end)

local downAt, moved
orb.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		downAt, moved = i.Position, false
	end
end)
UserInputService.InputChanged:Connect(function(i)
	if downAt and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
		if (i.Position-downAt).Magnitude > 6 then moved = true end
	end
end)
orb.MouseButton1Click:Connect(function() if not moved then show() end end)
drag(orb, orb)

local side = bx(main, P.rail, 3); side.BackgroundTransparency = 0.05
side.Size = UDim2.new(0,SIDE,1,-TOP); side.Position = UDim2.new(0,0,0,TOP)
side.Active = true; side.Selectable = true; drag(side, main)

local bottom = bx(main, P.top, 8)
bottom.BackgroundTransparency = 0.35
bottom.Size = UDim2.new(1, 0, 0, 22)
bottom.Position = UDim2.new(0, 0, 1, -22)
bottom.Active = true; bottom.Selectable = true
corner(bottom, 0); drag(bottom, main)
local bottomHair = bx(bottom, P.line, 9)
bottomHair.Size = UDim2.new(1, 0, 0, 1)
bottomHair.Position = UDim2.new(0, 0, 0, 0)
grad(bottomHair, P.line, P.accent, 0)
local grip = tx(bottom, "═══", F_MO, 10, P.faint)
grip.Size = UDim2.fromScale(1, 1)
grip.TextXAlignment = Enum.TextXAlignment.Center
grip.ZIndex = 10

local rightEdge = fr(main, 8)
rightEdge.Size = UDim2.new(0, 18, 1, -TOP - 22)
rightEdge.Position = UDim2.new(1, -18, 0, TOP)
rightEdge.Active = true; rightEdge.Selectable = true
drag(rightEdge, main)

local leftEdge = fr(main, 8)
leftEdge.Size = UDim2.new(0, 10, 1, -TOP - 22)
leftEdge.Position = UDim2.new(0, 0, 0, TOP)
leftEdge.Active = true; leftEdge.Selectable = true
drag(leftEdge, main)

enableEdgeWindowDrag(main, 40)

local nav = Instance.new("ScrollingFrame"); nav.BackgroundTransparency = 1; nav.BorderSizePixel = 0
nav.Size = UDim2.new(1,-12,1,-16); nav.Position = UDim2.fromOffset(6,10); nav.ScrollBarThickness = 0
nav.CanvasSize = UDim2.new(); nav.AutomaticCanvasSize = Enum.AutomaticSize.Y; nav.ZIndex = 4; nav.Parent = side
local nl = Instance.new("UIListLayout"); nl.Padding = UDim.new(0,4); nl.SortOrder = Enum.SortOrder.LayoutOrder; nl.Parent = nav

local content = fr(main, 3)
content.Size = UDim2.new(1,-SIDE-12,1,-TOP-34); content.Position = UDim2.new(0,SIDE+6,0,TOP+6)

local pages, navBtns, active = {}, {}, nil
local tabOrder = 0
local function selectTab(name)
	active = name
	for n, page in pairs(pages) do page.Visible = (n == name) end
	for n, e in pairs(navBtns) do
		local on = n == name
		e.label.TextColor3 = on and P.hot or P.dim
		e.icon.TextColor3 = on and P.accent or P.dim
		e.btn.BackgroundTransparency = on and 0.35 or 1
	end
end

local function makeTab(name, icon)
	tabOrder += 1
	local b = Instance.new("TextButton"); b.Size = UDim2.new(1,0,0,38); b.BackgroundColor3 = P.cardHov
	b.BackgroundTransparency = 1; b.AutoButtonColor = false; b.Text = ""; b.BorderSizePixel = 0
	b.LayoutOrder = tabOrder; b.ZIndex = 5; b.Parent = nav; corner(b,12)
	local ic = tx(b, icon or "•", F_R, 14, P.dim); ic.Size = UDim2.fromOffset(28,38); ic.Position = UDim2.fromOffset(8,0)
	ic.TextXAlignment = Enum.TextXAlignment.Center; ic.ZIndex = 6
	local l = tx(b, name, F_M, 13, P.dim); l.Size = UDim2.new(1,-44,1,0); l.Position = UDim2.fromOffset(38,0); l.ZIndex = 6
	local page = Instance.new("ScrollingFrame"); page.BackgroundTransparency = 1; page.BorderSizePixel = 0
	page.Size = UDim2.fromScale(1,1); page.ScrollBarThickness = 3; page.ScrollBarImageColor3 = P.accent
	page.CanvasSize = UDim2.new(); page.AutomaticCanvasSize = Enum.AutomaticSize.Y
	page.Visible = false; page.ZIndex = 4; page.Parent = content
	local list = Instance.new("UIListLayout"); list.Padding = UDim.new(0,8); list.SortOrder = Enum.SortOrder.LayoutOrder; list.Parent = page
	local pad = Instance.new("UIPadding"); pad.PaddingTop = UDim.new(0,4); pad.PaddingRight = UDim.new(0,6)
	pad.PaddingBottom = UDim.new(0,12); pad.PaddingLeft = UDim.new(0,2); pad.Parent = page
	pages[name] = page; navBtns[name] = {btn=b, label=l, icon=ic}
	b.MouseButton1Click:Connect(function() selectTab(name) end)
	if not active then selectTab(name) end
	local rows = 0
	local tab = {}
	local function rowBase(h)
		rows += 1
		local r = bx(page, P.card, 3); r.Size = UDim2.new(1,-4,0,h); r.LayoutOrder = rows; r.BackgroundTransparency = 0
		corner(r,14); stroke(r, P.line,1,0.45); return r
	end
	function tab:Section(titleText, desc)
		rows += 1
		local s = fr(page, 3); s.Size = UDim2.new(1,-4,0,desc and 40 or 24); s.LayoutOrder = rows
		local l2 = tx(s, titleText, F_B, 15, P.text); l2.Size = UDim2.new(1,-8,0,18); l2.Position = UDim2.fromOffset(2,2); l2.ZIndex = 4
		if desc then
			local d = tx(s, titleText, F_R, 11, P.dim); d.Size = UDim2.new(1,-8,0,14); d.Position = UDim2.fromOffset(2,22); d.ZIndex = 4
		end
	end
	function tab:Label(text)
		rows += 1
		local l2 = tx(page, text, F_R, 11, P.dim); l2.Size = UDim2.new(1,-8,0,0); l2.AutomaticSize = Enum.AutomaticSize.Y
		l2.TextWrapped = true; l2.LayoutOrder = rows; l2.ZIndex = 4
	end
	function tab:Toggle(label, desc, default, cb)
		local on = default and true or false; local h = desc and 54 or 44; local r = rowBase(h)
		local l2 = tx(r, label, F_S, 13, P.text); l2.Size = UDim2.new(1,-70,0,18); l2.Position = UDim2.fromOffset(14, desc and 8 or 12); l2.ZIndex = 5
		if desc then local d = tx(r, desc, F_R, 11, P.dim); d.Size = UDim2.new(1,-70,0,14); d.Position = UDim2.fromOffset(14,28); d.ZIndex = 5 end
		local box = bx(r, on and P.accent or P.off, 5); box.AnchorPoint = Vector2.new(1,0.5); box.Position = UDim2.new(1,-14,0.5,0)
		box.Size = UDim2.fromOffset(22,22); corner(box,8); stroke(box, on and P.accent or P.line, 1, on and 0.2 or 0.45)
		local check = tx(box, "✓", F_B, 14, P.hot); check.Size = UDim2.fromScale(1,1); check.TextXAlignment = Enum.TextXAlignment.Center
		check.TextTransparency = on and 0 or 1; check.ZIndex = 6
		local function apply(v, fire)
			on = v and true or false
			box.BackgroundColor3 = on and P.accent or P.off
			check.TextTransparency = on and 0 or 1
			if fire and cb then pcall(cb, on) end
		end
		local hit = Instance.new("TextButton"); hit.Size = UDim2.fromScale(1,1); hit.BackgroundTransparency = 1; hit.Text = ""; hit.ZIndex = 8; hit.Parent = r
		hit.MouseButton1Click:Connect(function() apply(not on, true) end)
		return { Set = function(_, v) apply(v, false) end, Get = function() return on end }
	end
	function tab:Slider(label, desc, min, max, default, cb, suffix)
		local val = default or min; local r = rowBase(desc and 72 or 62)
		local l2 = tx(r, label, F_S, 13, P.text); l2.Size = UDim2.new(1,-20,0,16); l2.Position = UDim2.fromOffset(14,8); l2.ZIndex = 5
		local valL = tx(r, tostring(val)..(suffix or ""), F_MO, 11, P.accent); valL.Size = UDim2.fromOffset(80,16); valL.Position = UDim2.new(1,-90,0,8)
		valL.TextXAlignment = Enum.TextXAlignment.Right; valL.ZIndex = 5
		if desc then local d = tx(r, desc, F_R, 10, P.dim); d.Size = UDim2.new(1,-20,0,12); d.Position = UDim2.fromOffset(14,26); d.ZIndex = 5 end
		local track = bx(r, P.off, 5); track.Size = UDim2.new(1,-28,0,8); track.Position = UDim2.new(0,14,1,-18); corner(track, 4)
		local fill = bx(track, P.accent, 6); fill.Size = UDim2.fromScale((val-min)/math.max(max-min,1), 1); corner(fill, 4)
		local sliding = false
		local function setFromX(x)
			local abs = track.AbsolutePosition.X; local w = track.AbsoluteSize.X
			local a = math.clamp((x - abs) / math.max(w,1), 0, 1)
			val = math.floor((min + a * (max - min)) * 100 + 0.5) / 100
			fill.Size = UDim2.fromScale(a, 1); valL.Text = tostring(val)..(suffix or "")
			if cb then pcall(cb, val) end
		end
		local hit = Instance.new("TextButton"); hit.Size = UDim2.fromScale(1,1); hit.BackgroundTransparency = 1; hit.Text = ""; hit.ZIndex = 8; hit.Parent = r
		hit.InputBegan:Connect(function(i)
			if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
				sliding = true; setFromX(i.Position.X)
			end
		end)
		UserInputService.InputEnded:Connect(function(i)
			if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then sliding = false end
		end)
		UserInputService.InputChanged:Connect(function(i)
			if sliding and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
				setFromX(i.Position.X)
			end
		end)
		return { Set = function(_, v) val = v; fill.Size = UDim2.fromScale((val-min)/math.max(max-min,1),1); valL.Text = tostring(val)..(suffix or "") end }
	end
	function tab:Button(label, primary, cb)
		local r = rowBase(44)
		local b = Instance.new("TextButton"); b.Size = UDim2.new(1,-20,0,32); b.Position = UDim2.fromOffset(10,6)
		b.BackgroundColor3 = primary and P.accent or P.cardHov; b.AutoButtonColor = false; b.Text = label
		b.FontFace = F_S; b.TextSize = 13; b.TextColor3 = primary and P.hot or P.text; b.BorderSizePixel = 0; b.ZIndex = 6; b.Parent = r
		corner(b, 10); stroke(b, primary and P.glow or P.line, 1, primary and 0.35 or 0.5)
		b.MouseButton1Click:Connect(function() if cb then pcall(cb) end end)
	end
	function tab:Dropdown(label, options, default, cb)
		options = options or {}
		local idx = 1
		for i, o in ipairs(options) do
			if o == default then idx = i break end
		end
		local cur = options[idx] or default or "?"
		local r = rowBase(54)
		local l2 = tx(r, label, F_S, 13, P.text); l2.Size = UDim2.new(1,-20,0,16); l2.Position = UDim2.fromOffset(14,6); l2.ZIndex = 5
		local b = Instance.new("TextButton"); b.Size = UDim2.new(1,-20,0,26); b.Position = UDim2.fromOffset(10,24)
		b.BackgroundColor3 = P.cardHov; b.AutoButtonColor = false
		b.Text = "  "..tostring(cur).."  ▾"; b.FontFace = F_S; b.TextSize = 12
		b.TextColor3 = P.accent; b.TextXAlignment = Enum.TextXAlignment.Left
		b.BorderSizePixel = 0; b.ZIndex = 6; b.Parent = r
		corner(b, 8); stroke(b, P.line, 1, 0.4)
		b.MouseButton1Click:Connect(function()
			if #options == 0 then return end
			idx = (idx % #options) + 1
			cur = options[idx]
			b.Text = "  "..tostring(cur).."  ▾"
			if cb then pcall(cb, cur) end
		end)
		return {
			Set = function(_, v)
				for i, o in ipairs(options) do
					if o == v then idx = i; cur = v; b.Text = "  "..tostring(cur).."  ▾"; break end
				end
			end,
			Get = function() return cur end,
		}
	end
	return tab
end

Notify = function(text, duration)
	local kind = "good"
	local sec = 3
	if type(duration) == "string" then kind = duration; sec = 3
	elseif type(duration) == "number" then sec = duration end
	local col = kind == "bad" and P.bad or (kind == "warn" and P.warn or P.good)
	local n = bx(gui, P.card, 50); n.Size = UDim2.fromOffset(280, 40); n.Position = UDim2.new(1, -300, 0, 20)
	corner(n, 12); stroke(n, col, 1.4, 0.25)
	local t = tx(n, tostring(text), F_S, 13, P.text); t.Size = UDim2.new(1,-16,1,0); t.Position = UDim2.fromOffset(12,0); t.ZIndex = 51
	task.delay(sec, function()
		pcall(function() tw(n, 0.25, {BackgroundTransparency=1}); task.wait(0.25); n:Destroy() end)
	end)
end

---------------------------------------------------------------------------
-- TABS
---------------------------------------------------------------------------
local statusTab = makeTab("Status", "◆")
statusTab:Section("Valkyrie.ui", "Fivelyss systems · TPS / MPS")
statusTab:Label("v3.0 · mobile opacity · circle axis scales · configs · air-tracking GK · limb reach")
statusTab:Label("Toggle menu: RightControl  ·  Mobile: top orb + GK button")
statusTab:Label("Player: "..LocalPlayer.Name.."  ·  Place: "..tostring(game.PlaceId))

local reachTab = makeTab("Reach", "⊕")
reachTab:Section("Limb Reach", "Hitbox expand per limb")
UIControls.HeadEnabled = reachTab:Toggle("Head Reach", nil, false, function(v) reachSettings.HeadEnabled=v; UpdateLimbVisualizer("Head") end)
UIControls.ReachX_Head = reachTab:Slider("Head X", nil, 0, 10, 1, function(v) reachSettings.ReachX_Head=v; UpdateLimbVisualizer("Head") end)
UIControls.ReachY_Head = reachTab:Slider("Head Y", nil, 0, 10, 1, function(v) reachSettings.ReachY_Head=v; UpdateLimbVisualizer("Head") end)
UIControls.ReachZ_Head = reachTab:Slider("Head Z", nil, 0, 10, 1, function(v) reachSettings.ReachZ_Head=v; UpdateLimbVisualizer("Head") end)
UIControls.RightArmEnabled = reachTab:Toggle("Right Arm", nil, false, function(v) reachSettings.RightArmEnabled=v; UpdateLimbVisualizer("Right Arm") end)
UIControls.LeftArmEnabled = reachTab:Toggle("Left Arm", nil, false, function(v) reachSettings.LeftArmEnabled=v; UpdateLimbVisualizer("Left Arm") end)
UIControls.ReachX_Arm = reachTab:Slider("Arm X", nil, 0, 10, 1, function(v) reachSettings.ReachX_Arm=v; UpdateLimbVisualizer("Right Arm"); UpdateLimbVisualizer("Left Arm") end)
UIControls.ReachY_Arm = reachTab:Slider("Arm Y", nil, 0, 10, 1, function(v) reachSettings.ReachY_Arm=v; UpdateLimbVisualizer("Right Arm"); UpdateLimbVisualizer("Left Arm") end)
UIControls.ReachZ_Arm = reachTab:Slider("Arm Z", nil, 0, 10, 1, function(v) reachSettings.ReachZ_Arm=v; UpdateLimbVisualizer("Right Arm"); UpdateLimbVisualizer("Left Arm") end)
UIControls.RightLegEnabled = reachTab:Toggle("Right Leg", nil, false, function(v) reachSettings.RightLegEnabled=v; UpdateLimbVisualizer("Right Leg") end)
UIControls.LeftLegEnabled = reachTab:Toggle("Left Leg", nil, false, function(v) reachSettings.LeftLegEnabled=v; UpdateLimbVisualizer("Left Leg") end)
UIControls.ReachX_Leg = reachTab:Slider("Leg X", nil, 0, 10, 1, function(v) reachSettings.ReachX_Leg=v; UpdateLimbVisualizer("Right Leg"); UpdateLimbVisualizer("Left Leg") end)
UIControls.ReachY_Leg = reachTab:Slider("Leg Y", nil, 0, 10, 1, function(v) reachSettings.ReachY_Leg=v; UpdateLimbVisualizer("Right Leg"); UpdateLimbVisualizer("Left Leg") end)
UIControls.ReachZ_Leg = reachTab:Slider("Leg Z", nil, 0, 10, 1, function(v) reachSettings.ReachZ_Leg=v; UpdateLimbVisualizer("Right Leg"); UpdateLimbVisualizer("Left Leg") end)
UIControls.ReachDistance = reachTab:Slider("Reach Multiplier", nil, 1, 200, 50, function(v)
	reachSettings.ReachDistance=v
	for _, p in ipairs(LimbBodyParts) do UpdateLimbVisualizer(p) end
end)
reachTab:Button("Refresh Hitboxes", false, function() RemoveAllLimbHitboxes(); CreateAllLimbHitboxes(); Notify("Limb refreshed", 2) end)
reachTab:Button("Disable All Reach", false, function()
	reachSettings.HeadEnabled=false; reachSettings.RightArmEnabled=false; reachSettings.LeftArmEnabled=false
	reachSettings.RightLegEnabled=false; reachSettings.LeftLegEnabled=false
	RemoveAllLimbHitboxes(); Notify("All Limb Reach OFF", 2)
	if SyncUIFromState then SyncUIFromState() end
end)
reachTab:Section("Visualizers", nil)
UIControls.HeadVisualizer = reachTab:Toggle("Head Vis", nil, false, function(v) reachSettings.HeadVisualizer=v; UpdateLimbVisualizer("Head") end)
UIControls.RightArmVisualizer = reachTab:Toggle("Right Arm Vis", nil, false, function(v) reachSettings.RightArmVisualizer=v; UpdateLimbVisualizer("Right Arm") end)
UIControls.LeftArmVisualizer = reachTab:Toggle("Left Arm Vis", nil, false, function(v) reachSettings.LeftArmVisualizer=v; UpdateLimbVisualizer("Left Arm") end)
UIControls.RightLegVisualizer = reachTab:Toggle("Right Leg Vis", nil, false, function(v) reachSettings.RightLegVisualizer=v; UpdateLimbVisualizer("Right Leg") end)
UIControls.LeftLegVisualizer = reachTab:Toggle("Left Leg Vis", nil, false, function(v) reachSettings.LeftLegVisualizer=v; UpdateLimbVisualizer("Left Leg") end)

---------------------------------------------------------------------------
-- CIRCLE REACH TAB  (ellipsoid · independent X / Y / Z radii)
---------------------------------------------------------------------------
local circleTab = makeTab("Circle", "○")
circleTab:Section("Circle Reach", "Ellipsoid around body · firetouchinterest · arms / legs / both")
circleTab:Label("Volume centered on HRP. Set Radius X / Y / Z independently to stretch the sphere.")
UIControls.CircleEnabled = circleTab:Toggle("Enable Circle Reach", nil, false, function(v)
	SetCircleReachEnabled(v)
end)
UIControls.CircleMode = circleTab:Dropdown("Limbs", { "Both", "Arms", "Legs" }, "Both", function(v)
	CircleReach.Mode = v
end)
UIControls.CircleRadiusX = circleTab:Slider("Radius X", "left-right", 1, 40, 12, function(v)
	CircleReach.RadiusX = v
	CircleReach.Radius = (CircleReach.RadiusX + CircleReach.RadiusY + CircleReach.RadiusZ) / 3
	UpdateCircleReachVisual()
end, " studs")
UIControls.CircleRadiusY = circleTab:Slider("Radius Y", "up-down", 1, 40, 12, function(v)
	CircleReach.RadiusY = v
	CircleReach.Radius = (CircleReach.RadiusX + CircleReach.RadiusY + CircleReach.RadiusZ) / 3
	UpdateCircleReachVisual()
end, " studs")
UIControls.CircleRadiusZ = circleTab:Slider("Radius Z", "forward-back", 1, 40, 12, function(v)
	CircleReach.RadiusZ = v
	CircleReach.Radius = (CircleReach.RadiusX + CircleReach.RadiusY + CircleReach.RadiusZ) / 3
	UpdateCircleReachVisual()
end, " studs")
UIControls.CircleVisible = circleTab:Toggle("Show Volume", nil, true, function(v)
	CircleReach.Visible = v
	UpdateCircleReachVisual()
end)
UIControls.CircleTransparency = circleTab:Slider("Vis Opacity", "0 = solid · 1 = invisible", 0, 1, 0.65, function(v)
	CircleReach.Transparency = v
	UpdateCircleReachVisual()
end)
circleTab:Button("Reset Axes to 12", false, function()
	CircleReach.RadiusX = 12
	CircleReach.RadiusY = 12
	CircleReach.RadiusZ = 12
	CircleReach.Radius = 12
	if UIControls.CircleRadiusX then UIControls.CircleRadiusX:Set(12) end
	if UIControls.CircleRadiusY then UIControls.CircleRadiusY:Set(12) end
	if UIControls.CircleRadiusZ then UIControls.CircleRadiusZ:Set(12) end
	UpdateCircleReachVisual()
	Notify("Axes reset", 2)
end)
circleTab:Button("Refresh Volume", false, function()
	UpdateCircleReachVisual()
	Notify("Volume refreshed", 2)
end)

local gkTab = makeTab("GK", "🧤")
gkTab:Section("GK Helper", "Auto move · BlueGoal / GreenGoal · air + high")
UIControls.GKHelper = gkTab:Toggle("GK Helper", "Stand in front of your goal", false, function(v)
	if v then StartGKHelper() else StopGKHelper() end
end)
gkTab:Label("Only helps when ball is coming at your goal. Caps retreat. Mobile GK button toggles too.")
gkTab:Section("Mobile Macros", "Keep1 + Save tools")
UIControls.ShowMobile = gkTab:Toggle("Show Mobile Buttons", nil, true, function(v) CC.btn.visible=v; ccSyncVisible() end)
UIControls.LockButtons = gkTab:Toggle("Lock Buttons", nil, false, function(v) CC.btn.locked=v; ccSyncLock() end)
UIControls.EnableMacros = gkTab:Toggle("Enable Macros", nil, true, function(v) CC.mobEnabled=v end)
UIControls.BtnSize = gkTab:Slider("Button Size", nil, 40, 180, 88, function(v) CC.btn.size=v; ccSyncSizes() end, " px")
-- NEW: full-range transparency — 1.0 = completely invisible, still fully clickable
UIControls.BtnTransparency = gkTab:Slider("Button Opacity", "0 = solid · 1 = invisible (still works)", 0, 1, 0.12, function(v)
	CC.btn.transparency = v
	if ccSyncOpacity then ccSyncOpacity() end
end)
UIControls.KeepToSave = gkTab:Slider("Keep→Save Gap", nil, 0, 0.5, 0.08, function(v) CC.keepToSave=v end, " s")
gkTab:Button("Test Keep+Save", false, function() ccRunCombo("KeepSave") end)
gkTab:Button("Test Jump+Keep+Save", false, function() ccRunCombo("JumpKeepSave") end)
gkTab:Button("Make Buttons Invisible", false, function()
	CC.btn.transparency = 1
	if UIControls.BtnTransparency then UIControls.BtnTransparency:Set(1) end
	if ccSyncOpacity then ccSyncOpacity() end
	Notify("Buttons invisible · still active", 2)
end)
gkTab:Button("Make Buttons Solid", false, function()
	CC.btn.transparency = 0
	if UIControls.BtnTransparency then UIControls.BtnTransparency:Set(0) end
	if ccSyncOpacity then ccSyncOpacity() end
	Notify("Buttons solid", 2)
end)

---------------------------------------------------------------------------
-- REACT / TOOLS / SKY / PERF / PROTECT  (feature tables defined here so config can see them)
---------------------------------------------------------------------------
NRC = { Enabled=false, Magnitude=4, Fires=2, Distance=15, Conn=nil }
ReactKiller = { Enabled=false, Fires=30, Distance=20, Conn=nil }
ReactBreak = { Enabled=false, Strength=0.7, Distance=16, Conn=nil, LastTouch = {} }

local ENEMY_LIMBS = { "Head", "Torso", "UpperTorso", "LowerTorso", "HumanoidRootPart",
    "Left Arm", "Right Arm", "Left Leg", "Right Leg",
    "LeftUpperArm", "RightUpperArm", "LeftLowerArm", "RightLowerArm",
    "LeftUpperLeg", "RightUpperLeg", "LeftLowerLeg", "RightLowerLeg",
    "LeftHand", "RightHand", "LeftFoot", "RightFoot" }

SetReactBreakEnabled = function(v, silent)
	ReactBreak.Enabled = v and true or false
	if ReactBreak.Conn then pcall(function() ReactBreak.Conn:Disconnect() end) ReactBreak.Conn = nil end
	ReactBreak.LastTouch = {}
	if ReactBreak.Enabled then
		ReactBreak.Conn = RunService.Heartbeat:Connect(function()
			if not ReactBreak.Enabled then return end
			local ball = GetMainBall()
			if not ball or not ball.Parent or not ball:IsA("BasePart") then return end
			local bpos = ball.Position
			local dist = ReactBreak.Distance or 16
			local strength = math.max(ReactBreak.Strength or 0.7, 0.1)
			local now = tick()
			for _, plr in ipairs(Players:GetPlayers()) do
				if plr == LocalPlayer then continue end
				local char = plr.Character
				if not char then continue end
				local root = char:FindFirstChild("HumanoidRootPart")
				if not root then continue end
				local offset = bpos - root.Position
				local mag = offset.Magnitude
				if mag < 0.08 or mag > dist then continue end

				-- 1) hard kick the ball OFF the enemy (breaks their react trajectory)
				local dir = offset.Unit
				local kick = strength * (42 + (dist - mag) * 2.8)
				pcall(function()
					local cur = ball.AssemblyLinearVelocity
					-- cancel any velocity component pointing at the enemy, then shove outward
					local towardEnemy = -dir
					local cancel = cur:Dot(towardEnemy)
					if cancel > 0 then
						cur = cur - towardEnemy * cancel
					end
					ball.AssemblyLinearVelocity = cur + dir * kick
					ball.AssemblyAngularVelocity = Vector3.new(
						(math.random() - 0.5) * strength * 55,
						(math.random() - 0.5) * strength * 55,
						(math.random() - 0.5) * strength * 55
					)
				end)

				-- 2) spam firetouchinterest on enemy limbs so their client react state gets eaten
				if firetouchinterest then
					for _, limbName in ipairs(ENEMY_LIMBS) do
						local limb = char:FindFirstChild(limbName)
						if limb and limb:IsA("BasePart") then
							local key = plr.UserId .. ":" .. limbName
							local last = ReactBreak.LastTouch[key] or 0
							if (now - last) >= 0.05 then
								ReactBreak.LastTouch[key] = now
								pcall(function()
									firetouchinterest(ball, limb, 0)
									firetouchinterest(ball, limb, 1)
								end)
							end
						end
					end
				end
			end
		end)
		if not silent and Notify then Notify("React Break ON", 2) end
	else
		if not silent and Notify then Notify("React Break OFF", 2) end
	end
	if UIControls.ReactBreak then pcall(function() UIControls.ReactBreak:Set(ReactBreak.Enabled) end) end
end

SetNRCEnabled = function(v, silent)
	NRC.Enabled = v and true or false
	if NRC.Conn then pcall(function() NRC.Conn:Disconnect() end) NRC.Conn = nil end
	if NRC.Enabled then
		NRC.Conn = RunService.Heartbeat:Connect(function()
			if not NRC.Enabled then return end
			local ball, root = GetMainBall(), Cache.Root
			if not ball or not root then return end
			if (ball.Position-root.Position).Magnitude <= NRC.Distance then
				pcall(function() ball.AssemblyLinearVelocity = (ball.Position-root.Position).Unit * NRC.Magnitude end)
			end
		end)
		if not silent and Notify then Notify("NRC ON", 2) end
	else
		if not silent and Notify then Notify("NRC OFF", 2) end
	end
	if UIControls.NRC then pcall(function() UIControls.NRC:Set(NRC.Enabled) end) end
end

SetReactKillerEnabled = function(v, silent)
	ReactKiller.Enabled = v and true or false
	if ReactKiller.Conn then pcall(function() ReactKiller.Conn:Disconnect() end) ReactKiller.Conn = nil end
	if ReactKiller.Enabled then
		ReactKiller.Conn = RunService.Heartbeat:Connect(function()
			if not ReactKiller.Enabled then return end
			local ball, root = GetMainBall(), Cache.Root
			if not ball or not root then return end
			if (ball.Position-root.Position).Magnitude > ReactKiller.Distance then return end
			local fe = GetFE(); if not fe then return end
			local scorer = fe:FindFirstChild("Scorer"); if not scorer then return end
			for _, remote in ipairs(scorer:GetChildren()) do
				if remote:IsA("RemoteEvent") then
					for i=1, ReactKiller.Fires do pcall(function() remote:FireServer(ball) end) end
				end
			end
		end)
		if not silent and Notify then Notify("React Killer ON", 2) end
	else
		if not silent and Notify then Notify("React Killer OFF", 2) end
	end
	if UIControls.ReactKiller then pcall(function() UIControls.ReactKiller:Set(ReactKiller.Enabled) end) end
end

local reactTab = makeTab("React", "⚡")
reactTab:Section("No React Decline", nil)
UIControls.NRC = reactTab:Toggle("NRC", nil, false, function(v) SetNRCEnabled(v) end)
UIControls.NRC_Magnitude = reactTab:Slider("Magnitude", nil, 1, 20, 4, function(v) NRC.Magnitude=v end)
UIControls.NRC_Distance = reactTab:Slider("Distance", nil, 1, 50, 15, function(v) NRC.Distance=v end, " studs")
reactTab:Section("React Break", "Breaks enemy react only · leaves your own touch clean")
UIControls.ReactBreak = reactTab:Toggle("React Break", "Pops ball off enemies when in range", false, function(v) SetReactBreakEnabled(v) end)
UIControls.ReactBreak_Strength = reactTab:Slider("Break Strength", "starts at 0.7", 0.1, 3, 0.7, function(v) ReactBreak.Strength=v end)
UIControls.ReactBreak_Distance = reactTab:Slider("Break Distance", nil, 4, 40, 16, function(v) ReactBreak.Distance=v end, " studs")
local function FireUniversalReact(mag)
	pcall(function()
		local ball, root = GetMainBall(), Cache.Root
		if ball and root then ball.AssemblyLinearVelocity = (ball.Position-root.Position).Unit * mag end
	end)
end
reactTab:Section("React Methods", nil)
reactTab:Button("Universal React", true, function() FireUniversalReact(8); Notify("Universal", 2) end)
reactTab:Button("Main React", false, function() FireUniversalReact(10); Notify("Main", 2) end)
reactTab:Button("Break React", false, function() FireUniversalReact(15); Notify("Break", 2) end)
reactTab:Button("Defence React", false, function() FireUniversalReact(6); Notify("Defence", 2) end)
reactTab:Section("React Killer", nil)
UIControls.ReactKiller = reactTab:Toggle("React Killer", nil, false, function(v) SetReactKillerEnabled(v) end)
UIControls.ReactKiller_Fires = reactTab:Slider("Fires / frame", nil, 1, 100, 30, function(v) ReactKiller.Fires=v end)

local toolsTab = makeTab("Tools", "🔧")
toolsTab:Section("Stamina / Rejoin", nil)
toolsTab:Toggle("Unlimited Stamina", nil, false, function(v)
	if v then
		pcall(function()
			local mt = getrawmetatable(game); if not mt then return end
			setreadonly(mt, false)
			local oldNC = mt.__namecall
			mt.__namecall = newcclosure(function(self, ...)
				if getnamecallmethod()=="FireServer" and tostring(self):lower():find("sprint") then return nil end
				return oldNC(self, ...)
			end)
			setreadonly(mt, true)
		end)
		Notify("Unlimited Stamina ON", 2)
	else Notify("Unlimited Stamina OFF", 2) end
end)
toolsTab:Button("Rejoin", false, function() TeleportService:Teleport(game.PlaceId, LocalPlayer) end)

---------------------------------------------------------------------------
-- CONFIGS TAB  (new)
---------------------------------------------------------------------------
local configsTab = makeTab("Configs", "💾")
configsTab:Section("Save / Load", "Writes to executor workspace + ValkyrieConfigs/")
configsTab:Label("Saves: reach + circle axes, GK, mobile positions/opacity, NRC, React Killer, AntiBan, sky")
configsTab:Label("Files: workspace/valkyrie_tps_ulti.json  and  ValkyrieConfigs/tps_ulti.json")

configsTab:Button("Save Config", true, function()
	local ok, res = SaveConfigToFile(CONFIG_FILE)
	Notify(ok and ("Saved · "..tostring(res)) or ("Fail · "..tostring(res)), ok and 3 or 4)
end)

configsTab:Button("Load Config", false, function()
	local ok, res = LoadConfigFromFile(CONFIG_FILE)
	if ok then
		task.defer(function()
			if PendingConfig then ApplyConfigSnapshot(PendingConfig) end
		end)
	end
	Notify(ok and ("Loaded · "..tostring(res)) or ("Fail · "..tostring(res)), ok and 3 or 4)
end)

configsTab:Button("Save Positions Now", false, function()
	pcall(function()
		if CC and ccRefs then
			for id, r in pairs(ccRefs) do
				if r.btn then CC.positions[id] = r.btn.Position end
			end
		end
	end)
	local ok, res = SaveConfigToFile(CONFIG_FILE)
	Notify(ok and "Positions saved" or ("Fail · "..tostring(res)), ok and 2 or 3)
end)

configsTab:Section("Named slots", "Cycle or pick a name then Save As / Load")
local namedSlot = "slot1"
local namedDrop = configsTab:Dropdown("Slot Name", {
	"slot1", "slot2", "slot3", "slot4", "slot5",
	"main", "gk", "reach", "mobile", "backup"
}, "slot1", function(v) namedSlot = v end)

configsTab:Button("Save As Selected Slot", true, function()
	local name = namedSlot
	if not name or name == "" then name = "slot1" end
	local ok, res = SaveConfigToFile(name)
	Notify(ok and ("Saved as "..name) or ("Fail · "..tostring(res)), ok and 3 or 4)
end)

configsTab:Button("Load Selected Slot", false, function()
	local name = namedSlot
	if not name or name == "" then name = "slot1" end
	local ok, res = LoadConfigFromFile(name)
	if ok then
		task.defer(function()
			if PendingConfig then ApplyConfigSnapshot(PendingConfig) end
		end)
	end
	Notify(ok and ("Loaded "..name) or ("Fail · "..tostring(res)), ok and 3 or 4)
end)

configsTab:Button("List Config Files", false, function()
	local files = ListConfigFiles()
	if #files == 0 then
		Notify("No config files found", 3)
	else
		Notify("Files: "..table.concat(files, " · "), 5)
		print("[Valkyrie Configs]")
		for _, f in ipairs(files) do print("  ", f) end
	end
end)

configsTab:Button("Delete Selected Slot", false, function()
	local name = namedSlot
	if not name or name == "" then name = "slot1" end
	local ok, res = DeleteConfigFile(name)
	Notify(ok and ("Deleted "..name) or ("Fail · "..tostring(res)), ok and 3 or 4)
end)

configsTab:Section("Paths", nil)
configsTab:Label("Primary: ValkyrieConfigs/tps_ulti.json")
configsTab:Label("Flat copy: valkyrie_tps_ulti.json (workspace root)")
configsTab:Label("Also writes the exact filename into workspace root on every save.")

---------------------------------------------------------------------------
-- SKY / PERF / PROTECT / MENU
---------------------------------------------------------------------------
local skyTab = makeTab("Sky", "☁")
SkyboxCfg = { Enabled = false, Current = "Default", OriginalSky = nil, OriginalSkyParent = nil }
pcall(function()
	local sky = Lighting:FindFirstChildOfClass("Sky")
	if sky then
		SkyboxCfg.OriginalSky = sky
		SkyboxCfg.OriginalSkyParent = sky.Parent
	end
end)

local SPACE = {
	SkyboxBk = "rbxassetid://159454299", SkyboxDn = "rbxassetid://159454296",
	SkyboxFt = "rbxassetid://159454293", SkyboxLf = "rbxassetid://159454286",
	SkyboxRt = "rbxassetid://159454300", SkyboxUp = "rbxassetid://159454288",
}
local SUNSET = {
	SkyboxBk = "rbxassetid://3210220142", SkyboxDn = "rbxassetid://3210220274",
	SkyboxFt = "rbxassetid://3210220405", SkyboxLf = "rbxassetid://3210220532",
	SkyboxRt = "rbxassetid://3210220652", SkyboxUp = "rbxassetid://3210220764",
}

local SkyPresets = {
	["Default"] = nil,
	["Space"] = { faces = SPACE, stars = 5000 },
	["Nebula"] = { faces = SPACE, stars = 8000 },
	["Galaxy"] = { faces = SPACE, stars = 12000 },
	["Deep Space"] = { faces = SPACE, stars = 20000 },
	["Night Sky"] = { faces = SPACE, stars = 3000 },
	["Sunset"] = { faces = SUNSET, stars = 0 },
	["Sunrise"] = { faces = SUNSET, stars = 500 },
	["Golden Hour"] = { faces = SUNSET, stars = 0 },
	["Blood Moon"] = { faces = SPACE, stars = 3000 },
	["Cyberpunk"] = { faces = SPACE, stars = 10000 },
	["Neon"] = { faces = SPACE, stars = 15000 },
}

local LightingPresets = {
	["Default"] = function()
		Lighting.Ambient = Color3.fromRGB(70, 70, 70)
		Lighting.OutdoorAmbient = Color3.fromRGB(128, 128, 128)
		Lighting.Brightness = 2
		Lighting.ClockTime = 14
		Lighting.FogEnd = 100000
		Lighting.GlobalShadows = true
	end,
	["Day"] = function()
		Lighting.Ambient = Color3.fromRGB(140, 140, 140)
		Lighting.OutdoorAmbient = Color3.fromRGB(160, 160, 160)
		Lighting.Brightness = 2.5
		Lighting.ClockTime = 12
		Lighting.FogEnd = 100000
	end,
	["Night"] = function()
		Lighting.Ambient = Color3.fromRGB(25, 25, 40)
		Lighting.OutdoorAmbient = Color3.fromRGB(35, 35, 55)
		Lighting.Brightness = 0.6
		Lighting.ClockTime = 0
		Lighting.FogEnd = 2000
	end,
	["Sunset"] = function()
		Lighting.Ambient = Color3.fromRGB(120, 70, 50)
		Lighting.OutdoorAmbient = Color3.fromRGB(150, 90, 60)
		Lighting.Brightness = 1.8
		Lighting.ClockTime = 18
		Lighting.FogEnd = 5000
	end,
	["Purple"] = function()
		Lighting.Ambient = Color3.fromRGB(80, 40, 120)
		Lighting.OutdoorAmbient = Color3.fromRGB(100, 50, 140)
		Lighting.Brightness = 1.5
		Lighting.ClockTime = 20
		Lighting.FogEnd = 4000
	end,
	["Dark"] = function()
		Lighting.Ambient = Color3.fromRGB(15, 15, 15)
		Lighting.OutdoorAmbient = Color3.fromRGB(25, 25, 25)
		Lighting.Brightness = 0.4
		Lighting.ClockTime = 0
		Lighting.FogEnd = 1500
	end,
	["Bright"] = function()
		Lighting.Ambient = Color3.fromRGB(180, 180, 180)
		Lighting.OutdoorAmbient = Color3.fromRGB(200, 200, 200)
		Lighting.Brightness = 3
		Lighting.ClockTime = 14
		Lighting.FogEnd = 100000
	end,
	["Blood"] = function()
		Lighting.Ambient = Color3.fromRGB(80, 20, 20)
		Lighting.OutdoorAmbient = Color3.fromRGB(100, 30, 30)
		Lighting.Brightness = 1.2
		Lighting.ClockTime = 22
		Lighting.FogColor = Color3.fromRGB(60, 10, 10)
		Lighting.FogEnd = 2500
	end,
}

RemoveCustomSky = function()
	pcall(function()
		local e = Lighting:FindFirstChild("Valkyrie_Sky")
		if e then e:Destroy() end
		e = Lighting:FindFirstChild("FireHub_Sky")
		if e then e:Destroy() end
	end)
end

ApplySkybox = function(name)
	SkyboxCfg.Current = name or "Default"
	RemoveCustomSky()
	if name == "Default" or not SkyPresets[name] or not SkyPresets[name].faces then
		if SkyboxCfg.OriginalSky then
			pcall(function() SkyboxCfg.OriginalSky.Parent = SkyboxCfg.OriginalSkyParent or Lighting end)
		end
		return
	end
	pcall(function()
		if SkyboxCfg.OriginalSky and SkyboxCfg.OriginalSky.Parent then
			SkyboxCfg.OriginalSky.Parent = nil
		end
	end)
	local faces = SkyPresets[name].faces
	local sky = Instance.new("Sky")
	sky.Name = "Valkyrie_Sky"
	sky.SkyboxBk = faces.SkyboxBk
	sky.SkyboxDn = faces.SkyboxDn
	sky.SkyboxFt = faces.SkyboxFt
	sky.SkyboxLf = faces.SkyboxLf
	sky.SkyboxRt = faces.SkyboxRt
	sky.SkyboxUp = faces.SkyboxUp
	sky.StarCount = SkyPresets[name].stars or 3000
	sky.Parent = Lighting
end

ApplyLighting = function(name)
	local fn = LightingPresets[name]
	if fn then pcall(fn) end
end

local skyNames = { "Default", "Space", "Nebula", "Galaxy", "Deep Space", "Night Sky", "Sunset", "Sunrise", "Golden Hour", "Blood Moon", "Cyberpunk", "Neon" }
local lightNames = { "Default", "Day", "Night", "Sunset", "Purple", "Dark", "Bright", "Blood" }

skyTab:Section("Skybox", "Tap dropdown to cycle presets")
skyTab:Toggle("Enable Skybox", "Apply custom sky locally", false, function(v)
	SkyboxCfg.Enabled = v
	if v then
		ApplySkybox(SkyboxCfg.Current ~= "Default" and SkyboxCfg.Current or "Space")
	else
		RemoveCustomSky()
		if SkyboxCfg.OriginalSky then
			pcall(function() SkyboxCfg.OriginalSky.Parent = SkyboxCfg.OriginalSkyParent or Lighting end)
		end
	end
	Notify(v and "Sky ON" or "Sky OFF", 2)
end)
skyTab:Dropdown("Sky Preset", skyNames, "Space", function(name)
	SkyboxCfg.Current = name
	if SkyboxCfg.Enabled or name ~= "Default" then
		SkyboxCfg.Enabled = true
		ApplySkybox(name)
	end
	Notify("Sky: "..name, 2)
end)

skyTab:Section("Lighting", "Atmosphere / time of day")
skyTab:Dropdown("Lighting Preset", lightNames, "Default", function(name)
	ApplyLighting(name)
	Notify("Lighting: "..name, 2)
end)
skyTab:Slider("Clock Time", "0-24 hours", 0, 24, 14, function(v)
	pcall(function() Lighting.ClockTime = v end)
end, "h")
skyTab:Slider("Brightness", nil, 0, 5, 2, function(v)
	pcall(function() Lighting.Brightness = v end)
end)
skyTab:Button("Bright Ambient", false, function()
	Lighting.Ambient = Color3.fromRGB(150,150,150)
	Lighting.OutdoorAmbient = Color3.fromRGB(150,150,150)
end)
skyTab:Button("Dark Ambient", false, function()
	Lighting.Ambient = Color3.fromRGB(20,20,20)
	Lighting.OutdoorAmbient = Color3.fromRGB(20,20,20)
end)
skyTab:Button("Reset Lighting", false, function()
	ApplyLighting("Default")
	Notify("Lighting reset", 2)
end)

local perfTab = makeTab("Perf", "🚀")
perfTab:Section("FPS / FOV", nil)
perfTab:Toggle("FPS Unlock", nil, false, function(v) pcall(function() if setfpscap then setfpscap(v and 240 or 60) end end) end)
perfTab:Slider("FOV", nil, 50, 120, 70, function(v) pcall(function() Workspace.CurrentCamera.FieldOfView=v end) end, "°")
perfTab:Button("Disable Shadows", false, function() pcall(function() Lighting.GlobalShadows=false end) end)

local protTab = makeTab("Protect", "🛡")
protTab:Section("Anti-Ban", nil)
UIControls.AntiBan = protTab:Toggle("Anti-Ban Extreme", nil, true, function(v) AntiBan.Enabled=v; Notify("Anti-Ban "..(v and "ON" or "OFF"), 2) end)
local AntiAFKConn = nil
protTab:Toggle("Anti-AFK", nil, false, function(v)
	if v then
		if AntiAFKConn then AntiAFKConn:Disconnect() end
		AntiAFKConn = LocalPlayer.Idled:Connect(function()
			pcall(function() VirtualUser:CaptureController(); VirtualUser:ClickButton2(Vector2.new()) end)
		end)
	else
		if AntiAFKConn then AntiAFKConn:Disconnect(); AntiAFKConn=nil end
	end
end)

local menuTab = makeTab("Menu", "☰")
menuTab:Section("Window", nil)
menuTab:Label("RightControl — toggle menu")
menuTab:Label("Top bar: mark · title · x close · − half")
menuTab:Label("Mobile bar: Open · Half · Close  ·  Orb when closed")
menuTab:Button("Show Menu", true, function() show() end)
menuTab:Button("Hide Menu", false, function() hide() end)

-- push live state into every bound UI control (called after Load)
SyncUIFromState = function()
	pcall(function()
		if not reachSettings then return end
		local function set(ctl, val)
			if ctl and ctl.Set then pcall(function() ctl:Set(val) end) end
		end
		set(UIControls.HeadEnabled, reachSettings.HeadEnabled)
		set(UIControls.ReachX_Head, reachSettings.ReachX_Head)
		set(UIControls.ReachY_Head, reachSettings.ReachY_Head)
		set(UIControls.ReachZ_Head, reachSettings.ReachZ_Head)
		set(UIControls.RightArmEnabled, reachSettings.RightArmEnabled)
		set(UIControls.LeftArmEnabled, reachSettings.LeftArmEnabled)
		set(UIControls.ReachX_Arm, reachSettings.ReachX_Arm)
		set(UIControls.ReachY_Arm, reachSettings.ReachY_Arm)
		set(UIControls.ReachZ_Arm, reachSettings.ReachZ_Arm)
		set(UIControls.RightLegEnabled, reachSettings.RightLegEnabled)
		set(UIControls.LeftLegEnabled, reachSettings.LeftLegEnabled)
		set(UIControls.ReachX_Leg, reachSettings.ReachX_Leg)
		set(UIControls.ReachY_Leg, reachSettings.ReachY_Leg)
		set(UIControls.ReachZ_Leg, reachSettings.ReachZ_Leg)
		set(UIControls.ReachDistance, reachSettings.ReachDistance)
		set(UIControls.HeadVisualizer, reachSettings.HeadVisualizer)
		set(UIControls.RightArmVisualizer, reachSettings.RightArmVisualizer)
		set(UIControls.LeftArmVisualizer, reachSettings.LeftArmVisualizer)
		set(UIControls.RightLegVisualizer, reachSettings.RightLegVisualizer)
		set(UIControls.LeftLegVisualizer, reachSettings.LeftLegVisualizer)

		if GKHelper then set(UIControls.GKHelper, GKHelper.Enabled == true) end
		if CC then
			set(UIControls.ShowMobile, CC.btn and CC.btn.visible)
			set(UIControls.LockButtons, CC.btn and CC.btn.locked)
			set(UIControls.EnableMacros, CC.mobEnabled)
			set(UIControls.BtnSize, CC.btn and CC.btn.size)
			set(UIControls.BtnTransparency, CC.btn and CC.btn.transparency)
			set(UIControls.KeepToSave, CC.keepToSave)
		end
		if NRC then
			set(UIControls.NRC, NRC.Enabled)
			set(UIControls.NRC_Magnitude, NRC.Magnitude)
			set(UIControls.NRC_Distance, NRC.Distance)
		end
		if ReactKiller then
			set(UIControls.ReactKiller, ReactKiller.Enabled)
			set(UIControls.ReactKiller_Fires, ReactKiller.Fires)
		end
		if ReactBreak then
			set(UIControls.ReactBreak, ReactBreak.Enabled)
			set(UIControls.ReactBreak_Strength, ReactBreak.Strength)
			set(UIControls.ReactBreak_Distance, ReactBreak.Distance)
		end
		if CircleReach then
			set(UIControls.CircleEnabled, CircleReach.Enabled)
			set(UIControls.CircleRadiusX, CircleReach.RadiusX or CircleReach.Radius)
			set(UIControls.CircleRadiusY, CircleReach.RadiusY or CircleReach.Radius)
			set(UIControls.CircleRadiusZ, CircleReach.RadiusZ or CircleReach.Radius)
			set(UIControls.CircleVisible, CircleReach.Visible)
			set(UIControls.CircleTransparency, CircleReach.Transparency)
			set(UIControls.CircleMode, CircleReach.Mode or "Both")
		end
		if AntiBan then set(UIControls.AntiBan, AntiBan.Enabled) end
	end)
end

UserInputService.InputBegan:Connect(function(Input, Processed)
	if Processed then return end
	if Input.KeyCode == Enum.KeyCode.RightControl then
		if main.Visible then hide() else show() end
	end
end)

-- auto-load on boot
task.defer(function()
	task.wait(0.5)
	pcall(function()
		local ok = LoadConfigFromFile(CONFIG_FILE)
		if ok then
			task.wait(0.35)
			if PendingConfig then ApplyConfigSnapshot(PendingConfig) end
			Notify("Config loaded", 2)
		end
	end)
end)

pcall(function()
	game:BindToClose(function()
		pcall(function() SaveConfigToFile(CONFIG_FILE) end)
	end)
end)

local function boot(steps, onDone)
	steps = steps or {"checking", "building", "ready"}
	local bg = mount("Valkyrie_Boot"); bg.DisplayOrder = 100001
	local shade = bx(bg, Color3.new(0,0,0), 1); shade.Size = UDim2.fromScale(1,1); shade.BackgroundTransparency = 1
	local stage = fr(bg, 3); stage.AnchorPoint = Vector2.new(0.5,0.5); stage.Position = UDim2.fromScale(0.5,0.5); stage.Size = UDim2.fromOffset(340,280)

	local dots = {}
	for i = 1, 10 do
		local d = bx(stage, P.accent, 4)
		d.Size = UDim2.fromOffset(5,5); d.AnchorPoint = Vector2.new(0.5,0.5)
		d.BackgroundTransparency = 1; corner(d, 5); dots[i] = d
	end

	local mark = bx(stage, P.accent, 5)
	mark.AnchorPoint = Vector2.new(0.5,0); mark.Position = UDim2.new(0.5,0,0,42)
	mark.Size = UDim2.fromOffset(64,64); mark.BackgroundTransparency = 0.15; corner(mark, 32)
	stroke(mark, P.glow, 2, 0.25)
	local msc = Instance.new("UIScale"); msc.Scale = 0; msc.Parent = mark
	local markTx = tx(mark, "V", F_B, 28, P.hot)
	markTx.Size = UDim2.fromScale(1,1); markTx.TextXAlignment = Enum.TextXAlignment.Center; markTx.ZIndex = 6

	local tLabel = tx(stage, "VALKYRIE", F_D, 22, P.text)
	tLabel.AnchorPoint = Vector2.new(0.5,0); tLabel.Position = UDim2.new(0.5,0,0,124)
	tLabel.Size = UDim2.fromOffset(260,28); tLabel.TextXAlignment = Enum.TextXAlignment.Center
	tLabel.TextTransparency = 1; tLabel.ZIndex = 6

	local subL = tx(stage, ". U I", F_MO, 12, P.faint)
	subL.AnchorPoint = Vector2.new(0.5,0); subL.Position = UDim2.new(0.5,0,0,152)
	subL.Size = UDim2.fromOffset(140,14); subL.TextXAlignment = Enum.TextXAlignment.Center
	subL.TextTransparency = 1; subL.ZIndex = 6

	local barBg = bx(stage, P.off, 5)
	barBg.AnchorPoint = Vector2.new(0.5,0); barBg.Position = UDim2.new(0.5,0,0,200)
	barBg.Size = UDim2.fromOffset(220,6); barBg.BackgroundTransparency = 1; corner(barBg, 3)
	local bar = bx(barBg, P.accent, 6)
	bar.Size = UDim2.fromScale(0,1); corner(bar, 3)

	local cap = tx(stage, "", F_R, 11, P.dim)
	cap.AnchorPoint = Vector2.new(0.5,0); cap.Position = UDim2.new(0.5,0,0,214)
	cap.Size = UDim2.fromOffset(260,16); cap.TextXAlignment = Enum.TextXAlignment.Center
	cap.TextTransparency = 1; cap.ZIndex = 6

	local t0 = 0
	local orbitConn = RunService.RenderStepped:Connect(function(dt)
		t0 += dt
		for i, d in ipairs(dots) do
			local a = t0 * 1.4 + (i / #dots) * math.pi * 2
			local rad = 78 + math.sin(t0 * 2 + i) * 4
			d.Position = UDim2.new(0.5, math.cos(a) * rad, 0, 74 + math.sin(a) * rad)
		end
	end)

	task.spawn(function()
		tw(shade, 0.5, {BackgroundTransparency = 0.42}); task.wait(0.15)
		for _, d in ipairs(dots) do tw(d, 0.35, {BackgroundTransparency = 0.15}) end
		tw(msc, 0.6, {Scale = 1}, Enum.EasingStyle.Back); task.wait(0.4)
		tw(tLabel, 0.4, {TextTransparency = 0}); task.wait(0.15)
		tw(subL, 0.35, {TextTransparency = 0}); task.wait(0.2)
		tw(barBg, 0.25, {BackgroundTransparency = 0}); tw(cap, 0.25, {TextTransparency = 0})
		for i, step in ipairs(steps) do
			cap.Text = tostring(step)
			tw(bar, 0.45, {Size = UDim2.fromScale(i / #steps, 1)}, Enum.EasingStyle.Quint)
			task.wait(0.38)
		end
		task.wait(0.2)
		for _, d in ipairs(dots) do tw(d, 0.2, {BackgroundTransparency = 1}) end
		tw(tLabel, 0.2, {TextTransparency = 1}); tw(subL, 0.2, {TextTransparency = 1}); tw(cap, 0.2, {TextTransparency = 1})
		tw(barBg, 0.2, {BackgroundTransparency = 1}); tw(bar, 0.2, {BackgroundTransparency = 1})
		tw(msc, 0.3, {Scale = 0.35}); tw(shade, 0.4, {BackgroundTransparency = 1}); task.wait(0.35)
		pcall(function() orbitConn:Disconnect() end)
		show()
		pcall(function() bg:Destroy() end)
		if onDone then pcall(onDone) end
	end)
end

boot({
	"checking executor",
	"loading reach",
	"wiring gk systems",
	"syncing theme",
	"loading configs",
	"ready",
}, function()
	Notify("Valkyrie.ui loaded", "good")
	pcall(function()
		if CC and CC.btn then
			CC.btn.visible = true
			if ccSyncVisible then ccSyncVisible() end
			if ccSyncOpacity then ccSyncOpacity() end
		end
		local roots = {}
		pcall(function() if gethui then table.insert(roots, gethui()) end end)
		table.insert(roots, CoreGui)
		pcall(function() table.insert(roots, LocalPlayer:FindFirstChild("PlayerGui")) end)
		for _, root in ipairs(roots) do
			if root then
				local g = root:FindFirstChild("Fivelyss_MobileGK")
				if g and g:IsA("ScreenGui") then
					g.Enabled = true
					g.DisplayOrder = 2147483646
					local eye = g:FindFirstChild("ccEyeToggle")
					if eye then eye.Visible = true end
				end
			end
		end
		if PendingConfig then
			ApplyConfigSnapshot(PendingConfig)
		else
			LoadConfigFromFile(CONFIG_FILE)
		end
		if getgenv().Valkyrie_SetGkHud then
			getgenv().Valkyrie_SetGkHud(CC and CC.gkHudVisible ~= false)
		elseif ccSyncVisible then
			ccSyncVisible()
		end
	end)
end)

print("[Valkyrie.ui] Fivelyss systems online · red theme · mobile opacity · circle X/Y/Z axes · FIXED SetCircleReachEnabled")
