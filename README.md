-- shako.win | v6.7.4 · damage numbers Elisium (quita los del juego)
-- Avatar mundo + crash fixes · stable boot · no ESP reconnect leak

-- Boot: 1 sola instancia. Si FALLA el load, se puede reintentar.
-- Si ya cargó bien, ejecutar otra vez NO abre segunda UI.
pcall(function()
    if not game:IsLoaded() then game.Loaded:Wait() end
end)

-- Si falló un load anterior (sin UI), permitir reintento
if getgenv().antiSleepy_loaded then
    if getgenv()._ShakoUIReady == true then
        return -- instancia real activa
    end
    -- load a medias → limpiar y reintentar
    getgenv().antiSleepy_loaded = false
end
getgenv().antiSleepy_loaded = true

-- ==================== AXIS/HARION ANTI-CHEAT BYPASS ====================
pcall(function()
(function()
    -- 1) Yukleme ekrani tamamen kapanana kadar bekle (autoexec'te oyun acilmadan donmayi onler)
    local players = game:GetService("Players")
    local coreGui = game:GetService("CoreGui")
    if not game:IsLoaded() then game.Loaded:Wait() end
    local player = players.LocalPlayer
    while not player do
        players:GetPropertyChangedSignal("LocalPlayer"):Wait()
        player = players.LocalPlayer
    end
    local playerGui = player:WaitForChild("PlayerGui")
    local function isActiveLoadingGui(object)
        local n = object.Name:lower():gsub("[%s_%-]", "")
        if n ~= "loadingscreen" and not n:find("loadingscreen", 1, true) then return false end
        local cur = object
        while cur and cur ~= playerGui and cur ~= coreGui do
            if cur:IsA("LayerCollector") and not cur.Enabled then return false end
            if cur:IsA("GuiObject") and not cur.Visible then return false end
            cur = cur.Parent
        end
        return object:IsDescendantOf(playerGui) or object:IsDescendantOf(coreGui)
    end
    local clearSince
    local t0 = os.clock()
    repeat
        local loading = false
        for _, root in ipairs({ playerGui, coreGui }) do
            local ok, list = pcall(function() return root:GetDescendants() end)
            if ok then
                for _, object in ipairs(list) do
                    if isActiveLoadingGui(object) then loading = true break end
                end
            end
            if loading then break end
        end
        clearSince = loading and nil or (clearSince or os.clock())
        task.wait(0.1)
    until (clearSince and os.clock() - clearSince >= 0.5) or os.clock() - t0 > 60

    -- 2) Anti-cheat hook'lari (oturum basina bir kez)
    local genvL = getgenv()
    if genvL.__ShakoStartupHooks then return end
    genvL.__ShakoStartupHooks = true
    local function ncc(fn) return (type(newcclosure) == "function" and newcclosure(fn)) or fn end
    local cref = (type(cloneref) == "function" and cloneref) or function(o) return o end
    if type(setthreadidentity) == "function" then pcall(setthreadidentity, 8) end

    -- anti-cheat modullerinin zayif (weak) tablolarini etkisizlestir
    local okEnv, renv = pcall(function() return getrenv() end)
    local smt = okEnv and type(renv) == "table" and renv.setmetatable or nil
    if type(hookfunction) == "function" and smt then
        local oldSmt = smt
        local okHook, hooked = pcall(hookfunction, smt, ncc(function(Table, Metatable)
            if Metatable and type(Metatable) == "table" and rawget(Metatable, "__mode") then
                local mode = rawget(Metatable, "__mode")
                if mode == "kv" or mode == "v" or mode == "k" then
                    local okT, trace = pcall(debug.traceback)
                    trace = okT and trace or ""
                    if trace:find("MiscellaneousController", 1, true)
                        or trace:find("CameraSecurity", 1, true)
                        or trace:find("AnalyticsPipelineController", 1, true) then
                        return oldSmt({ 1, 2, 3 }, {})
                    end
                end
            end
            return oldSmt(Table, Metatable)
        end))
        if okHook and hooked then oldSmt = hooked end
    end

    local lpl = cref(players).LocalPlayer
    if lpl and type(hookfunction) == "function" then
        -- istemciden kick'i engelle
        pcall(function()
            local oldKick
            oldKick = hookfunction(lpl.Kick, ncc(function(self, ...)
                if self == lpl then return nil end
                return oldKick(self, ...)
            end))
        end)
        -- MiscellaneousController'a sahte mouse ver
        pcall(function()
            local oldGetMouse
            oldGetMouse = hookfunction(lpl.GetMouse, ncc(function(self, ...)
                if self == lpl then
                    local okT, trace = pcall(debug.traceback)
                    if okT and trace:find("MiscellaneousController", 1, true) then
                        local realMouse = oldGetMouse(self, ...)
                        return setmetatable({}, {
                            __index = function(_, key)
                                if key == "X" or key == "Y" then
                                    local loc = game:GetService("UserInputService"):GetMouseLocation()
                                    return key == "X" and loc.X or loc.Y
                                end
                                local val = realMouse[key]
                                if type(val) == "function" then return function(_, ...) return val(realMouse, ...) end end
                                return val
                            end,
                            __newindex = function(_, key, value) realMouse[key] = value end,
                        })
                    end
                end
                return oldGetMouse(self, ...)
            end))
        end)
    end

    -- CameraSecurity modulunu kor et
    pcall(function()
        local cs = require(lpl:WaitForChild("PlayerScripts"):WaitForChild("Modules"):WaitForChild("CameraSecurity", 10))
        local mt = getrawmetatable(cs)
        if mt then
            if type(setreadonly) == "function" then pcall(setreadonly, mt, false) end
            mt.__index = function() return nil end
            mt.__tostring = function() return "LocalPlayer = nil" end
            mt.__newindex = function(E, G, f) return rawset(E, G, f) end
        end
    end)

    -- servis isimlerini degistir (Harion ile ayni)
    pcall(function()
        for _, service in pairs({ cref(players), workspace, cref(game:GetService("ReplicatedStorage")), cref(game:GetService("ReplicatedFirst")) }) do
            service.Name = service.Name .. " "
        end
    end)

    -- oyunun ping kalp atisini taklit et (anti-cheat kapaninca oyunun kendi ping'i durur)
    task.spawn(function()
        local ok, ServerPing = pcall(function() return workspace:WaitForChild("ServerPing", 30) end)
        local Remotes = game:GetService("ReplicatedStorage"):FindFirstChild("Remotes")
        local PingRemote = Remotes and Remotes:FindFirstChild("Ping")
        if not PingRemote then return end
        while task.wait(12 * math.random()) do
            local rnd = math.random(1, 9999)
            local sp = (ok and ServerPing and ServerPing.Value) or -1
            local value = rnd == 6961 and 2137 or rnd == sp and 2138 or rnd
            pcall(function() PingRemote:FireServer(value) end)
        end
    end)
    print("[shako] anti-cheat bypass aktif")
end)()

-- ============================================================
-- 5 SANIYE BASLATMA GECIKMESI
-- Anti-cheat bypass hemen calisir (donma korumasi korunur),
-- ardindan hem manuel execute hem de teleport queue (matchmaking)
-- sonrasi menu ve tum ozellikler 5 saniye sonra baslar.
-- ============================================================

end)
task.wait(2) -- breve delay post-bypass (AXIS usa 5s; 2s para no tardar tanto)
-- ======================================================================


