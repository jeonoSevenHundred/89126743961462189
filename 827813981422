--// Types

type ItemType = {
	Name: string;
	Worth: number;
	Rarity: string;
	ImageId: string;
	MinigameSteps: number;
	TimeLimit: number;
}

type CriminalCase = {
	RegisterTime: number;
	Client: Player;
	Item: ItemType;
	NPC: {NPC: Model};
	VotesToStopChasing: number?;
}

--// Services

local Players             = game:GetService("Players")
local ServerScriptService = game:GetService("ServerScriptService")
local PhysicsService      = game:GetService("PhysicsService")
local RunService          = game:GetService("RunService")
local ServerStorage       = game:GetService("ServerStorage")
local BadgeService        = game:GetService("BadgeService")
local Chat                = game:GetService("Chat")
local StarterPlayer       = game:GetService("StarterPlayer")

--// Framework

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VAULT99           = ReplicatedStorage.Framework

local Cache         = VAULT99.Cache
local Communication = VAULT99.Communication
local Modules       = VAULT99.Modules
local Dict          = ServerScriptService.Dict

--// Modules

local LIBs           = ReplicatedStorage.LIBs
local TokenGenerator = require(Modules.Utility.TokenGenerator)
local XPCalculator   = require(LIBs:WaitForChild("XPCalculator"))
local NPCLib         = require(LIBs:WaitForChild("NPCLib"))
local ResponseLIB    = require(LIBs:WaitForChild("ResponseLIB"))

--// Variables

local PossibleSpawns = workspace.Core.Backend.SpawnPoints:GetChildren()

-- or {} so hot reloads dont wipe live data
_G.ActiveCriminalCases = _G.ActiveCriminalCases or {}
_G.spawnedAmount       = _G.spawnedAmount       or {}
_G.PickpocketResultLog = _G.PickpocketResultLog or {}

--// Constants

local VOTES_NEEDED_TO_ESCAPE = 5
local NOTIFICATION_DURATION  = 3
local WANTED_COLOR           = Color3.fromRGB(255, 0, 0)
local ESCAPE_BADGE_ID        = 9541313380911
local TIMER_NO_ITEM          = 10
local TIMER_HAS_ITEM         = 40

--// Helper Functions

