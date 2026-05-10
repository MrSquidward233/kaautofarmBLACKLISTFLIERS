local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local localPlayer = Players.LocalPlayer
local playerGui = localPlayer:WaitForChild("PlayerGui")
local Live = workspace:WaitForChild("Live")
local spawnRemote = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("FireServer")

local DEFAULT_TARGET_NAME = ""
local STUDS_UNDER = -1
local TWEEN_TIME = 0.01
local KEY_SPAM_DELAY = 0.1
local FOLLOW_DELAY = 0.01
local MAX_RESPAWNS_PER_TARGET = 15
local TARGET_PRIORITY_OPTIONS = {
	"Lowest Level",
	"Highest Level",
	"Random",
	"Closest",
}
local BLACKLISTED_KAIJU = {
	["Space Godzilla"] = true,
	["Monster Zero"] = true,
	["Showa Rodan"] = true,
	["Male Muto"] = true,
	-- Add more here:
	-- ["Kaiju Name"] = true,
}

local enabled = false
local keyLoopRunning = false
local followLoopRunning = false
local respawnDebounce = false
local sessionStartTime = 0

local selectedTargetName = DEFAULT_TARGET_NAME
local currentTargetPlayer = nil
local playerButtons = {}
local targetTypeButtons = {}
local disconnectCellsTracker
local selectedTargetPriority = TARGET_PRIORITY_OPTIONS[1]

local cellsSessionTotal = 0
local cellsBaseline = 0
local cellsValueObject = nil
local cellsChangedConn = nil
local leaderstatsConn = nil

local respawnAttemptsByTarget = {}
local ignoredTargets = {}

local THEME_BLACK = Color3.fromRGB(10, 10, 12)
local THEME_PANEL = Color3.fromRGB(18, 18, 22)
local THEME_PANEL_ALT = Color3.fromRGB(26, 26, 31)
local THEME_RED = Color3.fromRGB(210, 34, 42)
local THEME_RED_DARK = Color3.fromRGB(120, 18, 24)
local THEME_WHITE = Color3.fromRGB(245, 245, 245)
local THEME_MUTED = Color3.fromRGB(170, 170, 176)
local THEME_OUTLINE = Color3.fromRGB(95, 20, 26)
local EXPECTED_TITLE_TEXT = "<font color=\"#D2222A\">Kaiju</font> <font color=\"#FFFFFF\">Alpha</font>"
local EXPECTED_CREDIT_TEXT = "made with love by studly <3"
local SIDE_DROPDOWN_WIDTH = 180

-- UI
local gui = Instance.new("ScreenGui")
gui.Name = "UnderPlayerGui"
gui.ResetOnSpawn = false
gui.Parent = playerGui

local shadow = Instance.new("Frame")
shadow.Size = UDim2.new(0, 320, 0, 400)
shadow.Position = UDim2.new(0, 25, 0, 25)
shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
shadow.BackgroundTransparency = 0.35
shadow.BorderSizePixel = 0
shadow.ZIndex = 0
shadow.Parent = gui

local shadowCorner = Instance.new("UICorner")
shadowCorner.CornerRadius = UDim.new(0, 22)
shadowCorner.Parent = shadow

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 320, 0, 400)
frame.Position = UDim2.new(0, 20, 0, 20)
frame.BackgroundColor3 = THEME_BLACK
frame.BorderSizePixel = 0
frame.Parent = gui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 20)
frameCorner.Parent = frame

local frameStroke = Instance.new("UIStroke")
frameStroke.Color = THEME_OUTLINE
frameStroke.Thickness = 2
frameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
frameStroke.Parent = frame

local header = Instance.new("Frame")
header.Size = UDim2.new(1, -24, 0, 58)
header.Position = UDim2.new(0, 12, 0, 10)
header.BackgroundColor3 = THEME_PANEL
header.BorderSizePixel = 0
header.Parent = frame

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 16)
headerCorner.Parent = header

