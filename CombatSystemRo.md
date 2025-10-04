# Roblox-Combat-System
This system allows u to upgrade your sword and it spawns npc to fight and get coins

-- ============================================================================
-- SERVICES - Roblox built-in services we need
-- ============================================================================
local Players = game:GetService("Players") -- Manages all players
local ReplicatedStorage = game:GetService("ReplicatedStorage") -- Shared storage
local ServerStorage = game:GetService("ServerStorage") -- Server-only storage
local Workspace = game.Workspace -- The 3D world

-- ============================================================================
-- CONFIGURATION - Customize the game settings here
-- ============================================================================
local CONFIG = {
	-- Starting values
	StartingCoins = 50, -- How many coins players start with
	StartingSword = "wooden_sword", -- Which sword players get at start

	-- NPC Settings
	NPCSpawnInterval = 8, -- Spawn NPC every 8 seconds
	MaxNPCs = 5, -- Maximum NPCs in the world at once
	NPCHealth = 100, -- How much health each NPC has
	NPCDamage = 10, -- How much damage NPCs deal to players
	NPCSpeed = 16, -- How fast NPCs move
	NPCAttackRange = 5, -- How close NPC needs to be to attack
	NPCAttackCooldown = 2, -- Seconds between NPC attacks

	-- Coin rewards
	CoinDropMin = 10, -- Minimum coins dropped by NPC
	CoinDropMax = 25, -- Maximum coins dropped by NPC

	-- Spawn area (where NPCs spawn)
	SpawnCenter = Vector3.new(0, 5, 0), -- Center point for spawning
	SpawnRadius = 50, -- How far from center NPCs can spawn
}

-- ============================================================================
-- SWORD DATABASE - All available swords with stats
-- ============================================================================
local SWORDS = {
	-- Tier 1: Wooden Sword (Starter)
	wooden_sword = {
		Name = "Wooden Sword", -- Display name
		Damage = 25, -- Damage per hit
		Price = 0, -- Free (starter weapon)
		Color = Color3.fromRGB(139, 90, 43), -- Brown color
		Tier = 1, -- Tier level
		Description = "A basic wooden training sword"
	},

	-- Tier 2: Stone Sword
	stone_sword = {
		Name = "Stone Sword",
		Damage = 50,
		Price = 100, -- Costs 100 coins
		Color = Color3.fromRGB(128, 128, 128), -- Gray color
		Tier = 2,
		Description = "A sturdy stone blade"
	},

	-- Tier 3: Iron Sword
	iron_sword = {
		Name = "Iron Sword",
		Damage = 100,
		Price = 300,
		Color = Color3.fromRGB(192, 192, 192), -- Silver color
		Tier = 3,
		Description = "A reliable iron weapon"
	},

	-- Tier 4: Gold Sword
	gold_sword = {
		Name = "Golden Sword",
		Damage = 200,
		Price = 750,
		Color = Color3.fromRGB(255, 215, 0), -- Gold color
		Tier = 4,
		Description = "A precious golden blade"
	},

	-- Tier 5: Diamond Sword
	diamond_sword = {
		Name = "Diamond Sword",
		Damage = 400,
		Price = 2000,
		Color = Color3.fromRGB(0, 255, 255), -- Cyan color
		Tier = 5,
		Description = "A legendary diamond weapon"
	},

	-- Tier 6: Legendary Sword (Final)
	legendary_sword = {
		Name = "Legendary Sword",
		Damage = 1000,
		Price = 5000,
		Color = Color3.fromRGB(255, 0, 255), -- Magenta/Purple
		Tier = 6,
		Description = "The ultimate weapon of legends"
	}
}

-- ============================================================================
-- PLAYER DATA MANAGER - Tracks each player's coins and current sword
-- ============================================================================
local PlayerData = {} -- Stores data for all players {UserId = {Coins = X, CurrentSword = "id"}}

-- Initialize data for a new player
local function InitializePlayerData(player)
	PlayerData[player.UserId] = {
		Coins = CONFIG.StartingCoins, -- Starting coins
		CurrentSword = CONFIG.StartingSword, -- Starting sword ID
		Kills = 0, -- Track kills for stats
	}
	print("[Player Data] Initialized data for " .. player.Name)
end

-- Get a player's data safely
local function GetPlayerData(player)
	return PlayerData[player.UserId]
end

-- Add coins to a player
local function AddCoins(player, amount)
	local data = GetPlayerData(player)
	if data then
		data.Coins = data.Coins + amount
		UpdateCoinGUI(player) -- Update the display
		return true
	end
	return false
end

