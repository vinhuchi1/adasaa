local MarketplaceService = game:GetService("MarketplaceService")

local AUTO_COLLECT_GAMEPASS_ID = 12345678 -- Thay bằng ID gamepass thật

local PlayersWithPass = {}

game.Players.PlayerAdded:Connect(function(player)
	local success, hasPass = pcall(function()
		return MarketplaceService:UserOwnsGamePassAsync(player.UserId, AUTO_COLLECT_GAMEPASS_ID)
	end)

	if success and hasPass then
		PlayersWithPass[player] = true
	end
end)

game.Players.PlayerRemoving:Connect(function(player)
	PlayersWithPass[player] = nil
end)

return PlayersWithPass
local Players = game:GetService("Players")
local Debris = game:GetService("Debris")
local Workspace = game:GetService("Workspace")

local PlayersWithPass = require(script.Parent:WaitForChild("GamepassHandler"))
local COLLECTION_RADIUS = 100 -- khoảng cách (studs) trong đó trái sẽ bị hút

-- Giả sử trái rơi được parent vào Folder "Fruits"
local fruitFolder = Workspace:WaitForChild("Fruits")

-- Hàm kiểm tra và hút trái
local function checkAndCollectFruits()
	for _, fruit in pairs(fruitFolder:GetChildren()) do
		if fruit:IsA("BasePart") then
			for player, _ in pairs(PlayersWithPass) do
				local character = player.Character
				if character and character:FindFirstChild("HumanoidRootPart") then
					local dist = (character.HumanoidRootPart.Position - fruit.Position).Magnitude
					if dist < COLLECTION_RADIUS then
						-- Di chuyển trái tới người chơi
						fruit.CFrame = character.HumanoidRootPart.CFrame + Vector3.new(0, 2, 0)
						
						-- Tùy game bạn: trigger hệ thống "nhặt"
						if fruit:FindFirstChild("TouchInterest") then
							-- giả lập chạm
							firetouchinterest(fruit, character:FindFirstChild("HumanoidRootPart"), 0)
							firetouchinterest(fruit, character:FindFirstChild("HumanoidRootPart"), 1)
						end

						-- Gỡ trái khỏi map nếu cần
						Debris:AddItem(fruit, 1)
						break
					end
				end
			end
		end
	end
end

-- Lặp lại mỗi vài giây

while true do
	checkAndCollectFruits()
	wait(5)
end
local DataStoreService = game:GetService("DataStoreService")
local InventoryStore = DataStoreService:GetDataStore("FruitInventory")

local InventoryManager = {}

function InventoryManager:GetInventory(userId)
	local success, data = pcall(function()
		return InventoryStore:GetAsync("inv_" .. userId)
	end)
	
	if success then
		return data or {}
	else
		warn("Failed to load inventory for userId:", userId)
		return {}
	end
end

function InventoryManager:HasFruit(userId, fruitName)
	local inventory = self:GetInventory(userId)
	for _, fruit in pairs(inventory) do
		if fruit == fruitName then
			return true
		end
	end
	return false
end

function InventoryManager:AddFruit(userId, fruitName)
	local inventory = self:GetInventory(userId)
	
	for _, fruit in pairs(inventory) do
		if fruit == fruitName then
			return false -- đã có rồi
		end
	end
	
	table.insert(inventory, fruitName)
	
	pcall(function()
		InventoryStore:SetAsync("inv_" .. userId, inventory)
	end)

	return true
end

return InventoryManager
local TeleportService = game:GetService("TeleportService")
local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")

local PlayersWithPass = require(script.Parent:WaitForChild("GamepassHandler"))
local InventoryManager = require(game.ServerStorage:WaitForChild("InventoryManager"))

local PLACE_ID = 12345678 -- placeId của game bạn (để server-hop)

local function getRandomServer()
	local servers = TeleportService:GetPlayerPlaceInstancesAsync(PLACE_ID)
	local list = servers:GetCurrentPage()
	for _, server in pairs(list) do
		if server.Playing < server.MaxPlayers then
			return server.Guid
		end
	end
	return nil
end

local function hopServer(player)
	local serverId = getRandomServer()
	if serverId then
		TeleportService:TeleportToPlaceInstance(PLACE_ID, serverId, player)
	else
		-- fallback: về lại main place
		TeleportService:Teleport(PLACE_ID, player)
	end
end

local COLLECTION_RADIUS = 100
local fruitFolder = Workspace:WaitForChild("Fruits")

local function checkAndCollectFruits()
	for _, fruit in pairs(fruitFolder:GetChildren()) do
		if fruit:IsA("BasePart") and fruit:FindFirstChild("FruitName") then
			local fruitName = fruit.FruitName.Value

			for player, _ in pairs(PlayersWithPass) do
				local character = player.Character
				if character and character:FindFirstChild("HumanoidRootPart") then
					local dist = (character.HumanoidRootPart.Position - fruit.Position).Magnitude
					if dist < COLLECTION_RADIUS then
						local userId = player.UserId
						if InventoryManager:HasFruit(userId, fruitName) then
							hopServer(player)
							return
						end

						-- Di chuyển trái đến người chơi
						fruit.CFrame = character.HumanoidRootPart.CFrame + Vector3.new(0, 2, 0)

						-- Thêm vào inventory
						InventoryManager:AddFruit(userId, fruitName)

						-- Xóa trái khỏi map
						fruit:Destroy()
						break
					end
				end
			end
		end
	end
end

while true do
	checkAndCollectFruits()
	wait(5)
end
local fruit = Instance.new("Part")
fruit.Name = "DevilFruit"

local fruitName = Instance.new("StringValue")
fruitName.Name = "FruitName"
fruitName.Value = "Mera" -- hoặc trái khác
fruitName.Parent = fruit