local headerStroke = Instance.new("UIStroke")
headerStroke.Color = THEME_OUTLINE
headerStroke.Thickness = 1.5
headerStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
headerStroke.Parent = header

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -18, 0, 26)
title.Position = UDim2.new(0, 10, 0, 7)
title.BackgroundTransparency = 1
title.RichText = true
title.Font = Enum.Font.GothamBold
title.TextSize = 20
title.TextXAlignment = Enum.TextXAlignment.Left
title.Text = EXPECTED_TITLE_TEXT
title.TextColor3 = THEME_WHITE
title.Parent = header

local creditLabel = Instance.new("TextLabel")
creditLabel.Size = UDim2.new(1, -18, 0, 18)
creditLabel.Position = UDim2.new(0, 10, 0, 32)
creditLabel.BackgroundTransparency = 1
creditLabel.Font = Enum.Font.GothamSemibold
creditLabel.TextSize = 11
creditLabel.TextXAlignment = Enum.TextXAlignment.Left
creditLabel.TextColor3 = THEME_MUTED
creditLabel.Text = EXPECTED_CREDIT_TEXT
creditLabel.Parent = header

local toggleButton = Instance.new("TextButton")
toggleButton.Size = UDim2.new(1, -24, 0, 40)
toggleButton.Position = UDim2.new(0, 12, 0, 78)
toggleButton.BackgroundColor3 = THEME_RED_DARK
toggleButton.TextColor3 = THEME_WHITE
toggleButton.Font = Enum.Font.GothamBold
toggleButton.TextSize = 16
toggleButton.Text = "OFF"
toggleButton.Parent = frame

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 12)
toggleCorner.Parent = toggleButton

local toggleStroke = Instance.new("UIStroke")
toggleStroke.Color = THEME_RED
toggleStroke.Thickness = 1.5
toggleStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
toggleStroke.Parent = toggleButton

local amountBox = Instance.new("TextBox")
amountBox.Size = UDim2.new(1, -24, 0, 36)
amountBox.Position = UDim2.new(0, 12, 0, 128)
amountBox.BackgroundColor3 = THEME_PANEL
amountBox.TextColor3 = THEME_WHITE
amountBox.PlaceholderColor3 = THEME_MUTED
amountBox.Font = Enum.Font.GothamSemibold
amountBox.TextSize = 15
amountBox.PlaceholderText = "Studs under target"
amountBox.Text = tostring(STUDS_UNDER)
amountBox.Parent = frame

local amountCorner = Instance.new("UICorner")
amountCorner.CornerRadius = UDim.new(0, 12)
amountCorner.Parent = amountBox

local amountStroke = Instance.new("UIStroke")
amountStroke.Color = THEME_OUTLINE
amountStroke.Thickness = 1.5
amountStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
amountStroke.Parent = amountBox

local targetTypeButton = Instance.new("TextButton")
targetTypeButton.Size = UDim2.new(1, -24, 0, 36)
targetTypeButton.Position = UDim2.new(0, 12, 0, 172)
targetTypeButton.BackgroundColor3 = THEME_PANEL_ALT
targetTypeButton.TextColor3 = THEME_WHITE
targetTypeButton.Font = Enum.Font.GothamSemibold
targetTypeButton.TextSize = 15
targetTypeButton.Text = "Target Type: " .. selectedTargetPriority
targetTypeButton.Parent = frame

local targetTypeCorner = Instance.new("UICorner")
targetTypeCorner.CornerRadius = UDim.new(0, 12)
targetTypeCorner.Parent = targetTypeButton

local targetTypeStroke = Instance.new("UIStroke")
targetTypeStroke.Color = THEME_OUTLINE
targetTypeStroke.Thickness = 1.5
targetTypeStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
targetTypeStroke.Parent = targetTypeButton

local targetTypeFrame = Instance.new("ScrollingFrame")
targetTypeFrame.Size = UDim2.new(0, 0, 0, 120)
targetTypeFrame.Position = UDim2.new(0, -176, 0, 172)
targetTypeFrame.BackgroundColor3 = THEME_PANEL
targetTypeFrame.BorderSizePixel = 0
targetTypeFrame.ScrollBarThickness = 4
targetTypeFrame.ScrollBarImageColor3 = THEME_RED
targetTypeFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
targetTypeFrame.Visible = false
targetTypeFrame.Parent = frame