-- Remove coins from a player (for purchases)
local function RemoveCoins(player, amount)
	local data = GetPlayerData(player)
	if data and data.Coins >= amount then
		data.Coins = data.Coins - amount
		UpdateCoinGUI(player)
		return true
	end
	return false
end

-- ============================================================================
-- SWORD CREATION - Creates physical sword tools
-- ============================================================================
-- Add combat functionality to a sword
local function AddSwordCombatScript(tool)
	local handle = tool:WaitForChild("Handle")
	local damageValue = tool:WaitForChild("Damage")
	local trail = handle:FindFirstChild("Trail")

	local debounce = false -- Prevents hitting multiple times per swing
	local swingCooldown = 0.5 -- Seconds between swings

	-- When sword is activated (clicked)
	tool.Activated:Connect(function()
		if debounce then return end -- Still on cooldown
		debounce = true

		-- Enable trail effect
		if trail then
			trail.Enabled = true
		end

		-- Play swing animation (visual feedback)
		local player = tool.Parent.Parent -- Get the player
		if player and player:IsA("Player") then
			local character = player.Character
			if character then
				local humanoid = character:FindFirstChild("Humanoid")
				if humanoid then
					-- Make character swing animation
					for i = 1, 10 do
						humanoid.RootPart.CFrame = humanoid.RootPart.CFrame * CFrame.Angles(0, math.rad(36), 0)
						wait(0.01)
					end
				end
			end
		end

		-- Detect hits (check what the sword touched)
		local hitConnection
		hitConnection = handle.Touched:Connect(function(hit)
			-- Check if we hit an NPC
			local npc = hit.Parent
			if npc and npc:FindFirstChild("Humanoid") and npc:FindFirstChild("IsNPC") then
				local npcHumanoid = npc.Humanoid

				-- Deal damage
				npcHumanoid.Health = npcHumanoid.Health - damageValue.Value

				-- Show damage number (visual feedback)
				ShowDamageNumber(hit.Position, damageValue.Value)

				print("[Combat] Dealt " .. damageValue.Value .. " damage to NPC")

				-- Disconnect after first hit (one hit per swing)
				hitConnection:Disconnect()
			end
		end)

		-- Disable trail after swing
		wait(0.3)
		if trail then
			trail.Enabled = false
		end

		-- Disconnect hit detection after swing duration
		wait(0.2)
		if hitConnection then
			hitConnection:Disconnect()
		end

		-- Reset cooldown
		wait(swingCooldown - 0.5)
		debounce = false
	end)
end
-- Create a sword tool from the database
local function CreateSword(swordId)
	local swordData = SWORDS[swordId]
	if not swordData then
		warn("[Sword] Invalid sword ID: " .. swordId)
		return nil
	end

	-- Create the Tool object (what players hold)
	local tool = Instance.new("Tool")
	tool.Name = swordData.Name -- Display name
	tool.ToolTip = swordData.Description .. " | Damage: " .. swordData.Damage -- Tooltip on hover
	tool.RequiresHandle = true -- Needs a Handle part

	-- Create the Handle (the visible 3D part)
	local handle = Instance.new("Part")
	handle.Name = "Handle" -- MUST be named "Handle"
	handle.Size = Vector3.new(0.5, 0.5, 4) -- Sword shape (thin and long)
	handle.Material = Enum.Material.SmoothPlastic
	handle.BrickColor = BrickColor.new(swordData.Color) -- Color from database
	handle.Parent = tool

	-- Add a mesh to make it look more like a sword
	local mesh = Instance.new("BlockMesh")
	mesh.Scale = Vector3.new(0.4, 0.3, 1) -- Make it thinner
	mesh.Parent = handle

	-- Add a trail effect for visual flair
	local attachment0 = Instance.new("Attachment")
	attachment0.Position = Vector3.new(0, 0, -2) -- Bottom of sword
	attachment0.Parent = handle

	local attachment1 = Instance.new("Attachment")
	attachment1.Position = Vector3.new(0, 0, 2) -- Top of sword
	attachment1.Parent = handle

	local trail = Instance.new("Trail")
	trail.Attachment0 = attachment0
	trail.Attachment1 = attachment1
	-- ColorSequence needs two keypoints minimum
	trail.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, swordData.Color),
		ColorSequenceKeypoint.new(1, swordData.Color)
	})
	trail.Lifetime = 0.5 -- Trail lasts 0.5 seconds
	trail.MinLength = 0.1
	trail.Enabled = false -- Only show when swinging
	trail.Parent = handle

	-- Store sword data in the tool for easy access
	local damageValue = Instance.new("IntValue")
	damageValue.Name = "Damage"
	damageValue.Value = swordData.Damage
	damageValue.Parent = tool

	local swordIdValue = Instance.new("StringValue")
	swordIdValue.Name = "SwordId"
	swordIdValue.Value = swordId
	swordIdValue.Parent = tool

	-- Add combat script to the sword
	AddSwordCombatScript(tool)

	print("[Sword] Created: " .. swordData.Name)
	return tool
