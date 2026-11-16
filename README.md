-- Vision GUI: переключатели + Visuals (Accessories) + фоны + Copy Job ID
repeat task.wait() until game:IsLoaded()

local Players       = game:GetService("Players")
local RunService    = game:GetService("RunService")
local UIS           = game:GetService("UserInputService")
local TweenSvc      = game:GetService("TweenService")
local Lighting      = game:GetService("Lighting")
local InsertService = game:GetService("InsertService")
local lp  = Players.LocalPlayer
local cam = workspace.CurrentCamera

local ACCENT = Color3.fromRGB(190, 120, 255)
local BG     = Color3.fromRGB(14,  15,  22)
local PANEL  = Color3.fromRGB(24,  25,  36)

-- удалить старый GUI
for _, p in ipairs({ game:GetService("CoreGui"), lp:WaitForChild("PlayerGui") }) do
	local old = p:FindFirstChild("VisionGUI")
	if old then old:Destroy() end
end

local gui = Instance.new("ScreenGui")
gui.Name           = "VisionGUI"
gui.ResetOnSpawn   = false
gui.DisplayOrder   = 100
gui.IgnoreGuiInset = true
if not pcall(function() gui.Parent = game:GetService("CoreGui") end) or not gui.Parent then
	gui.Parent = lp:WaitForChild("PlayerGui")
end

-- helpers
local function corner(o, r)
	Instance.new("UICorner", o).CornerRadius = UDim.new(0, r or 8)
end
local function mkStroke(o, c, tr, th)
	local s = Instance.new("UIStroke", o)
	s.Color = c or ACCENT; s.Transparency = tr or 0.5; s.Thickness = th or 1
	return s
end
local function lbl(parent, text, size, font, color)
	local l = Instance.new("TextLabel")
	l.BackgroundTransparency = 1
	l.Text = text; l.TextSize = size or 13
	l.Font = font or Enum.Font.GothamMedium
	l.TextColor3 = color or Color3.new(1,1,1)
	l.TextXAlignment = Enum.TextXAlignment.Left
	l.Parent = parent
	return l
end

-- уведомление-тост внутри GUI
local function toast(text)
	local t = Instance.new("TextLabel", gui)
	t.AnchorPoint = Vector2.new(0.5, 1)
	t.Position = UDim2.new(0.5, 0, 1, -30)
	t.Size = UDim2.new(0, 360, 0, 44); t.TextWrapped = true
	t.BackgroundColor3 = BG; t.BackgroundTransparency = 0.1
	t.Text = text; t.TextColor3 = Color3.new(1,1,1)
	t.Font = Enum.Font.GothamMedium; t.TextSize = 13
	t.ZIndex = 50; corner(t, 8); mkStroke(t, ACCENT, 0.3)
	task.delay(2, function() t:Destroy() end)
end

-- фоновый ImageLabel прямо в ScreenGui (позади main)
local bgImage = Instance.new("ImageLabel", gui)
bgImage.Name                   = "BGLayer"
bgImage.AnchorPoint            = Vector2.new(0.5, 0.5)
bgImage.Position               = UDim2.new(0.5, 0, 0.5, 0)
bgImage.Size                   = UDim2.new(0, 420, 0, 360)
bgImage.BackgroundTransparency = 1
bgImage.Image                  = ""
bgImage.ImageTransparency      = 0.15
bgImage.ScaleType              = Enum.ScaleType.Crop
bgImage.ZIndex                 = 1
corner(bgImage, 12)

-- главное окно
local main = Instance.new("Frame", gui)
main.Size             = UDim2.new(0, 420, 0, 360)
main.AnchorPoint      = Vector2.new(0.5, 0.5)
main.Position         = UDim2.new(0.5, 0, 0.5, 0)
main.BackgroundColor3 = BG
main.BackgroundTransparency = 0.55
main.BorderSizePixel  = 0
main.ZIndex           = 2
corner(main, 12)
mkStroke(main, ACCENT, 0.4)

local mainScale = Instance.new("UIScale", main)
mainScale.Scale = 1

local function syncBG()
	bgImage.Position = main.Position
	bgImage.Size = UDim2.new(0, main.AbsoluteSize.X, 0, main.AbsoluteSize.Y)
end

-- заголовок
local titleBar = Instance.new("Frame", main)
titleBar.Size = UDim2.new(1, 0, 0, 38)
titleBar.BackgroundTransparency = 1
titleBar.ZIndex = 3
local titleLbl = lbl(titleBar, "VISION", 16, Enum.Font.GothamBold, ACCENT)
titleLbl.Position = UDim2.new(0, 14, 0, 0)
titleLbl.Size = UDim2.new(1, -28, 1, 0)
titleLbl.ZIndex = 3

-- drag
local guiLocked = false
do
	local dragging, startMainPos, startInput
	titleBar.InputBegan:Connect(function(i)
		if guiLocked then return end
		if i.UserInputType == Enum.UserInputType.MouseButton1
		or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true; startMainPos = main.Position; startInput = i.Position
			i.Changed:Connect(function()
				if i.UserInputState == Enum.UserInputState.End then dragging = false end
			end)
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement
		or i.UserInputType == Enum.UserInputType.Touch) then
			local d = i.Position - startInput
			main.Position = UDim2.new(
				startMainPos.X.Scale, startMainPos.X.Offset + d.X,
				startMainPos.Y.Scale, startMainPos.Y.Offset + d.Y)
			syncBG()
		end
	end)
end

UIS.InputBegan:Connect(function(i, gp)
	if not gp and i.KeyCode == Enum.KeyCode.LeftControl then
		local v = not main.Visible
		main.Visible = v; bgImage.Visible = v
	end
end)

-- tabs
local tabBar = Instance.new("Frame", main)
tabBar.Size = UDim2.new(1, -20, 0, 30)
tabBar.Position = UDim2.new(0, 10, 0, 40)
tabBar.BackgroundTransparency = 1; tabBar.ZIndex = 3
local tabLayout = Instance.new("UIListLayout", tabBar)
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding = UDim.new(0, 6)

local contentFrame = Instance.new("Frame", main)
contentFrame.Size = UDim2.new(1, -20, 1, -84)
contentFrame.Position = UDim2.new(0, 10, 0, 76)
contentFrame.BackgroundTransparency = 1; contentFrame.ZIndex = 3

-- кнопки в заголовке: свернуть и закрепить (lock)
do
	local collapsed = false
	local FULL_H, MIN_H = 360, 38
	local function mkTitleBtn(txt, xOff)
		local b = Instance.new("TextButton", titleBar)
		b.Size = UDim2.new(0,28,0,22); b.Position = UDim2.new(1,xOff,0.5,-11)
		b.BackgroundColor3 = PANEL; b.Text = txt; b.TextColor3 = Color3.new(1,1,1)
		b.Font = Enum.Font.GothamBold; b.TextSize = 14; b.AutoButtonColor = false; b.ZIndex = 4
		corner(b, 6)
		return b
	end
	local lockBtn = mkTitleBtn("🔓", -38)
	local minBtn  = mkTitleBtn("–", -70)

	lockBtn.MouseButton1Click:Connect(function()
		guiLocked = not guiLocked
		lockBtn.Text = guiLocked and "🔒" or "🔓"
	end)

	minBtn.MouseButton1Click:Connect(function()
		collapsed = not collapsed
		local shift = (FULL_H - MIN_H) / 2 * mainScale.Scale
		if collapsed then
			for _, c in ipairs(main:GetChildren()) do
				if c:IsA("GuiObject") and c ~= titleBar then c.Visible = false end
			end
			main.Size = UDim2.new(0, 420, 0, MIN_H)
			main.Position = main.Position - UDim2.new(0, 0, 0, shift)
		else
			main.Size = UDim2.new(0, 420, 0, FULL_H)
			main.Position = main.Position + UDim2.new(0, 0, 0, shift)
			tabBar.Visible = true; contentFrame.Visible = true
		end
		minBtn.Text = collapsed and "+" or "–"
		syncBG(); task.defer(syncBG)
	end)
end

local tabs, tabButtons = {}, {}

local function selectTab(name)
	for n, page in pairs(tabs) do
		page.Visible = (n == name)
		local b = tabButtons[n]
		TweenSvc:Create(b, TweenInfo.new(0.15), {
			BackgroundColor3 = (n == name) and ACCENT or PANEL,
			TextColor3 = (n == name) and Color3.new(0,0,0) or Color3.new(1,1,1),
		}):Play()
	end
end

local function newTab(name)
	local b = Instance.new("TextButton", tabBar)
	b.Size = UDim2.new(0, 76, 1, 0); b.BackgroundColor3 = PANEL
	b.Text = name; b.TextSize = 12; b.Font = Enum.Font.GothamBold
	b.TextColor3 = Color3.new(1,1,1); b.AutoButtonColor = false; b.ZIndex = 4
	corner(b, 8)
	local page = Instance.new("ScrollingFrame", contentFrame)
	page.Size = UDim2.new(1, 0, 1, 0); page.BackgroundTransparency = 1
	page.BorderSizePixel = 0; page.ScrollBarThickness = 3
	page.ScrollBarImageColor3 = ACCENT; page.AutomaticCanvasSize = Enum.AutomaticSize.Y
	page.CanvasSize = UDim2.new(); page.Visible = false; page.ZIndex = 4
	local layout = Instance.new("UIListLayout", page); layout.Padding = UDim.new(0, 6)
	tabs[name] = page; tabButtons[name] = b
	b.MouseButton1Click:Connect(function() selectTab(name) end)
	return page
end

local function addToggle(page, text, default, callback)
	local row = Instance.new("TextButton", page)
	row.Size = UDim2.new(1, -6, 0, 36); row.BackgroundColor3 = PANEL
	row.Text = ""; row.AutoButtonColor = false; row.ZIndex = 5
	corner(row, 8)
	local l = lbl(row, text, 13); l.Position = UDim2.new(0, 12, 0, 0)
	l.Size = UDim2.new(1, -70, 1, 0); l.ZIndex = 5
	local pill = Instance.new("Frame", row)
	pill.Size = UDim2.new(0, 38, 0, 20); pill.Position = UDim2.new(1, -50, 0.5, -10)
	pill.BackgroundColor3 = Color3.fromRGB(50,50,66); pill.ZIndex = 5; corner(pill, 10)
	local knob = Instance.new("Frame", pill)
	knob.Size = UDim2.new(0, 16, 0, 16); knob.Position = UDim2.new(0, 2, 0.5, -8)
	knob.BackgroundColor3 = Color3.new(1,1,1); knob.ZIndex = 6; corner(knob, 8)
	local state = false
	local function set(v)
		state = v and true or false
		TweenSvc:Create(pill, TweenInfo.new(0.15), {
			BackgroundColor3 = state and ACCENT or Color3.fromRGB(50,50,66),
		}):Play()
		TweenSvc:Create(knob, TweenInfo.new(0.15), {
			Position = state and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8),
		}):Play()
		if callback then task.spawn(callback, state) end
	end
	row.MouseButton1Click:Connect(function() set(not state) end)
	if default then set(true) end
	return set
