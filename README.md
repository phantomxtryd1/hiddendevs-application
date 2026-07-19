# this is module script in serverscript storage

return {
	["Gun"] = {
		Damage = 10,
		Range = 250,

		MaxAmmo = 10000000000000,
		LoadAmmo = 15,
		ReloadTime = 2,

		FireMode = "Auto",

		FireRate = 0.1,
		
		Category = "Pistol",
		
		Price = 0,
		
		Sounds = {
			["FireSound"] = "",
			["ReloadSound"] = "",
			["EquipSound"] = "",
		},
		
		Animation = {
			["Idle"] = "",
			["Fire"] = "",
			["Reload"] = "",
			["Equip"] = "",
		}
	}
}





# this is script under the gundata module

local fireRemote = game.ReplicatedStorage.Main.Remotes.RemoteEvents.Gun.Fire
local handleSemi = game.ReplicatedStorage.Main.Remotes.RemoteEvents.Gun.HandleSemi
local reloadRemote = game.ReplicatedStorage.Main.Remotes.RemoteEvents.Gun.Reload
local hitAnimation = game.ReplicatedStorage.Main.Remotes.RemoteEvents.Gun.HitAnimation

local handleGunVfx = game.ReplicatedStorage.Main.Remotes.RemoteEvents.Gun.HandleGunVFX

local playersThatCantFire = {}
local playersLoadedGunAmmo = {}
local playersTotalGunAmmo = {}
local playersReloading = {}

local gunData = require(script.Parent)

local function fireGun(player, origin, direction, gun, tool)
	if not player or not origin or not direction then return end

	print(gun)
	print(gun.Range)
	local raycastParams = RaycastParams.new()
	local excluded = {player.Character}
	for i, v in ipairs(game.Players:GetPlayers()) do
		if v ~= player then
			local playerChar = v.Character
			if playerChar then
				for _, charPart in playerChar:GetDescendants() do
					if charPart:IsA("Accessory") then
						for _, accessoryPart in charPart:GetDescendants() do
							if accessoryPart:IsA("BasePart") then
								table.insert(excluded, accessoryPart)
							end
						end
					elseif charPart:IsA("BasePart") and (charPart.Name == "HumanoidRootPart" or charPart.Name == "CollisionPart") then
						table.insert(excluded, charPart)					
					end
				end
			end
		end
	end
	raycastParams.FilterDescendantsInstances = excluded
	raycastParams.FilterType = Enum.RaycastFilterType.Exclude
	
	local rayDirection = direction.Unit * gun.Range

	local rayResult = workspace:Raycast(origin, rayDirection, raycastParams)

	local endPosition

	if rayResult then
		endPosition = rayResult.Position

		local hitPart = rayResult.Instance
		local hitCharacter = hitPart:FindFirstAncestorOfClass("Model")

		if hitCharacter then
			local hitPlayer = game.Players:GetPlayerFromCharacter(hitCharacter)

			if hitPlayer then
				print("Hit player: " .. hitPlayer.Name)
				print("Hit CharPart: " .. hitPart.Name)
				local humanoid = hitCharacter:FindFirstChildOfClass("Humanoid")
				if humanoid and humanoid.Health > 0 then
					humanoid:TakeDamage(gun.Damage)
				end
				local highlight = Instance.new("Highlight")
				highlight.OutlineColor = Color3.fromRGB(0, 0, 0)
				highlight.DepthMode = Enum.HighlightDepthMode.Occluded
				highlight.Parent = hitPart
				
				if hitPart.Name == "Head" then
					hitAnimation:FireClient(player, "HeadShot")
					print("fired")
				else
					hitAnimation:FireClient(player, "Normal")
					print("fired")
				end
				
				game:GetService("Debris"):AddItem(highlight, 0.5)
			end
		else
			print("Hit object: " .. hitPart.Name)
		end
	else
		endPosition = origin + rayDirection
	end
	local distance = (endPosition - origin).Magnitude
	
	handleGunVfx:FireAllClients("BulletTrail", {EndPos = endPosition, Gun = tool})
end