end



-- Give a sword to a player
local function GiveSword(player, swordId)
	-- Remove old swords from player
	local character = player.Character
	if character then
		for _, item in pairs(character:GetChildren()) do
			if item:IsA("Tool") and SWORDS[item:FindFirstChild("SwordId") and item.SwordId.Value] then
				item:Destroy()
			end
		end
	end

	-- Remove from backpack too
	local backpack = player:FindFirstChild("Backpack")
	if backpack then
		for _, item in pairs(backpack:GetChildren()) do
			if item:IsA("Tool") and SWORDS[item:FindFirstChild("SwordId") and item.SwordId.Value] then
				item:Destroy()
			end
		end
	end

	-- Create and give new sword
	local sword = CreateSword(swordId)
	if sword and backpack then
		sword.Parent = backpack

		-- Update player data
		local data = GetPlayerData(player)
		if data then
			data.CurrentSword = swordId
		end

		print("[Sword] Gave " .. SWORDS[swordId].Name .. " to " .. player.Name)
		return true
	end
	return false
end

-- ============================================================================
-- NPC SYSTEM - Spawns and controls enemy NPCs
-- ============================================================================

local ActiveNPCs = {} -- Tracks all spawned NPCs
-- Create a health bar GUI above NPC
local function CreateHealthBar(npc)
	local head = npc:FindFirstChild("Head")
	if not head then return end

	-- Create BillboardGui (GUI that floats in 3D space)
	local billboard = Instance.new("BillboardGui")
	billboard.Name = "HealthBar"
	billboard.Size = UDim2.new(4, 0, 0.5, 0) -- Size of the billboard
	billboard.StudsOffset = Vector3.new(0, 3, 0) -- Position above head
	billboard.AlwaysOnTop = true -- Always visible through walls
	billboard.Adornee = head -- Attach to the head
	billboard.Parent = head

	-- Background frame
	local background = Instance.new("Frame")
	background.Size = UDim2.new(1, 0, 1, 0)
	background.BackgroundColor3 = Color3.fromRGB(50, 50, 50) -- Dark gray
	background.BorderSizePixel = 2
	background.Parent = billboard

	-- Health bar (green fill)
	local healthBar = Instance.new("Frame")
	healthBar.Name = "Bar"
	healthBar.Size = UDim2.new(1, 0, 1, 0) -- Full width = full health
	healthBar.BackgroundColor3 = Color3.fromRGB(0, 255, 0) -- Green
	healthBar.BorderSizePixel = 0
	healthBar.Parent = background

	-- Health text
	local healthText = Instance.new("TextLabel")
	healthText.Name = "Text"
	healthText.Size = UDim2.new(1, 0, 1, 0)
	healthText.BackgroundTransparency = 1
	healthText.Text = CONFIG.NPCHealth .. "/" .. CONFIG.NPCHealth
	healthText.TextColor3 = Color3.fromRGB(255, 255, 255)
	healthText.TextScaled = true
	healthText.Font = Enum.Font.GothamBold
	healthText.Parent = background

	-- Update health bar when NPC takes damage
	local humanoid = npc:FindFirstChild("Humanoid")
	if humanoid then
		humanoid.HealthChanged:Connect(function(health)
			local percentage = health / humanoid.MaxHealth
			healthBar.Size = UDim2.new(percentage, 0, 1, 0)
			healthText.Text = math.floor(health) .. "/" .. humanoid.MaxHealth

			-- Change color based on health
			if percentage > 0.5 then
				healthBar.BackgroundColor3 = Color3.fromRGB(0, 255, 0) -- Green
			elseif percentage > 0.25 then
				healthBar.BackgroundColor3 = Color3.fromRGB(255, 255, 0) -- Yellow
			else
				healthBar.BackgroundColor3 = Color3.fromRGB(255, 0, 0) -- Red
			end
		end)
	end