-- kill avatar lag loops from old sessions
pcall(function()
    if getgenv()._ShakoAvatarKeepThread then
        task.cancel(getgenv()._ShakoAvatarKeepThread)
        getgenv()._ShakoAvatarKeepThread = nil
    end
end)

getgenv()._ShakoUIReady = false
getgenv()._ShakoBootAt = tick()

-- ==================== SOLO RIVALS ====================
-- GameId oficial de Rivals
local RIVALS_GAME_ID = 6035872082
local function notSupportedKick()
    local msg = "this game not supported"
    warn("[shako.win] " .. msg .. " | GameId=" .. tostring(game.GameId))
    pcall(function()
        local lp = game:GetService("Players").LocalPlayer
        if lp then lp:Kick(msg) end
    end)
    -- si el kick no corre, forzar leave
    pcall(function()
        game:GetService("TeleportService"):Teleport(0) -- invalido = force exit intent
    end)
    getgenv().antiSleepy_loaded = false
end

local _gid = tonumber(game.GameId)
local _okGame = (_gid == RIVALS_GAME_ID)
if not _okGame then
    -- algunos executors reportan PlaceId distinto; solo warn + allow si PlaceId conocido
    warn("[shako.win] GameId=" .. tostring(_gid) .. " (esperado " .. tostring(RIVALS_GAME_ID) .. ")")
    -- no kick hard: deja cargar para debug; descomenta kick si quieres estricto
    -- notSupportedKick(); return
end
-- =====================================================

local function shakoFail(msg)
    warn("[shako]", msg)
    getgenv().antiSleepy_loaded = false -- permite re-ejecutar SOLO si falló
end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local VirtualUser = game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then
    for _ = 1, 100 do
        LocalPlayer = Players.LocalPlayer
        if LocalPlayer then break end
        task.wait(0.05)
    end
end
if not LocalPlayer then
    shakoFail("LocalPlayer nil — reintenta en 1s")
    return
end

-- PlayerGui listo (sin esto la UI a veces no sale al entrar)
pcall(function() LocalPlayer:WaitForChild("PlayerGui", 12) end)

local Camera = workspace.CurrentCamera

-- ==================== Obsidian UI ====================
local repo = "https://raw.githubusercontent.com/xyznick/UELinoriaLib/main/"
local Library, ThemeManager, SaveManager

local function httpGet(url)
    local ok, body = pcall(function()
        return game:HttpGet(url)
    end)
    if ok and body and #body > 100 then return body end
    return nil
end

local function try_load(url)
    local body = httpGet(url)
    if not body then return nil end
    local ok, res = pcall(function() return loadstring(body)() end)
    if ok and res then return res end
    return nil
end

-- UELinoriaLib (misma UI que Aurora spoof) + fallbacks
local UELINORIA_REPO = "https://raw.githubusercontent.com/xyznick/UELinoriaLib/main/"
local LINORIA_REPO = "https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/"
local OBSIDIAN_REPO = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"

Library = try_load(UELINORIA_REPO .. "Library.lua")
ThemeManager = try_load(UELINORIA_REPO .. "addons/ThemeManager.lua")
SaveManager = try_load(UELINORIA_REPO .. "addons/SaveManager.lua")
local _usingLinoria = Library ~= nil
local _usingUE = Library ~= nil

if not Library then
    task.wait(0.25)
    Library = try_load(LINORIA_REPO .. "Library.lua")
    ThemeManager = try_load(LINORIA_REPO .. "addons/ThemeManager.lua")
    SaveManager = try_load(LINORIA_REPO .. "addons/SaveManager.lua")
    _usingLinoria = Library ~= nil
    _usingUE = false
end
if not Library then
    task.wait(0.25)
    Library = try_load(OBSIDIAN_REPO .. "Library.lua")
    ThemeManager = try_load(OBSIDIAN_REPO .. "addons/ThemeManager.lua")
    SaveManager = try_load(OBSIDIAN_REPO .. "addons/SaveManager.lua")
    _usingLinoria = false
    _usingUE = false
end

if not Library or type(Library.CreateWindow) ~= "function" then
    shakoFail("UI Library no cargó (UELinoria/Linoria/Obsidian)")
    return
end