local targetTypeFrameCorner = Instance.new("UICorner")
targetTypeFrameCorner.CornerRadius = UDim.new(0, 12)
targetTypeFrameCorner.Parent = targetTypeFrame

local targetTypeFrameStroke = Instance.new("UIStroke")
targetTypeFrameStroke.Color = THEME_OUTLINE
targetTypeFrameStroke.Thickness = 1.5
targetTypeFrameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
targetTypeFrameStroke.Parent = targetTypeFrame

local targetTypeListLayout = Instance.new("UIListLayout")
targetTypeListLayout.Padding = UDim.new(0, 4)
targetTypeListLayout.Parent = targetTypeFrame

local dropdownButton = Instance.new("TextButton")
dropdownButton.Size = UDim2.new(1, -24, 0, 36)
dropdownButton.Position = UDim2.new(0, 12, 0, 258)
dropdownButton.BackgroundColor3 = THEME_PANEL_ALT
dropdownButton.TextColor3 = THEME_WHITE
dropdownButton.Font = Enum.Font.GothamSemibold
dropdownButton.TextSize = 15
dropdownButton.Text = "Select Player"
dropdownButton.Parent = frame

local dropdownCorner = Instance.new("UICorner")
dropdownCorner.CornerRadius = UDim.new(0, 12)
dropdownCorner.Parent = dropdownButton

local dropdownStroke = Instance.new("UIStroke")
dropdownStroke.Color = THEME_OUTLINE
dropdownStroke.Thickness = 1.5
dropdownStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
dropdownStroke.Parent = dropdownButton

local dropdownFrame = Instance.new("ScrollingFrame")
dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
dropdownFrame.Position = UDim2.new(0, -176, 0, 258)
dropdownFrame.BackgroundColor3 = THEME_PANEL
dropdownFrame.BorderSizePixel = 0
dropdownFrame.ScrollBarThickness = 4
dropdownFrame.ScrollBarImageColor3 = THEME_RED
dropdownFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
dropdownFrame.Visible = false
dropdownFrame.Parent = frame

local dropdownFrameCorner = Instance.new("UICorner")
dropdownFrameCorner.CornerRadius = UDim.new(0, 12)
dropdownFrameCorner.Parent = dropdownFrame

local dropdownFrameStroke = Instance.new("UIStroke")
dropdownFrameStroke.Color = THEME_OUTLINE
dropdownFrameStroke.Thickness = 1.5
dropdownFrameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
dropdownFrameStroke.Parent = dropdownFrame

local listLayout = Instance.new("UIListLayout")
listLayout.Padding = UDim.new(0, 4)
listLayout.Parent = dropdownFrame

local sessionTimeLabel = Instance.new("TextLabel")
sessionTimeLabel.Size = UDim2.new(1, -24, 0, 20)
sessionTimeLabel.Position = UDim2.new(0, 12, 1, -70)
sessionTimeLabel.BackgroundTransparency = 1
sessionTimeLabel.TextColor3 = THEME_MUTED
sessionTimeLabel.Font = Enum.Font.GothamMedium
sessionTimeLabel.TextSize = 14
sessionTimeLabel.TextXAlignment = Enum.TextXAlignment.Left
sessionTimeLabel.Text = "Session Time: 0s"
sessionTimeLabel.Parent = frame

local cellsLabel = Instance.new("TextLabel")
cellsLabel.Size = UDim2.new(1, -24, 0, 20)
cellsLabel.Position = UDim2.new(0, 12, 1, -46)
cellsLabel.BackgroundTransparency = 1
cellsLabel.TextColor3 = THEME_WHITE
cellsLabel.Font = Enum.Font.GothamBold
cellsLabel.TextSize = 14
cellsLabel.TextXAlignment = Enum.TextXAlignment.Left
cellsLabel.Text = "Cells: 0"
cellsLabel.Parent = frame

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, -24, 0, 20)
statusLabel.Position = UDim2.new(0, 12, 1, -22)
statusLabel.BackgroundTransparency = 1
statusLabel.TextColor3 = THEME_RED
statusLabel.Font = Enum.Font.GothamSemibold
statusLabel.TextSize = 14
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Text = "Target: none"
statusLabel.Parent = frame

local dragging = false
local dragStart
local startPos

