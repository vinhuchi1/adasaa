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