-- Shim: Linoria no usa Callback en AddToggle/Slider/Dropdown como Obsidian
do
    local function wrapGroupbox(gb)
        if not gb or gb.__shakoPatched then return gb end
        gb.__shakoPatched = true
        local function hook(method, valueKey)
            local old = gb[method]
            if type(old) ~= "function" then return end
            gb[method] = function(self, idx, opts)
                opts = opts or {}
                local cb = opts.Callback
                local ret = old(self, idx, opts)
                if cb and _usingLinoria then
                    task.defer(function()
                        local Toggles = Library.Toggles or getgenv().Toggles
                        local Options = Library.Options or getgenv().Options
                        if method == "AddToggle" and Toggles and Toggles[idx] then
                            pcall(function()
                                Toggles[idx]:OnChanged(function()
                                    pcall(cb, Toggles[idx].Value)
                                end)
                            end)
                        elseif Options and Options[idx] then
                            pcall(function()
                                Options[idx]:OnChanged(function()
                                    local v = Options[idx].Value
                                    pcall(cb, v)
                                end)
                            end)
                        end
                    end)
                end
                return ret
            end
        end
        hook("AddToggle")
        hook("AddSlider")
        hook("AddDropdown")
        -- AddButton may differ
        local oldBtn = gb.AddButton
        if type(oldBtn) == "function" then
            gb.AddButton = function(self, opts)
                if type(opts) == "table" and opts.Func and not opts.Callback then
                    -- Obsidian style {Text=, Func=}
                    if _usingLinoria then
                        return oldBtn(self, opts.Text or "Button", opts.Func)
                    end
                end
                return oldBtn(self, opts)
            end
        end
        return gb
    end

    local function wrapTab(tab)
        if not tab or tab.__shakoPatched then return tab end
        tab.__shakoPatched = true
        for _, m in ipairs({ "AddLeftGroupbox", "AddRightGroupbox", "AddGroupbox" }) do
            local old = tab[m]
            if type(old) == "function" then
                tab[m] = function(self, ...)
                    local gb = old(self, ...)
                    return wrapGroupbox(gb)
                end
            end
        end
        return tab
    end

    local oldCreate = Library.CreateWindow
    Library.CreateWindow = function(self, opts)
        opts = opts or {}
        -- Linoria no siempre soporta Footer/Resizable/NotifySide
        local ok, win = pcall(oldCreate, self, {
            Title = opts.Title or "shako.win",
            Center = opts.Center ~= false,
            AutoShow = opts.AutoShow ~= false,
            TabPadding = 8,
        })
        if not ok or not win then
            return oldCreate(self, opts)
        end
        local oldAddTab = win.AddTab
        if type(oldAddTab) == "function" then
            win.AddTab = function(self, name, ...)
                local tab = oldAddTab(self, name, ...)
                return wrapTab(tab)
            end
        end
        return win
    end

    -- Notify compatibility: Obsidian table vs Linoria string
    local oldNotify = Library.Notify
    if type(oldNotify) == "function" then
        Library.Notify = function(self, a, b, c)
            if type(a) == "table" then
                local title = a.Title or a.title or "shako"
                local desc = a.Description or a.description or a.Text or ""
                local t = a.Time or a.time or 3
                local msg = (desc ~= "" and (tostring(title) .. " · " .. tostring(desc))) or tostring(title)
                return oldNotify(self, msg, t)
            end
            return oldNotify(self, a, b, c)
        end
    end
end
getgenv()._ShakoUsingLinoria = _usingLinoria

pcall(function() Library.ShowCustomCursor = true end)

local LucideIcons = {}
pcall(function()
    LucideIcons = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/Icons/refs/heads/main/lucide/dist/Icons.lua"))() or {}
end)
local function Icon(n)
    local f = LucideIcons[string.lower(tostring(n or ""))]
    if type(f) == "string" then
        local id = f:match("rbxassetid://(%d+)") or f:match("(%d+)")
        if id then return id end
    end
    return nil
end

if ThemeManager and ThemeManager.SetLibrary then
    pcall(function()
        ThemeManager:SetLibrary(Library)
        ThemeManager:SetFolder("shako_win")
    end)
end
if SaveManager and SaveManager.SetLibrary then
    pcall(function()
        SaveManager:SetLibrary(Library)
        SaveManager:SetFolder("shako_win/configs")
        if ThemeManager and SaveManager.IgnoreThemeSettings then
            pcall(function() SaveManager:IgnoreThemeSettings() end)
        end
        pcall(function() SaveManager:SetIgnoreIndexes({"MenuKeybind"}) end)
    end)
end

local Window
do
    local ok, win = pcall(function()
        -- ventana mas ancha + fuente limpia (no Code)
        pcall(function()
            Library.Font = Enum.Font.Gotham
            if Library.SetFontSize then
                Library:SetFontSize(13)
            else
                Library.FontSize = 13
            end
        end)
        return Library:CreateWindow({
            Title = "shako.win",
            Footer = "v6.7.0 · Elisium + HARION · AXIS",
            Center = true,
            AutoShow = true,
            Resizable = true,
            NotifySide = "Right",
            ShowCustomCursor = true,
        })
    end)
    if not ok or not win then
        shakoFail("CreateWindow falló — vuelve a ejecutar")
        return
    end
    Window = win

-- Cursor UI visible al abrir
pcall(function()
    Library.ShowCustomCursor = true
    UserInputService.MouseBehavior = Enum.MouseBehavior.Default
    UserInputService.MouseIconEnabled = true
end)
end

-- Forzar label Rivals (no Marketplace)
pcall(function()
    if Library.SetWatermark then
        Library:SetWatermark("shako.win")
    end
    if Library.Watermark then
        pcall(function() Library.Watermark.Text = "shako.win · Rivals" end)
    end
end)

-- silenciar notificaciones de la UI (menos Config)
pcall(function()
    if Library and Library.Notify then
        getgenv()._ShakoLibraryNotify = Library.Notify
        Library.Notify = function(self, data)
            -- bloquear spam de toggles; configs usan Notify() → StarterGui
            return
        end
    end
end)

function Notify(title, text)
    if tostring(title) ~= "Config" then return end
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {
            Title = "Config",
            Text = tostring(text or ""),
            Duration = 3,
        })
    end)
    pcall(function()
        local fn = getgenv()._ShakoLibraryNotify
        if type(fn) == "function" then
            fn(Library, { Title = "Config", Description = tostring(text or ""), Time = 3 })
        end
    end)
end
getgenv()._ShakoBootAt = tick()



-- ==================== SPOOFERS REMOVIDOS (full rage + legit) ====================
-- stubs minimos por si algo queda referenciado
local states = { spoofer_state = {}, skinchanger_state = { equipped = {}, fake_owned = {}, fake_weapon_owned = {} } }
local modules = modules or {}
local function SpoofDevice() end
local function ApplyNameSpoof() end
local function RefreshAllNameSpoofs() end
local function ApplyThumbnailSpoof() end
local function UnlockAll() end
local function UnlockSelectedRarity() end
local function EquipApply() end
getgenv().RivalsSetAvatarSpoof = function() end
getgenv().RivalsClearAvatarSpoof = function() end
getgenv().RivalsBroadcastAvatarSpoof = function() end

-- stubs spoof (por si configs viejas los llaman)
RivalsSpoof = RivalsSpoof or {}
function RivalsRefreshNames() end
function RivalsUpdateBadges() end
function RivalsSetAnonymous() end
function RivalsApplyDevice() end


