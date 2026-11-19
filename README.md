local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local RS = game:GetService("ReplicatedStorage")

local CONFIG = {
	ADMINS = { [123456789] = true }, -- troque pelo seu UserId
	STUDIO_IS_ADMIN = true,          -- no Studio, todo mundo é admin (só para testes)
	DATASTORE = "PlayerInventory_v1",
	COOLDOWN = 1,
	TOGGLE_KEY = Enum.KeyCode.RightControl,
	THEME = {
		bg = Color3.fromRGB(22, 24, 32),
		panel = Color3.fromRGB(32, 35, 46),
		accent = Color3.fromRGB(88, 101, 242),
		text = Color3.fromRGB(235, 237, 245),
	},
	CATALOG = {
		{ Id = "Pistola", Name = "Pistola" },
		{ Id = "Rifle", Name = "Rifle" },
		{ Id = "Faca", Name = "Faca" },
		{ Id = "Escopeta", Name = "Escopeta" },
	},
}

local M = {}

local valid = {}
for _, w in ipairs(CONFIG.CATALOG) do valid[w.Id] = true end

---------------------------------------------------------------- SERVIDOR
function M.Server()
	local DSS = game:GetService("DataStoreService")
	local SS = game:GetService("ServerStorage")
	local store = DSS:GetDataStore(CONFIG.DATASTORE)

	local remotes = Instance.new("Folder")
	remotes.Name = "ModMenuRemotes"
	local give = Instance.new("RemoteEvent", remotes); give.Name = "GiveWeapon"
	local updated = Instance.new("RemoteEvent", remotes); updated.Name = "Updated"
	local getInfo = Instance.new("RemoteFunction", remotes); getInfo.Name = "GetInfo"
	remotes.Parent = RS

	local inventories, loaded, lastUse = {}, {}, {}

	local function isAdmin(p)
		if CONFIG.ADMINS[p.UserId] then return true end
		if game.CreatorType == Enum.CreatorType.User and p.UserId == game.CreatorId then return true end
		return CONFIG.STUDIO_IS_ADMIN and RunService:IsStudio()
	end

	local function list(p)
		local out = {}
		for id in pairs(inventories[p] or {}) do table.insert(out, id) end
		table.sort(out)
		return out
	end

	local function getTemplate(id)
		local folder = SS:FindFirstChild("Weapons")
		local t = folder and folder:FindFirstChild(id)
		if t then return t end
		local tool = Instance.new("Tool")
		tool.Name = id
		tool.ToolTip = id
		local h = Instance.new("Part")
		h.Name = "Handle"
		h.Size = Vector3.new(0.6, 1, 3)
		h.Color = Color3.fromRGB(60, 60, 70)
		h.Parent = tool
		return tool
	end

	local function equip(p, id)
		local backpack = p:FindFirstChildOfClass("Backpack")
		if not backpack then return end
		if backpack:FindFirstChild(id) or (p.Character and p.Character:FindFirstChild(id)) then return end
		getTemplate(id):Clone().Parent = backpack
	end

	local function save(p)
		if not loaded[p] then return end -- nunca sobrescreve se o carregamento falhou
		local ok, err = pcall(function() store:SetAsync(tostring(p.UserId), list(p)) end)
		if not ok then warn("Falha ao salvar:", err) end
	end

	Players.PlayerAdded:Connect(function(p)
		inventories[p] = {}
		local ok, saved = pcall(function() return store:GetAsync(tostring(p.UserId)) end)
		if ok then
			loaded[p] = true
			if type(saved) == "table" then
				for _, id in ipairs(saved) do
					if valid[id] then inventories[p][id] = true end
				end
			end
		else
			warn("Falha ao carregar:", saved)
		end
		p.CharacterAdded:Connect(function()
			task.wait(0.5)
			for id in pairs(inventories[p] or {}) do equip(p, id) end
		end)
		if p.Character then
			for id in pairs(inventories[p]) do equip(p, id) end
		end
	end)

	Players.PlayerRemoving:Connect(function(p)
		save(p)
		inventories[p], loaded[p], lastUse[p] = nil, nil, nil
	end)

	game:BindToClose(function()
		for _, p in ipairs(Players:GetPlayers()) do save(p) end
	end)

	give.OnServerEvent:Connect(function(p, targetName, weaponId)
		if not isAdmin(p) then return end
		if type(targetName) ~= "string" or type(weaponId) ~= "string" then return end
		if #targetName > 20 or not valid[weaponId] then return end
		local now = os.clock()
		if now - (lastUse[p] or 0) < CONFIG.COOLDOWN then return end
		lastUse[p] = now

		local target = Players:FindFirstChild(targetName)
		if not target or not inventories[target] then return end
		inventories[target][weaponId] = true
		equip(target, weaponId)
		save(target)
		updated:FireClient(target)
	end)

	getInfo.OnServerInvoke = function(p)
		return { items = list(p), admin = isAdmin(p) }
	end
end