-- Formatting
local function formatWithCommas(number)
	local formatted = tostring(math.floor(number))
	while true do
		local updated, count = formatted:gsub("^(-?%d+)(%d%d%d)", "%1,%2")
		formatted = updated
		if count == 0 then
			break
		end
	end
	return formatted
end

local function formatSessionTime(totalSeconds)
	local seconds = math.max(0, math.floor(totalSeconds))
	local minutes = math.floor(seconds / 60)
	local remainingSeconds = seconds % 60

	if minutes > 0 then
		return string.format("%dm %ds", minutes, remainingSeconds)
	end

	return string.format("%ds", remainingSeconds)
end

local function parseCellsValue(rawValue)
	if typeof(rawValue) == "number" then
		return rawValue
	end

	if typeof(rawValue) == "string" then
		local cleanedValue = rawValue:gsub(",", "")
		return tonumber(cleanedValue) or 0
	end

	return tonumber(rawValue) or 0
end

-- Leaderstats
local function getLeaderstats()
	return localPlayer:FindFirstChild("leaderstats")
end

local function getCurrentKaiju()
	local leaderstats = getLeaderstats()
	if not leaderstats then
		return nil
	end

	local kaijuValue = leaderstats:FindFirstChild("Kaiju")
	if not kaijuValue then
		return nil
	end

	return kaijuValue.Value
end

local function getAttackKeycodes()
	local keycodes = {
		Enum.KeyCode.One,
		Enum.KeyCode.Two,
		Enum.KeyCode.Three,
	}

	local currentKaiju = getCurrentKaiju()
	if currentKaiju == "Kong 2017" then
		table.insert(keycodes, Enum.KeyCode.Four)
		table.insert(keycodes, Enum.KeyCode.Five)
		return keycodes
	end

	if currentKaiju == "Kong 2021" then
		return {
			Enum.KeyCode.One,
			Enum.KeyCode.Two,
			Enum.KeyCode.Four,
			Enum.KeyCode.Five,
		}
	end

	if currentKaiju == "Godzilla Minus One" then
		local character = localPlayer.Character
		local humanoid = character and character:FindFirstChildOfClass("Humanoid")
		if humanoid and humanoid.MaxHealth > 0 and (humanoid.Health / humanoid.MaxHealth) <= 0.3 then
			table.insert(keycodes, Enum.KeyCode.Four)
			table.insert(keycodes, Enum.KeyCode.Four)
			table.insert(keycodes, Enum.KeyCode.Four)
		end
	end

	if currentKaiju == "Shin Godzilla" then
		table.insert(keycodes, Enum.KeyCode.Five)
	end

	return keycodes
end

local function getCellsValueObject()
	local leaderstats = getLeaderstats()
	if not leaderstats then
		return nil
	end

	return leaderstats:FindFirstChild("Cells")
end

local function isBrandingValid()
	return title.Text == EXPECTED_TITLE_TEXT and creditLabel.Text == EXPECTED_CREDIT_TEXT
end

local function disableAutofarm()
	enabled = false
	currentTargetPlayer = nil
	respawnAttemptsByTarget = {}
	ignoredTargets = {}
	sessionStartTime = 0
	disconnectCellsTracker()
	cellsSessionTotal = 0
	updateCellsLabel()
	updateSessionTimeLabel()
	toggleButton.Text = "LOCKED"
	toggleButton.BackgroundColor3 = THEME_RED_DARK
	statusLabel.Text = "Target: unavailable"
	dropdownFrame.Visible = false
	dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
	targetTypeFrame.Visible = false
	targetTypeFrame.Size = UDim2.new(0, 0, 0, 120)
end

local function ensureBranding()
	if isBrandingValid() then
		return true
	end

	disableAutofarm()
	gui.Enabled = false
	shadow.Visible = false
	return false
end

local function updateCellsLabel()
	cellsLabel.Text = "Cells: " .. formatWithCommas(cellsSessionTotal)
end

local function updateSessionTimeLabel()
	if enabled and sessionStartTime > 0 then
		sessionTimeLabel.Text = "Session Time: " .. formatSessionTime(os.clock() - sessionStartTime)
	else
		sessionTimeLabel.Text = "Session Time: 0s"
	end