getgenv().config = {

    Enabled = false,
    StickHeight = -2.8,
    StickOffsetX = 0,
    StickOffsetZ = 0,
    StickMultiPos = false,
    StickDesync = false, -- no ves el TP (igual que rage)
    StickAway = 6,
    StickPosInterval = 0.18,
    StickFireHead = false,

    MaxRange = 1e9, -- infinito efectivo anti-UE
    TeamCheck = false, -- NUNCA pegarse al team si ON

    PriorityMode = "Closest",
    TargetLock = false,
    FakePosition = false,
    FakePosStrength = 8,
    HitboxExpand = false,
    HitboxSize = 16,

    -- UE Counter (ex Anti Unnamed)
    UECounterEnabled = false,
    UECounterY = 500,
    UECounterCooldown = 0.2,
    UECounterHideDist = 500, -- hide / void height
    UECounterAttackSpeed = 1, -- velocidad ataque/reposición
    AntiUnnamed = false, -- alias de UECounterEnabled (otros sistemas)

    AntiSafeZone = false,
    SafeZoneMinY = 30,
    AntiVoidRescue = false,
    AntiVoidY = 40,

    StrafeEnabled = false,
    OrbitEnabled = false,
    OrbitRange = 10,
    OrbitSpeed = 16,
    StrafeSpeed = 18,

    -- Hit / Kill Songs
    HitSoundEnabled = false, -- legacy OFF (Elisium hitsounds)

    KillSoundEnabled = false, -- OFF (user quitó kill sounds)

    HitSoundVolume = 6,
    KillSoundVolume = 7,
    HitSoundMode = "Random",
    KillSoundMode = "Random",
    SelectedHitSound = "Rust HS",
    HitSoundPitch = 1,
    SelectedKillSound = "OOF",
    HitSoundCooldown = 0.08,
    KillSoundCooldown = 0.35,

    TargetName = "",
    UseNameTarget = false,

    FlyEnabled = false,
    FlySpeed = 120,
    FlyBoost = 1.8,
    JumpPowerEnabled = false,
    JumpPower = 100,
    JumpBoostForce = true, -- fuerza Y en Rivals (WalkSpeed/Jump se resetean)
    NoclipEnabled = false,
    AntiFling = true,
    SpeedEnabled = false,
    WalkSpeed = 50,
    SpeedVelocityMode = true,
    DefaultWalkSpeed = 16,
    DefaultJumpPower = 50,
    DefaultJumpHeight = 7.2,
    SlideBoostEnabled = false,
    SlideBoostSpeed = 60, -- Elisium style (direct speed on _sliding_velocity)
    -- Disable animations (Elisium)
    DisableAnimsEnabled = false,
    DisableAnimsSelect = {}, -- bobbing, landing, sway, equip, inspect, sliding, shoot, impulse, aim, sprint, attack, charge, reload, throw

    SlideBoostMult = 2.5,
    SlideBoostKey = "C", -- solo C (no Control)
    SlideBaseSpeed = 45,
    InfinityJump = false,

    AntiAimEnabled = false,
    AntiAimMode = "PitchYaw",
    -- Anti Aim estilo HARION (UpdateCameraRotation remote)
    AAPitchMode = "down",
    AAPitchOffset = 250,
    AAPitchAngle = 0,
    AAYawMode = "jitter",      -- disabled / backwards / spin / random / jitter / side / opposite / strong
    AAYawAngle = 180,
    SpinSpeed = 25,
    JitterAngle = 180,
    AAJitterAmount = 90,       -- grados max del jitter yaw
    AAStrongMode = false,      -- burst fuerte multi-packet
    AAPacketBurst = 5,         -- paquetes por frame en strong
    AASpinRate = 720,          -- deg/s spin
    ManualYaw = 180,
    AAPitch = 0,
    AAInvert = false,
    AAHideLocal = false,
    -- Underground AA (pitch down + jitter, NO tp bajo mapa)
    AntiAimUnderground = false,
    AntiAimUndergroundY = -25, -- degrees pitch down (legacy name, now pitch)
    UndergroundPitch = -25,    -- -15 a -50 recomendado
    UndergroundJitter = 35,    -- yaw jitter degrees
    UndergroundRandom = true,  -- random delay/variation
    UndergroundSpin = 0.4,     -- factor de spin suave (0-1)

    SpawnDelay = 1.0,

    RagebotEnabled = false,

    -- AXIS ragebot keys
    TeleportOffsetY = 1,
    RageShootAttempts = 1,
    RageTeleportDelay = 0.04,
    RageVoidSpamAfterKill = false,
    RageHover = false,
    RageVoidOnReload = false,
    RageShotgunVoidTime = 5,
    RageVoidMinDist = 1,
    RageVoidRange = 50000,
    RageVoidIterations = 120,
    RageAntiAim = false,
    BodyTwist = false,
    BodyTwistMode = "Rager", -- Rager | Broken | Twist | Random | Lay | Ultra
    BodyTwistPower = 1.8, -- ultra fold default
    BodyTwistSpeed = 8,
    BodyTwistWithRage = true,
    BodyTwistWithAA = true, -- auto con anti-aim
    RageNoclip = false,
    RagePreferredWeapon = "primary",
    RageAutoEquip = false,
    RageAutoSwap = false,
    RageTargetMode = "Closest",
    RageAutoPriority = false,
    RagePriorityAttackers = false,
    RagePriorityVoided = false,
    RageDistance = 2.5,
    HarionFastShoot = false,
    HarionNoSpread = false,
    WallCheckEnabled = false,
    WallCheckRadius = 2,
    WallCheckAttempts = 4,
    WallCheckFallbackY = 5,
    VoidLockEnabled = true,

    -- Sleepy Counter: TP al lado de la mano (lejitos) + auto head
    SleepyCounterEnabled = false,
    SleepyCounterSideDist = 14,   -- lejos de la mano
    SleepyCounterHandY = 0.5,
    SleepyCounterHeight = 1.2,
    SleepyCounterFireRate = 0.02,
    SleepyCounterTeamCheck = false,
    SleepyCounterRange = 1e12,
    -- Kicia Counter: anti kiciahook rage
    KiciaCounterEnabled = false,
    KiciaCounterSkyY = 450,          -- altura en el cielo
    KiciaCounterLineWidth = 80,      -- ancho de la linea lado a lado
    KiciaCounterMoveSpeed = 400,     -- studs/seg entre puntos (rapido)
    KiciaCounterFireRate = 0.015,
    KiciaCounterTeamCheck = false,
    KiciaCounterShoot = true,        -- seguir disparando a head mientras esta en cielo
    RageIndicatorEnabled = true,
    RageStatusEnabled = true,
    RageIndicatorSnapline = true,
    RageIndicatorColor = "Red",
    RageStatusMode = "static", -- static / crosshair
    RageStatusOffsetY = 48,

    RageDesync = true, -- no ves el TP (restore post-fire)
    RageHideTP = true,
    RageLookSyncOnShoot = false,
    RageHideLocal = false,
    RagebotHeight = 2.2,
    RagebotRange = math.huge,
    RagebotInfiniteRange = true,
    RagebotTeamCheck = true, -- no atacar team
    RagebotForceFFA = false,
 -- true = siempre FFA (ignora team)
    RagebotForceFFA = false,
    RagebotPriorityHP = true,
    RagebotSkipFF = false,
    RagebotLookAtHead = false,
    RagebotOffsetX = 0,
    RagebotOffsetZ = 0,
    RagebotPredict = true,
    RagebotPredictAmount = 0.12,
    -- Rivals UseItem rage
    RageWeaponSlot = "Primary", -- no Fists por defecto

    RageFireRate = 0,
    RageAttackSpeed = 10, -- UE-style: que tan rapido se tpea/ataca (1 lento - 20 insta)
    RageDesyncKnifeY = 6,
    RageDesyncNormalY = 1,
    RageDesyncNormalZ = 2,
    RageSkipDeflect = true,
    RageUseItem = true, -- always on with ragebot

    RageShotsPerTick = 10,         -- Harion shoot attempts
    RageAttackMode = "gun",        -- gun | knife | melee
    RagePreferredWeapon = "primary", -- primary | secondary | melee

    RageWeaponSpecialize = true,
    RageSwapWhenEmpty = true,
    RageAutoPriority = false,
    RagePriorityAttackers = false,
    RagePriorityVoided = false,

    RageMultipoint = true,         -- head + upper torso
    RageSwitchDelay = 0.05,        -- al morir target, siguiente
    RageResolver = true,           -- ajusta Y si el target se mueve raro
    RageBodyAimIfWall = false,     -- si head bloqueada usa torso


    VoidRage = false,
    VoidRageAwayDist = 99999999,
    RageVoidFirst = true,
    RageVoidOnKill = true,
    RageVoidHideTime = 0.25, -- Harion hide
    RageVoidAttackTime = 0.03, -- Harion attack window

    RagePrioritizeRagers = true,
    VoidRageInterval = 0.35,
    VoidRageMethods = "Side",
    VoidRageReturn = true,

    AimbotEnabled = false,
    AimbotFOV = 400,
    AimbotSmooth = 0.9,
    AimbotPart = "Head",
    AimbotTeamCheck = false,
    AimbotKeyHold = false, -- NO RMB: solo keybind ...
    MenuKey = "RightShift",
    KeybindListEnabled = true,
    KeybindListTransparency = 0.25,
    KeybindListOnlyActive = false,
    KeybindListPosX = 20,
    KeybindListPosY = 120,

    SilentAimEnabled = false,
    SilentHitPart = "Head",
    SilentPredict = false, -- OFF por defecto
    SilentPredictAmount = 0.06,
    SilentHeadOffsetY = 0.35, -- un poco arriba del centro de head
    SilentFOV = 2000, -- FOV grande anti-UE lejano
    SilentShowFOV = true,
    SilentTeamCheck = false,
    SilentForceAiming = true,
    SilentFOVFill = true,
    SilentFOVFillTransparency = 0.92,
    SilentFOVAnimated = true,
    SilentFOVSpin = true,
    SilentFOVSpinSpeed = 1.2,
    SilentFOVThickness = 1.5,
    SilentFOVColor = "White",
    SilentFOVOutlineColor3 = Color3.fromRGB(255, 255, 255),
    SilentFOVFillC1 = Color3.fromRGB(255, 105, 180),
    SilentFOVFillC2 = Color3.fromRGB(180, 100, 255),
    SilentFOVFillC3 = Color3.fromRGB(80, 180, 255),

    AimbotWallCheck = false,
    AimbotVisibleCheck = false,
    AimbotShowFOV = true,
    AimbotFOVFill = true,
    AimbotFOVFillTransparency = 0.92,
    AimbotFOVAnimated = false,
    AimbotFOVColor = "White",
    AimbotFOVOutlineColor3 = Color3.fromRGB(255, 230, 80),
    AimbotFOVFillC1 = Color3.fromRGB(255, 210, 80),
    AimbotFOVFillC2 = Color3.fromRGB(255, 140, 60),
    AimbotFOVFillC3 = Color3.fromRGB(255, 80, 120),
    AimbotSmooth = 0.15,

    WeatherEnabled = false,
    AmbientSoundEnabled = false,
    AmbientSoundType = "Rain",
    AmbientSoundVolume = 0.45,
    AmbientSoundPitch = 1,
    Weather = false,
    WeatherType = "Rain",
    WeatherIntensity = 1,
    WeatherSoundVolume = 0.6,
    WeatherStorm = false,
    WeatherStormMin = 8,
    WeatherStormVar = 6,
    WeatherMeteors = false,
    WeatherMeteorRate = 1,
    WeatherShootingStars = false,
    WeatherStarRate = 1,
    WeatherPuddles = false,
    WeatherMood = true,
    WeatherClockDial = false,
    WeatherRainbow = false,
    WeatherGodRays = false,
    SkyboxPreset = "Off",

    EmoteEnabled = false,
    EmoteId = "5917459365",
    EmoteName = "Floss",
    EmoteSpeed = 1,
    EmoteLoop = false,
    EmoteKeepPlaying = false,
    EmoteAlwaysOn = false,
    EmotePriority = "Action4",

    AutoClickEnabled = false,
    AutoClickCPS = 16,
    AutoClickOnlyOnTarget = true,

    NoRecoil = false,
    NoRecoilStrength = 1,
    NoRecoilOnlyShoot = true,
    NoRecoilScanGun = false,

    RapidFire = false,
    RapidFireValue = 999,
    RapidFireDelay = 0.001,
    RapidFireLoop = true,
    FastMelee = false,
    ItemLibraryKeepAlive = true,

    VoidSpamEnabled = false,
    VoidMethod = "Quantum",
    VoidSpeed = 100000000000,
    VoidChaos = 0.98,
    VoidBaseAltitude = 100000000000, -- distancia extrema
    VoidRadius = 100000000000,
    VoidEvade = true,
    VoidEvadeDist = 100000000000,
    VoidEvadeRadius = 100000000000,
    VoidEvadeSpeed = 100000000000,
    VoidEvadeCooldown = 0.05,
    VoidEvadeTrigger = 1.0, -- Lua x paid
    VoidEvadeStrength = 1.0,
    VoidEvadeVert = 3e9,
    VoidExtremeMode = true, -- speeds estilo Lua x paid
    VoidChaosVel = true, -- velocity chaos random


    PredDodgeEnabled = false,
    AutoCollectHeals = false, -- FFA packs (vida/balas)
    AutoCollectRadius = 60,
    AutoCollectAmmo = true,
    AutoCollectOnlyFFA = true,

    BlinkEnabled = false, -- meoww-style blink anti-rager
    BlinkRadius = 90,
    BlinkRate = 40,

    PredRadius = 90,
    PredDodgeDist = 120,
    PredCooldown = 0.12,
    PredThreshold = 55, -- HVH: detecta ragers rapido
    PredMult = 1,

    TriggerbotEnabled = false,
    TriggerbotFOV = 120,
    TriggerbotCPS = 12,
    TriggerbotPart = "Head",

    AutoEquipMelee = false,
    AutoEquipEnabled = false,
    AutoEquipName = "Chainsaw",
    AutoEquipAny = false,
    AutoEquipSlot = 3, -- 1 primary 2 secondary 3 melee
    AutoEquipInterval = 0.35,
    AutoEquipDelay = 0.8,
    MeleeName = "Chainsaw",
    DesyncJitter = false,
    DesyncJitterPower = 12,

    -- HVH Defense
    HurtEvade = false,
    HurtEvadeDist = 55,
    HurtEvadeCooldown = 0.35,
    NearEnemyEvade = false,
    NearEnemyDist = 25,
    NearEnemyPush = 45,
    AdaptiveDesync = false,
    AdaptiveDesyncPower = 18,
    AdaptiveDesyncRange = 200,
    VelocityBreak = false,
    VelocityBreakMax = 140,
    RandomMicroMove = false,
    RandomMicroPower = 3,
    AntiMeleeRange = 20,
    AntiMeleePush = 40,
    AntiMeleeEnabled = false,
    LowHPPanic = false,
    LowHPThreshold = 30,
    LowHPBoostDist = 80,

    -- Anti UE / Kicia HVH
    InfiniteRange = true,          -- ignora MaxRange en target
    AntiUEClose = false,           -- si UE se pega cerca → push lejos
    -- Riot (Lua x paid)
    RiotEnabled = false,
    RiotSpeed = 0.03,
    RiotRange = 50,
    RiotEvadeRange = 30,
    RiotSpinSpeed = 180,
    RiotAbuseEnabled = false,
    RiotAbuseMode = "Stick",
    RiotAbuseHeight = 3,
    RiotAbuseForward = 0,
    RiotAbuseRight = 0,
    RiotAbuseDown = 0,

    AntiUECloseDist = 35,
    AntiUEClosePush = 80,
    AntiUEPeek = false,            -- side peek al detectar rager cerca
    AntiUEPeekDist = 40,
    RageInfinite = true,           -- rage sin límite de distancia
    StickFarMode = false,          -- stick más lejos (anti melee UE)
    StickFarMult = 2.5,
    DesyncSpam = false,            -- micro desync continuo anti hit
    DesyncSpamPower = 8,
    HitboxBig = false,
    HitboxBigSize = 18,


    
    -- Auto match / loadout (Rivals Duels)
    AutoLoadoutEnabled = false,
    AutoLoadoutPrimary = "Assault Rifle",
    AutoLoadoutSecondary = "Handgun",
    AutoLoadoutMelee = "Fists",
    AutoLoadoutUtility = "Grenade",
    AutoLoadoutInterval = 2,
    AutoVoteEnabled = false,
    AutoVoteMap = "Arena",
    AutoVoteInterval = 1,
    AutoBanWeaponEnabled = false,
    AutoBanWeapons = {}, -- list of names to cycle ban
    AutoBanWeaponCurrent = "Sniper",
    AutoBanInterval = 1,
    AntiKick = true,
    SoftStick = false,
    SoftStickSpeed = 180,
    SoftStickMaxStep = 5,
    -- TP atras del enemigo (sin delay)
    StickBehindTP = false,
    StickBehindDist = 10,   -- offset atras del enemigo
    StickBehindHeight = 1.5,
    StickBehindRange = math.huge, -- alcance infinito para TP
    CameraIndependent = false,
    BlockIllegalNotify = true,

    CameraLookAtTarget = false,
    ForceZeroVelocity = true,
    StickOnRender = true,
    LockYWhileStick = false,

    Fullbright = false,
    Brightness = 2,
    Exposure = 0,
    ClockTime = 14,
    NoFog = true,
    Saturation = 0,
    Contrast = 0,
    BloomEnabled = false,
    BloomIntensity = 0.4,
    BlurEnabled = false,
    BlurSize = 8,
    AmbientColor = false,
    LightingOverride = false,
    AmbientR = 120, AmbientG = 120, AmbientB = 140,
    OutdoorR = 140, OutdoorG = 140, OutdoorB = 160,
    ColorShiftEnabled = false,
    ColorShiftR = 0, ColorShiftG = 0, ColorShiftB = 0,
    TintEnabled = false,
    TintR = 255, TintG = 255, TintB = 255,
    ShadowsEnabled = true,
    GlobalShadows = true,
    EnvironmentDiffuse = 1,
    EnvironmentSpecular = 1,
    WallXray = false,
    WallTransparency = 0.55,
    WallLocalOnly = true, -- solo client
    MapColorEnabled = false,
    MapColorR = 80, MapColorG = 80, MapColorB = 100,
    RemoveTextures = false,
    GlassWindows = false,
    CharacterTransparency = 0,
    ApplyCharVisualsEnabled = false,
    ForceFieldVisual = false,
    ForceFieldColor = Color3.fromRGB(180, 0, 255),
    ForceFieldTransparency = 0.72,
    ForceFieldOutlineTransparency = 0.55,
    ForceFieldStyle = "Material", -- Material (solo skin) | Highlight | Both | Soft
    ForceFieldPulse = false,
    ForceFieldRemoveAcc = false,
    RemoveAccessories = false,

    ESPEnabled = false,
    ESPPerfMode = true, -- OPT: ESP ligero high FPS
    ESPMaxPlayers = 16, -- mas jugadores
    ESPUpdateInterval = 0.033, -- ~30 Hz fluido
    ESPRadar = false,
    ESPChams = false,

    -- LuaHook ESP
    ESP = false,
    ESPBox = true,
    ESPBoxStyle = "Full Box",
    ESPBoxBrackets = false,
    ESPBoxFill = false,
    ESPBoxThickness = 1,
    ESPBoxColorMode = "Solid",
    ESPBoxColor = Color3.fromRGB(255, 255, 255),
    ESPBoxGradA = Color3.fromRGB(255, 59, 78),
    ESPBoxGradB = Color3.fromRGB(255, 194, 75),
    ESPBoxFillColor = Color3.fromRGB(255, 59, 78),
    ESPBoxTransparency = 0,
    ESPBoxFillTransparency = 0.75,
    ESPBoxScale = 1,
    ESPName = true,
    ESPDistance = true,
    ESPHealth = true,
    ESPHealthBar = true,
    ESPSkeleton = false, -- OPT: skeleton mata FPS

    ESPTracers = false,
    ESPTeamCheck = false,
    ESPMaxDistance = 100000, -- 0 = infinite
    ESPDistanceScaling = true,
    ESPDistanceScalingRef = 50,
    ESPChams = false,
    ESPFlags = false, -- OPT


    ESPMaxDist = 1e9, -- infinito
    ESPInfiniteRange = true,
    ESPTeamCheck = false,
    ESPShowName = true,
    ESPShowDist = true,
    ESPShowHP = true,
    ESPShowBox = true,
    ESPBoxStyle = "Corner", -- Corner / Full / Both
    ESPSkeleton = false,
    ESPGlow = true,
    ESPSkeletonThickness = 1.5,
    ESPTracers = false,
    ESPTracerFrom = "Bottom",
    ESPColor = "Pink",
    ESPHealthBarColor = "Green",
    ESPUseTeamColor = false,
    ESPUseHPColor = false,
    ESPBoxThickness = 2,
    ESPCasingThickness = 2,
    ESPFilledBox = false,
    ESPFilledBoxTrans = 0.85,

    -- Bullet tracers (avanzados)
    BulletTracerEnabled = false, -- legacy OFF (Elisium tracers)

    BulletTracerColor = "Cyan",
    BulletTracerDuration = 0.6,
    BulletTracerThickness = 1.2,
    BulletTracerFromGun = true,
    BulletTracerStyle = "Neon",
    BulletTracerGlow = 8,
    BulletTracerFade = 0.25,
    BulletTracerEmission = 1,
    BulletTracerTextureLength = 1.2,
    BulletTracerTextureSpeed = 4,
    BulletTracerExpand = true,
    BulletTracerExpandSpeed = 12,
    BulletTracerExpandDamper = 0.85,
    BulletTracerCameraOffset = 0.15,

    -- Stretched resolution (CFrame matrix method)
    StretchedRes = false, -- legacy OFF (Elisium stretch)

    StretchedScale = 0.81, -- Y scale (0.5 = mas stretch, 1.0 = normal)

    ThirdPerson = false,
    ThirdPersonMode = "ThirdPerson",
    WeaponWireframe = false,
    WeaponWireframeColor = Color3.fromRGB(0, 255, 180),
    ThirdPersonDistance = 10,
    ThirdPersonHeight = 2.5,
    ThirdPersonSmooth = 0.2,
    Watermark = false,
    Crosshair = false,
    CrosshairSize = 25,
    CrosshairGap = 10,
    CrosshairThickness = 2,
    CrosshairOutline = true,
    CrosshairSpin = true,
    CrosshairSpinSpeed = 120,
    CrosshairText = "",
    CrosshairTextSize = 22,
    CrosshairRainbow = true,
    CrosshairOffsetX = 0,
    CrosshairOffsetY = 0,
    CrosshairTextOffsetY = 28,
    CurrentSky = "None",
    AutoRespawn = false,

    -- Spoofer
    SpoofEnabled = false,
    SpoofMyName = false,
    SpoofMyNameValue = "shako.win",
    SpoofMyUserValue = "",     -- @username falso en perfil/UI (si vacio usa MyName)
    SpoofOthers = false,
    SpoofOthersValue = "User",
    SpoofTargetName = "",
    SpoofTargetFake = "",
    SpoofHideReal = true,
    SpoofRandomOthers = false,
    SpoofProfileUI = true,     -- reescribir perfil / PlayerGui
    -- Device spoofer
    DeviceSpoofEnabled = false,
    DeviceSpoofType = "Mobile", -- PC, Mobile, Console, VR, Controller
    DeviceSpoofAttribute = true, -- setea atributos locales por si el juego los lee

    -- Skinchanger
    SkinchangerEnabled = false,
    SkinTargetUserId = 1,
    ShakoSyncEnabled = true, -- ver avatars de otros shako.win

    SkinTargetName = "",
    SkinOnRespawn = true,

    -- Keybinds (Enum.KeyCode name)
    KeybindStick = "V",
    KeybindRage = "R",
    KeybindSilent = "T",
    KeybindAimbot = "Y",
    KeybindFly = "F",
    KeybindNoclip = "G",
    KeybindAntiAim = "B",
    KeybindStrafe = "C",
    KeybindOrbit = "X",
    KeybindSpeed = "Z",
    KeybindVoidSpam = "H",
}


