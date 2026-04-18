	local weaponData = {}
	
	-- دالة إحداث ضرر
	local function dealDamage(target, damageAmount, source)
		local humanoid = target:FindFirstChild("Humanoid")
		if humanoid and humanoid.Health > 0 then
			humanoid.Health = math.max(0, humanoid.Health - damageAmount)
			
			-- تأثير احمرار
			for _, part in ipairs(target:GetChildren()) do
				if part:IsA("BasePart") then
					local originalColor = part.Color
					part.Color = Color3.fromRGB(255, 0, 0)
					task.delay(0.1, function()
						if part then part.Color = originalColor end
					end)
				end
			end
			
			-- تأثير دم
			local bloodPart = Instance.new("Part")
			bloodPart.Size = Vector3.new(0.3, 0.3, 0.3)
			bloodPart.Color = Color3.fromRGB(150, 0, 0)
			bloodPart.Material = Enum.Material.SmoothPlastic
			if target.PrimaryPart then
				bloodPart.CFrame = target.PrimaryPart.CFrame
			end
			bloodPart.Anchored = true
			bloodPart.CanCollide = false
			bloodPart.Parent = workspace
			debris:AddItem(bloodPart, 0.5)
			
			-- رسالة قتل
			if humanoid.Health <= 0 then
				local killer = source and source.Parent
				if killer and killer:IsA("Player") then
					local targetPlayer = players:GetPlayerFromCharacter(target)
					if targetPlayer then
						local message = Instance.new("Message")
						message.Text = killer.Name .. " 💀 قتل " .. targetPlayer.Name .. " بـ " .. weaponName
						message.Parent = workspace
						debris:AddItem(message, 3)
					end
				end
			end
			
			return true
		end
		return false
	end
	
	-- دالة إطلاق النار
	local function shoot(player, mousePosition)
		local data = weaponData[player]
		if not data or not data.canShoot or data.isReloading or data.currentAmmo <= 0 then
			if data and data.currentAmmo <= 0 and not data.isReloading then
				reload(player)
			end
			return
		end
		
		local now = tick()
		if now - data.lastShot < fireRate then
			return
		end
		data.lastShot = now
		
		data.currentAmmo = data.currentAmmo - 1
		data.canShoot = false
		
		-- تحديث واجهة المستخدم
		updateUIEvent:FireClient(player, "UpdateAmmo", data.currentAmmo, maxAmmo)
		
		-- حساب اتجاه الرمي
		local character = player.Character
		if not character then return end
		
		local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
		local toolHandle = player.Character:FindFirstChild(weaponName) and player.Character[weaponName]:FindFirstChild("Handle")
		
		if not humanoidRootPart or not toolHandle then return end
		
		local direction = (mousePosition - humanoidRootPart.Position).Unit
		
		-- رصاصة مرئية
		local bullet = Instance.new("Part")
		bullet.Size = Vector3.new(0.2, 0.2, 0.5)
		bullet.Shape = Enum.PartType.Cylinder
		bullet.Color = Color3.fromRGB(255, 255, 0)
		bullet.Material = Enum.Material.Neon
		bullet.CFrame = CFrame.new(toolHandle.Position, toolHandle.Position + direction)
		bullet.CanCollide = false
		bullet.Anchored = false
		bullet.Parent = workspace
		
		local pointLight = Instance.new("PointLight")
		pointLight.Range = 10
		pointLight.Brightness = 2
		pointLight.Color = Color3.fromRGB(255, 200, 0)
		pointLight.Parent = bullet
		
		local bodyVelocity = Instance.new("BodyVelocity")
		bodyVelocity.MaxForce = Vector3.new(1, 1, 1) * 1000
		bodyVelocity.Velocity = direction * bulletSpeed
		bodyVelocity.Parent = bullet
		
		-- Raycast للضرر
		local rayOrigin = toolHandle.Position
		local rayDirection = direction * range
		local raycastParams = RaycastParams.new()
		raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
		raycastParams.FilterDescendantsInstances = {character, tool}
		
		local raycastResult = workspace:Raycast(rayOrigin, rayDirection, raycastParams)
		
		if raycastResult then
			local hitPart = raycastResult.Instance
			local hitCharacter = hitPart.Parent
			
			if hitCharacter and hitCharacter:FindFirstChild("Humanoid") then
				local targetPlayer = players:GetPlayerFromCharacter(hitCharacter)
				if targetPlayer and targetPlayer ~= player then
					dealDamage(hitCharacter, damage, tool)
					
					local impact = Instance.new("Part")
					impact.Size = Vector3.new(0.3, 0.3, 0.3)
					impact.Color = Color3.fromRGB(255, 100, 0)
					impact.Material = Enum.Material.Neon
					impact.CFrame = CFrame.new(raycastResult.Position)
					impact.Anchored = true
					impact.CanCollide = false
					impact.Parent = workspace
					debris:AddItem(impact, 0.2)
				end
			end
		end
		
		debris:AddItem(bullet, 0.5)
		
		task.delay(fireRate, function()
			if weaponData[player] then
				weaponData[player].canShoot = true
			end
		end)
	end
	
	-- دالة إعادة التلقيم
	local function reload(player)
		local data = weaponData[player]
		if not data or data.isReloading or data.currentAmmo == maxAmmo then
			return
		end
		
		data.isReloading = true
		data.canShoot = false
		
		updateUIEvent:FireClient(player, "StartReload", reloadTime)
		
		task.wait(reloadTime)
		
		if weaponData[player] then
			data.currentAmmo = maxAmmo
			data.isReloading = false
			data.canShoot = true
			updateUIEvent:FireClient(player, "UpdateAmmo", data.currentAmmo, maxAmmo)
			updateUIEvent:FireClient(player, "EndReload")
		end
	end
	
	-- أحداث الأداة
	tool.Equipped:Connect(function(player)
		if not weaponData[player] then
			weaponData[player] = {
				currentAmmo = maxAmmo,
				isReloading = false,
				canShoot = true,
				lastShot = 0
			}
		else
			weaponData[player].canShoot = true
			weaponData[player].isReloading = false
		end
		
		updateUIEvent:FireClient(player, "UpdateAmmo", weaponData[player].currentAmmo, maxAmmo)
		updateUIEvent:FireClient(player, "ShowUI")
	end)
	
	tool.Unequipped:Connect(function(player)
		if weaponData[player] then
			weaponData[player].canShoot = false
		end
		updateUIEvent:FireClient(player, "HideUI")
	end)
	
	-- استقبال أحداث من العميل
	shootEvent.OnServerEvent:Connect(function(player, mousePosition)
		if player.Character and player.Character:FindFirstChild(weaponName) then
			shoot(player, mousePosition)
		end
	end)
	
	reloadEvent.OnServerEvent:Connect(function(player)
		if player.Character and player.Character:FindFirstChild(weaponName) then
			reload(player)
		end
	end)
	
	return tool