end
-- Create an NPC enemy
local function CreateNPC()
	-- Create the NPC model
	local npc = Instance.new("Model")
	npc.Name = "Enemy"

	-- Create body parts
	local head = Instance.new("Part")
	head.Name = "Head"
	head.Size = Vector3.new(2, 1, 1)
	head.BrickColor = BrickColor.new("Bright red") -- Red = enemy
	head.TopSurface = Enum.SurfaceType.Smooth
	head.BottomSurface = Enum.SurfaceType.Smooth
	head.Parent = npc

	local torso = Instance.new("Part")
	torso.Name = "Torso"
	torso.Size = Vector3.new(2, 2, 1)
	torso.BrickColor = BrickColor.new("Bright blue")
	torso.TopSurface = Enum.SurfaceType.Smooth
	torso.BottomSurface = Enum.SurfaceType.Smooth
	torso.Parent = npc

	-- Add humanoid (makes it alive and able to walk)
	local humanoid = Instance.new("Humanoid")
	humanoid.MaxHealth = CONFIG.NPCHealth
	humanoid.Health = CONFIG.NPCHealth
	humanoid.WalkSpeed = CONFIG.NPCSpeed
	humanoid.Parent = npc

	-- Add humanoid root part (required for movement)
	local rootPart = Instance.new("Part")
	rootPart.Name = "HumanoidRootPart"
	rootPart.Size = Vector3.new(2, 2, 1)
	rootPart.Transparency = 1 -- Invisible
	rootPart.CanCollide = false
	rootPart.Anchored = false -- Must not be anchored for movement
	rootPart.Parent = npc

	-- Weld parts together so they move as one
	local neckWeld = Instance.new("Weld")
	neckWeld.Part0 = torso
	neckWeld.Part1 = head
	neckWeld.C0 = CFrame.new(0, 1.5, 0) -- Position head above torso
	neckWeld.Parent = torso

	local rootWeld = Instance.new("Weld")
	rootWeld.Part0 = rootPart
	rootWeld.Part1 = torso
	rootWeld.Parent = rootPart

	-- Mark as NPC (so swords can detect it)
	local isNPCTag = Instance.new("BoolValue")
	isNPCTag.Name = "IsNPC"
	isNPCTag.Value = true
	isNPCTag.Parent = npc

	-- Set primary part (used for positioning)
	npc.PrimaryPart = rootPart

	-- Add health bar above NPC
	CreateHealthBar(npc)

	return npc
end

-- NPC AI - Makes NPC chase and attack players
local function StartNPCAI(npc)
	local humanoid = npc:FindFirstChild("Humanoid")
	local rootPart = npc:FindFirstChild("HumanoidRootPart")
	if not humanoid or not rootPart then return end

	local lastAttackTime = 0

	-- AI loop
	spawn(function()
		while npc.Parent and humanoid.Health > 0 do
			-- Find nearest player
			local nearestPlayer = nil
			local nearestDistance = math.huge

			for _, player in pairs(Players:GetPlayers()) do
				local character = player.Character
				if character and character:FindFirstChild("HumanoidRootPart") then
					local distance = (character.HumanoidRootPart.Position - rootPart.Position).Magnitude
					if distance < nearestDistance then
						nearestDistance = distance
						nearestPlayer = player
					end
				end
			end

			-- Chase nearest player
			if nearestPlayer and nearestPlayer.Character then
				local targetRoot = nearestPlayer.Character.HumanoidRootPart
				humanoid:MoveTo(targetRoot.Position) -- Move towards player

				-- Attack if in range
				if nearestDistance <= CONFIG.NPCAttackRange then
					local currentTime = tick()
					if currentTime - lastAttackTime >= CONFIG.NPCAttackCooldown then
						-- Deal damage to player
						local targetHumanoid = nearestPlayer.Character:FindFirstChild("Humanoid")
						if targetHumanoid then
							targetHumanoid:TakeDamage(CONFIG.NPCDamage)
							lastAttackTime = currentTime
							print("[NPC] Attacked " .. nearestPlayer.Name .. " for " .. CONFIG.NPCDamage .. " damage")
						end
					end
				end
			end

			wait(0.5) -- Update AI every 0.5 seconds
		end
	end)
end