end

-- кнопка-строка со стрелкой (открывает попап / выполняет действие)
local function addNavRow(page, text, arrowText, onClick)
	local row = Instance.new("TextButton", page)
	row.Size = UDim2.new(1,-6,0,36); row.BackgroundColor3 = PANEL
	row.Text = ""; row.AutoButtonColor = false; row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, text, 13)
	l.Position = UDim2.new(0,12,0,0); l.Size = UDim2.new(1,-50,1,0); l.ZIndex = 5
	local arrow = lbl(row, arrowText or "›", 18, Enum.Font.GothamBold, ACCENT)
	arrow.Position = UDim2.new(1,-30,0,0); arrow.Size = UDim2.new(0,18,1,0)
	arrow.TextXAlignment = Enum.TextXAlignment.Center; arrow.ZIndex = 5
	row.MouseButton1Click:Connect(function() onClick(row, l) end)
	return row, l
end

local function addSlider(page, text, mn, mx, default, callback, isInt)
	local row = Instance.new("Frame", page)
	row.Size = UDim2.new(1, -6, 0, 50); row.BackgroundColor3 = PANEL
	row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, text, 13); l.Position = UDim2.new(0, 12, 0, 4)
	l.Size = UDim2.new(1, -24, 0, 20); l.ZIndex = 5
	local valLbl = lbl(row, tostring(default), 13, Enum.Font.GothamBold, ACCENT)
	valLbl.Position = UDim2.new(0, 12, 0, 4); valLbl.Size = UDim2.new(1, -24, 0, 20)
	valLbl.TextXAlignment = Enum.TextXAlignment.Right; valLbl.ZIndex = 5
	local bar = Instance.new("Frame", row)
	bar.Size = UDim2.new(1, -24, 0, 6); bar.Position = UDim2.new(0, 12, 0, 34)
	bar.BackgroundColor3 = Color3.fromRGB(50,50,66); bar.ZIndex = 5; corner(bar, 3)
	local fill = Instance.new("Frame", bar)
	fill.BackgroundColor3 = ACCENT
	fill.Size = UDim2.new((default-mn)/(mx-mn), 0, 1, 0)
	fill.ZIndex = 6; corner(fill, 3)
	local dragging = false
	local function update(x)
		local a = math.clamp((x - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
		local v = isInt and math.floor(mn + (mx-mn)*a + 0.5) or math.floor((mn + (mx-mn)*a)*100+0.5)/100
		fill.Size = UDim2.new(a, 0, 1, 0); valLbl.Text = tostring(v); callback(v)
	end
	bar.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1
		or i.UserInputType == Enum.UserInputType.Touch then dragging = true; update(i.Position.X) end
	end)
	UIS.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1
		or i.UserInputType == Enum.UserInputType.Touch then dragging = false end
	end)
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement
		or i.UserInputType == Enum.UserInputType.Touch) then update(i.Position.X) end
	end)
end

-- ════════ ВКЛАДКИ ════════

local mainTab = newTab("Main")
local combatTab = newTab("Combat")

-- Visuals
local visTab = newTab("Visuals")
local espOn, tracersOn = false, false
local espObjects, tracerLines = {}, {}

local function clearESP()
	for _, objs in pairs(espObjects) do
		for _, o in ipairs(objs) do pcall(function() o:Destroy() end) end
	end; espObjects = {}
end
local function clearTracers()
	for _, l in pairs(tracerLines) do pcall(function() l:Remove() end) end; tracerLines = {}
end
local function ensureESP(plr)
	local char = plr.Character; if not char then return end
	local existing = espObjects[plr]
	if existing and existing[1] and existing[1].Parent == char then return end
	if existing then for _, o in ipairs(existing) do pcall(function() o:Destroy() end) end end
	local hl = Instance.new("Highlight")
	hl.FillColor = ACCENT; hl.FillTransparency = 0.6
	hl.OutlineColor = Color3.new(1,1,1)
	hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop; hl.Parent = char
	local head = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
	local bb
	if head then
		bb = Instance.new("BillboardGui")
		bb.Size = UDim2.new(0,120,0,18); bb.StudsOffset = Vector3.new(0,2.5,0)
		bb.AlwaysOnTop = true; bb.Adornee = head; bb.Parent = char
		local t = Instance.new("TextLabel", bb)
		t.Size = UDim2.new(1,0,1,0); t.BackgroundTransparency = 1
		t.Text = plr.DisplayName; t.TextColor3 = Color3.new(1,1,1)
		t.TextStrokeTransparency = 0.4; t.Font = Enum.Font.GothamBold; t.TextSize = 13
	end
	espObjects[plr] = {hl, bb}
end

task.spawn(function()
	while gui.Parent do
		if espOn then
			for _, p in ipairs(Players:GetPlayers()) do
				if p ~= lp then pcall(ensureESP, p) end
			end
		end
		task.wait(0.5)
	end
end)

RunService.RenderStepped:Connect(function()
	if not tracersOn or not Drawing then return end
	local seen = {}
	for _, p in ipairs(Players:GetPlayers()) do
		local hrp = p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart")
		if hrp then
			local pos, onScreen = cam:WorldToViewportPoint(hrp.Position)
			local line = tracerLines[p]
			if not line then line = Drawing.new("Line"); line.Thickness = 1.5; line.Color = ACCENT; tracerLines[p] = line end
			line.From = Vector2.new(cam.ViewportSize.X/2, cam.ViewportSize.Y)
			line.To = Vector2.new(pos.X, pos.Y); line.Visible = onScreen; seen[p] = true
		end
	end
	for p, l in pairs(tracerLines) do if not seen[p] then l.Visible = false end end
end)

addToggle(visTab, "ESP", false, function(v) espOn = v; if not v then clearESP() end end)
addToggle(visTab, "ESP Tracers", false, function(v)
	tracersOn = v; if not v then clearTracers() end
	if v and not Drawing then warn("[Vision] Drawing API недоступен") end
end)

local fovOn, fovVal = false, 70
RunService.RenderStepped:Connect(function() if fovOn then cam.FieldOfView = fovVal end end)
addToggle(visTab, "Custom FOV", false, function(v) fovOn = v; if not v then cam.FieldOfView = 70 end end)
addSlider(visTab, "FOV", 30, 120, 70, function(v) fovVal = v end)

local stretchOn, stretchY = false, 0.7
RunService.RenderStepped:Connect(function()
	if stretchOn then cam.CFrame = cam.CFrame * CFrame.new(0,0,0, 1,0,0, 0,stretchY,0, 0,0,1) end
end)
addToggle(visTab, "Stretch Resolution", false, function(v) stretchOn = v end)
addSlider(visTab, "Stretch Y", 0.3, 1, 0.7, function(v) stretchY = v end)

local origMat = setmetatable({}, {__mode="k"})
local origShadows, antiConn
local function lighten(d)
	if d:IsA("BasePart") then
		if not origMat[d] then origMat[d] = {d.Material, d.CastShadow} end
		d.Material = Enum.Material.SmoothPlastic; d.CastShadow = false
	elseif d:IsA("ParticleEmitter") or d:IsA("Trail") or d:IsA("Smoke") or d:IsA("Fire") or d:IsA("Sparkles") then
		d.Enabled = false
	elseif d:IsA("Decal") or d:IsA("Texture") then d.Transparency = 1 end
end
addToggle(visTab, "Anti Lag", false, function(v)
	if v then
		origShadows = Lighting.GlobalShadows; Lighting.GlobalShadows = false
		for _, d in ipairs(workspace:GetDescendants()) do pcall(lighten, d) end
		antiConn = workspace.DescendantAdded:Connect(function(d) pcall(lighten, d) end)
	else
		if antiConn then antiConn:Disconnect(); antiConn = nil end
		if origShadows ~= nil then Lighting.GlobalShadows = origShadows end
		for part, data in pairs(origMat) do
			pcall(function() part.Material = data[1]; part.CastShadow = data[2] end)
		end
		origMat = setmetatable({}, {__mode="k"})
	end
end)

local fbOrig
addToggle(visTab, "Fullbright", false, function(v)
	if v then
		fbOrig = {Lighting.Brightness, Lighting.ClockTime, Lighting.FogEnd, Lighting.Ambient}
		Lighting.Brightness = 2; Lighting.ClockTime = 14
		Lighting.FogEnd = 1e6; Lighting.Ambient = Color3.fromRGB(200,200,200)
	elseif fbOrig then
		Lighting.Brightness, Lighting.ClockTime, Lighting.FogEnd, Lighting.Ambient =
			fbOrig[1], fbOrig[2], fbOrig[3], fbOrig[4]
	end
end)

-- ════════ ACCESSORIES (визуально, только у тебя на клиенте) ════════
-- Принимает ССЫЛКИ: roblox.com/catalog/ID/... (вещь) или roblox.com/bundles/ID/... (бандл)

local AccAPI = {}
local accPopup = Instance.new("Frame", main)
accPopup.Size = UDim2.new(1,-16,1,-48); accPopup.Position = UDim2.new(0,8,0,42)
accPopup.BackgroundColor3 = BG; accPopup.BackgroundTransparency = 0.02
accPopup.Visible = false; accPopup.ZIndex = 10
corner(accPopup, 10); mkStroke(accPopup, ACCENT, 0.45)