end

disconnectCellsTracker = function()
	if cellsChangedConn then
		cellsChangedConn:Disconnect()
		cellsChangedConn = nil
	end

	if leaderstatsConn then
		leaderstatsConn:Disconnect()
		leaderstatsConn = nil
	end

	cellsValueObject = nil
end

local function bindCellsTracker()
	if cellsChangedConn then
		cellsChangedConn:Disconnect()
		cellsChangedConn = nil
	end

	cellsValueObject = getCellsValueObject()
	if not cellsValueObject then
		return
	end

	cellsBaseline = parseCellsValue(cellsValueObject.Value)
	cellsChangedConn = cellsValueObject:GetPropertyChangedSignal("Value"):Connect(function()
		local currentCells = parseCellsValue(cellsValueObject.Value)
		local gained = currentCells - cellsBaseline

		if enabled and gained > 0 then
			cellsSessionTotal += gained
		end

		cellsBaseline = currentCells
		updateCellsLabel()
	end)
end

local function startCellsTracker()
	cellsSessionTotal = 0
	updateCellsLabel()
	bindCellsTracker()

	local leaderstats = getLeaderstats()
	if not leaderstats then
		return
	end

	if leaderstatsConn then
		leaderstatsConn:Disconnect()
	end

	leaderstatsConn = leaderstats.ChildAdded:Connect(function(child)
		if child.Name == "Cells" and enabled then
			bindCellsTracker()
		end
	end)
end

-- Targeting
local function isAlive(player)
	local character = player and player.Character
	local humanoid = character and character:FindFirstChildOfClass("Humanoid")
	local hrp = character and character:FindFirstChild("HumanoidRootPart")
	return humanoid and humanoid.Health > 0 and hrp
end

local function clearIgnoredTargets()
	for name in pairs(ignoredTargets) do
		local player = Players:FindFirstChild(name)
		if not player or not isAlive(player) then
			ignoredTargets[name] = nil
			respawnAttemptsByTarget[name] = nil
		end
	end
end

local function getLevelLabelFromLiveModel(model)
	local overhead = model and model:FindFirstChild("Overhead")
	if not overhead then
		return nil
	end

	local outerFrame = overhead:FindFirstChildOfClass("Frame")
	if not outerFrame then
		return nil
	end

	local levelFrame = outerFrame:FindFirstChild("Level")
	if not levelFrame then
		return nil
	end

	return levelFrame:FindFirstChildOfClass("TextLabel")
end

local function getPlayerLevel(player)
	local liveModel = Live:FindFirstChild(player.Name)
	if not liveModel then
		return nil
	end

	local levelLabel = getLevelLabelFromLiveModel(liveModel)
	if not levelLabel then
		return nil
	end

	return tonumber(string.match(levelLabel.Text, "%d+"))
end

local function getPlayerKaiju(player)
	local leaderstats = player and player:FindFirstChild("leaderstats")
	local kaijuValue = leaderstats and leaderstats:FindFirstChild("Kaiju")
	return kaijuValue and kaijuValue.Value or nil
end

local function isBlacklistedKaiju(player)
	local kaijuName = getPlayerKaiju(player)
	return kaijuName ~= nil and BLACKLISTED_KAIJU[kaijuName] == true
end

local function isEligibleTarget(player)
	return player and not ignoredTargets[player.Name] and not isBlacklistedKaiju(player) and isAlive(player)
end

local function getClosestLivingPlayer()
	local myCharacter = localPlayer.Character
	local myHrp = myCharacter and myCharacter:FindFirstChild("HumanoidRootPart")
	if not myHrp then
		return nil
	end

	local closestPlayer = nil
	local closestDistance = math.huge

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer and isEligibleTarget(player) then
			local targetHrp = player.Character.HumanoidRootPart
			local distance = (myHrp.Position - targetHrp.Position).Magnitude
			if distance < closestDistance then
				closestDistance = distance
				closestPlayer = player
			end
		end
	end

	return closestPlayer
end