local config = getgenv().config

-- ==================== RIVALS MODULES + ITEMLIBRARY ====================
local RS = game:GetService("ReplicatedStorage")
local util, enum, FighterController, SpectateController
local modulesOK = false
local ItemBackup = {}
local gunExceptions = { RPG = true } -- sniper/bow/crossbow SI tienen rapid + no spread
local rageLastFire = 0
local deflecting = {}
local rageSlots = { Primary = 1, Secondary = 2, Melee = 3 }

function loadRivalsModules()
    local ok1, u = pcall(function() return require(RS.Modules.Utility) end)
    local ok2, e = pcall(function() return require(RS.Modules.EnumLibrary) end)
    local ok3, fc = pcall(function()
        return require(LocalPlayer.PlayerScripts.Controllers.FighterController)
    end)
    local ok4, sc = pcall(function()
        return require(LocalPlayer.PlayerScripts.Controllers:WaitForChild("SpectateController"))
    end)
    if ok1 and ok2 and ok3 then
        util, enum, FighterController = u, e, fc
        if ok4 then SpectateController = sc end
        modulesOK = true
        return true
    end
    modulesOK = false
    return false
end

task.spawn(function()
    for _ = 1, 40 do
        if loadRivalsModules() then
            pcall(function() Notify("Rivals", "FighterController OK") end)
            break
        end
        task.wait(0.4)
    end
end)