-- picks a random npc response
local function GetResponse(): string
	if not ResponseLIB or #ResponseLIB == 0 then
		return "Hey!"
	end
	return ResponseLIB[math.random(1, #ResponseLIB)] or "bro wyd"
end

-- teleports npc to a part, offset up so it dont clip
local function SpawnNPC(npc: Model, location: BasePart)
	if not npc or not location then return end

	local humanoidRootPart = npc:FindFirstChild("HumanoidRootPart")
	if not humanoidRootPart then return end

	local success, err = pcall(function()
		npc:PivotTo(location.CFrame + Vector3.new(0, 3, 0))
	end)

	if not success then
		warn("SpawnNPC: PivotTo failed —", err)
	end
end

-- tells all police npcs to either chase or go back to patrolling
local function ChangePoliceAttitude(shouldChase: boolean)
	if not _G.Entities then return end

	local targetState = shouldChase and "ChasingPickpocketer" or "Patrolling"

	for _, entity in pairs(_G.Entities) do
		if entity and entity.IsPolice and entity.ChangeState then
			local success, err = pcall(function()
				entity:ChangeState(targetState)
			end)
			if not success then
				warn("ChangePoliceAttitude: state change failed —", err)
			end
		end
	end
end

-- finds the item in the loot bag closest in value to targetWorth
local function GetSimilarValueItem(lootData: {}, targetWorth: number)
	if not lootData or not next(lootData) then return nil end
	if not targetWorth or type(targetWorth) ~= "number" then return nil end

	local closestItem        = nil
	local closestKey         = nil
	local smallestDifference = math.huge

	for itemIdentifier, item in pairs(lootData) do
		if item and type(item) == "table" and item.Worth then
			local difference = math.abs(item.Worth - targetWorth)
			if difference < smallestDifference then
				smallestDifference = difference
				closestItem        = item
				closestKey         = itemIdentifier
			end
		end
	end

	return closestItem, closestKey
end

-- returns a random key from any table, fallback for GetSimilarValueItem
local function GetRandomKey(tbl)
	if not tbl or not next(tbl) then return nil end

	local keys = {}
	for key in pairs(tbl) do
		table.insert(keys, key)
	end

	return #keys > 0 and keys[math.random(1, #keys)] or nil
end

-- how many players are currently wanted
local function CountActiveCases(): number
	local count = 0
	for _ in pairs(_G.ActiveCriminalCases) do
		count += 1
	end
	return count
end

-- makes an npc say something, wrapped in pcall in case the npc got removed
local function NPCChat(npc: Model?, message: string)
	if not npc or not message then return end

	local success, err = pcall(function()
		local head = npc:FindFirstChild("Head")
		if head then
			Chat:Chat(head, message)
		end
	end)

	if not success then
		warn("NPCChat: Chat:Chat failed —", err)
	end
end

--// ReviewLevel
-- checks if player crossed the xp threshold, levels them up if so and gives rewards

local ReviewLevel = function(Client: Player)
	local LeveledUpData, CurrentLevelData, RequiredXPData, CurrentXPData

	if not Client then return end

	local leaderstats = Client:FindFirstChild("leaderstats")
	if not leaderstats then return end

	local LevelValue: IntValue = leaderstats:FindFirstChild("Level")
	if not LevelValue then return end

	local CurrentLevel = LevelValue.Value
	CurrentLevelData   = CurrentLevel

	local XPValue = Client:GetAttribute("XP")
	if not XPValue then return end

	local CurrentXP = XPValue
	local GoalXP    = XPCalculator(CurrentLevel + 1)
	CurrentXPData   = CurrentXP
	RequiredXPData  = GoalXP
	LeveledUpData   = false

	if CurrentXP >= RequiredXPData then
		LevelValue.Value += 1
		Client:SetAttribute("XP", 250) -- 250 is the starting bonus for the new level

		local LevelRewards = NPCLib[LevelValue.Value]
		if LevelRewards and LevelRewards.Rewards then
			local FreeCash      = LevelRewards.Rewards.Cash
			local FreeDisguises = LevelRewards.Rewards.Disguises

			if FreeCash and FreeDisguises then
				local csh: NumberValue = leaderstats:FindFirstChild("Cash")
				if csh then
					csh.Value += FreeCash
				end
				Client:SetAttribute("DisguiseAmount", (Client:GetAttribute("DisguiseAmount") or 0) + FreeDisguises)
			end
		end

		LeveledUpData = true

		-- level 4 unlocks pickpocketing
		if LevelValue.Value == 4 then
			task.spawn(function()
				pcall(function()
					Communication.Functions.NotificationFromServer:InvokeClient(Client, "Player pickpocketing unlocked, watch your pockets..", 5, Color3.fromRGB(255, 0, 0))
					task.wait(0.3)
					Communication.Functions.NotificationFromServer:InvokeClient(Client, "You can't attack cops, you will be arrested instantly!", 6, Color3.fromRGB(255, 0, 0))
				end)
			end)

			task.spawn(function()
				pcall(function()
					Communication.Functions.PromtClientPopup:Invoke("Equip the slap tool!", {[1] = "Ok!"})
				end)
			end)

			local suc, err = pcall(function()
				return Communication.Functions.ForceRedeemTool:Invoke(Client, "Slap")
			end)
			if not suc then
				warn("ReviewLevel: ForceRedeemTool failed —", err)
			end
		end
	end

	return LeveledUpData, CurrentLevelData, RequiredXPData, CurrentXPData
end

--// RewardXP
-- adds xp, checks for levelup, fires the xp bar update to client

local RewardXP = function(Client: Player, Amount: number)
	local s, e = pcall(function()
		local XP: number = Client:GetAttribute("XP")
		if not XP then return end

		Amount = math.floor(Amount)
		Client:SetAttribute("XP", math.floor(XP) + Amount)

		Communication.Functions.NotificationFromServer:InvokeClient(
			Client, `+{Amount} XP!`, 2, Color3.fromRGB(85, 255, 0)
		)

		local LeveledUp, CurrentLevel, RequiredXP, CurrentXP = ReviewLevel(Client)
		Communication.Remotes.ClientLevelProgressUpdate:FireClient(Client, LeveledUp, CurrentLevel, RequiredXP, CurrentXP)
	end)

	if not s then
		warn("RewardXP error —", e)
	end
end

--// HandleWantedUpdate
-- the main state machine, handles going wanted and getting arrested/escaping

local function HandleWantedUpdate(client: Player, isWanted: boolean, itemData: ItemType?, npcData: {NPC: Model}?, Forced: boolean)
	if not RunService:IsServer() then
		warn("HandleWantedUpdate: called from client — rejected")
		return false
	end

	if not client or not client.Parent then
		warn("HandleWantedUpdate: invalid client")
		return false
	end

	local wasWanted = client:GetAttribute("Wanted") == true

	-- calc xp before any state changes so the value stays consistent
	local calculatedXP = (function()
		local bestItemWorth         = client:GetAttribute("BestItemWorth") or 0
		local HasRewardedXPMultiply = client:GetAttribute("5xXPMultiply")
		local BaseXP                = math.floor(bestItemWorth * 1.2)

		-- 5x multiplier lasts 5 minutes
		if HasRewardedXPMultiply and (tick() - HasRewardedXPMultiply) < 300 then
			BaseXP = math.floor(BaseXP * 5)
		end

		return BaseXP
	end)()

	if isWanted then

		pcall(function()
			Communication.Functions.ToggleWanted:InvokeClient(client, true)
		end)

		client:SetAttribute("Wanted", true)

		if npcData and npcData.NPC then
			NPCChat(npcData.NPC, GetResponse())
		end

		pcall(function()
			Communication.Functions.ForceCancelPickpocket:InvokeClient(client)
		end)

		-- new players get a small speed boost
		if client:GetAttribute("NewerThan5Days") then
			client:SetAttribute("BoostedSpeed", 17.5)
		end

		-- cancel old timer before making a new case, avoids two timers running at once
		local existingCase = _G.ActiveCriminalCases[client.UserId]
		if existingCase then
			if existingCase.TimerThread then
				task.cancel(existingCase.TimerThread)
			end
			_G.ActiveCriminalCases[client.UserId] = nil
		end

		-- token lets the timer verify its still working on the right case
		local uniqueToken   = TokenGenerator.Generate() or (tostring(tick()) .. "_" .. tostring(math.random(1000, 9999)))
		local timerDuration = client:GetAttribute("BestItemWorth") and TIMER_HAS_ITEM or TIMER_NO_ITEM

		_G.ActiveCriminalCases[client.UserId] = {
			RegisterTime       = tick(),
			Client             = client,
			Item               = itemData,
			NPC                = npcData,
			VotesToStopChasing = 0,
			TimerThread        = nil,
			Token              = uniqueToken,
			IsActive           = true,
			TimerDuration      = timerDuration,
		}

		-- fires when player survives the full timer without getting caught
		local timerThread = task.delay(timerDuration, function()
			local currentCase = _G.ActiveCriminalCases[client.UserId]

			-- abort if this is a stale timer from an old case
			if not currentCase or currentCase.Token ~= uniqueToken or not currentCase.IsActive then
				return
			end

			if not client or not client.Parent or not client:GetAttribute("Wanted") then
				return
			end

			currentCase.IsActive = false
			client:SetAttribute("Wanted", false)

			pcall(function()
				Communication.Functions.ToggleWanted:InvokeClient(client, false)
			end)

			-- reset walk speed
			pcall(function()
				local Character = client.Character
				if Character then
					local Humanoid: Humanoid = Character:FindFirstChild("Humanoid")
					if Humanoid then
						client:SetAttribute("BoostedSpeed", false)
						Humanoid.WalkSpeed = StarterPlayer.CharacterWalkSpeed or 16.5
					end
				end
			end)

			_G.ActiveCriminalCases[client.UserId] = nil

			pcall(function()
				Communication.Functions.NotificationFromServer:InvokeClient(
					client, "You escaped! The police stopped chasing you.",
					NOTIFICATION_DURATION, Color3.fromRGB(0, 255, 0)
				)
			end)

			RewardXP(client, calculatedXP)

			if CountActiveCases() == 0 then
				ChangePoliceAttitude(false)
			end
		end)

		_G.ActiveCriminalCases[client.UserId].TimerThread = timerThread

		-- update nameplates for everyone so they can see whos wanted
		pcall(function()
			for _, v in pairs(Players:GetPlayers()) do
				Communication.Functions.UpdatePlayerHeadGui:InvokeClient(v, client.Name)
			end
		end)

		if itemData and itemData.Name then
			local log = _G.PickpocketResultLog[client.UserId] or {}
			table.insert(log, false)
			_G.PickpocketResultLog[client.UserId] = log

			pcall(function()
				Communication.Functions.NotificationFromServer:InvokeClient(
					client, "Pickpocket failed! Run or lose an item!", 3, WANTED_COLOR
				)
			end)
		end

		-- only flip police to chase on the first case, theyre already chasing after that
		if CountActiveCases() == 1 then
			ChangePoliceAttitude(true)
		end

	else

		pcall(function()
			Communication.Functions.ToggleWanted:InvokeClient(client, false)
		end)

		client:SetAttribute("Wanted", false)

		-- reset walk speed
		pcall(function()
			local Character = client.Character
			if Character then
				local Humanoid: Humanoid = Character:FindFirstChild("Humanoid")
				if Humanoid then
					client:SetAttribute("BoostedSpeed", false)
					Humanoid.WalkSpeed = StarterPlayer.CharacterWalkSpeed or 16.5
				end
			end
		end)

		local criminalCase = _G.ActiveCriminalCases[client.UserId]

		if criminalCase then
			criminalCase.IsActive = false

			-- cancel escape timer so it dont award xp after arrest
			if criminalCase.TimerThread then
				task.cancel(criminalCase.TimerThread)
				criminalCase.TimerThread = nil
			end

			local success, lootData = pcall(function()
				return Communication.Functions.GetClientLootBag:Invoke(client) or {}
			end)
			if not success then lootData = {} end

			if wasWanted then
				-- player was actually chased and caught, take an item and jail them
				if lootData and type(lootData) == "table" and next(lootData) then
					local itemWorth            = criminalCase.Item and criminalCase.Item.Worth or 0
					local similarItem, itemKey = GetSimilarValueItem(lootData, itemWorth)

					-- fallback to random item if worth matching fails
					if not similarItem then
						itemKey     = GetRandomKey(lootData)
						similarItem = itemKey and lootData[itemKey] or nil
					end

					if similarItem and similarItem.Name then
						local removeSuccess = pcall(function()
							Communication.Functions.DeregisterLoot:Invoke(client, {
								Name   = similarItem.Name,
								Rarity = similarItem.Rarity,
								Worth  = similarItem.Worth,
							})
						end)

						if removeSuccess then
							pcall(function()
								Communication.Functions.NotificationFromServer:InvokeClient(
									client,
									string.format("The cops took your %s!", similarItem.Name),
									NOTIFICATION_DURATION, WANTED_COLOR
								)
							end)

							-- small delay so player can read the notification before getting jailed
							task.delay(2, function()
								pcall(function()
									Communication.Functions.ThrowInJail:Invoke(client, similarItem)
								end)
							end)
						end
					else
						-- loot exists but data is broken, jail with no item
						pcall(function()
							Communication.Functions.NotificationFromServer:InvokeClient(
								client, "You had no loot, but you're still going to jail!",
								NOTIFICATION_DURATION, WANTED_COLOR
							)
						end)
						task.delay(2, function()
							pcall(function() Communication.Functions.ThrowInJail:Invoke(client, nil) end)
						end)
					end
				else
					-- empty loot bag, still jail them
					pcall(function()
						Communication.Functions.NotificationFromServer:InvokeClient(
							client, "You had no loot, but you're still going to jail!",
							NOTIFICATION_DURATION, WANTED_COLOR
						)
					end)
					task.delay(2, function()
						pcall(function() Communication.Functions.ThrowInJail:Invoke(client, nil) end)
					end)
				end
			else
				-- force clear, give badge and xp as courtesy
				pcall(function()
					if not BadgeService:UserHasBadgeAsync(client.UserId, ESCAPE_BADGE_ID) then
						BadgeService:AwardBadge(client.UserId, ESCAPE_BADGE_ID)
					end
				end)
				RewardXP(client, calculatedXP)
			end

			_G.ActiveCriminalCases[client.UserId] = nil
		end

		if CountActiveCases() == 0 then
			ChangePoliceAttitude(false)
		end
	end

	return true
end

--// VoteToStopChasing

local function VoteToStopChasing(client: Player, reason: string?)
	if not RunService:IsServer() then
		warn("VoteToStopChasing: called from client — rejected")
		return false
	end

	if not client or not client.Parent then
		warn("VoteToStopChasing: invalid client")
		return false
	end

	local criminalCase = _G.ActiveCriminalCases[client.UserId]
	if not criminalCase or not criminalCase.IsActive then return false end

	if not client:GetAttribute("Wanted") then return false end

	criminalCase.VotesToStopChasing = (criminalCase.VotesToStopChasing or 0) + 1
	local newVotes = criminalCase.VotesToStopChasing

	if RunService:IsStudio() then
		warn(string.format("[Studio] Vote for %s: %d/%d", client.Name, newVotes, VOTES_NEEDED_TO_ESCAPE))
	end

	if newVotes >= VOTES_NEEDED_TO_ESCAPE and client:GetAttribute("Wanted") and criminalCase.IsActive then
		pcall(function()
			if not BadgeService:UserHasBadgeAsync(client.UserId, ESCAPE_BADGE_ID) then
				BadgeService:AwardBadge(client.UserId, ESCAPE_BADGE_ID)
			end
		end)

		HandleWantedUpdate(client, false)
		return true
	end

	return false
end

--// CleanupDisconnectedPlayers

local function CleanupDisconnectedPlayers()
	local playersRemoved = false

	for userId, criminalCase in pairs(_G.ActiveCriminalCases) do
		local player = Players:GetPlayerByUserId(userId)

		if not player or not player.Parent then
			if criminalCase.TimerThread then
				task.cancel(criminalCase.TimerThread)
				criminalCase.TimerThread = nil
			end

			criminalCase.IsActive = false
			_G.ActiveCriminalCases[userId] = nil
			playersRemoved = true
		end
	end

	if playersRemoved and CountActiveCases() == 0 then
		ChangePoliceAttitude(false)
	end
end

--// Player Removing

Players.PlayerRemoving:Connect(function(player)
	if not player then return end

	local criminalCase = _G.ActiveCriminalCases[player.UserId]
	if criminalCase then
		if criminalCase.TimerThread then
			task.cancel(criminalCase.TimerThread)
			criminalCase.TimerThread = nil
		end

		criminalCase.IsActive = false
		_G.ActiveCriminalCases[player.UserId] = nil
		_G.spawnedAmount[player.UserId]       = nil

		if CountActiveCases() == 0 then
			ChangePoliceAttitude(false)
		end
	end
end)

--// Loops

-- cleans up disconnected players every 20s
task.spawn(function()
	while true do
		task.wait(20)
		pcall(CleanupDisconnectedPlayers)
	end
end)

-- force resolves cases open longer than 2 mins, last resort for stuck cases
task.spawn(function()
	while true do
		task.wait(60)
		pcall(function()
			local now = tick()
			for userId, criminalCase in pairs(_G.ActiveCriminalCases) do
				if criminalCase.RegisterTime and (now - criminalCase.RegisterTime) > 120 then
					local player = Players:GetPlayerByUserId(userId)
					if player and player.Parent then
						warn(string.format(
							"Stale case for %s (%.1fs old) — force resolving",
							player.Name, now - criminalCase.RegisterTime
							))
						HandleWantedUpdate(player, false)
					end
				end
			end
		end)
	end
end)

--// Remote Binding

-- ive got into the habit of using this, should probably just use a networking module atp
local function SafeBindRemote(remote, func)
	if not remote then return end

	remote.OnInvoke = function(...)
		local success, result = pcall(func, ...)
		if not success then
			warn("Remote error on", remote.Name, "—", result)
			return false
		end
		return result
	end
end

SafeBindRemote(Communication.Functions.SetWanted,         HandleWantedUpdate)
SafeBindRemote(Communication.Functions.VoteToStopChasing, VoteToStopChasing)