---------------------------------------------------------------- CLIENTE
function M.Client()
	local TS = game:GetService("TweenService")
	local UIS = game:GetService("UserInputService")
	local player = Players.LocalPlayer
	local remotes = RS:WaitForChild("ModMenuRemotes")
	local T = CONFIG.THEME
	local TWEEN = TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

	local function make(class, props, parent)
		local o = Instance.new(class)
		for k, v in pairs(props) do o[k] = v end
		o.Parent = parent
		return o
	end
	local function round(o, r) make("UICorner", { CornerRadius = UDim.new(0, r or 8) }, o) end

	local gui = make("ScreenGui", { Name = "ModMenu", ResetOnSpawn = false }, player:WaitForChild("PlayerGui"))
	local FULL, MINI = UDim2.fromOffset(460, 320), UDim2.fromOffset(460, 40)
	local main = make("Frame", {
		Size = FULL, Position = UDim2.fromScale(0.5, 0.5), AnchorPoint = Vector2.new(0.5, 0.5),
		BackgroundColor3 = T.bg, ClipsDescendants = true, Active = true, Draggable = true,
	}, gui)
	round(main, 12)
	local scale = make("UIScale", { Scale = 0.9 }, main)
	TS:Create(scale, TWEEN, { Scale = 1 }):Play()

	local top = make("Frame", { Size = UDim2.new(1, 0, 0, 40), BackgroundColor3 = T.panel }, main)
	make("TextLabel", {
		Size = UDim2.new(1, -90, 1, 0), Position = UDim2.fromOffset(12, 0), BackgroundTransparency = 1,
		Text = "Menu de Inventário", TextXAlignment = Enum.TextXAlignment.Left,
		Font = Enum.Font.GothamBold, TextSize = 16, TextColor3 = T.text,
	}, top)
	local function topBtn(txt, x)
		local b = make("TextButton", {
			Size = UDim2.fromOffset(28, 28), Position = UDim2.new(1, x, 0, 6), BackgroundColor3 = T.accent,
			Text = txt, Font = Enum.Font.GothamBold, TextSize = 16, TextColor3 = T.text,
		}, top)
		round(b, 6)
		return b
	end
	local closeBtn, miniBtn = topBtn("X", -36), topBtn("-", -70)

	local content = make("Frame", {
		Size = UDim2.new(1, 0, 1, -40), Position = UDim2.fromOffset(0, 40), BackgroundTransparency = 1,
	}, main)
	local tabs = make("Frame", {
		Size = UDim2.new(1, -20, 0, 32), Position = UDim2.fromOffset(10, 8), BackgroundTransparency = 1,
	}, content)
	make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 8) }, tabs)

	local pages, tabBtns = {}, {}
	local function select(name)
		for n, pg in pairs(pages) do pg.Visible = (n == name) end
		for n, b in pairs(tabBtns) do
			TS:Create(b, TWEEN, { BackgroundColor3 = (n == name) and T.accent or T.panel }):Play()
		end
	end
	local function newPage(name, onOpen)
		local pg = make("Frame", {
			Size = UDim2.new(1, -20, 1, -56), Position = UDim2.fromOffset(10, 48),
			BackgroundTransparency = 1, Visible = false,
		}, content)
		pages[name] = pg
		local b = make("TextButton", {
			Size = UDim2.fromOffset(110, 32), BackgroundColor3 = T.panel, Text = name,
			Font = Enum.Font.GothamMedium, TextSize = 14, TextColor3 = T.text,
		}, tabs)
		round(b, 6)
		tabBtns[name] = b
		b.MouseButton1Click:Connect(function()
			select(name)
			if onOpen then onOpen() end
		end)
		return pg
	end

	local ok, info = pcall(function() return remotes.GetInfo:InvokeServer() end)
	info = ok and info or { items = {}, admin = false }

	-- Aba Armas (só aparece para admin; o servidor revalida)
	if info.admin then
		local armas = newPage("Armas")
		local target = make("TextBox", {
			Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = T.panel, Text = player.Name,
			PlaceholderText = "Nome do jogador", Font = Enum.Font.Gotham, TextSize = 14,
			TextColor3 = T.text, ClearTextOnFocus = false,
		}, armas)
		round(target, 6)
		local grid = make("ScrollingFrame", {
			Size = UDim2.new(1, 0, 1, -40), Position = UDim2.fromOffset(0, 40), BackgroundTransparency = 1,
			CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarThickness = 4,
		}, armas)
		make("UIGridLayout", { CellSize = UDim2.fromOffset(130, 44), CellPadding = UDim2.fromOffset(8, 8) }, grid)
		for _, w in ipairs(CONFIG.CATALOG) do
			local b = make("TextButton", {
				BackgroundColor3 = T.panel, Text = w.Name, Font = Enum.Font.GothamMedium,
				TextSize = 14, TextColor3 = T.text,
			}, grid)
			round(b, 8)
			b.MouseButton1Click:Connect(function()
				remotes.GiveWeapon:FireServer(target.Text, w.Id)
			end)
		end
	end

	-- Aba Inventário
	local invList
	local function refresh()
		local ok2, data = pcall(function() return remotes.GetInfo:InvokeServer() end)
		if not ok2 or not data then return end
		for _, c in ipairs(invList:GetChildren()) do
			if c:IsA("TextLabel") then c:Destroy() end
		end
		for _, id in ipairs(data.items) do
			local l = make("TextLabel", {
				Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = T.panel, Text = "  " .. id,
				TextXAlignment = Enum.TextXAlignment.Left, Font = Enum.Font.Gotham,
				TextSize = 14, TextColor3 = T.text,
			}, invList)
			round(l, 6)
		end
	end
	local inv = newPage("Inventário", refresh)
	invList = make("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarThickness = 4,
	}, inv)
	make("UIListLayout", { Padding = UDim.new(0, 6) }, invList)
	remotes.Updated.OnClientEvent:Connect(refresh)

	if info.admin then select("Armas") else select("Inventário"); refresh() end

	-- Minimizar, fechar e atalho
	local minimized = false
	miniBtn.MouseButton1Click:Connect(function()
		minimized = not minimized
		TS:Create(main, TWEEN, { Size = minimized and MINI or FULL }):Play()
	end)
	closeBtn.MouseButton1Click:Connect(function() gui.Enabled = false end)
	UIS.InputBegan:Connect(function(input, processed)
		if not processed and input.KeyCode == CONFIG.TOGGLE_KEY then gui.Enabled = not gui.Enabled end
	end)
end

return M