function getItems()
    local ok, mod = pcall(function() return require(RS.Modules.ItemLibrary) end)
    if ok and mod and mod.Items then return mod.Items end
    return nil
end

function ilSave(name, data, prop)
    if data[prop] == nil then return end
    ItemBackup[name] = ItemBackup[name] or {}
    if ItemBackup[name][prop] == nil then ItemBackup[name][prop] = data[prop] end
end

function ilSet(name, data, prop, val)
    if data[prop] == nil then return end
    ilSave(name, data, prop)
    data[prop] = val
end

function ilRestore(name, data)
    local b = ItemBackup[name]
    if not b then return end
    for prop, old in pairs(b) do
        if data[prop] ~= nil then data[prop] = old end
    end
end

function ApplyItemLibraryGuns(on)
    local Items = getItems()
    if not Items then return false end
    for name, data in pairs(Items) do
        if typeof(data) ~= "table" then continue end
        local lname = string.lower(tostring(name))
        if gunExceptions[name] or lname:find("rpg", 1, true) or lname:find("rocket", 1, true) then
            continue
        end
        if on then
            if config.NoRecoil or config.RapidFire then
                ilSet(name, data, "ShootSpread", 0)
                ilSet(name, data, "ShootRecoil", 0)
                ilSet(name, data, "Recoil", 0)
                ilSet(name, data, "Spread", 0)
                ilSet(name, data, "MinSpread", 0)
                ilSet(name, data, "MaxSpread", 0)
                ilSet(name, data, "AimSpread", 0)
                ilSet(name, data, "HipSpread", 0)
            end
            ilSet(name, data, "ShootAccuracy", 0)
            -- SOLO fire rate de disparo — NUNCA ReloadTime / Cooldown generico / Charge
            -- (Cooldown/ChargeTime a 0 rompe la recarga infinita)
            local delay = config.RapidFireDelay or 0.001
            ilSet(name, data, "ShootCooldown", delay)
            ilSet(name, data, "ShootBurstCooldown", delay)
            if data.FireRate ~= nil and type(data.FireRate) == "number" and data.FireRate > 0 then
                ilSet(name, data, "FireRate", math.max(delay, 0.001))
            end
            -- NO: Cooldown, ReloadTime, ChargeTime, BoltTime, PumpTime
        else
            ilRestore(name, data)
        end
    end
    return true