-- Spawn a coin pickup in the world
local function SpawnCoinDrop(position, amount)
	-- Create coin part
	local coin = Instance.new("Part")
	coin.Name = "Coin"
	coin.Size = Vector3.new(1, 0.2, 1)
	coin.Shape = Enum.PartType.Cylinder
	coin.BrickColor = BrickColor.new("Bright yellow")
	coin.Material = Enum.Material.Neon -- Glowing effect
	coin.Position = position + Vector3.new(0, 2, 0) -- Slightly above ground
	coin.Anchored = true
	coin.CanCollide = false
	coin.Parent = Workspace

	-- Store coin value
	local valueTag = Instance.new("IntValue")
	valueTag.Name = "CoinValue"
	valueTag.Value = amount
	valueTag.Parent = coin

	-- Add spinning animation
	spawn(function()
		while coin.Parent do
			coin.CFrame = coin.CFrame * CFrame.Angles(0, math.rad(5), 0)
			wait(0.05)
		end
	end)

	-- Add pickup detection
	coin.Touched:Connect(function(hit)
		local character = hit.Parent
		local player = Players:GetPlayerFromCharacter(character)

		if player then
			-- Give coins to player
			AddCoins(player, amount)
			ShowNotification(player, "💰 +" .. amount .. " Coins!")
			coin:Destroy()
			print("[Coin] " .. player.Name .. " collected " .. amount .. " coins")
		end
	end)

	-- Remove coin after 30 seconds if not picked up
	wait(30)
	if coin.Parent then
		coin:Destroy()
	end
end

-- Handle NPC death (drop coins)
local function OnNPCDeath(npc)
	-- Remove from active list
	for i, activeNPC in ipairs(ActiveNPCs) do
		if activeNPC == npc then
			table.remove(ActiveNPCs, i)
			break
		end
	end

	-- Drop coins
	local coinAmount = math.random(CONFIG.CoinDropMin, CONFIG.CoinDropMax)
	SpawnCoinDrop(npc.PrimaryPart.Position, coinAmount)

	-- Remove NPC after delay
	wait(2)
	npc:Destroy()

	print("[NPC] Enemy defeated, dropped " .. coinAmount .. " coins")
end

-- Spawn an NPC at a random location
local function SpawnNPC()
	if #ActiveNPCs >= CONFIG.MaxNPCs then
		return -- Max NPCs reached
	end

	local npc = CreateNPC()

	-- Random spawn position within radius
	local angle = math.random() * math.pi * 2 -- Random angle
	local distance = math.random() * CONFIG.SpawnRadius -- Random distance
	local spawnPos = CONFIG.SpawnCenter + Vector3.new(
		math.cos(angle) * distance,
		0,
		math.sin(angle) * distance
	)

	-- Position the NPC
	npc:SetPrimaryPartCFrame(CFrame.new(spawnPos))
	npc.Parent = Workspace

	-- Add to active list
	table.insert(ActiveNPCs, npc)

	-- Start NPC AI
	StartNPCAI(npc)

	-- Handle NPC death
	local humanoid = npc:FindFirstChild("Humanoid")
	if humanoid then
		humanoid.Died:Connect(function()
			OnNPCDeath(npc)
		end)
	end

	print("[NPC] Spawned enemy at " .. tostring(spawnPos))
end

-- ============================================================================
-- SHOP GUI SYSTEM - Allows players to buy sword upgrades
-- ============================================================================