fireRemote.OnServerEvent:Connect(function(player, origin, direction)
	local character = player.Character
	if not character then return end

	local tool = character:FindFirstChildOfClass("Tool")
	if not tool or not tool:HasTag("Gun") then return end

	local gun = gunData[tool.Name]
	if not gun then return end
	
	local assetFolder = tool:FindFirstChild("Assets")
	if not assetFolder then  warn("error") return end
	
	if typeof(origin) ~= "Vector3" then return end
	if typeof(direction) ~= "Vector3" then return end
	
	playersLoadedGunAmmo[player] = playersLoadedGunAmmo[player] or {}
	playersTotalGunAmmo[player] = playersTotalGunAmmo[player] or {}
	playersTotalGunAmmo[player][tool.Name] = playersTotalGunAmmo[player][tool.Name] or gun.MaxAmmo
	playersLoadedGunAmmo[player][tool.Name] = playersLoadedGunAmmo[player][tool.Name] or gun.LoadAmmo
	
	if playersThatCantFire[player] or playersLoadedGunAmmo[player][tool.Name] <= 0 then return end
	
	if playersReloading[player] then return end
	
	if gun.FireMode == "Auto" then
		playersThatCantFire[player] = true
		playersLoadedGunAmmo[player][tool.Name] -= 1
		print(player.Name .. " fired, loaded ammo: " .. playersLoadedGunAmmo[player][tool.Name] .. " total ammo: " .. playersTotalGunAmmo[player][tool.Name])
		
		fireGun(player, origin, direction, gun, tool)
		
		task.delay(gun.FireRate, function()
			playersThatCantFire[player] = nil
		end)
	elseif gun.FireMode == "Semi" then
		playersThatCantFire[player] = tick()
		playersLoadedGunAmmo[player][tool.Name] -= 1
		print(player.Name .. " fired, loaded ammo: " .. playersLoadedGunAmmo[player][tool.Name] .. " total ammo: " .. playersTotalGunAmmo[player][tool.Name])
		
		fireGun(player, origin, direction, gun, tool)
	else
		warn("Unknown FireMode")
		return
	end
end)

handleSemi.OnServerEvent:Connect(function(player)
	local character = player.Character
	if not character then return end

	local tool = character:FindFirstChildOfClass("Tool")
	if not tool or not tool:HasTag("Gun") then return end

	local gun = gunData[tool.Name]
	if not gun then return end
	
	if gun.FireMode ~= "Semi" then return end
	local now = tick()
	if (playersThatCantFire[player] or 0) + gun.FireRate > now then return end
	
	if not playersThatCantFire[player] then return end
	playersThatCantFire[player] = nil
end)

reloadRemote.OnServerEvent:Connect(function(player)
	local character = player.Character
	if not character then return end

	local tool = character:FindFirstChildOfClass("Tool")
	if not tool or not tool:HasTag("Gun") then return end

	local gun = gunData[tool.Name]
	if not gun then return end
	
	playersLoadedGunAmmo[player] = playersLoadedGunAmmo[player] or {}
	playersTotalGunAmmo[player] = playersTotalGunAmmo[player] or {}
	playersTotalGunAmmo[player][tool.Name] = playersTotalGunAmmo[player][tool.Name] or gun.MaxAmmo
	playersLoadedGunAmmo[player][tool.Name] = playersLoadedGunAmmo[player][tool.Name] or gun.LoadAmmo
	
	local ammoNeeded = gun.LoadAmmo - playersLoadedGunAmmo[player][tool.Name]
	if ammoNeeded <= 0 or playersTotalGunAmmo[player][tool.Name] <= 0 then return end
	if playersReloading[player] then return end

	playersReloading[player] = true
	task.wait(gun.ReloadTime)
	playersReloading[player] = nil
	
	local currentCharacter = player.Character
	if not currentCharacter then return end
	local currentTool = currentCharacter:FindFirstChildOfClass("Tool")
	if not currentTool or currentTool.Name ~= tool.Name or not currentTool:HasTag("Gun") then return end
	
	if ammoNeeded > playersTotalGunAmmo[player][tool.Name] then
		playersLoadedGunAmmo[player][tool.Name] = playersTotalGunAmmo[player][tool.Name]
		playersTotalGunAmmo[player][tool.Name] = 0
	elseif ammoNeeded <= playersTotalGunAmmo[player][tool.Name] then
		playersLoadedGunAmmo[player][tool.Name] += ammoNeeded
		playersTotalGunAmmo[player][tool.Name] -= ammoNeeded
	end
	
end)

game.Players.PlayerRemoving:Connect(function(player)
	playersThatCantFire[player] = nil
	playersLoadedGunAmmo[player] = nil
	playersTotalGunAmmo[player] = nil
	playersReloading[player] = nil
end)