end

function ApplyItemLibraryMelee(on)
    local Items = getItems()
    if not Items then return false end
    for name, data in pairs(Items) do
        if typeof(data) ~= "table" then continue end
        if on then
            ilSet(name, data, "AttackCooldown", 0.001)
            ilSet(name, data, "SwingCooldown", 0.001)
            ilSet(name, data, "MeleeCooldown", 0.001)
            ilSet(name, data, "Cooldown", 0.001)
            ilSet(name, data, "RecoveryTime", 0.001)
            ilSet(name, data, "ResetTime", 0.001)
        else
            ilRestore(name, data)
        end
    end
    return true
end

task.spawn(function()
    while true do
        task.wait(5)
        if config.ItemLibraryKeepAlive then
            if config.RapidFire then pcall(ApplyItemLibraryGuns, true) end
            if config.FastMelee then pcall(ApplyItemLibraryMelee, true) end
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(4) -- OPT
        if not config.RagebotEnabled or not modulesOK then continue end
        -- NO forzar Melee/Fists: usar slot elegido (Primary por defecto)
        local slotName = config.RageWeaponSlot or "Primary"
        local pref = tostring(config.RagePreferredWeapon or "primary"):lower()
        if pref == "melee" or pref == "knife" then
            slotName = "Melee"
        elseif pref == "secondary" then
            slotName = "Secondary"
        elseif pref == "primary" then
            slotName = "Primary"
        end
        -- si el user eligió Primary/Secondary, NUNCA poner Fists
        if slotName == "Melee" and pref ~= "melee" and pref ~= "knife" then
            slotName = "Primary"
        end
        local slot = rageSlots[slotName] or 1
        local lf = FighterController and FighterController.LocalFighter
        if lf and config.RageWeaponSpecialize ~= false then
            pcall(function() lf:EquipItem(slot) end)
        end
    end
end)