-- Populate shop with sword items
local function PopulateShop(player, scrollFrame)
	-- Clear existing items (except layout objects)
	for _, child in pairs(scrollFrame:GetChildren()) do
		if child:IsA("Frame") then
			child:Destroy()
		end
	end

	-- Make sure we have a UIListLayout in the scrollFrame
	local listLayout = scrollFrame:FindFirstChildOfClass("UIListLayout")
	if not listLayout then
		listLayout = Instance.new("UIListLayout")
		listLayout.Parent = scrollFrame
		listLayout.SortOrder = Enum.SortOrder.LayoutOrder
		listLayout.Padding = UDim.new(0, 10)
	end

	local data = GetPlayerData(player)
	if not data then return end

	-- Create item for each sword
	for swordId, swordData in pairs(SWORDS) do
		local itemFrame = Instance.new("Frame")
		itemFrame.Size = UDim2.new(1, -10, 0, 100)
		itemFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
		itemFrame.BorderSizePixel = 2
		itemFrame.BorderColor3 = swordData.Color
		itemFrame.LayoutOrder = swordData.Price -- optional sorting
		itemFrame.Parent = scrollFrame

		local itemCorner = Instance.new("UICorner")
		itemCorner.CornerRadius = UDim.new(0, 10)
		itemCorner.Parent = itemFrame

		-- Sword name
		local nameLabel = Instance.new("TextLabel")
		nameLabel.Size = UDim2.new(1, -120, 0, 30)
		nameLabel.Position = UDim2.new(0, 10, 0, 5)
		nameLabel.BackgroundTransparency = 1
		nameLabel.Text = swordData.Name
		nameLabel.TextColor3 = swordData.Color
		nameLabel.TextSize = 22
		nameLabel.Font = Enum.Font.GothamBold
		nameLabel.TextXAlignment = Enum.TextXAlignment.Left
		nameLabel.Parent = itemFrame

		-- Damage stat
		local damageLabel = Instance.new("TextLabel")
		damageLabel.Size = UDim2.new(1, -120, 0, 25)
		damageLabel.Position = UDim2.new(0, 10, 0, 35)
		damageLabel.BackgroundTransparency = 1
		damageLabel.Text = "⚔️ Damage: " .. swordData.Damage
		damageLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
		damageLabel.TextSize = 18
		damageLabel.Font = Enum.Font.Gotham
		damageLabel.TextXAlignment = Enum.TextXAlignment.Left
		damageLabel.Parent = itemFrame

		-- Description
		local descLabel = Instance.new("TextLabel")
		descLabel.Size = UDim2.new(1, -120, 0, 25)
		descLabel.Position = UDim2.new(0, 10, 0, 60)
		descLabel.BackgroundTransparency = 1
		descLabel.Text = swordData.Description
		descLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
		descLabel.TextSize = 14
		descLabel.Font = Enum.Font.Gotham
		descLabel.TextXAlignment = Enum.TextXAlignment.Left
		descLabel.Parent = itemFrame

		-- Buy button
		local buyButton = Instance.new("TextButton")
		buyButton.Size = UDim2.new(0, 100, 0, 80)
		buyButton.Position = UDim2.new(1, -110, 0, 10)
		buyButton.Font = Enum.Font.GothamBold
		buyButton.TextSize = 18
		buyButton.Parent = itemFrame

		local buyCorner = Instance.new("UICorner")
		buyCorner.CornerRadius = UDim.new(0, 8)
		buyCorner.Parent = buyButton

		-- Check if player owns this sword
		if data.CurrentSword == swordId then
			-- Player already has this sword
			buyButton.Text = "✓ OWNED"
			buyButton.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
			buyButton.TextColor3 = Color3.fromRGB(200, 200, 200)
			buyButton.Active = false
		elseif swordData.Price == 0 then
			-- Free sword
			buyButton.Text = "FREE"
			buyButton.BackgroundColor3 = Color3.fromRGB(50, 150, 50)
			buyButton.TextColor3 = Color3.fromRGB(255, 255, 255)
			buyButton.MouseButton1Click:Connect(function()
				GiveSword(player, swordId)
				ShowNotification(player, "✓ Claimed " .. swordData.Name .. "!")
				PopulateShop(player, scrollFrame)
			end)
		else
			-- Available for purchase
			buyButton.Text = "BUY\n💰 " .. swordData.Price
			buyButton.BackgroundColor3 = Color3.fromRGB(50, 100, 200)
			buyButton.TextColor3 = Color3.fromRGB(255, 255, 255)

			-- Handle purchase
			buyButton.MouseButton1Click:Connect(function()
				if data.Coins >= swordData.Price then
					-- Can afford
					RemoveCoins(player, swordData.Price)
					GiveSword(player, swordId)
					ShowNotification(player, "✓ Purchased " .. swordData.Name .. "!")
					PopulateShop(player, scrollFrame) -- Refresh shop
				else
					-- Can't afford
					ShowNotification(player, "❌ Not enough coins!")
				end
			end)
		end
	end

	-- Update scroll canvas size after all items are added
	scrollFrame.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 10)
end