local function getRandomLivingPlayer()
	local eligiblePlayers = {}

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer and isEligibleTarget(player) then
			table.insert(eligiblePlayers, player)
		end
	end

	if #eligiblePlayers == 0 then
		return nil
	end

	return eligiblePlayers[math.random(1, #eligiblePlayers)]
end

local function getSortedLivingPlayersByLevel()
	local rankedPlayers = {}

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer and isEligibleTarget(player) then
			local level = getPlayerLevel(player)
			if level then
				table.insert(rankedPlayers, {
					player = player,
					level = level,
				})
			end
		end
	end

	table.sort(rankedPlayers, function(a, b)
		if a.level == b.level then
			return a.player.Name < b.player.Name
		end
		return a.level < b.level
	end)

	return rankedPlayers
end

local function getNextLowestLevelLivingPlayer()
	local rankedPlayers = getSortedLivingPlayersByLevel()
	return rankedPlayers[1] and rankedPlayers[1].player or nil
end

local function getNextHighestLevelLivingPlayer()
	local rankedPlayers = getSortedLivingPlayersByLevel()
	return rankedPlayers[#rankedPlayers] and rankedPlayers[#rankedPlayers].player or nil
end

local function getPriorityTarget()
	if selectedTargetPriority == "Highest Level" then
		return getNextHighestLevelLivingPlayer() or getClosestLivingPlayer()
	end

	if selectedTargetPriority == "Random" then
		return getRandomLivingPlayer() or getClosestLivingPlayer()
	end

	if selectedTargetPriority == "Closest" then
		return getClosestLivingPlayer() or getNextLowestLevelLivingPlayer()
	end

	return getNextLowestLevelLivingPlayer() or getClosestLivingPlayer()
end

local function getSelectedPlayer()
	if selectedTargetName == "" then
		return nil
	end

	local direct = Players:FindFirstChild(selectedTargetName)
	if direct then
		return direct
	end

	for _, player in ipairs(Players:GetPlayers()) do
		if player.Name:lower():find(selectedTargetName:lower(), 1, true) then
			return player
		end
	end

	return nil
end

local function refreshStatus()
	updateCellsLabel()
	updateSessionTimeLabel()
	clearIgnoredTargets()

	local selectedPlayer = getSelectedPlayer()
	if isEligibleTarget(selectedPlayer) then
		currentTargetPlayer = selectedPlayer
	elseif not isEligibleTarget(currentTargetPlayer) then
		currentTargetPlayer = getPriorityTarget()
	end

	if currentTargetPlayer then
		local level = getPlayerLevel(currentTargetPlayer)
		if level then
			statusLabel.Text = "Target: " .. currentTargetPlayer.Name .. " | Level: " .. level
		else
			statusLabel.Text = "Target: " .. currentTargetPlayer.Name
		end
	else
		statusLabel.Text = "Target: none"
	end
end

local function getActiveTarget()
	refreshStatus()
	return currentTargetPlayer
end

local function rebuildDropdown()
	for _, button in ipairs(playerButtons) do
		button:Destroy()
	end
	table.clear(playerButtons)

	local ySize = 0

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer then
			local button = Instance.new("TextButton")
			button.Size = UDim2.new(1, -8, 0, 28)
			button.BackgroundColor3 = THEME_PANEL_ALT
			button.TextColor3 = THEME_WHITE
			button.Font = Enum.Font.GothamSemibold
			button.TextSize = 14
			button.Text = player.Name
			button.Parent = dropdownFrame

			local buttonCorner = Instance.new("UICorner")
			buttonCorner.CornerRadius = UDim.new(0, 10)
			buttonCorner.Parent = button

			local buttonStroke = Instance.new("UIStroke")
			buttonStroke.Color = THEME_OUTLINE
			buttonStroke.Thickness = 1.2
			buttonStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
			buttonStroke.Parent = button

			button.MouseButton1Click:Connect(function()
				ignoredTargets[player.Name] = nil
				respawnAttemptsByTarget[player.Name] = nil
				selectedTargetName = player.Name
				currentTargetPlayer = player
				dropdownButton.Text = "Selected: " .. player.Name
				dropdownFrame.Visible = false
				dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
				refreshStatus()
			end)

			table.insert(playerButtons, button)
			ySize += 32
		end
	end

	dropdownFrame.CanvasSize = UDim2.new(0, 0, 0, ySize)
	if dropdownFrame.Visible then
		dropdownFrame.Size = UDim2.new(0, SIDE_DROPDOWN_WIDTH, 0, math.min(ySize, 120))
	end
end

local function rebuildTargetTypeDropdown()
	for _, button in ipairs(targetTypeButtons) do
		button:Destroy()
	end
	table.clear(targetTypeButtons)

	local ySize = 0

	for _, option in ipairs(TARGET_PRIORITY_OPTIONS) do
		local button = Instance.new("TextButton")
		button.Size = UDim2.new(1, -8, 0, 28)
		button.BackgroundColor3 = THEME_PANEL_ALT
		button.TextColor3 = THEME_WHITE
		button.Font = Enum.Font.GothamSemibold
		button.TextSize = 14
		button.Text = option
		button.Parent = targetTypeFrame

		local buttonCorner = Instance.new("UICorner")
		buttonCorner.CornerRadius = UDim.new(0, 10)
		buttonCorner.Parent = button

		local buttonStroke = Instance.new("UIStroke")
		buttonStroke.Color = THEME_OUTLINE
		buttonStroke.Thickness = 1.2
		buttonStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
		buttonStroke.Parent = button

		button.MouseButton1Click:Connect(function()
			selectedTargetPriority = option
			targetTypeButton.Text = "Target Type: " .. option
			targetTypeFrame.Visible = false
			targetTypeFrame.Size = UDim2.new(0, 0, 0, 120)
			currentTargetPlayer = nil
			refreshStatus()
		end)

		table.insert(targetTypeButtons, button)
		ySize += 32
	end

	targetTypeFrame.CanvasSize = UDim2.new(0, 0, 0, ySize)
	if targetTypeFrame.Visible then
		targetTypeFrame.Size = UDim2.new(0, SIDE_DROPDOWN_WIDTH, 0, math.min(ySize, 120))
	end
end

-- Actions
local function pressKey(keyCode)
	VirtualInputManager:SendKeyEvent(true, keyCode, false, game)
	task.wait(0.02)
	VirtualInputManager:SendKeyEvent(false, keyCode, false, game)
end

local function fireRespawn()
	if respawnDebounce or not enabled then
		return
	end

	respawnDebounce = true
	task.delay(5, function()
		if not enabled then
			respawnDebounce = false
			return
		end

		local target = currentTargetPlayer
		if target and target.Parent and isAlive(target) then
			local targetName = target.Name
			respawnAttemptsByTarget[targetName] = (respawnAttemptsByTarget[targetName] or 0) + 1

			if respawnAttemptsByTarget[targetName] >= MAX_RESPAWNS_PER_TARGET then
				ignoredTargets[targetName] = true
				respawnAttemptsByTarget[targetName] = nil

				if selectedTargetName == targetName then
					selectedTargetName = ""
					dropdownButton.Text = "Select Player"
				end

				currentTargetPlayer = nil
			end
		end

		local currentKaiju = getCurrentKaiju()
		if currentKaiju then
			pcall(function()
				spawnRemote:FireServer("Spawn", currentKaiju, "Tokyo")
			end)
		end

		respawnDebounce = false
		refreshStatus()
	end)
end

local function hookCharacter(character)
	local humanoid = character:WaitForChild("Humanoid", 10)
	if humanoid then
		humanoid.Died:Connect(fireRespawn)
	end
end

local function startKeyLoop()
	if keyLoopRunning then
		return
	end

	keyLoopRunning = true
	task.spawn(function()
		while enabled do
			if not ensureBranding() then
				break
			end

			local target = getActiveTarget()
			local character = localPlayer.Character
			local hrp = character and character:FindFirstChild("HumanoidRootPart")
			local targetHrp = target and target.Character and target.Character:FindFirstChild("HumanoidRootPart")

			if hrp and targetHrp then
				for _, keyCode in ipairs(getAttackKeycodes()) do
					pressKey(keyCode)
				end
			end

			task.wait(KEY_SPAM_DELAY)
		end

		keyLoopRunning = false
	end)
end

local function startFollowLoop()
	if followLoopRunning then
		return
	end

	followLoopRunning = true
	task.spawn(function()
		while enabled do
			if not ensureBranding() then
				break
			end

			local target = getActiveTarget()
			local character = localPlayer.Character
			local hrp = character and character:FindFirstChild("HumanoidRootPart")

			if target and target.Character and hrp then
				local targetHrp = target.Character:FindFirstChild("HumanoidRootPart")
				if targetHrp then
					local offset = tonumber(amountBox.Text) or STUDS_UNDER
					local goalCFrame = targetHrp.CFrame * CFrame.new(0, -offset, 0)

					local tween = TweenService:Create(
						hrp,
						TweenInfo.new(TWEEN_TIME, Enum.EasingStyle.Linear),
						{CFrame = goalCFrame}
					)
					tween:Play()
				end
			end

			refreshStatus()
			task.wait(FOLLOW_DELAY)
		end

		followLoopRunning = false
	end)
end

-- Input and UI events
header.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = frame.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if not dragging then
		return
	end

	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		local delta = input.Position - dragStart
		frame.Position = UDim2.new(
			startPos.X.Scale,
			startPos.X.Offset + delta.X,
			startPos.Y.Scale,
			startPos.Y.Offset + delta.Y
		)
		shadow.Position = UDim2.new(
			frame.Position.X.Scale,
			frame.Position.X.Offset + 5,
			frame.Position.Y.Scale,
			frame.Position.Y.Offset + 5
		)
	end
end)

dropdownButton.MouseButton1Click:Connect(function()
	targetTypeFrame.Visible = false
	targetTypeFrame.Size = UDim2.new(0, 0, 0, 120)
	dropdownFrame.Visible = not dropdownFrame.Visible
	if dropdownFrame.Visible then
		rebuildDropdown()
		local canvasHeight = dropdownFrame.CanvasSize.Y.Offset
		dropdownFrame.Size = UDim2.new(0, SIDE_DROPDOWN_WIDTH, 0, math.min(canvasHeight, 120))
	else
		dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
	end
end)

targetTypeButton.MouseButton1Click:Connect(function()
	dropdownFrame.Visible = false
	dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
	targetTypeFrame.Visible = not targetTypeFrame.Visible
	if targetTypeFrame.Visible then
		rebuildTargetTypeDropdown()
		local canvasHeight = targetTypeFrame.CanvasSize.Y.Offset
		targetTypeFrame.Size = UDim2.new(0, SIDE_DROPDOWN_WIDTH, 0, math.min(canvasHeight, 120))
	else
		targetTypeFrame.Size = UDim2.new(0, 0, 0, 120)
	end
end)

-- Toggle
toggleButton.MouseButton1Click:Connect(function()
	if not ensureBranding() then
		return
	end

	enabled = not enabled

	if enabled then
		respawnAttemptsByTarget = {}
		ignoredTargets = {}
		sessionStartTime = os.clock()
		startCellsTracker()
		toggleButton.Text = "ON"
		toggleButton.BackgroundColor3 = THEME_RED
		startKeyLoop()
		startFollowLoop()
	else
		currentTargetPlayer = nil
		respawnAttemptsByTarget = {}
		ignoredTargets = {}
		sessionStartTime = 0
		disconnectCellsTracker()
		cellsSessionTotal = 0
		updateCellsLabel()
		updateSessionTimeLabel()
		toggleButton.Text = "OFF"
		toggleButton.BackgroundColor3 = THEME_RED_DARK
	end

	refreshStatus()
end)

-- Player events
Players.PlayerAdded:Connect(function()
	task.wait(0.2)
	rebuildDropdown()
	refreshStatus()
end)

Players.PlayerRemoving:Connect(function()
	task.wait(0.2)
	if currentTargetPlayer and not Players:FindFirstChild(currentTargetPlayer.Name) then
		currentTargetPlayer = nil
	end
	rebuildDropdown()
	refreshStatus()
end)

if localPlayer.Character then
	hookCharacter(localPlayer.Character)
end

localPlayer.CharacterAdded:Connect(function(character)
	hookCharacter(character)
	task.wait(1)
	refreshStatus()
end)

-- Startup
updateCellsLabel()
updateSessionTimeLabel()
rebuildDropdown()
refreshStatus()