Players.PlayerRemoving:Connect(function(p) deflecting[p] = nil end)

function updateDeflection()
    if not FighterController or not FighterController.Objects then return end
    for _, fighterObj in pairs(FighterController.Objects) do
        local player = fighterObj.Player
        if not player then continue end
        if not fighterObj.Entity or not fighterObj.Entity:IsAlive() or fighterObj:Get("IsSpectating") then
            deflecting[player] = false
            continue
        end
        local equipped = fighterObj.EquippedItem
        local isKatana = equipped and equipped.ViewModel and equipped.ViewModel.Name == "Katana"
        local isDeflecting = false
        if isKatana then
            isDeflecting = (equipped._attack_cooldown and equipped._attack_cooldown > tick()) or false
        end
        deflecting[player] = isDeflecting
    end
end

-- FFA: si casi todos comparten el mismo TeamID → Free For All
function isFFAMode()
    if config.RagebotForceFFA then return true end
    local teams, alive = {}, 0
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        if not plr.Character then continue end
        local hum = plr.Character:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health <= 0 then continue end
        alive = alive + 1
        local tid = getPlayerTeamId and getPlayerTeamId(plr) or nil
        -- nil / 0 / "0" / "FFA" cuentan como FFA-ish
        local key = (tid == nil or tid == "0" or tid == "nil" or tid == "FFA") and "__ffa" or tostring(tid)
        teams[key] = true
    end
    if alive <= 1 then return true end
    local unique = 0
    for _ in pairs(teams) do unique = unique + 1 end
    -- un solo "equipo" en el server = Free For All
    if unique <= 1 then return true end
    return false
end

function isEnemyRageRivals(player)
    if not player or player == LocalPlayer then return false end
    if not player.Character then return false end
    -- solo targets en pelea real (FighterController), no lobby/spectate
    if FighterController and FighterController.Objects then
        local foundAlive = false
        for _, fo in pairs(FighterController.Objects) do
            if fo and fo.Player == player then
                if fo.Entity and fo.Entity:IsAlive() and not fo:Get("IsSpectating") then
                    foundAlive = true
                end
                break
            end
        end
        if not foundAlive then return false end
    end
    -- Team check OFF o FFA forzado = atacar a todos
    if config.RagebotTeamCheck == false then return true end
    if config.RagebotForceFFA == true then return true end
    -- Team Check ON: solo enemigos (FFA real sigue permitiendo todos)
    if isFFAMode and isFFAMode() then return true end
    if IsSameTeam and IsSameTeam(player) then return false end
    if player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then return false end
    if player.Character and player.Character:FindFirstChild("TeammateLabel") then return false end
    return true
end

function hasKnifeViewModel(targetPlayer)
    if not targetPlayer then return false end
    local viewModels = workspace:FindFirstChild("ViewModels")
    if not viewModels then return false end
    local targetName = tar... (Restantes: 784 KB)