-- Create shop GUI for a player
local function CreateShopGUI(player)
	local playerGui = player:WaitForChild("PlayerGui")

	-- Main screen GUI
	local screenGui = Instance.new("ScreenGui")
	screenGui.Name = "ShopGUI"
	screenGui.ResetOnSpawn = false
	screenGui.Parent = playerGui

	-- Coin display (top-left)
	local coinFrame = Instance.new("Frame")
	coinFrame.Name = "CoinDisplay"
	coinFrame.Size = UDim2.new(0, 200, 0, 60)
	coinFrame.Position = UDim2.new(0, 10, 0, 10)
	coinFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	coinFrame.BorderSizePixel = 3
	coinFrame.BorderColor3 = Color3.fromRGB(255, 215, 0)
	coinFrame.Parent = screenGui

	local coinCorner = Instance.new("UICorner")
	coinCorner.CornerRadius = UDim.new(0, 10)
	coinCorner.Parent = coinFrame

	local coinLabel = Instance.new("TextLabel")
	coinLabel.Name = "CoinLabel"
	coinLabel.Size = UDim2.new(1, 0, 1, 0)
	coinLabel.BackgroundTransparency = 1
	coinLabel.Text = "💰 Coins: " .. CONFIG.StartingCoins
	coinLabel.TextColor3 = Color3.fromRGB(255, 215, 0)
	coinLabel.TextSize = 24
	coinLabel.Font = Enum.Font.GothamBold
	coinLabel.Parent = coinFrame

	-- Shop button (top-right)
	local shopButton = Instance.new("TextButton")
	shopButton.Name = "ShopButton"
	shopButton.Size = UDim2.new(0, 150, 0, 60)
	shopButton.Position = UDim2.new(1, -160, 0, 10)
	shopButton.BackgroundColor3 = Color3.fromRGB(50, 150, 50)
	shopButton.BorderSizePixel = 3
	shopButton.BorderColor3 = Color3.fromRGB(255, 255, 255)
	shopButton.Text = "🛒 SHOP"
	shopButton.TextColor3 = Color3.fromRGB(255, 255, 255)
	shopButton.TextSize = 24
	shopButton.Font = Enum.Font.GothamBold
	shopButton.Parent = screenGui

	local shopCorner = Instance.new("UICorner")
	shopCorner.CornerRadius = UDim.new(0, 10)
	shopCorner.Parent = shopButton

	-- Shop panel (hidden by default)
	local shopPanel = Instance.new("Frame")
	shopPanel.Name = "ShopPanel"
	shopPanel.Size = UDim2.new(0, 600, 0, 500)
	shopPanel.Position = UDim2.new(0.5, -300, 0.5, -250)
	shopPanel.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
	shopPanel.BorderSizePixel = 4
	shopPanel.BorderColor3 = Color3.fromRGB(100, 200, 255)
	shopPanel.Visible = false
	shopPanel.Parent = screenGui

	local panelCorner = Instance.new("UICorner")
	panelCorner.CornerRadius = UDim.new(0, 15)
	panelCorner.Parent = shopPanel

	-- Shop title
	local shopTitle = Instance.new("TextLabel")
	shopTitle.Size = UDim2.new(1, 0, 0, 50)
	shopTitle.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
	shopTitle.BorderSizePixel = 0
	shopTitle.Text = "⚔️ SWORD SHOP ⚔️"
	shopTitle.TextColor3 = Color3.fromRGB(255, 215, 0)
	shopTitle.TextSize = 28
	shopTitle.Font = Enum.Font.GothamBold
	shopTitle.Parent = shopPanel

	-- Close button
	local closeButton = Instance.new("TextButton")
	closeButton.Size = UDim2.new(0, 40, 0, 40)
	closeButton.Position = UDim2.new(1, -45, 0, 5)
	closeButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
	closeButton.Text = "✖"
	closeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
	closeButton.TextSize = 24
	closeButton.Font = Enum.Font.GothamBold
	closeButton.Parent = shopPanel

	local closeCorner = Instance.new("UICorner")
	closeCorner.CornerRadius = UDim.new(0, 8)
	closeCorner.Parent = closeButton

	-- Scrolling frame for sword items
	local scrollFrame = Instance.new("ScrollingFrame")
	scrollFrame.Name = "SwordList"
	scrollFrame.Size = UDim2.new(1, -20, 1, -70)
	scrollFrame.Position = UDim2.new(0, 10, 0, 60)
	scrollFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	scrollFrame.BorderSizePixel = 0
	scrollFrame.ScrollBarThickness = 10
	scrollFrame.Parent = shopPanel

	-- Layout for sword items
	local listLayout = Instance.new("UIListLayout")
	listLayout.Padding = UDim.new(0, 10)
	listLayout.Parent = scrollFrame

	-- Populate shop with swords
	PopulateShop(player, scrollFrame)

	-- Toggle shop visibility
	shopButton.MouseButton1Click:Connect(function()
		shopPanel.Visible = not shopPanel.Visible
		if shopPanel.Visible then
			PopulateShop(player, scrollFrame) -- Refresh shop
		end
	end)

	closeButton.MouseButton1Click:Connect(function()
		shopPanel.Visible = false
	end)

	print("[GUI] Created shop interface for " .. player.Name)
end

-- Update coin display
function UpdateCoinGUI(player)
	local gui = player.PlayerGui:FindFirstChild("ShopGUI")
	if not gui then return end

	local coinDisplay = gui:FindFirstChild("CoinDisplay")
	if not coinDisplay then return end

	local coinLabel = coinDisplay:FindFirstChild("CoinLabel")
	if coinLabel then
		local data = GetPlayerData(player)
		if data then
			coinLabel.Text = "💰 Coins: " .. data.Coins
		end
	end
end