end

-- ========== كود العميل (LocalScript) داخل نفس الكود ==========
local function createClientScript(player)
	local localScript = Instance.new("LocalScript")
	localScript.Name = "WeaponClient"
	
	local scriptCode = [[
		local player = game.Players.LocalPlayer
		local mouse = player:GetMouse()
		local replicatedStorage = game:GetService("ReplicatedStorage")
		local userInputService = game:GetService("UserInputService")
		local runService = game:GetService("RunService")
		
		local shootEvent = replicatedStorage:WaitForChild("ShootEvent")
		local reloadEvent = replicatedStorage:WaitForChild("ReloadEvent")
		local updateUIEvent = replicatedStorage:WaitForChild("UpdateUIEvent")
		
		local currentTool = nil
		local ui = nil
		local isAiming = false
		local originalMouseIcon = "rbxasset://textures/arrow.png"
		
		-- إنشاء الواجهة
		local function createUI()
			local screenGui = Instance.new("ScreenGui")
			screenGui.Name = "WeaponUI"
			screenGui.ResetOnSpawn = false
			screenGui.Parent = player:WaitForChild("PlayerGui")
			
			-- إطار الذخيرة
			local ammoFrame = Instance.new("Frame")
			ammoFrame.Size = UDim2.new(0, 150, 0, 60)
			ammoFrame.Position = UDim2.new(0.5, -75, 1, -80)
			ammoFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
			ammoFrame.BackgroundTransparency = 0.4
			ammoFrame.BorderSizePixel = 0
			ammoFrame.Parent = screenGui
			
			local ammoCorner = Instance.new("UICorner")
			ammoCorner.CornerRadius = UDim.new(0, 10)
			ammoCorner.Parent = ammoFrame
			
			local ammoLabel = Instance.new("TextLabel")
			ammoLabel.Name = "AmmoLabel"
			ammoLabel.Size = UDim2.new(1, 0, 1, 0)
			ammoLabel.BackgroundTransparency = 1
			ammoLabel.Text = "30 / 30"
			ammoLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
			ammoLabel.TextScaled = true
			ammoLabel.Font = Enum.Font.GothamBold
			ammoLabel.Parent = ammoFrame
			
			-- شريط إعادة التلقيم
			local reloadBar = Instance.new("Frame")
			reloadBar.Name = "ReloadBar"
			reloadBar.Size = UDim2.new(0, 0, 0, 5)
			reloadBar.Position = UDim2.new(0, 0, 1, 0)
			reloadBar.BackgroundColor3 = Color3.fromRGB(255, 100, 0)
			reloadBar.BorderSizePixel = 0
			reloadBar.Parent = ammoFrame
			
			-- زر إطلاق النار
			local shootButton = Instance.new("ImageButton")
			shootButton.Size = UDim2.new(0, 90, 0, 90)
			shootButton.Position = UDim2.new(1, -110, 1, -110)
			shootButton.AnchorPoint = Vector2.new(1, 1)
			shootButton.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
			shootButton.BackgroundTransparency = 0.3
			shootButton.BorderSizePixel = 0
			shootButton.Image = "rbxasset://textures/ui/GuiImagePlaceholder.png"
			shootButton.Parent = screenGui
			
			local shootCorner = Instance.new("UICorner")
			shootCorner.CornerRadius = UDim.new(1, 0)
			shootCorner.Parent = shootButton
			
			local shootText = Instance.new("TextLabel")
			shootText.Size = UDim2.new(1, 0, 1, 0)
			shootText.BackgroundTransparency = 1
			shootText.Text = "🔫"
			shootText.TextColor3 = Color3.fromRGB(255, 255, 255)
			shootText.TextScaled = true
			shootText.Font = Enum.Font.GothamBold
			shootText.Parent = shootButton
			
			-- زر إعادة التلقيم
			local reloadButton = Instance.new("ImageButton")
			reloadButton.Size = UDim2.new(0, 65, 0, 65)
			reloadButton.Position = UDim2.new(0, 20, 1, -110)
			reloadButton.AnchorPoint = Vector2.new(0, 1)
			reloadButton.BackgroundColor3 = Color3.fromRGB(50, 50, 255)
			reloadButton.BackgroundTransparency = 0.3
			reloadButton.BorderSizePixel = 0
			reloadButton.Parent = screenGui
			
			local reloadCorner = Instance.new("UICorner")
			reloadCorner.CornerRadius = UDim.new(1, 0)
			reloadCorner.Parent = reloadButton
			
			local reloadText = Instance.new("TextLabel")
			reloadText.Size = UDim2.new(1, 0, 1, 0)
			reloadText.BackgroundTransparency = 1
			reloadText.Text = "🔄"
			reloadText.TextColor3 = Color3.fromRGB(255, 255, 255)
			reloadText.TextScaled = true
			reloadText.Font = Enum.Font.GothamBold
			reloadText.Parent = reloadButton
			
			--十字瞄准器 (Crosshair)
			local crosshair = Instance.new("Frame")
			crosshair.Size = UDim2.new(0, 30, 0, 30)
			crosshair.Position = UDim2.new(0.5, -15, 0.5, -15)
			crosshair.BackgroundTransparency = 1
			crosshair.Parent = screenGui
			
			local line1 = Instance.new("Frame")
			line1.Size = UDim2.new(0, 20, 0, 3)
			line1.Position = UDim2.new(0.5, -10, 0, 13.5)
			line1.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
			line1.Parent = crosshair
			
			local line2 = Instance.new("Frame")
			line2.Size = UDim2.new(0, 3, 0, 20)
			line2.Position = UDim2.new(0, 13.5, 0.5, -10)
			line2.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
			line2.Parent = crosshair
			
			-- إخفاء الواجهة افتراضياً
			screenGui.Enabled = false
			
			return {
				screenGui = screenGui,
				ammoLabel = ammoLabel,
				reloadBar = reloadBar,
				shootButton = shootButton,
				reloadButton = reloadButton,
				crosshair = crosshair
			}
		end
		
		-- أحداث واجهة المستخدم
		updateUIEvent.OnClientEvent:Connect(function(action, data)
			if action == "ShowUI" and ui then
				ui.screenGui.Enabled = true
				player.PlayerGui:FindFirstChild("WeaponUI").Enabled = true
			elseif action == "HideUI" and ui then
				ui.screenGui.Enabled = false
			elseif action == "UpdateAmmo" and ui then
				ui.ammoLabel.Text = data .. " / " .. data2
			elseif action == "StartReload" and ui then
				ui.reloadBar.Size = UDim2.new(0, 0, 0, 5)
				local tweenService = game:GetService("TweenService")
				local tween = tweenService:Create(ui.reloadBar, TweenInfo.new(data, Enum.EasingStyle.Linear), {Size = UDim2.new(1, 0, 0, 5)})
				tween:Play()
			elseif action == "EndReload" and ui then
				ui.reloadBar.Size = UDim2.new(0, 0, 0, 5)
			end
		end)
		
		-- أحداث الأزرار
		local function setupButtons()
			if not ui then return end
			
			ui.shootButton.MouseButton1Down:Connect(function()
				if currentTool then
					shootEvent:FireServer(mouse.Hit.Position)
				end
			end)
			
			ui.reloadButton.MouseButton1Click:Connect(function()
				if currentTool then
					reloadEvent:FireServer()
				end
			end)
		end
		
		-- مراقبة تغيير الأداة
		player.CharacterAdded:Connect(function(character)
			task.wait(0.5)
			character:WaitForChild("Humanoid").ToolEquipped:Connect(function(tool)
				if tool.Name == "AssaultRifle" then
					currentTool = tool
					if not ui then
						ui = createUI()
						setupButtons()
					end
					updateUIEvent:FireServer("ShowUI")
				end
			end)
			
			character:WaitForChild("Humanoid").ToolUnequipped:Connect(function()
				currentTool = nil
				updateUIEvent:FireServer("HideUI")
			end)
		end)
		
		-- إطلاق النار بزر الماوس أيضاً
		mouse.Button1Down:Connect(function()
			if currentTool then
				shootEvent:FireServer(mouse.Hit.Position)
			end
		end)
		
		-- زر R لإعادة التلقيم
		userInputService.InputBegan:Connect(function(input, gameProcesse
