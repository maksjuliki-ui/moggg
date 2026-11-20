--[[
    David & Tixan Code Sniper
    Redeems codes from game announcements, plus a few small utilities.
    Set hub.FAST = false further down if you want the slower, safer redeem.

    MOD: Anti Lag now also auto-enables FOV and Unwalk.
         hub.LAG_FOV below is set to 120 (Roblox caps Camera.FieldOfView at 120).
    MOD: hub.SKIP_WORDS (code, top, ...) are never added to the code buffer,
         so "codeTop" is no longer typed into the game's box.
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local RealTweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

local hub = {}

if _G.HamsterHubInstance and _G.HamsterHubInstance.cleanup then
    pcall(_G.HamsterHubInstance.cleanup)
end

_G.HamsterHubInstance = hub

hub.alive = true
hub.conns = {}
hub.previews = {}
hub.windows = {}
hub.popups = {}
hub.rows = {}
hub.FONT_FAMILY = "rbxasset://fonts/families/GothamSSm.json"

function hub.track(connection)
    hub.conns[#hub.conns + 1] = connection
    return connection
end

local SETTINGS_FILE = "HamsterHub_settings.json"
local settings = {
    animations = true,
    animWindow = true,
    animButtons = true,
    animPanels = true,
    speed = 1,
    scale = 0.75,
    pulse = true,
    transparency = 0.05,
    hue = 0.555,
    sat = 1,
    theme = "traced",
    bloxTheme = "traced",
    bloxHue = 0.78,
    bloxSat = 1,
    sound = true,
    volume = 0.5,
    autoUseCode = true,
    antiLag = false,
}

pcall(function()
    if isfile and isfile(SETTINGS_FILE) then
        local data = HttpService:JSONDecode(readfile(SETTINGS_FILE))

        for key, value in pairs(data) do
            if settings[key] ~= nil and type(value) == type(settings[key]) then
                settings[key] = value
            end
        end
    end
end)

settings.antiLag = true
-- Halloween look is the default, even when an older settings file exists.
settings.theme = "halloween"
settings.bloxTheme = "halloween"

for key, range in pairs({
    speed = {0.5, 2}, scale = {0.65, 0.85}, transparency = {0, 0.6}, volume = {0, 1},
    hue = {0, 1}, sat = {0, 1}, bloxHue = {0, 1}, bloxSat = {0, 1},
}) do
    settings[key] = math.clamp(tonumber(settings[key]) or range[1], range[1], range[2])
end

-- needs writefile, silently skipped if the executor doesn't have it
local function saveSettings()
    pcall(function()
        if writefile then
            writefile(SETTINGS_FILE, HttpService:JSONEncode(settings))
        end
    end)
end

-- small wrapper so tweens follow the animation toggle and speed slider
local TweenService = {}

function TweenService:Create(object, info, properties, category)
    local enabled = settings.animations and (category == nil or settings[category])

    if enabled then
        info = TweenInfo.new(
            info.Time / settings.speed,
            info.EasingStyle,
            info.EasingDirection,
            info.RepeatCount,
            info.Reverses,
            info.DelayTime
        )
    else
        info = TweenInfo.new(0)
    end

    return RealTweenService:Create(object, info, properties)
end

local THEME_ROLES = {
    bg          = {0.130, 0.98},
    bar         = {0.230, 0.97},
    button      = {0.255, 0.95},
    hover       = {0.340, 0.94},
    active      = {0.480, 0.97},
    input       = {0.205, 1.00},
    stroke      = {0.400, 0.97},
    text        = {0.760, 0.55},
    textMain    = {0.820, 0.50},
    textInput   = {0.830, 0.40},
    textSub     = {0.600, 0.74},
    placeholder = {0.420, 0.76},
}

local function rgb(r, g, b)
    return Color3.fromRGB(r, g, b)
end

local THEME_PRESETS = {
    traced = {
        bg = rgb(20, 14, 12),        bar = rgb(35, 23, 17),
        button = rgb(43, 30, 23),    hover = rgb(67, 43, 27),
        active = rgb(218, 125, 35),  input = rgb(29, 20, 16),
        stroke = rgb(105, 62, 28),   text = rgb(224, 205, 184),
        textMain = rgb(255, 225, 190), textInput = rgb(255, 239, 220),
        textSub = rgb(177, 145, 112), placeholder = rgb(132, 103, 78),
    },
    dark = {
        bg = rgb(22, 22, 27),        bar = rgb(32, 32, 40),
        button = rgb(46, 46, 56),    hover = rgb(68, 68, 82),
        active = rgb(70, 120, 210),  input = rgb(34, 34, 42),
        stroke = rgb(86, 86, 102),   text = rgb(232, 232, 240),
        textMain = rgb(244, 244, 250), textInput = rgb(255, 255, 255),
        textSub = rgb(164, 164, 178),  placeholder = rgb(120, 120, 136),
    },
    light = {
        bg = rgb(250, 250, 253),     bar = rgb(236, 237, 245),
        button = rgb(226, 228, 240), hover = rgb(206, 210, 228),
        active = rgb(150, 182, 240), input = rgb(243, 244, 250),
        stroke = rgb(196, 200, 216), text = rgb(38, 40, 52),
        textMain = rgb(28, 30, 42),  textInput = rgb(14, 16, 26),
        textSub = rgb(106, 110, 128), placeholder = rgb(148, 152, 170),
    },
    amoled = {
        bg = rgb(0, 0, 0),           bar = rgb(14, 14, 18),
        button = rgb(24, 24, 30),    hover = rgb(42, 42, 52),
        active = rgb(0, 150, 255),   input = rgb(10, 10, 14),
        stroke = rgb(52, 52, 64),    text = rgb(226, 226, 236),
        textMain = rgb(246, 246, 252), textInput = rgb(255, 255, 255),
        textSub = rgb(150, 150, 166),  placeholder = rgb(100, 100, 116),
    },
    newyear = {
        bg = rgb(14, 50, 38),        bar = rgb(156, 30, 40),
        button = rgb(34, 98, 68),    hover = rgb(196, 46, 58),
        active = rgb(28, 146, 78),   input = rgb(10, 38, 30),
        stroke = rgb(236, 196, 76),  text = rgb(255, 246, 228),
        textMain = rgb(255, 255, 255), textInput = rgb(255, 250, 236),
        textSub = rgb(244, 208, 130),  placeholder = rgb(150, 192, 172),
    },
    halloween = {
        bg = rgb(16, 10, 22),        bar = rgb(48, 18, 70),
        button = rgb(40, 22, 58),    hover = rgb(96, 44, 122),
        active = rgb(232, 102, 14),  input = rgb(26, 15, 36),
        stroke = rgb(255, 122, 20),  text = rgb(255, 172, 62),
        textMain = rgb(255, 238, 208), textInput = rgb(255, 216, 144),
        textSub = rgb(178, 132, 222),  placeholder = rgb(126, 98, 150),
    },
    ocean = {
        bg = rgb(9, 27, 46),         bar = rgb(14, 58, 94),
        button = rgb(20, 76, 118),   hover = rgb(32, 108, 158),
        active = rgb(0, 172, 204),   input = rgb(11, 35, 58),
        stroke = rgb(58, 170, 222),  text = rgb(190, 230, 255),
        textMain = rgb(232, 247, 255), textInput = rgb(255, 255, 255),
        textSub = rgb(120, 190, 232),  placeholder = rgb(88, 140, 172),
    },
    sunset = {
        bg = rgb(38, 17, 34),        bar = rgb(122, 38, 70),
        button = rgb(150, 52, 72),   hover = rgb(204, 82, 82),
        active = rgb(255, 122, 60),  input = rgb(50, 22, 40),
        stroke = rgb(255, 152, 92),  text = rgb(255, 216, 192),
        textMain = rgb(255, 241, 232), textInput = rgb(255, 247, 241),
        textSub = rgb(255, 172, 142),  placeholder = rgb(182, 122, 122),
    },
    mint = {
        bg = rgb(13, 37, 35),        bar = rgb(22, 78, 70),
        button = rgb(30, 100, 88),   hover = rgb(44, 136, 118),
        active = rgb(38, 202, 150),  input = rgb(15, 47, 43),
        stroke = rgb(90, 232, 190),  text = rgb(202, 255, 241),
        textMain = rgb(236, 255, 250), textInput = rgb(255, 255, 255),
        textSub = rgb(130, 212, 192),  placeholder = rgb(100, 162, 146),
    },
    sakura = {
        bg = rgb(255, 241, 247),     bar = rgb(250, 216, 231),
        button = rgb(246, 202, 221), hover = rgb(238, 172, 201),
        active = rgb(232, 122, 172), input = rgb(255, 250, 252),
        stroke = rgb(230, 152, 192), text = rgb(112, 42, 72),
        textMain = rgb(92, 32, 62),  textInput = rgb(62, 22, 42),
        textSub = rgb(162, 92, 122), placeholder = rgb(192, 142, 166),
    },
}

local BLOX_PRESETS = {
    traced = THEME_PRESETS.traced,
    purple = {
        bg = rgb(45, 20, 65),       bar = rgb(75, 35, 105),
        button = rgb(75, 35, 105),  hover = rgb(105, 55, 140),
        active = rgb(70, 180, 100), input = rgb(75, 35, 105),
        stroke = rgb(75, 35, 105),  text = rgb(210, 160, 255),
        textMain = rgb(200, 140, 255), textInput = rgb(240, 225, 250),
        textSub = rgb(190, 120, 255),  textSoft = rgb(205, 180, 225),
        placeholder = rgb(175, 145, 195),
    },
}

for key, preset in pairs(THEME_PRESETS) do
    BLOX_PRESETS[key] = preset
end

if settings.theme ~= "custom" and not THEME_PRESETS[settings.theme] then
    settings.theme = "custom"
end

if settings.bloxTheme ~= "custom" and not BLOX_PRESETS[settings.bloxTheme] then
    settings.bloxTheme = "purple"
end

-- a skin remembers every (object, property, role) so a theme change can recolor them all
function hub.newSkin(themeKey, hueKey, satKey, presets)
    local skin = {items = {}, themeKey = themeKey, hueKey = hueKey, satKey = satKey, presets = presets}

    function skin.color(role)
        if type(role) == "function" then
            role = role()
        end

        local name = settings[themeKey]
        local preset = name ~= "custom" and presets[name]

        if preset then
            return preset[role] or preset.text
        end

        local part = THEME_ROLES[role] or THEME_ROLES.text
        return Color3.fromHSV(settings[hueKey], part[1] * settings[satKey], part[2])
    end

    function skin.reg(object, property, role)
        skin.items[#skin.items + 1] = {object, property, role}
        object[property] = skin.color(role)
        return object
    end

    function skin.apply(instant)
        local info = TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

        for i = #skin.items, 1, -1 do
            local item = skin.items[i]
            local object = item[1]

            if not object.Parent then
                table.remove(skin.items, i)
            elseif instant then
                object[item[2]] = skin.color(item[3])
            else
                TweenService:Create(object, info, {[item[2]] = skin.color(item[3])}):Play()
            end
        end
    end

    return skin
end

hub.skinMain = hub.newSkin("theme", "hue", "sat", THEME_PRESETS)
hub.skinBlox = hub.newSkin("bloxTheme", "bloxHue", "bloxSat", BLOX_PRESETS)

hub.DECOR = {
    newyear = {
        {"🎄", -6, 128, 46, -6}, {"🎄", 262, 122, 44, 7}, {"🎄", 100, 146, 34, 3},
        {"🎄", 196, 140, 36, -4}, {"🎄", 18, 62, 24, -8, 0.6}, {"🎄", 258, 58, 22, 8, 0.6},
        {"🎁", 50, 168, 26, -8}, {"🎁", 150, 172, 28, 6}, {"🎁", 232, 170, 24, -5},
        {"🎁", 8, 100, 20, 10, 0.6}, {"🎁", 276, 96, 20, -10, 0.6},
    },
    halloween = {
        -- bottom row: big pumpkins, ghost and bats
        {"🎃", -6, 326, 50, -8, 0.3}, {"🎃", 236, 330, 46, 8, 0.3},
        {"👻", 112, 332, 34, 0, 0.4}, {"🦇", 58, 318, 26, -10, 0.5},
        {"🦇", 190, 314, 24, 12, 0.5}, {"🍬", 84, 356, 20, -14, 0.55},
        {"🍭", 170, 358, 20, 14, 0.55},
        -- sides
        {"🕷", 16, 292, 20, 0, 0.6}, {"💀", 244, 288, 20, 8, 0.6},
        {"🕸", 238, 66, 30, 0, 0.55}, {"🦇", 12, 150, 22, -12, 0.7},
        {"👻", 246, 196, 24, 10, 0.7}, {"🎃", 10, 214, 20, -10, 0.7},
        {"🕷", 250, 250, 18, 6, 0.7},
    },
}

function hub.makeDecor(parent, kind)
    local decor = Instance.new("Frame")
    decor.Name = kind .. "Decor"
    decor.Size = UDim2.fromScale(1, 1)
    decor.BackgroundTransparency = 1
    decor.BorderSizePixel = 0
    decor.ClipsDescendants = true
    decor.Active = false
    decor.Visible = false
    decor.ZIndex = 1
    decor.Parent = parent

    for _, item in ipairs(hub.DECOR[kind]) do
        local size = item[4]
        local label = Instance.new("TextLabel")
        label.Size = UDim2.fromOffset(size + 10, size + 10)
        label.Position = UDim2.fromOffset(item[2], item[3])
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.Text = item[1]
        label.TextSize = size
        label.TextTransparency = item[6] or 0.45
        label.Rotation = item[5] or 0
        label.ZIndex = 1
        label.Parent = decor
    end

    return function(active)
        decor.Visible = active and true or false
    end
end

local lower = string.lower
local find = string.find
local match = string.match
local concat = table.concat
local clear = table.clear
local spawnTask = task.defer

local scanning = false
local processing = false
local workerScheduled = false

local buffer = {}
local bufferHead = 1

local mode = 6
local delay = 0

local cachedGui, cachedFrame, cachedBox, cachedButton

function hub.need()
    return (mode == 5 or mode == 6) and 1 or mode
end

function hub.modeText()
    return "Mode: all words (stop manually)"
end

hub.autoTried = 0

local function clearCache()
    cachedGui = nil
    cachedFrame = nil
    cachedBox = nil
    cachedButton = nil
end

-- ignore anything that belongs to this script's own gui
local function isOwnGui(object)
    local parent = object

    for _ = 1, 8 do
        if not parent then
            return false
        end

        if parent.Name == "SourcesHubRedeemerGui" then
            return true
        end

        parent = parent.Parent
    end

    return false
end

local function isGuiButton(object)
    return object:IsA("TextButton") or object:IsA("ImageButton")
end

hub.lastScan = 0

local BAD_BUTTON_WORDS = {"close", "toggle", "cancel", "back", "exit"}

local function isBadButton(object)
    local name = lower(object.Name)
    local text = object:IsA("TextButton") and lower(object.Text or "") or ""

    if name == "x" or text == "x" or text == "✕" then
        return true
    end

    for _, word in ipairs(BAD_BUTTON_WORDS) do
        if find(name, word, 1, true) or find(text, word, 1, true) then
            return true
        end
    end

    return false
end

local function submitScore(object)
    if not isGuiButton(object) or isOwnGui(object) or isBadButton(object) then
        return 0
    end

    local name = lower(object.Name)
    local text = object:IsA("TextButton") and lower(object.Text or "") or ""
    local score = 0

    if find(name, "redeem", 1, true) or find(text, "redeem", 1, true) then score += 4 end
    if find(name, "submit", 1, true) or find(text, "submit", 1, true) then score += 4 end
    if find(name, "claim", 1, true) or find(text, "claim", 1, true) then score += 3 end
    if find(name, "confirm", 1, true) or find(text, "confirm", 1, true) then score += 2 end
    if name == "enter" or text == "enter" then score += 1 end

    return score
end

local function cacheValid()
    return (cachedGui
        and cachedGui.Parent
        and cachedFrame
        and cachedFrame.Parent
        and cachedBox
        and cachedBox.Parent) and true or false
end

-- looks for the textbox and the submit button in the game's Codes gui
local function findControls()
    local gui = PlayerGui:FindFirstChild("Codes")

    if not gui then
        clearCache()
        return
    end

    if cacheValid() and cachedGui == gui and (cachedButton or os.clock() - hub.lastScan < 1) then
        return cachedBox, cachedButton
    end

    hub.lastScan = os.clock()

    if gui:IsA("ScreenGui") then
        gui.Enabled = true
    end

    local frame = gui:FindFirstChild("Codes") or gui

    if frame:IsA("GuiObject") then
        frame.Visible = true
    end

    local box
    local button
    local bestScore = 0
    local descendants = frame:GetDescendants()

    for i = 1, #descendants do
        local object = descendants[i]

        if not isOwnGui(object) then
            if object:IsA("TextBox") then
                if not box or (not box.Visible and object.Visible) then
                    box = object
                end
            else
                local score = submitScore(object)

                if score > bestScore then
                    bestScore = score
                    button = object
                end
            end
        end
    end

    if box and not button then
        for i = 1, #descendants do
            local object = descendants[i]

            if isGuiButton(object) and not isOwnGui(object) and not isBadButton(object) then
                button = object
                break
            end
        end
    end

    cachedGui = gui
    cachedFrame = frame
    cachedBox = box
    cachedButton = button

    return box, button
end

-- firesignal first, if that fails call the connections by hand
function hub.fireSignal(signal, ...)
    if typeof(firesignal) == "function" and pcall(firesignal, signal, ...) then
        return true
    end

    if typeof(getconnections) == "function" then
        local ok, connections = pcall(getconnections, signal)

        if ok and connections then
            local fired = false
            local args = table.pack(...)

            for _, connection in ipairs(connections) do
                local done = pcall(function()
                    if connection.Function then
                        task.spawn(connection.Function, table.unpack(args, 1, args.n))
                    else
                        connection:Fire(table.unpack(args, 1, args.n))
                    end
                end)

                fired = fired or done
            end

            if fired then
                return true
            end
        end
    end

    return false
end

local function activateButton(button)
    if not button or not button.Parent then
        return false
    end

    if hub.fireSignal(button.Activated) then
        return true
    end

    if hub.fireSignal(button.MouseButton1Click) then
        return true
    end

    return pcall(function()
        button.Activated:Fire()
    end)
end

hub.lastCode = nil
hub.lastCodeAt = 0
hub.sentAt = 0
hub.REDEEM_COOLDOWN = 0.15

-- if the box still has the last code in it, only append the new word
function hub.setBoxText(box, code, piece)
    return pcall(function()
        if piece and hub.lastCode and box.Text == hub.lastCode then
            box.Text = box.Text .. piece
        else
            box.Text = code
        end

        if box.Text ~= code then
            box.Text = code
        end
    end)
end

-- types the code into the game's box and presses submit
local function redeem(code, piece)
    if not code or code == "" then
        return false
    end

    local now = os.clock()

    if hub.lastCode == code and now - hub.lastCodeAt < hub.REDEEM_COOLDOWN then
        return false
    end

    local box, button

    if cacheValid() and cachedButton then
        box, button = cachedBox, cachedButton
    else
        box, button = findControls()
    end

    if not box then
        return false
    end

    if not hub.setBoxText(box, code, piece) then
        clearCache()
        box, button = findControls()

        if not box or not hub.setBoxText(box, code) then
            return false
        end
    end

    hub.lastCode = code
    hub.lastCodeAt = now
    hub.sentAt = os.clock()

    -- press submit first, nothing else is allowed to delay it
    local submitted = false

    if button and button.Parent then
        submitted = activateButton(button)
    end

    if not submitted then
        submitted = hub.fireSignal(box.FocusLost, true)
    end

    -- show the code being entered in our own CODE field (after submit)
    if hub.codeInput and hub.codeInput.Parent then
        hub.codeInput.Text = code

        if hub.animateCode then
            task.defer(hub.animateCode)
        end
    end

    return submitted
end

local function validCode(code)
    return code
        and #code > 0
        and match(code, "^[A-Za-z0-9]+$") ~= nil
end

local function bufferSize()
    return #buffer - bufferHead + 1
end

local function resetBuffer()
    clear(buffer)
    bufferHead = 1
    hub.autoTried = 0
    hub.lastCode = nil
end

function hub.setPreview(text)
    for _, box in ipairs(hub.previews) do
        if box.Parent then
            box.Text = text
        end
    end
end

local function updatePreview()
    local size = bufferSize()
    local text = ""

    if size > 0 then
        local count = size
        local values = table.create(count)

        for i = 1, count do
            values[i] = buffer[bufferHead + i - 1]
        end

        text = concat(values)
    end

    hub.setPreview(text)
end

local function takeCode()
    if bufferSize() < mode then
        return nil
    end

    local values = table.create(mode)

    for i = 1, mode do
        values[i] = buffer[bufferHead]
        buffer[bufferHead] = nil
        bufferHead += 1
    end

    if bufferHead > #buffer then
        resetBuffer()
    end

    return concat(values)
end

hub.waiting = false
hub.resultKind = nil
hub.waitThread = nil
hub.waitToken = 0
hub.latency = 0.2
hub.resultTimeout = 0.15
hub.MIN_TIMEOUT = 0.08
hub.MAX_TIMEOUT = 0.4
hub.FAST = true
hub.FAST_WINDOW = 1.5

-- Words that are never part of a code; they are dropped before the buffer.
-- Lowercase only (the comparison is case-insensitive).
hub.SKIP_WORDS = {
    code = true, codes = true, use = true, the = true, is = true,
    redeem = true, top = true, codetop = true,
}
hub.LOWERCASE = false   -- true = always type the code in lowercase

hub.OK_WORDS = {
    "success", "redeemed", "claimed", "congrat", "you got", "you received",
    "you've received", "reward", "already",
}

hub.FAIL_WORDS = {
    "invalid", "wrong", "not valid", "incorrect", "doesn't exist", "does not exist",
    "not exist", "expired", "not found", "unknown", "try again", "no such",
    "no longer", "not active", "doesn't match",
}

function hub.wake()
    local thread = hub.waitThread

    if thread then
        hub.waitThread = nil
        task.spawn(thread)
    end
end

-- reads the game's reply and works out if the code got accepted
function hub.noteResult(text)
    local fastActive = hub.FAST and mode == 6 and scanning and not hub.waiting
        and os.clock() - hub.lastCodeAt < hub.FAST_WINDOW

    if not (hub.waiting or fastActive) or not text then
        return false
    end

    local low = lower(tostring(text))

    if #low > 300 then
        return false
    end

    low = string.gsub(low, "<[^>]+>", "")

    local kind

    for _, word in ipairs(hub.OK_WORDS) do
        if find(low, word, 1, true) then
            kind = "ok"
            break
        end
    end

    if not kind then
        for _, word in ipairs(hub.FAIL_WORDS) do
            if find(low, word, 1, true) then
                kind = "fail"
                break
            end
        end
    end

    if not kind then
        return false
    end

    if fastActive then
        if kind == "ok" then
            hub.acceptFast()
        end

        return true
    end

    if not hub.resultKind then
        hub.resultKind = kind

        if hub.sentAt > 0 then
            local took = math.clamp(os.clock() - hub.sentAt, 0.02, 1)

            hub.latency = hub.latency * 0.7 + took * 0.3
            hub.resultTimeout = math.clamp(hub.latency * 2.5 + 0.1, hub.MIN_TIMEOUT, hub.MAX_TIMEOUT)
            hub.sentAt = 0
        end

        hub.wake()
    end

    return true
end

function hub.tryRedeem(code, piece)
    hub.resultKind = nil
    hub.waiting = true
    hub.waitToken += 1

    local token = hub.waitToken

    if not redeem(code, piece) then
        hub.waiting = false
        return false
    end

    if not hub.resultKind and hub.alive then
        local thread = coroutine.running()
        hub.waitThread = thread

        task.delay(hub.resultTimeout, function()
            if hub.waitThread == thread and hub.waitToken == token then
                hub.waitThread = nil
                task.spawn(thread)
            end
        end)

        coroutine.yield()
    end

    hub.waitThread = nil
    hub.waiting = false
    hub.sentAt = 0

    return hub.resultKind == "ok"
end

-- code accepted, wipe the buffer so the next code starts clean
function hub.acceptFast()
    clear(buffer)
    bufferHead = 1
    hub.autoTried = 0
    hub.lastCode = nil
    hub.lastCodeAt = 0
    hub.setPreview("")
end

-- fast mode: send every new word right away, don't wait for the reply
function hub.processFast()
    while hub.alive and scanning and mode == 6 do
        local n = hub.autoTried + 1

        if #buffer < n then
            break
        end

        local code = concat(buffer, "", 1, n)
        local piece = n > 1 and buffer[n] or nil

        hub.autoTried = n
        redeem(code, piece)
        hub.setPreview(code)

        if delay > 0 then
            task.wait(delay)
        end
    end
end

function hub.processAuto()
    if bufferHead > 1 then
        local rest = {}

        for i = bufferHead, #buffer do
            rest[#rest + 1] = buffer[i]
        end

        clear(buffer)

        for i = 1, #rest do
            buffer[i] = rest[i]
        end

        bufferHead = 1
    end

    while hub.alive and scanning and (mode == 5 or mode == 6) do
        local n = hub.autoTried + 1

        if mode == 5 and n > 5 then
            table.remove(buffer, 1)
            hub.autoTried = 0
            hub.lastCode = nil
            updatePreview()
        elseif #buffer < n then
            break
        else
            local code = concat(buffer, "", 1, n)

            hub.setPreview(code)

            local piece = (n > 1 and hub.autoTried == n - 1) and buffer[n] or nil
            local accepted = hub.tryRedeem(code, piece)

            if scanning and (mode == 5 or mode == 6) and #buffer >= n then
                if accepted then
                    for _ = 1, n do
                        table.remove(buffer, 1)
                    end

                    hub.autoTried = 0
                    hub.lastCode = nil
                else
                    hub.autoTried = n
                end
            end

            updatePreview()

            if delay > 0 then
                task.wait(delay)
            end
        end
    end
end

local function processBuffer()
    if processing or not scanning or not hub.alive then
        return
    end

    processing = true

    if mode == 5 or mode == 6 then
        pcall((hub.FAST and mode == 6) and hub.processFast or hub.processAuto)
        processing = false
        updatePreview()
        return
    end

    pcall(function()
        while scanning and bufferSize() >= mode do
            local code = takeCode()

            if not code then
                return
            end

            updatePreview()
            redeem(code)

            if delay > 0 then
                task.wait(delay)
            end
        end
    end)

    processing = false
    updatePreview()

    if scanning and mode ~= 5 and mode ~= 6 and bufferSize() >= mode then
        if not workerScheduled then
            workerScheduled = true

            spawnTask(function()
                workerScheduled = false
                processBuffer()
            end)
        end
    end
end

local function scheduleProcess()
    if not scanning or processing or workerScheduled then
        return
    end

    workerScheduled = true

    task.spawn(function()
        workerScheduled = false
        processBuffer()
    end)
end

function hub.toggleScan()
    if not hub.alive then
        return
    end

    scanning = not scanning

    resetBuffer()
    hub.setPreview("")

    if scanning then
        task.spawn(function()
            pcall(findControls)
        end)
    end

    if not scanning then
        hub.waiting = false
    end

    hub.refreshRows()
end

hub.lastAuto = 0

-- game says "use code" -> turn scanning on automatically
function hub.tryAutoCode(text)
    if not hub.alive or not settings.autoUseCode or not text then
        return false
    end

    text = tostring(text)

    if #text > 400 then
        return false
    end

    local low = lower(text)

    if not find(low, "code", 1, true) then
        return false
    end

    low = string.gsub(low, "<[^>]+>", "")

    if not (
        find(low, "use%s+code")
        or find(low, "use%s+the%s+code")
        or find(low, "code%s+is")
    ) then
        return false
    end

    local now = os.clock()

    if now - hub.lastAuto < 30 then
        return true
    end

    hub.lastAuto = now

    if not scanning then
        spawnTask(hub.toggleScan)
    end

    return true
end

local function receiveMessage(text, skipAnnouncementDisplay)
    if type(text) ~= "string" then
        return
    end

    local plainText = lower(tostring(text))
    plainText = string.gsub(plainText, "<[^>]->", "")
    plainText = string.gsub(plainText, "&nbsp;", " ")
    plainText = string.gsub(plainText, "%s+", " ")
    plainText = string.gsub(plainText, "^%s*(.-)%s*$", "%1")

    local isBaseStatus =
        (find(plainText, "base", 1, true) and (
            find(plainText, "already locked", 1, true)
            or find(plainText, "locked your base", 1, true)
            or find(plainText, "lock your base", 1, true)
            or find(plainText, "base is locked", 1, true)
        ))

    local isSoundPath =
        find(plainText, "sounds.sfx.", 1, true)
        or find(plainText, "sound.sfx.", 1, true)

    if isBaseStatus or isSoundPath then
        return
    end

    if not skipAnnouncementDisplay and hub.showAnnouncement then
        task.defer(function()
            pcall(hub.showAnnouncement, text)
        end)
    end

    if hub.noteResult(text) then
        return
    end

    if hub.tryAutoCode(text) then
        return
    end

    if not scanning then
        return
    end

    local code = match(text, "^%s*(.-)%s*$")

    if not validCode(code) then
        return
    end

    -- never type filler words such as "code" or "Top" into the game's box
    if hub.SKIP_WORDS[lower(code)] then
        return
    end

    if hub.LOWERCASE then
        code = lower(code)
    end

    buffer[#buffer + 1] = code

    -- submit first, preview is refreshed afterwards
    if bufferSize() >= hub.need() then
        scheduleProcess()
    end

    updatePreview()
end

local function findNotifyRemote()
    if _G.PhiNotifyRemote then
        return _G.PhiNotifyRemote
    end

    local packages = ReplicatedStorage:FindFirstChild("Packages")
    local net = packages and packages:FindFirstChild("Net")

    if not net then
        return nil
    end

    if not getconnections or not debug or not debug.getinfo then
        return nil
    end

    local descendants = net:GetDescendants()

    for i = 1, #descendants do
        local object = descendants[i]

        if object:IsA("RemoteEvent") then
            local success, connections = pcall(getconnections, object.OnClientEvent)

            if success and connections then
                for j = 1, #connections do
                    local fn

                    pcall(function()
                        fn = connections[j].Function
                    end)

                    if type(fn) == "function" then
                        local info

                        pcall(function()
                            info = debug.getinfo(fn)
                        end)

                        if info then
                            local source = tostring(info.short_src or info.source or "")

                            if find(source, "NotificationController", 1, true) then
                                return object
                            end
                        end
                    end
                end
            end
        end
    end

    return nil
end

-- hook the game's notification remote, retry a few times because the game can load late
function hub.startListener()
    task.spawn(function()
        for _ = 1, 20 do
            if not hub.alive then
                return
            end

            local remote = findNotifyRemote()

            if remote then
                hub.track(remote.OnClientEvent:Connect(function(...)
                    local foundTexts = {}
                    local seenTables = {}
                    local seenStrings = {}

                    local function collectText(value, depth)
                        if depth > 6 then
                            return
                        end

                        if type(value) == "string" then
                            local clean = match(value, "^%s*(.-)%s*$")
                            if clean ~= "" and not seenStrings[clean] then
                                seenStrings[clean] = true
                                foundTexts[#foundTexts + 1] = clean
                            end
                        elseif type(value) == "table" and not seenTables[value] then
                            seenTables[value] = true
                            for _, nested in pairs(value) do
                                collectText(nested, depth + 1)
                            end
                        end
                    end

                    for i = 1, select("#", ...) do
                        collectText(select(i, ...), 0)
                    end

                    if #foundTexts > 0 then
                        -- codes first, the announcement picture is drawn after submit
                        for _, text in ipairs(foundTexts) do
                            receiveMessage(text, true)
                        end

                        if hub.showAnnouncement then
                            local joined = table.concat(foundTexts, "\n")

                            task.defer(function()
                                pcall(hub.showAnnouncement, joined)
                            end)
                        end
                    end
                end))

                return
            end

            task.wait(3)
        end
    end)
end

hub.antiRagdoll = false
hub.autoBuy = false
hub.buyInterval = 0.25
hub.buyToken = 0
hub.ragdollConn = nil

function hub.refreshRows()
    for _, refresh in ipairs(hub.rows) do
        pcall(refresh)
    end
end

hub.RAGDOLL_ATTRS = {"Ragdoll", "Ragdolled", "IsRagdoll", "RagdollState", "Stunned"}

hub.BAD_STATES = {
    [Enum.HumanoidStateType.Ragdoll] = true,
    [Enum.HumanoidStateType.FallingDown] = true,
    [Enum.HumanoidStateType.Physics] = true,
}

function hub.charParts()
    local char = Player.Character

    if not char then
        return
    end

    return char, char:FindFirstChildOfClass("Humanoid"), char:FindFirstChild("HumanoidRootPart")
end

function hub.fixRagdoll()
    local char, humanoid, root = hub.charParts()

    if not (char and humanoid and root) or humanoid.Health <= 0 then
        return
    end

    for _, name in ipairs(hub.RAGDOLL_ATTRS) do
        if char:GetAttribute(name) == true then
            char:SetAttribute(name, false)
        end
    end

    for _, object in ipairs(char:GetDescendants()) do
        if object:IsA("Motor6D") then
            if not object.Enabled then
                object.Enabled = true
            end
        elseif object:IsA("BallSocketConstraint") then
            object:Destroy()
        end
    end

    if humanoid.PlatformStand then
        humanoid.PlatformStand = false
    end

    if hub.BAD_STATES[humanoid:GetState()] then
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end

    if root.AssemblyAngularVelocity.Magnitude > 50 then
        root.AssemblyAngularVelocity = Vector3.zero
    end
end

function hub.setRagdollStates(humanoid, state)
    pcall(function()
        humanoid:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, state)
        humanoid:SetStateEnabled(Enum.HumanoidStateType.FallingDown, state)
    end)
end

-- checked 10 times a second instead of every frame
function hub.setAntiRagdoll(state)
    hub.antiRagdoll = state

    local _, humanoid = hub.charParts()

    if humanoid then
        hub.setRagdollStates(humanoid, not state)
    end

    if hub.ragdollConn then
        hub.ragdollConn:Disconnect()
        hub.ragdollConn = nil
    end

    if state then
        local acc = 0

        hub.ragdollConn = RunService.Heartbeat:Connect(function(dt)
            acc += dt

            if acc < 0.1 then
                return
            end

            acc = 0
            pcall(hub.fixRagdoll)
        end)
    end
end

hub.track(Player.CharacterAdded:Connect(function(char)
    if not hub.alive or not hub.antiRagdoll then
        return
    end

    local humanoid = char:WaitForChild("Humanoid", 5)

    if humanoid then
        hub.setRagdollStates(humanoid, false)
    end
end))

hub.prompts = {}
hub.promptCool = setmetatable({}, {__mode = "k"})
hub.promptConns = {}
hub.promptCacheOn = false

hub.promptHold = setmetatable({}, {__mode = "k"})
hub.BUY_COOLDOWN = 0.12
hub.buyConns = {}

function hub.isBuyPrompt(prompt)
    local action = lower(prompt.ActionText or "")

    return find(action, "purchase", 1, true) ~= nil or find(action, "buy", 1, true) ~= nil
end

-- remove the hold time so the purchase doesn't have to be held
function hub.zeroHold(prompt)
    if prompt.HoldDuration > 0 and hub.isBuyPrompt(prompt) then
        hub.promptHold[prompt] = prompt.HoldDuration
        prompt.HoldDuration = 0
    end
end

function hub.restoreHolds()
    for prompt, value in pairs(hub.promptHold) do
        pcall(function()
            prompt.HoldDuration = value
        end)
    end

    clear(hub.promptHold)
end

function hub.trackPrompt(object)
    if object:IsA("ProximityPrompt") then
        hub.prompts[object] = true

        if hub.autoBuy then
            pcall(hub.zeroHold, object)
        end
    end
end

function hub.startPromptCache()
    if hub.promptCacheOn then
        return
    end

    hub.promptCacheOn = true
    hub.promptConns[1] = workspace.DescendantAdded:Connect(hub.trackPrompt)
    hub.promptConns[2] = workspace.DescendantRemoving:Connect(function(object)
        hub.prompts[object] = nil
    end)

    task.spawn(function()
        local descendants = workspace:GetDescendants()

        for i = 1, #descendants do
            if not hub.promptCacheOn then
                return
            end

            hub.trackPrompt(descendants[i])

            if i % 500 == 0 then
                task.wait()
            end
        end
    end)
end

function hub.stopPromptCache()
    hub.promptCacheOn = false

    for i, connection in pairs(hub.promptConns) do
        connection:Disconnect()
        hub.promptConns[i] = nil
    end

    clear(hub.prompts)
    hub.restoreHolds()
end

function hub.firePrompt(prompt)
    if typeof(fireproximityprompt) == "function" then
        return pcall(fireproximityprompt, prompt)
    end

    return pcall(function()
        prompt:InputHoldBegin()
        task.wait(math.max(prompt.HoldDuration, 0.05))
        prompt:InputHoldEnd()
    end)
end

-- fires one prompt if it is a buy prompt and not on cooldown
function hub.buyPrompt(prompt)
    local now = os.clock()

    if not prompt.Enabled or now - (hub.promptCool[prompt] or 0) < hub.BUY_COOLDOWN then
        return false
    end

    if not hub.isBuyPrompt(prompt) then
        return false
    end

    hub.promptCool[prompt] = now
    pcall(hub.zeroHold, prompt)
    task.spawn(hub.firePrompt, prompt)

    return true
end

-- checks every cached prompt that is close enough, runs every frame
function hub.tryBuy()
    local _, _, root = hub.charParts()

    if not root then
        return
    end

    local position = root.Position

    for prompt in pairs(hub.prompts) do
        if prompt.Parent and prompt.Enabled then
            local part = prompt.Parent

            if part:IsA("Attachment") then
                part = part.Parent
            end

            if part and part:IsA("BasePart")
                and (part.Position - position).Magnitude <= prompt.MaxActivationDistance + 3 then

                hub.buyPrompt(prompt)
            end
        end
    end
end

function hub.setAutoBuy(state)
    hub.autoBuy = state
    hub.buyToken += 1

    for i, connection in pairs(hub.buyConns) do
        connection:Disconnect()
        hub.buyConns[i] = nil
    end

    if state then
        hub.startPromptCache()

        -- instant: buy the moment a prompt is shown to the player
        pcall(function()
            hub.buyConns[1] = game:GetService("ProximityPromptService").PromptShown:Connect(function(prompt)
                if hub.alive and hub.autoBuy then
                    hub.buyPrompt(prompt)
                end
            end)
        end)

        -- backup: every frame, covers prompts that were already visible
        hub.buyConns[2] = RunService.Heartbeat:Connect(function()
            if hub.alive and hub.autoBuy then
                pcall(hub.tryBuy)
            end
        end)

        -- prompts found before auto buy was turned on
        task.spawn(function()
            local n = 0

            for prompt in pairs(hub.prompts) do
                pcall(hub.zeroHold, prompt)
                n += 1

                if n % 300 == 0 then
                    task.wait()
                end
            end
        end)
    else
        hub.stopPromptCache()
    end
end

function hub.stopFeatures()
    hub.setAntiRagdoll(false)
    hub.setAutoBuy(false)
end

-- ========================= Anti Lag =========================

hub.lagOn = false
hub.lagToken = 0
hub.lagSaved = setmetatable({}, {__mode = "k"})
hub.lagConns = {}
hub.lagLighting = nil
hub.lagQuality = nil
hub.lagTerrain = nil
hub.lagStoppedTracks = {}
hub.lagAnimatorSeen = setmetatable({}, {__mode = "k"})

function hub.lagStopBrainrotAnimations(object)
    if not object or not object:IsA("Animator") or hub.lagAnimatorSeen[object] then
        return
    end

    local model = object:FindFirstAncestorOfClass("Model")
    if not model or Players:GetPlayerFromCharacter(model) then
        return
    end

    hub.lagAnimatorSeen[object] = true

    local function stopTrack(track)
        pcall(function()
            if track.IsPlaying then
                hub.lagStoppedTracks[#hub.lagStoppedTracks + 1] = {
                    track = track,
                    time = track.TimePosition,
                    speed = track.Speed,
                    weight = track.WeightCurrent,
                }
                track:Stop(0)
            end
        end)
    end

    for _, track in ipairs(object:GetPlayingAnimationTracks()) do
        stopTrack(track)
    end

    local connection = object.AnimationPlayed:Connect(stopTrack)
    hub.lagConns[#hub.lagConns + 1] = connection
end

hub.LAG_CLASSES = {
    ParticleEmitter = true, Trail = true, Beam = true, Smoke = true, Fire = true,
    Sparkles = true, BloomEffect = true, BlurEffect = true, DepthOfFieldEffect = true,
    SunRaysEffect = true, ColorCorrectionEffect = true,
    PointLight = true, SpotLight = true, SurfaceLight = true,
    Atmosphere = false,
}

function hub.lagObject(object)
    local saved = hub.lagSaved

    if object:IsA("Decal") or object:IsA("Texture") then
        local data = saved[object] or {}
        if data.Transparency == nil then
            data.Transparency = object.Transparency
        end
        saved[object] = data
        object.Transparency = 1
    end

    if object:IsA("SurfaceAppearance") then
        local data = saved[object] or {}
        if data.ColorMap == nil then
            data.ColorMap = object.ColorMap
            data.NormalMap = object.NormalMap
            data.RoughnessMap = object.RoughnessMap
            data.MetalnessMap = object.MetalnessMap
        end
        saved[object] = data
        pcall(function()
            object.ColorMap = ""
            object.NormalMap = ""
            object.RoughnessMap = ""
            object.MetalnessMap = ""
        end)
    end

    if object:IsA("MeshPart") then
        local data = saved[object] or {}
        if data.TextureID == nil then
            data.TextureID = object.TextureID
        end
        saved[object] = data
        pcall(function() object.TextureID = "" end)
    elseif object:IsA("SpecialMesh") then
        local data = saved[object] or {}
        if data.TextureId == nil then
            data.TextureId = object.TextureId
        end
        saved[object] = data
        pcall(function() object.TextureId = "" end)
    end

    if hub.LAG_CLASSES[object.ClassName] then
        local data = saved[object] or {}
        if data.Enabled == nil then
            data.Enabled = object.Enabled
        end
        saved[object] = data
        object.Enabled = false
    end

    if object:IsA("BasePart") then
        local data = saved[object] or {}
        if data.Material == nil then
            data.Material = object.Material
            data.CastShadow = object.CastShadow
            data.Reflectance = object.Reflectance
            data.Color = object.Color
        end
        saved[object] = data

        object.Material = Enum.Material.SmoothPlastic
        object.CastShadow = false
        object.Reflectance = 0

        local size = object.Size
        if size.X >= 18 and size.Z >= 18 and size.Y <= 8 then
            if data.FloorColor == nil then
                data.FloorColor = object.Color
            end
            object.Color = Color3.fromRGB(35, 220, 35)
        end
    end
end

-- ===== Anti Lag extras: FOV + Unwalk (auto-enabled together with Anti Lag) =====
hub.LAG_FOV = 120           -- Roblox caps Camera.FieldOfView at 120
hub.lagFovOld = nil
hub.lagAnimate = {}

function hub.lagStartFov()
    local cam = workspace.CurrentCamera
    if not cam then
        return
    end

    hub.lagFovOld = hub.lagFovOld or cam.FieldOfView
    cam.FieldOfView = hub.LAG_FOV

    -- if the game resets the FOV, put it back
    hub.lagConns[#hub.lagConns + 1] = cam:GetPropertyChangedSignal("FieldOfView"):Connect(function()
        if hub.lagOn and cam.FieldOfView ~= hub.LAG_FOV then
            cam.FieldOfView = hub.LAG_FOV
        end
    end)
end

function hub.lagStopFov()
    local cam = workspace.CurrentCamera

    if cam and hub.lagFovOld then
        cam.FieldOfView = hub.lagFovOld
    end

    hub.lagFovOld = nil
end

function hub.lagUnwalkChar(char)
    task.spawn(function()
        local humanoid = char:WaitForChild("Humanoid", 5)
        local animate = char:WaitForChild("Animate", 5)

        if not (hub.lagOn and humanoid) then
            return
        end

        if animate and animate:IsA("LocalScript") then
            animate.Disabled = true
            hub.lagAnimate[#hub.lagAnimate + 1] = animate
        end

        local animator = humanoid:FindFirstChildOfClass("Animator")

        if animator then
            for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                track:Stop(0)
            end

            hub.lagConns[#hub.lagConns + 1] = animator.AnimationPlayed:Connect(function(track)
                track:Stop(0)
            end)
        end
    end)
end

function hub.lagStartUnwalk()
    hub.lagConns[#hub.lagConns + 1] = Player.CharacterAdded:Connect(hub.lagUnwalkChar)

    if Player.Character then
        hub.lagUnwalkChar(Player.Character)
    end
end

function hub.lagStopUnwalk()
    for _, animate in ipairs(hub.lagAnimate) do
        pcall(function()
            animate.Disabled = false
        end)
    end

    table.clear(hub.lagAnimate)
end

-- strips effects and textures, everything is restored when you turn it off
function hub.setAntiLag(state)
    state = state and true or false

    if state == hub.lagOn then
        return
    end

    hub.lagOn = state
    hub.lagToken += 1
    local token = hub.lagToken

    for i, connection in pairs(hub.lagConns) do
        connection:Disconnect()
        hub.lagConns[i] = nil
    end

    if state then
        hub.lagLighting = {
            GlobalShadows = Lighting.GlobalShadows,
            FogEnd = Lighting.FogEnd,
            EnvironmentDiffuseScale = Lighting.EnvironmentDiffuseScale,
            EnvironmentSpecularScale = Lighting.EnvironmentSpecularScale,
        }
        hub.lagQuality = nil
        hub.lagTerrain = nil

        pcall(function()
            Lighting.GlobalShadows = false
            Lighting.FogEnd = 9e9
        end)

        pcall(function()
            local renderSettings = getfenv(0).settings().Rendering
            hub.lagQuality = renderSettings.QualityLevel
            renderSettings.QualityLevel = Enum.QualityLevel.Level01
        end)

        pcall(function()
            Lighting.GlobalShadows = false
            Lighting.EnvironmentDiffuseScale = 0
            Lighting.EnvironmentSpecularScale = 0
        end)

        pcall(function()
            local terrain = workspace.Terrain
            hub.lagTerrain = {
                Decoration = terrain.Decoration,
                WaterWaveSize = terrain.WaterWaveSize,
                WaterWaveSpeed = terrain.WaterWaveSpeed,
                WaterReflectance = terrain.WaterReflectance,
                WaterTransparency = terrain.WaterTransparency,
                GrassColor = terrain:GetMaterialColor(Enum.Material.Grass),
            }
            terrain:SetMaterialColor(Enum.Material.Grass, Color3.fromRGB(35, 220, 35))
            terrain.Decoration = false
            terrain.WaterWaveSize = 0
            terrain.WaterWaveSpeed = 0
            terrain.WaterReflectance = 0
            terrain.WaterTransparency = 1
        end)

        -- FOV + Unwalk turn on together with Anti Lag
        pcall(hub.lagStartFov)
        pcall(hub.lagStartUnwalk)

        hub.lagConns[#hub.lagConns + 1] = workspace.DescendantAdded:Connect(function(object)
            pcall(hub.lagObject, object)
        end)

        hub.lagConns[#hub.lagConns + 1] = Lighting.DescendantAdded:Connect(function(object)
            pcall(hub.lagObject, object)
        end)

        hub.lagConns[#hub.lagConns + 1] = workspace.DescendantAdded:Connect(function(object)
            if object:IsA("Animator") then
                pcall(hub.lagStopBrainrotAnimations, object)
            end
        end)

        for _, object in ipairs(workspace:GetDescendants()) do
            if object:IsA("Animator") then
                pcall(hub.lagStopBrainrotAnimations, object)
            end
        end

        task.spawn(function()
            for _, root in ipairs({workspace, Lighting}) do
                local descendants = root:GetDescendants()

                for i = 1, #descendants do
                    if not hub.alive or hub.lagToken ~= token then
                        return
                    end

                    pcall(hub.lagObject, descendants[i])

                    if i % 400 == 0 then
                        task.wait()
                    end
                end
            end
        end)
    else
        -- FOV + Unwalk turn off together with Anti Lag
        pcall(hub.lagStopFov)
        pcall(hub.lagStopUnwalk)

        for _, savedTrack in ipairs(hub.lagStoppedTracks) do
            pcall(function()
                local track = savedTrack.track
                track:Play(0, savedTrack.weight or 1, savedTrack.speed or 1)
                track.TimePosition = savedTrack.time or 0
            end)
        end
        table.clear(hub.lagStoppedTracks)
        hub.lagAnimatorSeen = setmetatable({}, {__mode = "k"})

        for object, data in pairs(hub.lagSaved) do
            pcall(function()
                if data.Enabled ~= nil then object.Enabled = data.Enabled end
                if data.Transparency ~= nil then object.Transparency = data.Transparency end
                if data.ColorMap ~= nil then
                    object.ColorMap = data.ColorMap
                    object.NormalMap = data.NormalMap
                    object.RoughnessMap = data.RoughnessMap
                    object.MetalnessMap = data.MetalnessMap
                end
                if data.TextureID ~= nil then object.TextureID = data.TextureID end
                if data.TextureId ~= nil then object.TextureId = data.TextureId end
                if data.Material ~= nil then
                    object.Material = data.Material
                    object.CastShadow = data.CastShadow
                    object.Reflectance = data.Reflectance
                    object.Color = data.Color
                end
            end)
        end

        clear(hub.lagSaved)

        if hub.lagLighting then
            pcall(function()
                Lighting.GlobalShadows = hub.lagLighting.GlobalShadows
                Lighting.FogEnd = hub.lagLighting.FogEnd
                Lighting.EnvironmentDiffuseScale = hub.lagLighting.EnvironmentDiffuseScale
                Lighting.EnvironmentSpecularScale = hub.lagLighting.EnvironmentSpecularScale
            end)
            hub.lagLighting = nil
        end

        if hub.lagQuality ~= nil then
            pcall(function()
                getfenv(0).settings().Rendering.QualityLevel = hub.lagQuality
            end)
            hub.lagQuality = nil
        end

        if hub.lagTerrain then
            pcall(function()
                local terrain = workspace.Terrain
                terrain.Decoration = hub.lagTerrain.Decoration
                terrain.WaterWaveSize = hub.lagTerrain.WaterWaveSize
                terrain.WaterWaveSpeed = hub.lagTerrain.WaterWaveSpeed
                terrain.WaterReflectance = hub.lagTerrain.WaterReflectance
                terrain.WaterTransparency = hub.lagTerrain.WaterTransparency
                terrain:SetMaterialColor(Enum.Material.Grass, hub.lagTerrain.GrassColor)
            end)
            hub.lagTerrain = nil
        end
    end
end

-- full shutdown (close button or running the script again)
function hub.cleanup()
    hub.alive = false
    scanning = false
    processing = false
    workerScheduled = false
    hub.waiting = false

    pcall(hub.stopFeatures)
    pcall(hub.setAntiLag, false)

    for _, connection in ipairs(hub.conns) do
        pcall(function()
            connection:Disconnect()
        end)
    end

    clear(hub.conns)
    resetBuffer()
    clearCache()

    if hub.gui then
        pcall(function()
            hub.gui:Destroy()
        end)

        hub.gui = nil
    end

    if _G.HamsterHubInstance == hub then
        _G.HamsterHubInstance = nil
    end
end

hub.fontFace = nil

pcall(function()
    hub.fontFace = Font.new(hub.FONT_FAMILY, Enum.FontWeight.Medium)
end)

hub.TEXT_CLASSES = {TextLabel = true, TextButton = true, TextBox = true}

function hub.make(class, props, parent)
    local object = Instance.new(class)

    for key, value in pairs(props or {}) do
        object[key] = value
    end

    if hub.TEXT_CLASSES[class] then
        if hub.fontFace then
            pcall(function()
                object.FontFace = hub.fontFace
            end)
        else
            object.Font = Enum.Font.GothamMedium
        end
    end

    object.Parent = parent
    return object
end

function hub.applyFont(root)
    if not hub.fontFace then
        return
    end

    for _, object in ipairs(root:GetDescendants()) do
        if hub.TEXT_CLASSES[object.ClassName] then
            pcall(function()
                object.FontFace = hub.fontFace
            end)
        end
    end
end

function hub.click()
    if not settings.sound then
        return
    end

    pcall(function()
        local sound = Instance.new("Sound")
        sound.SoundId = "rbxassetid://6895079853"
        sound.Volume = settings.volume
        sound.Parent = hub.gui or PlayerGui
        sound:Play()
        game:GetService("Debris"):AddItem(sound, 2)
    end)
end

function hub.buildGui()
    local gui = Instance.new("ScreenGui")
    gui.Name = "SourcesHubRedeemerGui"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 999
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    local parent = PlayerGui

    pcall(function()
        if gethui then
            parent = gethui()
        elseif syn and syn.protect_gui then
            syn.protect_gui(gui)
            parent = game:GetService("CoreGui")
        end
    end)

    gui.Parent = parent
    hub.gui = gui
    return gui
end

hub.activePopup = nil

function hub.closePopup(popup)
    if not popup then
        return
    end

    if hub.activePopup == popup then
        hub.activePopup = nil
    end

    popup.frame.Visible = false
end

function hub.openPopup(popup)
    local current = hub.activePopup

    if current and current ~= popup then
        hub.closePopup(current)
    end

    hub.activePopup = popup

    local frame = popup.frame
    frame.Visible = true

    frame.Position = UDim2.new(0.5, 0, 0.5, 14)
    TweenService:Create(frame, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        Position = UDim2.fromScale(0.5, 0.5),
    }, "animWindow"):Play()
end

function hub.togglePopup(popup)
    if hub.activePopup == popup then
        hub.closePopup(popup)
    else
        hub.openPopup(popup)
    end
end

-- drag a window by its title bar
function hub.makeDraggable(frame, handle)
    local dragging, startInput, startMouse, startPos

    handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true
            startInput = input
            startMouse = input.Position
            startPos = frame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    hub.track(UserInputService.InputChanged:Connect(function(input)
        if not dragging then
            return
        end

        if input.UserInputType == Enum.UserInputType.MouseMovement
            or (input.UserInputType == Enum.UserInputType.Touch and input == startInput) then

            local scale = settings.scale
            local delta = (input.Position - startMouse) / scale

            frame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end))
end

function hub.makeWindow(skin, title, size, position, onClose)
    local frame = hub.make("Frame", {
        Size = UDim2.fromOffset(size.X, size.Y),
        Position = position,
        BorderSizePixel = 0,
        ClipsDescendants = true,
        Active = true,
        ZIndex = 2,
    }, hub.gui)

    skin.reg(frame, "BackgroundColor3", "bg")
    frame.BackgroundTransparency = settings.transparency

    hub.make("UICorner", {CornerRadius = UDim.new(0, 12)}, frame)

    local stroke = hub.make("UIStroke", {Thickness = 1.5}, frame)
    skin.reg(stroke, "Color", "stroke")

    hub.make("UIScale", {Scale = settings.scale}, frame)

    local bar = hub.make("Frame", {
        Size = UDim2.new(1, 0, 0, 30),
        BorderSizePixel = 0,
        ZIndex = 5,
    }, frame)
    skin.reg(bar, "BackgroundColor3", "bar")
    hub.make("UICorner", {CornerRadius = UDim.new(0, 12)}, bar)

    local label = hub.make("TextLabel", {
        Size = UDim2.new(1, -70, 1, 0),
        Position = UDim2.fromOffset(10, 0),
        BackgroundTransparency = 1,
        Text = title,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, bar)
    skin.reg(label, "TextColor3", "textMain")

    local closeButton = hub.make("TextButton", {
        Size = UDim2.fromOffset(26, 22),
        Position = UDim2.new(1, -30, 0, 4),
        BorderSizePixel = 0,
        Text = "X",
        TextSize = 13,
        AutoButtonColor = false,
        ZIndex = 6,
    }, bar)
    skin.reg(closeButton, "BackgroundColor3", "button")
    skin.reg(closeButton, "TextColor3", "text")
    hub.make("UICorner", {CornerRadius = UDim.new(0, 5)}, closeButton)

    closeButton.MouseButton1Click:Connect(function()
        hub.click()
        onClose()
    end)

    local body = hub.make("ScrollingFrame", {
        Size = UDim2.new(1, -12, 1, -38),
        Position = UDim2.fromOffset(6, 34),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 3,
        CanvasSize = UDim2.new(),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ZIndex = 3,
    }, frame)

    hub.make("UIListLayout", {
        Padding = UDim.new(0, 6),
        SortOrder = Enum.SortOrder.LayoutOrder,
    }, body)

    hub.makeDraggable(frame, bar)

    hub.windows[#hub.windows + 1] = frame

    return {frame = frame, bar = bar, body = body, label = label}
end

function hub.addButton(skin, parent, text, callback)
    local button = hub.make("TextButton", {
        Size = UDim2.new(1, -4, 0, 30),
        BorderSizePixel = 0,
        Text = text,
        TextSize = 13,
        AutoButtonColor = false,
        ZIndex = 4,
    }, parent)

    skin.reg(button, "BackgroundColor3", "button")
    skin.reg(button, "TextColor3", "text")
    hub.make("UICorner", {CornerRadius = UDim.new(0, 8)}, button)

    button.MouseEnter:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.12), {BackgroundColor3 = skin.color("hover")}, "animButtons"):Play()
    end)

    button.MouseLeave:Connect(function()
        if button:GetAttribute("Toggled") then
            return
        end

        TweenService:Create(button, TweenInfo.new(0.12), {BackgroundColor3 = skin.color("button")}, "animButtons"):Play()
    end)

    button.MouseButton1Click:Connect(function()
        hub.click()
        callback(button)
    end)

    return button
end

function hub.addToggle(skin, parent, label, getter, setter)
    local button = hub.addButton(skin, parent, "", function()
        setter(not getter())
        hub.refreshRows()
    end)

    local function refresh()
        local on = getter()

        button.Text = label .. ": " .. (on and "ON" or "OFF")
        button:SetAttribute("Toggled", on)
        button.BackgroundColor3 = skin.color(on and "active" or "button")
    end

    hub.rows[#hub.rows + 1] = refresh
    refresh()

    return button
end

function hub.addSlider(skin, parent, label, minValue, maxValue, getter, setter)
    local holder = hub.make("Frame", {
        Size = UDim2.new(1, -4, 0, 40),
        BackgroundTransparency = 1,
        ZIndex = 4,
    }, parent)

    local text = hub.make("TextLabel", {
        Size = UDim2.new(1, 0, 0, 18),
        BackgroundTransparency = 1,
        TextSize = 12,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 4,
    }, holder)
    skin.reg(text, "TextColor3", "textSub")

    local track = hub.make("Frame", {
        Size = UDim2.new(1, 0, 0, 8),
        Position = UDim2.fromOffset(0, 24),
        BorderSizePixel = 0,
        ZIndex = 4,
    }, holder)
    skin.reg(track, "BackgroundColor3", "input")
    hub.make("UICorner", {CornerRadius = UDim.new(1, 0)}, track)

    local fill = hub.make("Frame", {
        Size = UDim2.fromScale(0, 1),
        BorderSizePixel = 0,
        ZIndex = 5,
    }, track)
    skin.reg(fill, "BackgroundColor3", "active")
    hub.make("UICorner", {CornerRadius = UDim.new(1, 0)}, fill)

    local function refresh()
        local alpha = (getter() - minValue) / (maxValue - minValue)

        fill.Size = UDim2.fromScale(math.clamp(alpha, 0, 1), 1)
        text.Text = string.format("%s: %.2f", label, getter())
    end

    local dragging = false

    local function update(input)
        local alpha = math.clamp((input.Position.X - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)

        setter(minValue + (maxValue - minValue) * alpha)
        refresh()
    end

    track.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true
            update(input)

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                    saveSettings()
                end
            end)
        end
    end)

    hub.track(UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch) then

            update(input)
        end
    end))

    hub.rows[#hub.rows + 1] = refresh
    refresh()

    return holder
end

function hub.makePopup(skin, title, size)
    local popup = {}

    local window = hub.makeWindow(skin, title, size, UDim2.fromScale(0.5, 0.5), function()
        hub.closePopup(popup)
    end)

    window.frame.AnchorPoint = Vector2.new(0.5, 0.5)
    window.frame.ZIndex = 50
    window.frame.Visible = false

    popup.frame = window.frame
    popup.body = window.body

    hub.popups[#hub.popups + 1] = popup

    return popup
end

function hub.applyThemes()
    hub.skinMain.apply(false)
    hub.skinBlox.apply(false)

    for _, setDecor in ipairs(hub.decorMain or {}) do
        setDecor(false)
    end

    local kind = settings.theme
    local halloween = kind == "halloween"

    if hub.decorByTheme and hub.decorByTheme[kind] then
        hub.decorByTheme[kind](true)
    end

    if hub.mainGradient then
        hub.mainGradient.Enabled = halloween
    end

    if hub.mainTitle then
        hub.mainTitle.Text = halloween and "🎃 David & Tixan Code Sniper" or "David & Tixan Code Sniper"
    end

    task.delay(0.3, hub.refreshRows)
    saveSettings()
end

function hub.buildUI()
    local gui = hub.buildGui()
    local skin = hub.skinMain
    local blox = hub.skinBlox

    local main
    local reopenButton

    local function showMainUI()
        if main and main.frame and main.frame.Parent then
            main.frame.Visible = true
        end
        if reopenButton and reopenButton.Parent then
            reopenButton.Visible = false
        end
    end

    local function hideMainUI()
        if main and main.frame and main.frame.Parent then
            main.frame.Visible = false
        end
        for _, popup in ipairs(hub.popups or {}) do
            if popup.frame and popup.frame.Parent then
                popup.frame.Visible = false
            end
        end
        hub.activePopup = nil
        if reopenButton and reopenButton.Parent then
            reopenButton.Visible = true
        end
    end

    main = hub.makeWindow(skin, "David & Tixan Code Sniper", Vector2.new(276, 390), UDim2.new(0, 30, 0.5, -195), function()
        hideMainUI()
    end)

    hub.mainTitle = main.label

    -- soft purple fade on the window background (halloween only)
    hub.mainGradient = hub.make("UIGradient", {
        Rotation = 90,
        Color = ColorSequence.new(Color3.fromRGB(255, 255, 255), Color3.fromRGB(176, 140, 210)),
        Enabled = false,
    }, main.frame)

    reopenButton = hub.make("TextButton", {
        Name = "OpenHubButton",
        Size = UDim2.fromOffset(120, 34),
        Position = UDim2.new(0, 14, 1, -52),
        BackgroundColor3 = Color3.fromRGB(20, 24, 42),
        BorderSizePixel = 0,
        Text = "Open menu",
        TextColor3 = Color3.fromRGB(245, 248, 255),
        TextSize = 13,
        AutoButtonColor = false,
        Visible = false,
        ZIndex = 100,
    }, gui)
    hub.make("UICorner", {CornerRadius = UDim.new(0, 6)}, reopenButton)
    local reopenStroke = hub.make("UIStroke", {
        Color = Color3.fromRGB(90, 90, 110),
        Thickness = 1,
        Transparency = 0,
    }, reopenButton)

    reopenButton.MouseEnter:Connect(function()
        reopenStroke.Color = Color3.fromRGB(150, 150, 180)
    end)
    reopenButton.MouseLeave:Connect(function()
        reopenStroke.Color = Color3.fromRGB(90, 90, 110)
    end)
    reopenButton.MouseButton1Click:Connect(function()
        hub.click()
        showMainUI()
    end)

    local redeemer = hub.makeWindow(blox, "Redeemer", Vector2.new(276, 390), UDim2.new(0, 400, 0.5, -215), function()
    end)
    redeemer.frame.Visible = false
    redeemer.body.Parent = main.frame
    redeemer.body.Position = UDim2.new(0, 6, 0, 68)
    redeemer.body.Size = UDim2.new(1, -12, 1, -74)
    redeemer.body.ZIndex = 3
    main.body.Position = UDim2.new(0, 6, 0, 68)
    main.body.Size = UDim2.new(1, -12, 1, -74)

    local tabBar = hub.make("Frame", {
        Size = UDim2.new(1, -12, 0, 28), Position = UDim2.fromOffset(6, 34),
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 8,
    }, main.frame)
    local tabLayout = hub.make("UIListLayout", {
        FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 6),
    }, tabBar)
    local function makeTab(label, width)
        local b = hub.make("TextButton", {
            Size = UDim2.new(0.5, -3, 1, 0), BorderSizePixel = 0, Text = label,
            TextSize = 12, AutoButtonColor = false, ZIndex = 9,
        }, tabBar)
        skin.reg(b, "BackgroundColor3", "button")
        skin.reg(b, "TextColor3", "text")
        hub.make("UICorner", {CornerRadius = UDim.new(0, 8)}, b)
        return b
    end
    local mainTab = makeTab("Main", 1)
    local redeemTab = makeTab("Redeemer", 1)
    local function showPage(which)
        if not main or not redeemer then return end
        main.body.Visible = which == "main"
        redeemer.body.Visible = which == "redeemer"
        mainTab.BackgroundTransparency = which == "main" and 0 or 0.35
        redeemTab.BackgroundTransparency = which == "redeemer" and 0 or 0.35
    end
    mainTab.MouseButton1Click:Connect(function() showPage("main") end)
    redeemTab.MouseButton1Click:Connect(function() showPage("redeemer") end)
    showPage("main")

    hub.decorByTheme = {}
    hub.decorMain = {}

    for kind in pairs(hub.DECOR) do
        local setter = hub.makeDecor(main.frame, kind)

        hub.decorByTheme[kind] = setter
        hub.decorMain[#hub.decorMain + 1] = setter
    end

    -- ---- Redeemer ----
    local codeLabel = hub.make("TextLabel", {
        Size = UDim2.new(1, -4, 0, 15),
        BackgroundTransparency = 1,
        Text = "CODE",
        TextSize = 10,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 4,
    }, redeemer.body)
    blox.reg(codeLabel, "TextColor3", "textSub")

    local codeInput = hub.make("TextBox", {
        Size = UDim2.new(1, -4, 0, 30),
        BorderSizePixel = 0,
        Text = "",
        PlaceholderText = "Type or paste a code...",
        TextSize = 13,
        ClearTextOnFocus = false,
        ZIndex = 4,
    }, redeemer.body)
    blox.reg(codeInput, "BackgroundColor3", "input")
    blox.reg(codeInput, "TextColor3", "textInput")
    blox.reg(codeInput, "PlaceholderColor3", "placeholder")
    hub.make("UICorner", {CornerRadius = UDim.new(0, 8)}, codeInput)

    -- remember the field so redeem() can show the code being entered
    hub.codeInput = codeInput

    -- entry animation: glowing border, quick pop and a background flash
    local codeStroke = hub.make("UIStroke", {
        Thickness = 2,
        Transparency = 1,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, codeInput)
    blox.reg(codeStroke, "Color", "active")

    local codeScale = hub.make("UIScale", {Scale = 1}, codeInput)

    hub.animateCode = function()
        if not codeInput.Parent then
            return
        end

        codeStroke.Transparency = 0
        TweenService:Create(codeStroke, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Transparency = 1,
        }):Play()

        codeScale.Scale = 1.05
        TweenService:Create(codeScale, TweenInfo.new(0.28, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Scale = 1,
        }):Play()

        codeInput.BackgroundColor3 = blox.color("hover")
        TweenService:Create(codeInput, TweenInfo.new(0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = blox.color("input"),
        }):Play()
    end

    local actionRow = hub.make("Frame", {
        Size = UDim2.new(1, -4, 0, 28),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ZIndex = 4,
    }, redeemer.body)
    hub.make("UIListLayout", {
        FillDirection = Enum.FillDirection.Horizontal,
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 5),
    }, actionRow)

    local function smallAction(label, callback)
        local button = hub.make("TextButton", {
            Size = UDim2.new(1/3, -4, 1, 0),
            BorderSizePixel = 0,
            Text = label,
            TextSize = 10,
            AutoButtonColor = false,
            ZIndex = 5,
        }, actionRow)
        blox.reg(button, "BackgroundColor3", "button")
        blox.reg(button, "TextColor3", "text")
        hub.make("UICorner", {CornerRadius = UDim.new(0, 7)}, button)
        button.MouseButton1Click:Connect(function()
            hub.click()
            callback()
        end)
        return button
    end

    smallAction("COPY", function()
        local value = tostring(codeInput.Text or "")
        if value == "" then
            local latestPreview = hub.previews[1]
            value = latestPreview and tostring(latestPreview.Text or "") or ""
        end
        if value ~= "" then
            pcall(function()
                if setclipboard then
                    setclipboard(value)
                end
            end)
        end
    end)

    smallAction("CLEAR", function()
        codeInput.Text = ""
        resetBuffer()
        updatePreview()
    end)

    smallAction("REDEEM", function()
        local value = tostring(codeInput.Text or "")
        if value ~= "" then
            redeem(value)
        end
    end)

    codeInput.FocusLost:Connect(function(enterPressed)
        if enterPressed and codeInput.Text ~= "" then
            redeem(codeInput.Text)
        end
    end)

    hub.addToggle(blox, redeemer.body, "Listen", function() return scanning end, function()
        hub.toggleScan()
    end)

    local preview = hub.make("TextBox", {
        Size = UDim2.new(1, -4, 0, 30),
        BorderSizePixel = 0,
        Text = "",
        PlaceholderText = "Latest code",
        TextSize = 13,
        TextEditable = false,
        ClearTextOnFocus = false,
        ZIndex = 4,
    }, redeemer.body)
    blox.reg(preview, "BackgroundColor3", "input")
    blox.reg(preview, "TextColor3", "textInput")
    blox.reg(preview, "PlaceholderColor3", "placeholder")
    hub.make("UICorner", {CornerRadius = UDim.new(0, 8)}, preview)
    hub.previews[#hub.previews + 1] = preview
    preview.Visible = false

    local settingsPopup = hub.makePopup(skin, "Settings", Vector2.new(260, 320))
    local themesPopup = hub.makePopup(skin, "Themes", Vector2.new(260, 340))
    local announcementsPopup = hub.makePopup(skin, "Announcement test", Vector2.new(280, 190))

    local announcementBox = hub.make("TextBox", {
        Size = UDim2.new(1, -4, 0, 62),
        BorderSizePixel = 0,
        Text = "",
        PlaceholderText = "Enter announcement text...",
        TextSize = 13,
        TextWrapped = true,
        TextYAlignment = Enum.TextYAlignment.Top,
        TextXAlignment = Enum.TextXAlignment.Left,
        ClearTextOnFocus = false,
        MultiLine = false,
        ZIndex = 4,
    }, announcementsPopup.body)
    skin.reg(announcementBox, "BackgroundColor3", "input")
    skin.reg(announcementBox, "TextColor3", "textInput")
    skin.reg(announcementBox, "PlaceholderColor3", "placeholder")
    hub.make("UICorner", {CornerRadius = UDim.new(0, 5)}, announcementBox)
    hub.make("UIPadding", {
        PaddingTop = UDim.new(0, 7), PaddingBottom = UDim.new(0, 7),
        PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8),
    }, announcementBox)

    local announcementPreview
    local announcementToken = 0

    local function showTestAnnouncement(message)
        message = tostring(message or "")
        if message == "" then
            message = "use code"
        end

        announcementToken += 1
        local thisToken = announcementToken

        if announcementPreview then
            announcementPreview:Destroy()
            announcementPreview = nil
        end

        local holder = hub.make("Frame", {
            Name = "TestAnnouncementPreview",
            AnchorPoint = Vector2.new(0.5, 0),
            Position = UDim2.new(0.5, 0, 0, 58),
            Size = UDim2.new(0.9, 0, 0, 76),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            ZIndex = 200,
        }, gui)
        hub.make("UIListLayout", {
            FillDirection = Enum.FillDirection.Horizontal,
            HorizontalAlignment = Enum.HorizontalAlignment.Center,
            VerticalAlignment = Enum.VerticalAlignment.Center,
            SortOrder = Enum.SortOrder.LayoutOrder,
            Padding = UDim.new(0, 10),
        }, holder)

        local textLabel = hub.make("TextLabel", {
            Name = "AnnouncementText",
            LayoutOrder = 2,
            Size = UDim2.new(1, 0, 1, 0),
            BackgroundTransparency = 1,
            Text = message,
            TextColor3 = Color3.fromRGB(255, 255, 255),
            TextSize = 32,
            TextWrapped = true,
            TextScaled = true,
            Font = Enum.Font.GothamBlack,
            TextXAlignment = Enum.TextXAlignment.Center,
            TextYAlignment = Enum.TextYAlignment.Center,
            ZIndex = 200,
        }, holder)
        hub.make("UITextSizeConstraint", {
            MinTextSize = 16,
            MaxTextSize = 32,
        }, textLabel)
        hub.make("UIStroke", {
            Color = Color3.fromRGB(0, 0, 0),
            Thickness = 2.5,
            LineJoinMode = Enum.LineJoinMode.Round,
        }, textLabel)

        announcementPreview = holder
        task.delay(5, function()
            if hub.alive and announcementToken == thisToken and announcementPreview == holder then
                holder:Destroy()
                announcementPreview = nil
            end
        end)
    end

    hub.showAnnouncement = showTestAnnouncement

    local function submitTestAnnouncement()
        local testText = tostring(announcementBox.Text or "")
        if testText == "" then
            return
        end

        showTestAnnouncement(testText)

        local normalized = string.lower(string.gsub(testText, "<[^>]+>", ""))
        local isCodePrompt = find(normalized, "use%s+code")
            or find(normalized, "use%s+the%s+code")
            or find(normalized, "code%s+is")

        if isCodePrompt then
            if not scanning then
                hub.toggleScan()
            end
            return
        end

        receiveMessage(testText, true)
    end

    hub.addButton(skin, announcementsPopup.body, "Preview announcement", submitTestAnnouncement)

    announcementBox.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            submitTestAnnouncement()
        end
    end)

    -- Settings
    hub.addToggle(skin, settingsPopup.body, "Animations", function() return settings.animations end, function(v)
        settings.animations = v
        saveSettings()
    end)

    hub.addToggle(skin, settingsPopup.body, "Sound", function() return settings.sound end, function(v)
        settings.sound = v
        saveSettings()
    end)

    hub.addToggle(skin, settingsPopup.body, "Auto use code", function() return settings.autoUseCode end, function(v)
        settings.autoUseCode = v
        saveSettings()
    end)

    hub.addSlider(skin, settingsPopup.body, "Scale", 0.65, 0.85, function() return settings.scale end, function(v)
        settings.scale = v

        for _, frame in ipairs(hub.windows) do
            local uiScale = frame:FindFirstChildOfClass("UIScale")

            if uiScale then
                uiScale.Scale = v
            end
        end
    end)

    hub.addSlider(skin, settingsPopup.body, "Transparency", 0, 0.6, function() return settings.transparency end, function(v)
        settings.transparency = v

        for _, frame in ipairs(hub.windows) do
            frame.BackgroundTransparency = v
        end
    end)

    hub.addSlider(skin, settingsPopup.body, "Animation speed", 0.5, 2, function() return settings.speed end, function(v)
        settings.speed = v
    end)

    hub.addSlider(skin, settingsPopup.body, "Volume", 0, 1, function() return settings.volume end, function(v)
        settings.volume = v
    end)

    -- Themes
    hub.themeTarget = "main"

    local targetButton = hub.addButton(skin, themesPopup.body, "Theme target: Main", function(button)
        hub.themeTarget = hub.themeTarget == "main" and "blox" or "main"
        button.Text = "Theme target: " .. (hub.themeTarget == "main" and "Main" or "Redeemer")
    end)

    local function setTheme(name)
        if hub.themeTarget == "main" then
            settings.theme = name
        else
            settings.bloxTheme = name
        end

        hub.applyThemes()
    end

    hub.addButton(skin, themesPopup.body, "custom", function()
        setTheme("custom")
    end)

    local names = {}

    for key in pairs(BLOX_PRESETS) do
        names[#names + 1] = key
    end

    table.sort(names)

    for _, name in ipairs(names) do
        hub.addButton(skin, themesPopup.body, name, function()
            setTheme(name)
        end)
    end

    hub.addSlider(skin, themesPopup.body, "Hue", 0, 1, function()
        return hub.themeTarget == "main" and settings.hue or settings.bloxHue
    end, function(v)
        if hub.themeTarget == "main" then
            settings.hue = v
            settings.theme = "custom"
        else
            settings.bloxHue = v
            settings.bloxTheme = "custom"
        end

        hub.skinMain.apply(true)
        hub.skinBlox.apply(true)
    end)

    hub.addSlider(skin, themesPopup.body, "Saturation", 0, 1, function()
        return hub.themeTarget == "main" and settings.sat or settings.bloxSat
    end, function(v)
        if hub.themeTarget == "main" then
            settings.sat = v
            settings.theme = "custom"
        else
            settings.bloxSat = v
            settings.bloxTheme = "custom"
        end

        hub.skinMain.apply(true)
        hub.skinBlox.apply(true)
    end)

    hub.addToggle(skin, main.body, "Anti ragdoll", function() return hub.antiRagdoll end, function(v)
        hub.setAntiRagdoll(v)
    end)

    hub.addToggle(skin, main.body, "Auto buy", function() return hub.autoBuy end, function(v)
        hub.setAutoBuy(v)
    end)

    hub.addToggle(skin, main.body, "Anti lag", function() return hub.lagOn end, function(v)
        settings.antiLag = v
        hub.setAntiLag(v)
        saveSettings()
    end)

    hub.addButton(skin, main.body, "Open Redeemer", function()
        showPage("redeemer")
    end)

    hub.addButton(skin, main.body, "Themes", function()
        hub.togglePopup(themesPopup)
    end)

    hub.addButton(skin, main.body, "Settings", function()
        hub.togglePopup(settingsPopup)
    end)

    hub.addButton(skin, main.body, "Announcement test", function()
        hub.togglePopup(announcementsPopup)
    end)

    -- glowing pulse on the main window border (halloween only)
    task.spawn(function()
        local stroke = main.frame:FindFirstChildOfClass("UIStroke")

        while hub.alive and stroke and stroke.Parent do
            if settings.pulse and settings.animations and settings.theme == "halloween" then
                TweenService:Create(stroke, TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
                    Transparency = 0.6,
                }):Play()
                task.wait(1.4 / settings.speed)

                TweenService:Create(stroke, TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
                    Transparency = 0,
                }):Play()
                task.wait(1.4 / settings.speed)
            else
                stroke.Transparency = 0
                task.wait(1)
            end
        end
    end)

    hub.applyFont(gui)
    hub.applyThemes()
    hub.refreshRows()

    if settings.antiLag then
        hub.setAntiLag(true)
        hub.refreshRows()
    end
end

hub.buildUI()
hub.startListener()