-- Show notification to player
function ShowNotification(player, message)
	local gui = player.PlayerGui:FindFirstChild("ShopGUI")
	if not gui then return end

	-- Create notification
	local notif = Instance.new("Frame")
	notif.Size = UDim2.new(0, 300, 0, 60)
	notif.Position = UDim2.new(0.5, -150, 0, -80)
	notif.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	notif.BorderSizePixel = 3
	notif.BorderColor3 = Color3.fromRGB(255, 215, 0)
	notif.Parent = gui

	local notifCorner = Instance.new("UICorner")
	notifCorner.CornerRadius = UDim.new(0, 10)
	notifCorner.Parent = notif

	local notifLabel = Instance.new("TextLabel")
	notifLabel.Size = UDim2.new(1, 0, 1, 0)
	notifLabel.BackgroundTransparency = 1
	notifLabel.Text = message
	notifLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	notifLabel.TextSize = 20
	notifLabel.Font = Enum.Font.GothamBold
	notifLabel.TextWrapped = true
	notifLabel.Parent = notif

	-- Animate in
	notif:TweenPosition(
		UDim2.new(0.5, -150, 0, 20),
		Enum.EasingDirection.Out,
		Enum.EasingStyle.Back,
		0.5,
		true
	)

	-- Remove after delay
	wait(2.5)
	notif:TweenPosition(
		UDim2.new(0.5, -150, 0, -80),
		Enum.EasingDirection.In,
		Enum.EasingStyle.Back,
		0.5,
		true
	)
	wait(0.5)
	notif:Destroy()
end

-- Show damage number in 3D space
function ShowDamageNumber(position, damage)
	-- Create part for damage number
	local part = Instance.new("Part")
	part.Size = Vector3.new(1, 1, 1)
	part.Position = position + Vector3.new(0, 3, 0)
	part.Anchored = true
	part.CanCollide = false
	part.Transparency = 1
	part.Parent = Workspace

	-- Add billboard GUI
	local billboard = Instance.new("BillboardGui")
	billboard.Size = UDim2.new(4, 0, 2, 0)
	billboard.AlwaysOnTop = true
	billboard.Adornee = part -- Attach to the part
	billboard.Parent = part

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(1, 0, 1, 0)
	label.BackgroundTransparency = 1
	label.Text = "-" .. damage
	label.TextColor3 = Color3.fromRGB(255, 50, 50)
	label.TextSize = 48
	label.Font = Enum.Font.GothamBold
	label.TextStrokeTransparency = 0.5
	label.Parent = billboard

	-- Animate upward and fade
	spawn(function()
		for i = 1, 20 do
			part.Position = part.Position + Vector3.new(0, 0.1, 0)
			label.TextTransparency = i / 20
			wait(0.05)
		end
		part:Destroy()
	end)
end

-- ============================================================================
-- NPC SPAWNER - Continuously spawns NPCs
-- ============================================================================
spawn(function()
	while true do
		wait(CONFIG.NPCSpawnInterval)
		SpawnNPC()
	end
end)

-- ============================================================================
-- PLAYER CONNECTION HANDLERS
-- ============================================================================

-- When player joins
Players.PlayerAdded:Connect(function(player)
	print("[System] Player joined: " .. player.Name)

	-- Initialize player data
	InitializePlayerData(player)

	-- Wait for character to load
	player.CharacterAdded:Connect(function(character)
		wait(1) -- Small delay to ensure everything loads

		-- Give starter sword
		local data = GetPlayerData(player)
		if data then
			GiveSword(player, data.CurrentSword)
		end

		-- Track kills
		local humanoid = character:WaitForChild("Humanoid")
		humanoid.Died:Connect(function()
			-- Reset on death (optional)
			wait(5)
			if player.Character then
				local newData = GetPlayerData(player)
				if newData then
					GiveSword(player, newData.CurrentSword)
				end
			end
		end)
	end)

	-- Create GUI
	CreateShopGUI(player)

	-- Welcome message
	wait(2)
	ShowNotification(player, "👋 Welcome! Defeat enemies to earn coins!")
end)

-- When player leaves
Players.PlayerRemoving:Connect(function(player)
	PlayerData[player.UserId] = nil
	print("[System] Player left: " .. player.Name)
end)

-- ============================================================================
-- INITIALIZATION
-- ============================================================================
print("═══════════════════════════════════════════════════════════════")
print("✓ COMBAT & SHOP SYSTEM INITIALIZED")
print("✓ NPCs will spawn every " .. CONFIG.NPCSpawnInterval .. " seconds")
-- Count swords properly (tables with string keys need manual counting)
local swordCount = 0
for _ in pairs(SWORDS) do swordCount = swordCount + 1 end
print("✓ " .. swordCount .. " swords available in shop")
print("✓ Ready for combat!")
print("═══════════════════════════════════════════════════════════════")