do
	local entries     = {}   -- key -> {name=, ids={assetId,...}}
	local loadedCache = {}   -- assetId -> Accessory (шаблон)
	local equipped    = {}   -- key -> {Accessory,...} на персонаже
	local selected    = {}   -- key -> true (хочет быть надетым)
	local cards       = {}   -- key -> {btn=, stroke=}

	local function getHumanoid()
		local c = lp.Character
		return c and c:FindFirstChildOfClass("Humanoid")
	end

	-- HTTP GET с запасными вариантами под разные executor'ы
	local function httpGet(url)
		local ok, res = pcall(function() return game:HttpGet(url) end)
		if ok and type(res) == "string" and res ~= "" then return res end
		local req = (syn and syn.request) or request or http_request or (http and http.request)
		if req then
			local ok2, r = pcall(function() return req({ Url = url, Method = "GET" }) end)
			if ok2 and r and r.Body and r.Body ~= "" then return r.Body end
		end
		return nil
	end

	-- разбор ссылки: "asset" / "bundle" / "share"
	local function parseLink(text)
		text = tostring(text or "")
		local b = text:match("/bundles/(%d+)")
		if b then return "bundle", tonumber(b) end
		local a = text:match("/catalog/(%d+)") or text:match("/library/(%d+)")
		if a then return "asset", tonumber(a) end
		if text:find("share") then return "share" end
		local n = text:match("^%s*(%d+)%s*$")   -- голое число тоже принимаем
		if n then return "asset", tonumber(n) end
		return nil
	end

	-- бандл -> имя + список asset id входящих в него вещей
	local function resolveBundle(bid)
		local body = httpGet("https://catalog.roblox.com/v1/bundles/" .. bid .. "/details")
		if not body then return nil, "can't reach catalog API (HttpGet blocked)" end
		local ok, data = pcall(function() return game:GetService("HttpService"):JSONDecode(body) end)
		if not ok or type(data) ~= "table" or type(data.items) ~= "table" then
			return nil, "bad bundle response"
		end
		local ids = {}
		for _, it in ipairs(data.items) do
			if it.type == "Asset" then table.insert(ids, it.id) end
		end
		if #ids == 0 then return nil, "bundle has no assets" end
		return data.name or ("Bundle " .. bid), ids
	end

	local function fetchAccessory(id)
		if loadedCache[id] then return loadedCache[id] end
		local obj, err
		local ok, res = pcall(function() return game:GetObjects("rbxassetid://" .. id) end)
		if ok and res and res[1] then obj = res[1] else err = tostring(res) end
		if not obj then
			local ok2, model = pcall(function() return InsertService:LoadAsset(id) end)
			if ok2 and model then obj = model else err = (err or "") .. " | " .. tostring(model) end
		end
		if not obj then return nil, "load failed: " .. tostring(err) end
		local acc = obj:IsA("Accessory") and obj or obj:FindFirstChildWhichIsA("Accessory", true)
		if not acc then return nil, "not an accessory (" .. obj.ClassName .. ")" end
		loadedCache[id] = acc
		return acc
	end

	local function unequip(key)
		local list = equipped[key]
		if list then
			for _, a in ipairs(list) do pcall(function() a:Destroy() end) end
		end
		equipped[key] = nil
	end

	-- надеть одну вещь на персонажа (+ ручная приварка, если AddAccessory не справился)
	local function attach(clone, hum, char)
		local ok, e = pcall(function() hum:AddAccessory(clone) end)
		if not ok then ok, e = pcall(function() clone.Parent = char end) end
		if not ok then return false, e end
		local handle = clone:FindFirstChild("Handle")
		if not handle then return false, "accessory has no Handle" end
		handle.Anchored = false; handle.CanCollide = false; handle.Massless = true
		task.wait(0.2)
		if char and not handle:FindFirstChildWhichIsA("JointInstance") then
			local att = handle:FindFirstChildWhichIsA("Attachment")
			local target = att and char:FindFirstChild(att.Name, true)
			local w = Instance.new("Weld"); w.Name = "AccessoryWeld"
			w.Part0 = handle
			if target and target:IsA("Attachment") then
				w.Part1 = target.Parent; w.C0 = att.CFrame; w.C1 = target.CFrame
			else
				w.Part1 = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
				w.C0 = CFrame.new(0, -0.5, 0)
			end
			w.Parent = handle
		end
		return true
	end

	AccAPI.attach = attach

	-- надеть запись целиком (одна вещь или весь бандл)
	local function equipEntry(key)
		local e = entries[key]
		if not e then return false, "no entry" end
		local hum, char = getHumanoid(), lp.Character
		if not hum then return false, "no character" end
		unequip(key)
		local list, lastErr = {}, nil
		for _, aid in ipairs(e.ids) do
			local tpl, err = fetchAccessory(aid)
			if tpl then
				local clone = tpl:Clone(); clone.Name = "VisionAcc_" .. aid
				local ok, e2 = attach(clone, hum, char)
				if ok then table.insert(list, clone) else lastErr = e2 end
			else
				lastErr = err   -- например голова/одежда из бандла — пропускаем
			end
		end
		equipped[key] = list
		if #list == 0 then return false, lastErr or "nothing to equip" end
		toast("Equipped " .. #list .. " item(s)")
		return true
	end

	local function setCardState(key, on)
		local c = cards[key]
		if c then
			c.stroke.Transparency = on and 0 or 0.75
			c.stroke.Thickness = on and 2.5 or 1
		end
	end

	local function setSelected(key, on)
		if on then
			task.spawn(function()
				local ok, err = equipEntry(key)
				if ok then
					selected[key] = true; setCardState(key, true)
				else
					selected[key] = nil; setCardState(key, false)
					warn("[Vision] equip " .. key .. ": " .. tostring(err))
					toast("Equip failed: " .. string.sub(tostring(err), 1, 70))
				end
			end)
		else
			selected[key] = nil; unequip(key); setCardState(key, false)
		end
	end

	-- переодеваем после респавна
	lp.CharacterAdded:Connect(function(char)
		char:WaitForChild("Humanoid")
		task.wait(1)
		equipped = {}
		for key in pairs(selected) do task.spawn(equipEntry, key) end
	end)

	-- шапка
	local headLbl = lbl(accPopup, "Accessories", 15, Enum.Font.GothamBold, ACCENT)
	headLbl.Position = UDim2.new(0,12,0,7); headLbl.Size = UDim2.new(1,-120,0,24); headLbl.ZIndex = 11

	local function mkBtn(text, xOff)
		local b = Instance.new("TextButton", accPopup)
		b.Size = UDim2.new(0,52,0,24); b.Position = UDim2.new(1,xOff,0,7)
		b.BackgroundColor3 = PANEL; b.Text = text; b.TextColor3 = Color3.new(1,1,1)
		b.Font = Enum.Font.GothamBold; b.TextSize = 12; b.AutoButtonColor = false; b.ZIndex = 11
		corner(b, 6); return b
	end
	local noneBtn  = mkBtn("None", -112)
	local closeBtn = mkBtn("Back",  -56)
	closeBtn.MouseButton1Click:Connect(function() accPopup.Visible = false end)
	noneBtn.MouseButton1Click:Connect(function()
		local keys = {}
		for key in pairs(selected) do table.insert(keys, key) end
		for _, key in ipairs(keys) do setSelected(key, false) end
	end)

	-- подсказка: какую ссылку вставлять
	local hint = Instance.new("TextLabel", accPopup)
	hint.Size = UDim2.new(1,-24,0,48); hint.Position = UDim2.new(0,12,0,34)
	hint.BackgroundTransparency = 1; hint.RichText = true; hint.TextWrapped = true
	hint.TextXAlignment = Enum.TextXAlignment.Left; hint.TextYAlignment = Enum.TextYAlignment.Top
	hint.Font = Enum.Font.GothamMedium; hint.TextSize = 12
	hint.TextColor3 = Color3.fromRGB(200,200,215); hint.ZIndex = 11
	hint.Text = 'Paste the <u><font color="rgb(190,120,255)">item link</font></u> (catalog or bundle):\n'
		.. 'roblox.com/catalog/85015102365405/Anonymous-Hood\n'
		.. 'roblox.com/bundles/<font color="rgb(190,120,255)">ID</font>/Name'

	-- сетка с превью
	local grid = Instance.new("ScrollingFrame", accPopup)
	grid.Size = UDim2.new(1,-12,1,-130); grid.Position = UDim2.new(0,6,0,86)
	grid.BackgroundTransparency = 1; grid.BorderSizePixel = 0
	grid.ScrollBarThickness = 3; grid.ScrollBarImageColor3 = ACCENT
	grid.AutomaticCanvasSize = Enum.AutomaticSize.Y; grid.CanvasSize = UDim2.new()
	grid.ZIndex = 11
	local gl = Instance.new("UIGridLayout", grid)
	gl.CellSize = UDim2.new(0,118,0,100); gl.CellPadding = UDim2.new(0,6,0,6)
	Instance.new("UIPadding", grid).PaddingTop = UDim.new(0,4)

	local function addCard(key, name, ids, thumb)
		if cards[key] then return end
		local b = Instance.new("ImageButton", grid)
		b.Name = "ACC" .. key; b.BackgroundColor3 = Color3.fromRGB(30,31,44)
		b.Image = thumb or ""
		b.ScaleType = Enum.ScaleType.Fit
		b.AutoButtonColor = false; b.ZIndex = 12
		corner(b, 8)
		local st = mkStroke(b, ACCENT, 0.75)
		local nameBg = Instance.new("TextLabel", b)
		nameBg.Size = UDim2.new(1,0,0,20); nameBg.Position = UDim2.new(0,0,1,-20)
		nameBg.BackgroundColor3 = BG; nameBg.BackgroundTransparency = 0.3
		nameBg.Text = name; nameBg.TextColor3 = Color3.new(1,1,1)
		nameBg.Font = Enum.Font.GothamMedium; nameBg.TextSize = 11
		nameBg.TextTruncate = Enum.TextTruncate.AtEnd; nameBg.ZIndex = 13
		corner(nameBg, 6)
		cards[key] = { btn = b, stroke = st }

		-- 3D-превью из первой загруженной вещи (для бандла — первой подходящей)
		task.spawn(function()
			local acc
			for _, aid in ipairs(ids) do
				acc = fetchAccessory(aid)
				if acc then break end
			end
			if not acc or not b.Parent then return end
			local handle = acc:FindFirstChild("Handle")
			if not handle then return end
			local vp = Instance.new("ViewportFrame", b)
			vp.Size = UDim2.new(1,0,1,-20); vp.BackgroundTransparency = 1
			vp.Active = false; vp.ZIndex = 13
			local h = handle:Clone()
			for _, d in ipairs(h:GetDescendants()) do
				if d:IsA("LuaSourceContainer") or d:IsA("JointInstance") then d:Destroy() end
			end
			h.Anchored = true; h.CFrame = CFrame.new(0, 0, 0); h.Parent = vp
			local vcam = Instance.new("Camera", vp); vp.CurrentCamera = vcam
			local s = h.Size
			local dist = math.max(s.X, s.Y, s.Z) * 2.2 + 0.5
			vcam.CFrame = CFrame.new(Vector3.new(dist * 0.5, s.Y * 0.3, dist), Vector3.new(0, 0, 0))
		end)

		-- реальное название одиночной вещи
		if key:sub(1,1) == "a" then
			task.spawn(function()
				local ok, info = pcall(function()
					return game:GetService("MarketplaceService"):GetProductInfo(ids[1])
				end)
				if ok and info and info.Name and b.Parent then
					nameBg.Text = info.Name
					entries[key].name = info.Name
				end
			end)
		end

		b.MouseButton1Click:Connect(function()
			setSelected(key, not selected[key])
		end)

		-- крестик: убрать карточку и снять вещи
		local x = Instance.new("TextButton", b)
		x.Size = UDim2.new(0,18,0,18); x.Position = UDim2.new(1,-20,0,2)
		x.BackgroundColor3 = BG; x.BackgroundTransparency = 0.2
		x.Text = "×"; x.TextColor3 = Color3.new(1,1,1); x.TextSize = 14
		x.Font = Enum.Font.GothamBold; x.AutoButtonColor = false; x.ZIndex = 14
		corner(x, 9)
		x.MouseButton1Click:Connect(function()
			setSelected(key, false); cards[key] = nil; entries[key] = nil; b:Destroy()
		end)
	end

	-- поле для ссылки
	local box = Instance.new("TextBox", accPopup)
	box.Size = UDim2.new(1,-96,0,30); box.Position = UDim2.new(0,8,1,-38)
	box.BackgroundColor3 = Color3.fromRGB(40,41,58); box.Text = ""
	box.PlaceholderText = "Paste item link (catalog / bundle)"; box.TextColor3 = ACCENT
	box.PlaceholderColor3 = Color3.fromRGB(130,130,150)
	box.Font = Enum.Font.GothamBold; box.TextSize = 12
	box.ClearTextOnFocus = false; box.ClipsDescendants = true
	box.ZIndex = 12; corner(box, 6); mkStroke(box, ACCENT, 0.6)

	local addBtn = Instance.new("TextButton", accPopup)
	addBtn.Size = UDim2.new(0,76,0,30); addBtn.Position = UDim2.new(1,-84,1,-38)
	addBtn.BackgroundColor3 = ACCENT; addBtn.Text = "Equip"
	addBtn.TextColor3 = Color3.new(0,0,0); addBtn.Font = Enum.Font.GothamBold
	addBtn.TextSize = 13; addBtn.AutoButtonColor = false; addBtn.ZIndex = 12; corner(addBtn, 6)

	addBtn.MouseButton1Click:Connect(function()
		local kind, id = parseLink(box.Text)
		if not kind then toast("Paste an item link (catalog or bundle)"); return end
		if kind == "share" then
			toast("Share links have no ID. Open the item and copy the roblox.com/catalog/... link")
			return
		end
		task.spawn(function()
			local key = (kind == "bundle" and "b" or "a") .. id
			if not entries[key] then
				if kind == "asset" then
					entries[key] = { name = "ID " .. id, ids = { id } }
					addCard(key, "ID " .. id, { id },
						"rbxthumb://type=Asset&id=" .. id .. "&w=150&h=150")
				else
					toast("Loading bundle...")
					local name, ids = resolveBundle(id)
					if not name then toast("Bundle failed: " .. tostring(ids)); return end
					entries[key] = { name = name, ids = ids }
					addCard(key, name, ids,
						"rbxthumb://type=BundleThumbnail&id=" .. id .. "&w=150&h=150")
				end
			end
			if not selected[key] then setSelected(key, true) end
			box.Text = ""
		end)
	end)
end

addNavRow(visTab, "Accessories", "›", function() accPopup.Visible = true end)

-- ════════ SELECTOR (< значение >) ════════
local function addSelector(page, text, options, onChange)
	local row = Instance.new("Frame", page)
	row.Size = UDim2.new(1,-6,0,36); row.BackgroundColor3 = PANEL; row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, text, 13); l.Position = UDim2.new(0,12,0,0)
	l.Size = UDim2.new(1,-190,1,0); l.ZIndex = 5
	local idx = 1
	local val = lbl(row, options[1], 12, Enum.Font.GothamBold, ACCENT)
	val.Size = UDim2.new(0,100,1,0); val.Position = UDim2.new(1,-144,0,0)
	val.TextXAlignment = Enum.TextXAlignment.Center; val.TextTruncate = Enum.TextTruncate.AtEnd
	val.ZIndex = 5
	local function mk(txt, xOff)
		local b = Instance.new("TextButton", row)
		b.Size = UDim2.new(0,24,0,24); b.Position = UDim2.new(1,xOff,0.5,-12)
		b.BackgroundColor3 = Color3.fromRGB(40,41,58); b.Text = txt
		b.TextColor3 = Color3.new(1,1,1); b.Font = Enum.Font.GothamBold; b.TextSize = 13
		b.AutoButtonColor = false; b.ZIndex = 6; corner(b, 6)
		return b
	end
	local left, right = mk("<", -170), mk(">", -40)
	local function setIndex(i, silent)
		idx = i; val.Text = options[idx]
		if not silent and onChange then task.spawn(onChange, options[idx], idx) end
	end
	left.MouseButton1Click:Connect(function() setIndex(((idx - 2) % #options) + 1) end)
	right.MouseButton1Click:Connect(function() setIndex((idx % #options) + 1) end)
	return setIndex
end

-- ════════ ANIM PACK (данные из DemonDuels) ════════
local ANIM_PACKS = {
    ["Zombie"] = { idle1="rbxassetid://616158929", idle2="rbxassetid://616160636", walk="rbxassetid://616168032", run="rbxassetid://616163682", jump="rbxassetid://616161997", fall="rbxassetid://616157476", climb="rbxassetid://616156119", swim="rbxassetid://616165109", swimidle="rbxassetid://616166655" },
    ["Ninja"] = { idle1="rbxassetid://656117400", idle2="rbxassetid://656117400", walk="rbxassetid://656121766", run="rbxassetid://656118852", jump="rbxassetid://656117878", fall="rbxassetid://656115606", climb="rbxassetid://656114359", swim="rbxassetid://656117400", swimidle="rbxassetid://656117400" },
    ["Knight"] = { idle1="rbxassetid://657595757", idle2="rbxassetid://657595757", walk="rbxassetid://657552124", run="rbxassetid://657564596", jump="rbxassetid://658409194", fall="rbxassetid://657600338", climb="rbxassetid://658360781", swim="rbxassetid://657595757", swimidle="rbxassetid://657595757" },
    ["Elder"] = { idle1="rbxassetid://845397899", idle2="rbxassetid://845397899", walk="rbxassetid://845403856", run="rbxassetid://845386501", jump="rbxassetid://845398858", fall="rbxassetid://845397673", climb="rbxassetid://845392038", swim="rbxassetid://845397899", swimidle="rbxassetid://845397899" },
    ["Levitate"] = { idle1="rbxassetid://616006778", idle2="rbxassetid://616006778", walk="rbxassetid://616013216", run="rbxassetid://616013216", jump="rbxassetid://616008936", fall="rbxassetid://616005863", climb="rbxassetid://616003713", swim="rbxassetid://616006778", swimidle="rbxassetid://616006778" },
    ["Astronaut"] = { idle1="rbxassetid://891621366", idle2="rbxassetid://891621366", walk="rbxassetid://891636393", run="rbxassetid://891636393", jump="rbxassetid://891627522", fall="rbxassetid://891617961", climb="rbxassetid://891609353", swim="rbxassetid://891621366", swimidle="rbxassetid://891621366" },
    ["Pirate"] = { idle1="rbxassetid://750781874", idle2="rbxassetid://750781874", walk="rbxassetid://750785693", run="rbxassetid://750783738", jump="rbxassetid://750782230", fall="rbxassetid://750780242", climb="rbxassetid://750779899", swim="rbxassetid://750781874", swimidle="rbxassetid://750781874" },
    ["Toy"] = { idle1="rbxassetid://782841498", idle2="rbxassetid://782841498", walk="rbxassetid://782843345", run="rbxassetid://782842708", jump="rbxassetid://782847020", fall="rbxassetid://782846423", climb="rbxassetid://782843869", swim="rbxassetid://782841498", swimidle="rbxassetid://782841498" },
    ["Vampire"] = { idle1="rbxassetid://1083445855", idle2="rbxassetid://1083445855", walk="rbxassetid://1083473930", run="rbxassetid://1083462077", jump="rbxassetid://1083455352", fall="rbxassetid://1083443587", climb="rbxassetid://1083439238", swim="rbxassetid://1083445855", swimidle="rbxassetid://1083445855" },
    ["Werewolf"] = { idle1="rbxassetid://1083195517", idle2="rbxassetid://1083195517", walk="rbxassetid://1083178339", run="rbxassetid://1083216690", jump="rbxassetid://1083218792", fall="rbxassetid://1083189019", climb="rbxassetid://1083182000", swim="rbxassetid://1083195517", swimidle="rbxassetid://1083195517" },
    ["Rthro"] = { idle1="rbxassetid://2510196951", idle2="rbxassetid://2510196951", walk="rbxassetid://2510202577", run="rbxassetid://2510198475", jump="rbxassetid://2510197830", fall="rbxassetid://2510195892", climb="rbxassetid://2510192778", swim="rbxassetid://2510196951", swimidle="rbxassetid://2510196951" },
    ["Stylish"] = { idle1="rbxassetid://616136790", idle2="rbxassetid://616136790", walk="rbxassetid://616146177", run="rbxassetid://616140816", jump="rbxassetid://616139451", fall="rbxassetid://616134815", climb="rbxassetid://616133594", swim="rbxassetid://616136790", swimidle="rbxassetid://616136790" },
}

do
	local animOrder = { "Off" }
	for name in pairs(ANIM_PACKS) do table.insert(animOrder, name) end
	table.sort(animOrder, function(a, b)
		if a == "Off" then return true elseif b == "Off" then return false end
		return a < b
	end)

	local packIds = {}
	for _, pack in pairs(ANIM_PACKS) do for _, v in pairs(pack) do packIds[v] = true end end

	local currentPack, animConn, orig = "Off", nil, nil

	local function slots(char)
		local a = char and char:FindFirstChild("Animate")
		if not a then return nil end
		local function g(folder, name) local f = a:FindFirstChild(folder); return f and f:FindFirstChild(name) end
		return {
			idle1 = g("idle","Animation1"), idle2 = g("idle","Animation2"),
			walk = g("walk","WalkAnim"), run = g("run","RunAnim"),
			jump = g("jump","JumpAnim"), fall = g("fall","FallAnim"),
			climb = g("climb","ClimbAnim"), swim = g("swim","Swim"),
			swimidle = g("swimidle","SwimIdle"),
		}
	end

	local function saveOrig(char)
		local s = slots(char); if not s or orig then return end
		if s.walk and packIds[s.walk.AnimationId] then return end
		orig = {}
		for k, inst in pairs(s) do orig[k] = inst.AnimationId end
	end

	local function setAll(ids)
		local s = slots(lp.Character); if not s then return end
		for k, inst in pairs(s) do
			if ids[k] and inst.AnimationId ~= ids[k] then inst.AnimationId = ids[k] end
		end
	end

	local function applyPack(name)
		currentPack = name
		if animConn then animConn:Disconnect(); animConn = nil end
		local char = lp.Character
		if name == "Off" then
			if orig then setAll(orig) end
		else
			if char then saveOrig(char) end
			local pack = ANIM_PACKS[name]
			if not pack then return end
			animConn = RunService.Heartbeat:Connect(function() setAll(pack) end)
		end
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if hum then
			for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop(0) end
			hum:ChangeState(Enum.HumanoidStateType.Running)
		end
	end

	addSelector(visTab, "Anim Pack", animOrder, applyPack)
end

-- ════════ OUTFITS / BEST AVATARS (локально, только у тебя) ════════
local AVATARS = {
	{ "ALIEN",    { 10997283529, 125085889448130 } },
	{ "OUTFIT",   { 125157548394108, 123231522991173, 74827642361405 } },
	{ "OUTFIT 2", { 136644276218918, 132022106987543, 131740907052260 } },
	{ "OUTFIT 3", { 119331031817124, 84952305140948 } },
	{ "OUTFIT 4", { 18902416696, 17739456422, 116750288839357, 12842417046, 122223069519702 } },
	{ "OUTFIT 5", { 13935333090, 16110951997, 10632503818 } },
	{ "OUTFIT 6", { 74891470, 4390891467, 91274263831244, 128424248603010, 91151766440767, 17044638424 } },
	{ "DEMON",    { 76479271580913, 1402432199, 140627490971265, 128912609563885, 17449551560, 5796531729, 139607718 } },
}
local OUTFITS_SIMPLE = {
	{ "Outfit 1", { 10159600649, 9683332638, 93182020184041 } },
	{ "Outfit 2", { 1744060292,  9683332638, 93182020184041 } },
	{ "Outfit 5", { 18552805597, 5414143509, 84008082880128 } },
	{ "Outfit 6", { 11416036966, 5842013905, 86475919719501, 121789170766733 } },
}

do
	local dressed, removed = {}, {}
	local currentItems = nil
	local setOutfitSel, setAvatarSel

	local function clearDressed()
		for _, o in ipairs(dressed) do pcall(function() o:Destroy() end) end
		dressed = {}
		for _, o in ipairs(removed) do pcall(function() o.Parent = lp.Character end) end
		removed = {}
	end

	local function loadAny(id)
		local ok, res = pcall(function() return game:GetObjects("rbxassetid://" .. id) end)
		if ok and res and res[1] then return res[1] end
		local ok2, model = pcall(function() return InsertService:LoadAsset(id) end)
		if ok2 and model then return model end
		return nil
	end

	local function dress(items)
		clearDressed()
		currentItems = items
		if not items then return end
		local char = lp.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not hum then toast("No character"); return end
		local n = 0
		for _, id in ipairs(items) do
			local obj = loadAny(id)
			if obj then
				if obj:IsA("Shirt") or obj:IsA("Pants") or obj:IsA("ShirtGraphic") then
					for _, old in ipairs(char:GetChildren()) do
						if old.ClassName == obj.ClassName then
							table.insert(removed, old); old.Parent = nil
						end
					end
					local c = obj:Clone(); c.Parent = char
					table.insert(dressed, c); n = n + 1
				else
					local acc = obj:IsA("Accessory") and obj or obj:FindFirstChildWhichIsA("Accessory", true)
					if acc then
						local c = acc:Clone(); c.Name = "VisionAv_" .. id
						local ok = AccAPI.attach and AccAPI.attach(c, hum, char)
						if ok then table.insert(dressed, c); n = n + 1 else pcall(function() c:Destroy() end) end
					end
				end
			end
		end
		toast("Applied " .. n .. "/" .. #items .. " items")
	end

	lp.CharacterAdded:Connect(function()
		removed = {}; dressed = {}
		if currentItems then
			task.wait(1.5)
			local items = currentItems
			task.spawn(dress, items)
		end
	end)

	local outfitNames, avatarNames = { "Off" }, { "Off" }
	for _, o in ipairs(OUTFITS_SIMPLE) do table.insert(outfitNames, o[1]) end
	for _, a in ipairs(AVATARS) do table.insert(avatarNames, a[1]) end

	setOutfitSel = addSelector(visTab, "Outfits", outfitNames, function(name, idx)
		if setAvatarSel then setAvatarSel(1, true) end
		dress(idx > 1 and OUTFITS_SIMPLE[idx - 1][2] or nil)
	end)
	setAvatarSel = addSelector(visTab, "Best Avatars", avatarNames, function(name, idx)
		if setOutfitSel then setOutfitSel(1, true) end
		dress(idx > 1 and AVATARS[idx - 1][2] or nil)
	end)
end

-- ════════ VIVID GRAPHICS + SKY THEME (данные из DemonDuels) ════════
local vividGraphicsEnabled, _vividEffects, skyTheme = false, {}, "Off"

local SKY_PRESETS_LIST = {"Off","Night","Aurora","Sunset","Galaxy","Cyber","Sakura","Pink Night","Blood Moon","Emerald Dawn","Volcanic","Arctic","Midnight Ocean","Vaporwave","Toxic","Solar Eclipse","Hellscape","Heaven","Storm","Sunrise","Deep Space","Lavender Dream","Inferno","Mint Sky"}

local SKY_PRESETS = {
    ["Off"]={kind="off"},
    ["Night"]={clock=22,brightness=2,ambient={110,100,130},outAmb={120,110,140},sky={stars=4000,moon=18,sun=0,moonTex=true},atm={dens=0.45,color={120,60,180},decay={60,20,100},glare=0.5,haze=1.2}},
    ["Aurora"]={clock=14,brightness=3,ambient={150,120,150},outAmb={160,130,150},atm={dens=0.55,color={255,80,200},decay={255,20,150},glare=2.5,haze=3},clouds={cover=0.7,dens=0.7,color={255,240,250}}},
    ["Sunset"]={clock=17.2,brightness=2.5,ambient={170,120,100},outAmb={180,130,110},sky={stars=0,sun=25,moon=0},atm={dens=0.5,color={255,130,60},decay={255,80,30},glare=2,haze=2.5},clouds={cover=0.55,dens=0.55,color={255,200,140}}},
    ["Galaxy"]={clock=0,brightness=1.5,ambient={70,60,100},outAmb={80,70,110},sky={stars=10000,moon=30,sun=0},atm={dens=0.15,color={40,20,80},decay={20,10,50},glare=0.3,haze=0.5}},
    ["Cyber"]={clock=21,brightness=2.2,ambient={90,130,170},outAmb={100,140,180},sky={stars=2000,moon=12},atm={dens=0.4,color={0,200,255},decay={150,0,255},glare=2,haze=2},clouds={cover=0.4,dens=0.6,color={100,200,255}}},
    ["Sakura"]={clock=11,brightness=3.5,ambient={170,150,160},outAmb={180,160,170},sky={sun=8},atm={dens=0.3,color={255,200,220},decay={255,170,200},glare=1,haze=1.5},clouds={cover=0.6,dens=0.4,color={255,250,252}}},
    ["Pink Night"]={clock=23,brightness=2.2,ambient={120,60,110},outAmb={140,70,120},sky={stars=5000,moon=22,sun=0,moonTex=true},atm={dens=0.5,color={255,80,180},decay={140,30,100},glare=0.7,haze=1.4},clouds={cover=0.3,dens=0.5,color={180,90,150}}},
    ["Blood Moon"]={clock=22.5,brightness=1.6,ambient={130,40,40},outAmb={150,50,50},sky={stars=1500,moon=28,sun=0,moonTex=true},atm={dens=0.6,color={220,30,30},decay={120,10,10},glare=1.4,haze=2},clouds={cover=0.5,dens=0.7,color={120,30,30}}},
    ["Emerald Dawn"]={clock=6.5,brightness=2.8,ambient={130,170,140},outAmb={140,180,150},sky={sun=18,moon=0,stars=0},atm={dens=0.4,color={80,200,140},decay={40,150,90},glare=1.8,haze=2.2},clouds={cover=0.5,dens=0.5,color={200,255,220}}},
    ["Volcanic"]={clock=19,brightness=2,ambient={180,80,40},outAmb={200,90,50},sky={stars=200,sun=12,moon=0},atm={dens=0.75,color={255,60,0},decay={180,20,0},glare=3,haze=3.5},clouds={cover=0.8,dens=0.9,color={120,40,20}}},
    ["Arctic"]={clock=9,brightness=3.2,ambient={200,220,235},outAmb={210,230,245},sky={sun=10,stars=0,moon=0},atm={dens=0.3,color={180,220,255},decay={140,200,240},glare=1.5,haze=1.8},clouds={cover=0.7,dens=0.6,color={250,253,255}}},
    ["Midnight Ocean"]={clock=1.5,brightness=1.7,ambient={60,90,130},outAmb={70,100,140},sky={stars=6000,moon=24,sun=0,moonTex=true},atm={dens=0.5,color={20,60,140},decay={10,30,90},glare=0.6,haze=1.5}},
    ["Vaporwave"]={clock=19.5,brightness=2.4,ambient={180,120,200},outAmb={190,130,210},sky={stars=1000,moon=14},atm={dens=0.45,color={255,100,220},decay={120,60,255},glare=2.2,haze=2.4},clouds={cover=0.55,dens=0.55,color={200,150,255}}},
    ["Toxic"]={clock=13,brightness=2.5,ambient={140,180,80},outAmb={150,190,90},atm={dens=0.55,color={100,220,40},decay={60,150,20},glare=1.8,haze=2.6},clouds={cover=0.65,dens=0.7,color={180,255,120}}},
    ["Solar Eclipse"]={clock=12,brightness=0.9,ambient={50,40,60},outAmb={60,50,70},sky={stars=3500,sun=22,moon=0},atm={dens=0.5,color={255,140,40},decay={30,20,40},glare=2.8,haze=1.8}},
    ["Hellscape"]={clock=18,brightness=1.8,ambient={200,60,30},outAmb={220,70,40},sky={stars=100,sun=30,moon=0},atm={dens=0.85,color={255,30,0},decay={120,0,0},glare=3.5,haze=4},clouds={cover=0.95,dens=0.95,color={80,20,10}}},
    ["Heaven"]={clock=12,brightness=4,ambient={240,235,210},outAmb={250,245,220},sky={sun=16,moon=0,stars=0},atm={dens=0.25,color={255,250,220},decay={255,240,200},glare=3,haze=1.5},clouds={cover=0.85,dens=0.5,color={255,255,255}}},
    ["Storm"]={clock=15,brightness=1.4,ambient={90,90,110},outAmb={100,100,120},sky={stars=0,sun=6,moon=0},atm={dens=0.65,color={80,90,120},decay={40,50,80},glare=0.5,haze=3},clouds={cover=0.95,dens=0.95,color={60,65,80}}},
    ["Sunrise"]={clock=6.2,brightness=2.8,ambient={220,180,130},outAmb={230,190,140},sky={sun=22,stars=0,moon=0},atm={dens=0.45,color={255,180,100},decay={255,140,80},glare=2.4,haze=2.2},clouds={cover=0.4,dens=0.4,color={255,220,180}}},
    ["Deep Space"]={clock=0,brightness=1,ambient={30,25,50},outAmb={40,35,60},sky={stars=15000,moon=0,sun=0},atm={dens=0.08,color={15,5,40},decay={5,0,20},glare=0.2,haze=0.3}},
    ["Lavender Dream"]={clock=18.5,brightness=2.6,ambient={180,160,220},outAmb={190,170,230},sky={stars=800,moon=16,sun=0},atm={dens=0.4,color={200,160,255},decay={160,120,220},glare=1.4,haze=1.8},clouds={cover=0.55,dens=0.5,color={220,200,255}}},
    ["Inferno"]={clock=17.5,brightness=2.2,ambient={220,100,40},outAmb={235,110,50},sky={sun=26,moon=0,stars=0},atm={dens=0.6,color={255,90,20},decay={200,40,0},glare=3,haze=3.2},clouds={cover=0.7,dens=0.7,color={200,80,40}}},
    ["Mint Sky"]={clock=10,brightness=3.2,ambient={180,230,210},outAmb={190,240,220},sky={sun=10},atm={dens=0.32,color={150,255,210},decay={100,220,180},glare=1.6,haze=1.6},clouds={cover=0.55,dens=0.45,color={240,255,250}}},
}

local function _vC3(t) return Color3.fromRGB(t[1], t[2], t[3]) end

function _v4mpClearSky()
    for _, child in ipairs(Lighting:GetChildren()) do
        if child:GetAttribute("_AdaptDuelsSky") then
            pcall(function() child:Destroy() end)
        end
    end
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if terrain then
        for _, child in ipairs(terrain:GetChildren()) do
            if child:GetAttribute("_AdaptDuelsSky") then
                pcall(function() child:Destroy() end)
            end
        end
    end
end

function applyCustomSky(mode)
    _v4mpClearSky()
    local preset = SKY_PRESETS[mode]
    if not preset or preset.kind == "off" then
        Lighting.ClockTime = 14
        Lighting.Brightness = 2
        Lighting.OutdoorAmbient = Color3.fromRGB(127,127,127)
        Lighting.Ambient = Color3.fromRGB(127,127,127)
        Lighting.FogEnd = 100000
        Lighting.GlobalShadows = true
        skyTheme = "Off"
        return
    end
    Lighting.FogStart = 0
    Lighting.FogEnd = 100000
    Lighting.FogColor = Color3.fromRGB(200,200,200)
    Lighting.ColorShift_Top = Color3.fromRGB(0,0,0)
    Lighting.ColorShift_Bottom = Color3.fromRGB(0,0,0)
    Lighting.GlobalShadows = true
    Lighting.ClockTime = preset.clock or 14
    Lighting.Brightness = math.max(tonumber(preset.brightness) or 2, 2.5)
    if preset.outAmb then Lighting.OutdoorAmbient = _vC3(preset.outAmb) end
    if preset.ambient then Lighting.Ambient = _vC3(preset.ambient) end
    local amb = Lighting.Ambient
    local out = Lighting.OutdoorAmbient
    Lighting.Ambient = Color3.fromRGB(math.max(amb.R * 255, 135), math.max(amb.G * 255, 135), math.max(amb.B * 255, 135))
    Lighting.OutdoorAmbient = Color3.fromRGB(math.max(out.R * 255, 145), math.max(out.G * 255, 145), math.max(out.B * 255, 145))
    if preset.sky then
        local skyInst = Instance.new("Sky")
        skyInst:SetAttribute("_AdaptDuelsSky", true)
        if preset.sky.stars then skyInst.StarCount = preset.sky.stars end
        if preset.sky.moon then skyInst.MoonAngularSize = preset.sky.moon end
        if preset.sky.sun then skyInst.SunAngularSize = preset.sky.sun end
        if preset.sky.moonTex then skyInst.MoonTextureId = "rbxasset://sky/moon.jpg" end
        skyInst.Parent = Lighting
    end
    if preset.atm then
        local atm = Instance.new("Atmosphere")
        atm:SetAttribute("_AdaptDuelsSky", true)
        atm.Density = preset.atm.dens or 0.3
        atm.Color = _vC3(preset.atm.color)
        atm.Decay = _vC3(preset.atm.decay)
        atm.Glare = preset.atm.glare or 1
        atm.Haze = preset.atm.haze or 1
        atm.Parent = Lighting
    end
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if preset.clouds and terrain then
        local clouds = Instance.new("Clouds")
        clouds:SetAttribute("_AdaptDuelsSky", true)
        clouds.Cover = preset.clouds.cover or 0.5
        clouds.Density = preset.clouds.dens or 0.5
        clouds.Color = _vC3(preset.clouds.color)
        clouds.Parent = terrain
    end
    skyTheme = mode
end

function enableVividGraphics()
    if vividGraphicsEnabled and #_vividEffects > 0 then return end
    vividGraphicsEnabled = true
    for _, eff in ipairs(_vividEffects) do
        pcall(function() eff:Destroy() end)
    end
    _vividEffects = {}
    local color = Instance.new("ColorCorrectionEffect")
    color.Parent = Lighting
    color.Saturation = 0.6; color.Contrast = 0.4; color.Brightness = 0.05
    color.TintColor = Color3.fromRGB(255, 240, 220)
    table.insert(_vividEffects, color)
    local bloom = Instance.new("BloomEffect")
    bloom.Parent = Lighting
    bloom.Intensity = 0.8; bloom.Size = 24; bloom.Threshold = 1
    table.insert(_vividEffects, bloom)
    local atmosphere = Instance.new("Atmosphere")
    atmosphere.Parent = Lighting
    atmosphere.Density = 0.3; atmosphere.Offset = 0.25
    atmosphere.Color = Color3.fromRGB(199, 199, 255)
    atmosphere.Decay = Color3.fromRGB(106, 112, 125)
    atmosphere.Glare = 0.2; atmosphere.Haze = 1
    table.insert(_vividEffects, atmosphere)
    local sun = Instance.new("SunRaysEffect")
    sun.Parent = Lighting
    sun.Intensity = 0.2; sun.Spread = 0.8
    table.insert(_vividEffects, sun)
    local dof = Instance.new("DepthOfFieldEffect")
    dof.Parent = Lighting
    dof.FocusDistance = 25; dof.InFocusRadius = 10
    dof.NearIntensity = 0.2; dof.FarIntensity = 0.4
    table.insert(_vividEffects, dof)
    task.spawn(function()
        while vividGraphicsEnabled and color and color.Parent do
            color.Contrast = 0.35 + math.sin(tick() * 2) * 0.05
            task.wait(0.03)
        end
    end)
end

function disableVividGraphics()
    vividGraphicsEnabled = false
    for _, eff in ipairs(_vividEffects) do
        pcall(function() eff:Destroy() end)
    end
    _vividEffects = {}
end


addToggle(visTab, "Vivid Graphics", false, function(v)
	if v then pcall(enableVividGraphics) else pcall(disableVividGraphics) end
end)
addSelector(visTab, "Sky Theme", SKY_PRESETS_LIST, function(name) pcall(applyCustomSky, name) end)


-- ════════ ВИДИМОСТЬ ФОНА (1-100) ════════
do
	local function setBgVisibility(v)
		bgImage.ImageTransparency = 1 - v / 100
		-- до 85% тёмная подложка окна как раньше, выше — плавно исчезает, чтобы фон было видно полностью
		local overlay = 0.55
		if v > 85 then overlay = 0.55 + 0.45 * (v - 85) / 15 end
		main.BackgroundTransparency = overlay
	end
	addSlider(visTab, "Background Visibility", 1, 100, 85, setBgVisibility, true)
end

-- ════════ THEME COLOR (общий цвет интерфейса) ════════
do
	local function close(a, b)
		return math.abs(a.R - b.R) < 0.004 and math.abs(a.G - b.G) < 0.004 and math.abs(a.B - b.B) < 0.004
	end

	local function setAccent(c)
		local old = ACCENT
		if close(old, c) then return end
		for _, d in ipairs(gui:GetDescendants()) do
			if d:IsA("UIStroke") then
				if close(d.Color, old) then d.Color = c end
			elseif d:IsA("GuiObject") then
				if close(d.BackgroundColor3, old) then d.BackgroundColor3 = c end
				if (d:IsA("TextLabel") or d:IsA("TextButton") or d:IsA("TextBox")) and close(d.TextColor3, old) then
					d.TextColor3 = c
				end
				if (d:IsA("ImageLabel") or d:IsA("ImageButton")) and close(d.ImageColor3, old) then
					d.ImageColor3 = c
				end
				if d:IsA("ScrollingFrame") and close(d.ScrollBarImageColor3, old) then
					d.ScrollBarImageColor3 = c
				end
			end
		end
		ACCENT = c
	end

	local COLORS = {
		{ "Purple", Color3.fromRGB(190, 120, 255) },
		{ "Red",    Color3.fromRGB(220, 40, 40) },
		{ "Blue",   Color3.fromRGB(70, 130, 255) },
		{ "Green",  Color3.fromRGB(60, 220, 120) },
		{ "Pink",   Color3.fromRGB(255, 105, 180) },
		{ "Orange", Color3.fromRGB(255, 150, 50) },
		{ "Cyan",   Color3.fromRGB(0, 220, 255) },
		{ "Yellow", Color3.fromRGB(255, 220, 60) },
		{ "White",  Color3.fromRGB(235, 235, 235) },
		{ "Black",  Color3.fromRGB(12, 12, 12) },
	}
	local names = {}
	for _, c in ipairs(COLORS) do table.insert(names, c[1]) end

	local hexBox
	local function toHex(c)
		return string.format("%02X%02X%02X", math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5))
	end

	addSelector(visTab, "Theme Color", names, function(_, idx)
		setAccent(COLORS[idx][2])
		if hexBox then hexBox.Text = toHex(COLORS[idx][2]) end
	end)

	-- свой цвет: HEX вида BE78FF
	local row = Instance.new("Frame", visTab)
	row.Size = UDim2.new(1,-6,0,36); row.BackgroundColor3 = PANEL; row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, "Custom Color (HEX)", 13); l.Position = UDim2.new(0,12,0,0)
	l.Size = UDim2.new(1,-120,1,0); l.ZIndex = 5
	hexBox = Instance.new("TextBox", row)
	hexBox.Size = UDim2.new(0,90,0,24); hexBox.Position = UDim2.new(1,-102,0.5,-12)
	hexBox.BackgroundColor3 = Color3.fromRGB(40,41,58); hexBox.Text = toHex(ACCENT)
	hexBox.PlaceholderText = "RRGGBB"; hexBox.TextColor3 = ACCENT
	hexBox.Font = Enum.Font.GothamBold; hexBox.TextSize = 13
	hexBox.ClearTextOnFocus = false; hexBox.ZIndex = 6; corner(hexBox, 6); mkStroke(hexBox, ACCENT, 0.6)
	hexBox.FocusLost:Connect(function()
		local s = hexBox.Text:gsub("[^%x]", "")
		if #s == 6 then
			local r, g, b = tonumber(s:sub(1,2), 16), tonumber(s:sub(3,4), 16), tonumber(s:sub(5,6), 16)
			setAccent(Color3.fromRGB(r, g, b))
			hexBox.Text = s:upper()
		else
			hexBox.Text = toHex(ACCENT)
		end
	end)
end

-- ════════ ESP ROLE (Murderer / Sheriff / Innocent / Lobby, под Murder Mystery) ════════
do
	local ROLE_COLORS = {
		Murderer = Color3.fromRGB(255, 40, 40),
		Sheriff  = Color3.fromRGB(60, 130, 255),
		Innocent = Color3.fromRGB(60, 220, 120),
		Lobby    = Color3.fromRGB(160, 160, 170),
	}
	local roleOn = false
	local objs = {}   -- plr -> {hl=, bb=, txt=}

	-- есть ли у игрока предмет с нужным словом в названии (в рюкзаке или в руках)
	local function hasTool(plr, word)
		local function scan(container)
			if not container then return false end
			for _, c in ipairs(container:GetChildren()) do
				if c:IsA("Tool") and c.Name:lower():find(word, 1, true) then return true end
			end
			return false
		end
		return scan(plr:FindFirstChildOfClass("Backpack")) or scan(plr.Character)
	end

	local lobbyCF, lobbySize
	local function refreshLobby()
		lobbyCF, lobbySize = nil, nil
		local lobby = workspace:FindFirstChild("Lobby")
		if lobby and lobby:IsA("Model") then
			local ok, cf, size = pcall(function() return lobby:GetBoundingBox() end)
			if ok then lobbyCF, lobbySize = cf, size end
		end
	end
	local function inLobbyArea(plr)
		local root = plr.Character and plr.Character:FindFirstChild("HumanoidRootPart")
		if not root or not lobbyCF then return false end
		local rel = lobbyCF:PointToObjectSpace(root.Position)
		return math.abs(rel.X) <= lobbySize.X / 2 + 8
			and math.abs(rel.Y) <= lobbySize.Y / 2 + 30
			and math.abs(rel.Z) <= lobbySize.Z / 2 + 8
	end

	local function clearRole(plr)
		local o = objs[plr]
		if o then
			pcall(function() o.hl:Destroy() end)
			pcall(function() o.bb:Destroy() end)
			objs[plr] = nil
		end
	end
	local function clearAllRoles()
		for plr in pairs(objs) do clearRole(plr) end
	end

	local function applyRole(plr, role)
		local char = plr.Character
		local head = char and (char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart"))
		if not char or not head then clearRole(plr); return end
		local o = objs[plr]
		if not o or not o.hl.Parent or o.hl.Parent ~= char then
			clearRole(plr)
			local hl = Instance.new("Highlight")
			hl.FillTransparency = 0.55; hl.OutlineColor = Color3.new(1,1,1)
			hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop; hl.Parent = char
			local bb = Instance.new("BillboardGui")
			bb.Size = UDim2.new(0,140,0,32); bb.StudsOffset = Vector3.new(0,3,0)
			bb.AlwaysOnTop = true; bb.Adornee = head; bb.Parent = char
			local txt = Instance.new("TextLabel", bb)
			txt.Size = UDim2.new(1,0,1,0); txt.BackgroundTransparency = 1
			txt.Font = Enum.Font.GothamBold; txt.TextSize = 13
			txt.TextStrokeTransparency = 0.3
			o = { hl = hl, bb = bb, txt = txt }
			objs[plr] = o
		end
		local c = ROLE_COLORS[role]
		o.hl.FillColor = c
		o.txt.TextColor3 = c
		o.txt.Text = plr.DisplayName .. "\n[" .. role .. "]"
	end

	task.spawn(function()
		while gui.Parent do
			if roleOn then
				refreshLobby()
				local roundActive = false
				for _, p in ipairs(Players:GetPlayers()) do
					if hasTool(p, "knife") then roundActive = true; break end
				end
				for _, p in ipairs(Players:GetPlayers()) do
					if p ~= lp then
						pcall(function()
							local role
							if hasTool(p, "knife") then role = "Murderer"
							elseif hasTool(p, "gun") or hasTool(p, "revolver") then role = "Sheriff"
							elseif not roundActive or inLobbyArea(p) then role = "Lobby"
							else role = "Innocent" end
							applyRole(p, role)
						end)
					end
				end
			end
			task.wait(0.3)
		end
	end)
	Players.PlayerRemoving:Connect(clearRole)

	addToggle(visTab, "ESP Role", false, function(on)
		roleOn = on
		if not on then clearAllRoles() end
	end)
end

-- ════════ ПОЛЯ ВВОДА ════════
local function mkNumBox(row, xOff, default, onCommit)
	local box = Instance.new("TextBox", row)
	box.Size = UDim2.new(0,66,0,24); box.Position = UDim2.new(1,xOff,0.5,-12)
	box.BackgroundColor3 = Color3.fromRGB(40,41,58); box.Text = tostring(default)
	box.TextColor3 = ACCENT; box.Font = Enum.Font.GothamBold; box.TextSize = 13
	box.ClearTextOnFocus = false; box.ZIndex = 6; corner(box, 6); mkStroke(box, ACCENT, 0.6)
	local last = default
	box.FocusLost:Connect(function()
		local n = tonumber(box.Text)
		if not n then box.Text = tostring(last); return end
		n = math.clamp(n, 0, 1000)
		last = n; box.Text = tostring(n)
		onCommit(n)
	end)
	return box
end

local function addNumberRow(page, text, default, onChange)
	local row = Instance.new("Frame", page)
	row.Size = UDim2.new(1,-6,0,36); row.BackgroundColor3 = PANEL; row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, text, 13); l.Position = UDim2.new(0,12,0,0)
	l.Size = UDim2.new(1,-100,1,0); l.ZIndex = 5
	mkNumBox(row, -78, default, onChange)
end

-- два поля: от (min) и до (max)
local function addRangeRow(page, text, defMin, defMax, onChange)
	local row = Instance.new("Frame", page)
	row.Size = UDim2.new(1,-6,0,36); row.BackgroundColor3 = PANEL; row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, text, 13); l.Position = UDim2.new(0,12,0,0)
	l.Size = UDim2.new(1,-190,1,0); l.ZIndex = 5
	local mn, mx = defMin, defMax
	local function commit()
		local a, b = mn, mx
		if a > b then a, b = b, a end
		onChange(a, b)
	end
	mkNumBox(row, -150, defMin, function(n) mn = n; commit() end)
	local dash = lbl(row, "-", 14, Enum.Font.GothamBold, ACCENT)
	dash.Size = UDim2.new(0,12,1,0); dash.Position = UDim2.new(1,-80,0,0)
	dash.TextXAlignment = Enum.TextXAlignment.Center; dash.ZIndex = 6
	mkNumBox(row, -72, defMax, function(n) mx = n; commit() end)
end

-- ════════ MAIN: Normal Speed + Anti Ragdoll ════════
-- ════════ COMBAT: Wallhop (Spam Jump) / Wallhop (A-D) ════════
local Move = {
	normalOn = false, normal = 60,
	mode = nil,                       -- nil / "spam" / "ad"
	spam = { min = 16, max = 60 },
	ad   = { min = 16, max = 60 },
	arBoost = false,                  -- true, пока Anti Ragdoll V1 разгоняет выход из рэгдолла
}

addToggle(mainTab, "Normal Speed", false, function(on) Move.normalOn = on end)
addNumberRow(mainTab, "Normal Speed value", 60, function(n) Move.normal = n end)

do
	-- кнопка прыжка на телефоне: запоминаем само касание и проверяем его состояние
	local jumpTouch = nil
	task.spawn(function()
		local tg = lp:WaitForChild("PlayerGui"):WaitForChild("TouchGui", 20)
		local frame = tg and tg:WaitForChild("TouchControlFrame", 5)
		local btn = frame and frame:WaitForChild("JumpButton", 5)
		if not btn then return end
		btn.InputBegan:Connect(function(i)
			if i.UserInputType == Enum.UserInputType.Touch then jumpTouch = i end
		end)
	end)

	-- ввод движения (стик на телефоне / WASD / геймпад): X < 0 влево, X > 0 вправо
	local controls
	task.spawn(function()
		local pm = lp:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 20)
		if not pm then return end
		local ok, mod = pcall(require, pm)
		if ok and mod then
			local ok2, c = pcall(function() return mod:GetControls() end)
			if ok2 then controls = c end
		end
	end)
	local function moveVec()
		if controls then
			local ok, v = pcall(function() return controls:GetMoveVector() end)
			if ok and v then return v end
		end
		local x = (UIS:IsKeyDown(Enum.KeyCode.D) and 1 or 0) - (UIS:IsKeyDown(Enum.KeyCode.A) and 1 or 0)
		local z = (UIS:IsKeyDown(Enum.KeyCode.S) and 1 or 0) - (UIS:IsKeyDown(Enum.KeyCode.W) and 1 or 0)
		return Vector3.new(x, 0, z)
	end

	local function jumpHeld()
		if UIS:IsKeyDown(Enum.KeyCode.Space) then return true end
		if jumpTouch then
			local st = jumpTouch.UserInputState
			if st == Enum.UserInputState.Begin or st == Enum.UserInputState.Change then return true end
			jumpTouch = nil
		end
		return false
	end

	local wh = { speed = 16, lastDir = nil, lastSwitch = 0 }
	local touched = false

	RunService.Heartbeat:Connect(function(dt)
		local char = lp.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not hum or hum.Health <= 0 then return end
		if Move.arBoost then return end

		local target
		if Move.mode then
			local cfg = (Move.mode == "spam") and Move.spam or Move.ad
			local mn, mx = cfg.min, cfg.max
			-- стоишь на месте (нет ввода движения) или не держишь прыжок -> скорость сразу сбрасывается
			if jumpHeld() and moveVec().Magnitude >= 0.1 then
				if Move.mode == "spam" then
					-- держишь прыжок -> плавно разгоняешься от min до max (~1 сек)
					wh.speed = math.min(mx, math.max(wh.speed, mn) + (mx - mn) * dt)
				else
					-- A-D: каждая смена A <-> D даёт прирост скорости, без смен скорость падает
					-- влево/вправо по вводу (стик или A/D), порог чтобы не ловить дрожание стика
					local x = moveVec().X
					local dir = (x < -0.35) and "A" or ((x > 0.35) and "D" or nil)
					if dir and dir ~= wh.lastDir then
						wh.lastDir = dir; wh.lastSwitch = os.clock()
						wh.speed = math.min(mx, math.max(wh.speed, mn) + (mx - mn) * 0.2)
					end
					if os.clock() - wh.lastSwitch > 0.35 then
						wh.speed = math.max(mn, wh.speed - (mx - mn) * 1.5 * dt)
					end
				end
				if hum.FloorMaterial ~= Enum.Material.Air then hum.Jump = true end
				target = wh.speed
			else
				wh.speed = mn; wh.lastDir = nil
				target = Move.normalOn and Move.normal or mn
			end
		elseif Move.normalOn then
			target = Move.normal
		end

		if target then
			if hum.WalkSpeed ~= target then hum.WalkSpeed = target end
			touched = true
		elseif touched then
			hum.WalkSpeed = 16; touched = false
		end
	end)

	-- A-D режим: пока держишь прыжок и двигаешь влево/вправо (стик или A/D), персонаж всё равно идёт ПРЯМО вперёд (как в бхопе)
	RunService:BindToRenderStep("VisionWallhopForward", Enum.RenderPriority.Last.Value, function()
		if Move.mode ~= "ad" or Move.arBoost or not jumpHeld() then return end
		local char = lp.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not hum or hum.Health <= 0 then return end
		if math.abs(moveVec().X) > 0.35 then
			hum:Move(Vector3.new(0, 0, -1), true)
		end
	end)

	local setSpam, setAD
	setSpam = addToggle(combatTab, "Wallhop (Spam Jump)", false, function(on)
		if on then
			Move.mode = "spam"; wh.speed = Move.spam.min
			if setAD then setAD(false) end
		elseif Move.mode == "spam" then
			Move.mode = nil
		end
	end)
	addRangeRow(combatTab, "Speed  min - max", 16, 60, function(a, b)
		Move.spam.min, Move.spam.max = a, b
	end)

	setAD = addToggle(combatTab, "Wallhop (A-D)", false, function(on)
		if on then
			Move.mode = "ad"; wh.speed = Move.ad.min; wh.lastDir = nil
			if setSpam then setSpam(false) end
		elseif Move.mode == "ad" then
			Move.mode = nil
		end
	end)
	addRangeRow(combatTab, "Speed  min - max", 16, 60, function(a, b)
		Move.ad.min, Move.ad.max = a, b
	end)
end

-- Anti Ragdoll (логика V1/V2 из DemonDuels)
do
	local BOOST_SPEED, DEFAULT_SPEED = 400, 16
	local v1 = { active = false, boosting = false, conns = {} }
	local v2 = { conn = nil, cooldown = 0 }

	local function ragdolled(hum)
		local st = hum:GetState()
		return st == Enum.HumanoidStateType.Physics
			or st == Enum.HumanoidStateType.Ragdoll
			or st == Enum.HumanoidStateType.FallingDown
	end

	local function v1Stop()
		v1.active = false
		local hum = lp.Character and lp.Character:FindFirstChildOfClass("Humanoid")
		if v1.boosting and hum then hum.WalkSpeed = DEFAULT_SPEED end
		v1.boosting = false; Move.arBoost = false
		for _, c in ipairs(v1.conns) do pcall(function() c:Disconnect() end) end
		v1.conns = {}
	end

	local function v1Start()
		if v1.active then return end
		v1.active = true
		table.insert(v1.conns, RunService.RenderStepped:Connect(function()
			local hum = lp.Character and lp.Character:FindFirstChildOfClass("Humanoid")
			if hum then cam.CameraSubject = hum end
		end))
		task.spawn(function()
			while v1.active do
				task.wait()
				local char = lp.Character
				local hum = char and char:FindFirstChildOfClass("Humanoid")
				local root = char and char:FindFirstChild("HumanoidRootPart")
				if hum and root then
					if ragdolled(hum) then
						pcall(function() lp:SetAttribute("RagdollEndTime", workspace:GetServerTimeNow()) end)
						for _, d in ipairs(char:GetDescendants()) do
							if d:IsA("BallSocketConstraint")
							or (d:IsA("Attachment") and d.Name:find("RagdollAttachment")) then
								pcall(function() d:Destroy() end)
							end
						end
						if not v1.boosting then
							v1.boosting = true; Move.arBoost = true
							hum.WalkSpeed = BOOST_SPEED
						end
						if hum.Health > 0 then hum:ChangeState(Enum.HumanoidStateType.Running) end
						root.Anchored = false
					elseif v1.boosting then
						v1.boosting = false; Move.arBoost = false
						hum.WalkSpeed = DEFAULT_SPEED
					end
				end
			end
		end)
	end

	local function v2Stop()
		if v2.conn then v2.conn:Disconnect(); v2.conn = nil end
		v2.cooldown = 0
	end

	local function v2Start()
		if v2.conn then return end
		v2.conn = RunService.Heartbeat:Connect(function()
			local char = lp.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			local root = char and char:FindFirstChild("HumanoidRootPart")
			if not hum or not root or hum.Health <= 0 then return end
			if ragdolled(hum) and tick() - v2.cooldown > 0.15 then
				v2.cooldown = tick()
				pcall(function()
					hum:ChangeState(Enum.HumanoidStateType.GettingUp)
					root.AssemblyLinearVelocity = Vector3.zero
					root.AssemblyAngularVelocity = Vector3.zero
					for _, o in ipairs(char:GetDescendants()) do
						if o:IsA("Motor6D") or o:IsA("Constraint") then o.Enabled = true end
					end
					cam.CameraSubject = hum
					local pm = lp.PlayerScripts:FindFirstChild("PlayerModule")
					if pm then
						local cm = require(pm:FindFirstChild("ControlModule"))
						if cm then cm:Enable() end
					end
					hum.AutoRotate = true; hum.PlatformStand = false; hum.Sit = false
				end)
			end
		end)
	end

	addSelector(mainTab, "Anti Ragdoll", { "Off", "V1", "V2" }, function(name)
		v1Stop(); v2Stop()
		if name == "V1" then v1Start() elseif name == "V2" then v2Start() end
	end)
end


-- Misc
local miscTab = newTab("Misc")
addToggle(miscTab, "Show UI On Execute", true)

-- Copy Job ID
addNavRow(miscTab, "Copy Job ID", "⧉", function()
	local id = game.JobId
	if id == "" then toast("JobId пустой (Studio / приватный сервер)"); return end
	local ok = pcall(function() setclipboard(id) end)
	if not ok then ok = pcall(function() toclipboard(id) end) end
	if ok then toast("Job ID скопирован") else toast("Нет доступа к буферу: " .. id) end
end)

-- GUI Size
do
	local row = Instance.new("Frame", miscTab)
	row.Size = UDim2.new(1,-6,0,36); row.BackgroundColor3 = PANEL; row.ZIndex = 5; corner(row, 8)
	local l = lbl(row, "GUI Size (%)", 13)
	l.Position = UDim2.new(0,12,0,0); l.Size = UDim2.new(1,-110,1,0); l.ZIndex = 5
	local box = Instance.new("TextBox", row)
	box.Size = UDim2.new(0,70,0,24); box.Position = UDim2.new(1,-82,0.5,-12)
	box.BackgroundColor3 = Color3.fromRGB(40,41,58); box.Text = "100"
	box.PlaceholderText = "50-200"; box.TextColor3 = ACCENT
	box.Font = Enum.Font.GothamBold; box.TextSize = 13
	box.ClearTextOnFocus = false; box.ZIndex = 6; corner(box, 6); mkStroke(box, ACCENT, 0.6)
	box.FocusLost:Connect(function()
		local n = tonumber(box.Text)
		if not n then box.Text = tostring(math.floor(mainScale.Scale*100+0.5)); return end
		n = math.clamp(n, 50, 200); box.Text = tostring(n)
		mainScale.Scale = n/100; syncBG()
	end)
end

-- Backgrounds — все 18 штук
local BACKGROUNDS = {
	"rbxassetid://101894744159774",
	"rbxassetid://136208130767349",
	"rbxassetid://126793180958099",
	"rbxassetid://105542572852370",
	"rbxassetid://128997600029394",
	"rbxassetid://99420703803809",
	"rbxassetid://117186687504218",
	"rbxassetid://123015462349687",
	"rbxassetid://111195293067618",
	"rbxassetid://133619037676439",
	"rbxassetid://116985758139639",
	"rbxassetid://132062039944824",
	"rbxassetid://139571679676131",
	"rbxassetid://70418952815837",
	"rbxassetid://98214837857183",
	"rbxassetid://126860692354524",
	"rbxassetid://87472506396981",
	"rbxassetid://77785523954153",
	"rbxassetid://122815453745063",
	"rbxassetid://77581705786454",
	"rbxassetid://86508089972764",
	"rbxassetid://99747773532595",
	"rbxassetid://139362780018712",
	"rbxassetid://81570223894096",
	"rbxassetid://94047991986188",
	"rbxassetid://118783198922289",
	"rbxassetid://101833990704596",
	"rbxassetid://109527418380162",
  "rbxassetid://119511177791737",
}

local bgPopup = Instance.new("Frame", main)
bgPopup.Size = UDim2.new(1,-16,1,-48); bgPopup.Position = UDim2.new(0,8,0,42)
bgPopup.BackgroundColor3 = BG; bgPopup.BackgroundTransparency = 0.02
bgPopup.Visible = false; bgPopup.ZIndex = 10
corner(bgPopup, 10); mkStroke(bgPopup, ACCENT, 0.45)

do
	local headLbl = lbl(bgPopup, "Background", 15, Enum.Font.GothamBold, ACCENT)
	headLbl.Position = UDim2.new(0,12,0,7); headLbl.Size = UDim2.new(1,-120,0,24); headLbl.ZIndex = 11

	local function mkBtn(text, xOff)
		local b = Instance.new("TextButton", bgPopup)
		b.Size = UDim2.new(0,52,0,24); b.Position = UDim2.new(1,xOff,0,7)
		b.BackgroundColor3 = PANEL; b.Text = text; b.TextColor3 = Color3.new(1,1,1)
		b.Font = Enum.Font.GothamBold; b.TextSize = 12; b.AutoButtonColor = false; b.ZIndex = 11
		corner(b, 6); return b
	end
	local noneBtn  = mkBtn("None", -112)
	local closeBtn = mkBtn("Back",  -56)
	closeBtn.MouseButton1Click:Connect(function() bgPopup.Visible = false end)

	local grid = Instance.new("ScrollingFrame", bgPopup)
	grid.Size = UDim2.new(1,-12,1,-40); grid.Position = UDim2.new(0,6,0,36)
	grid.BackgroundTransparency = 1; grid.BorderSizePixel = 0
	grid.ScrollBarThickness = 3; grid.ScrollBarImageColor3 = ACCENT
	grid.AutomaticCanvasSize = Enum.AutomaticSize.Y; grid.CanvasSize = UDim2.new()
	grid.ZIndex = 11
	local gl = Instance.new("UIGridLayout", grid)
	gl.CellSize = UDim2.new(0,118,0,70); gl.CellPadding = UDim2.new(0,6,0,6)
	Instance.new("UIPadding", grid).PaddingTop = UDim.new(0,4)

	local selectedBtn = nil
	local function pick(btn, id)
		if selectedBtn then
			local s = selectedBtn:FindFirstChildOfClass("UIStroke")
			if s then s.Transparency = 0.75; s.Thickness = 1 end
		end
		bgImage.Image = id or ""
		selectedBtn = btn
		if btn then
			local s = btn:FindFirstChildOfClass("UIStroke")
			if s then s.Transparency = 0; s.Thickness = 2.5 end
		end
	end
	noneBtn.MouseButton1Click:Connect(function() pick(nil, "") end)

	for i, id in ipairs(BACKGROUNDS) do
		local b = Instance.new("ImageButton", grid)
		b.Name = "BG"..i; b.BackgroundColor3 = Color3.fromRGB(30,31,44)
		b.Image = id; b.ScaleType = Enum.ScaleType.Crop
		b.AutoButtonColor = false; b.ZIndex = 12
		corner(b, 8); mkStroke(b, ACCENT, 0.75)
		b.MouseButton1Click:Connect(function() pick(b, id) end)
	end

	task.spawn(function()
		pcall(function() game:GetService("ContentProvider"):PreloadAsync(BACKGROUNDS) end)
	end)
end

-- кнопка Background в Misc
addNavRow(miscTab, "Background", "›", function() bgPopup.Visible = true end)

syncBG()
selectTab("Visuals")
