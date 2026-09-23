MY SCRIPT IS NOT INDEPENDENT THIS IS JUST THE MAIN SCRIPT PRESS Q TO LAUNCH WHEN IN GAME!!!!!!!!!!!!!!!!

local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local beybladeEvent = ReplicatedStorage:WaitForChild("BeybladeEvent")
local template = ReplicatedStorage:WaitForChild("Bloodfang")

local MAX_STAMINA = 100
local DRAIN_RATE = 1.5
local BASE_SPIN = 14
local MOVE_SPEED = 22
local DEATH_TIME = 6
local ACCEL = 55
local FRICTION = 18
local TIP_Y_OFFSET = 1.58
local FLAT_Y_OFFSET = 1.61

local activeBlades = {}

local function lerpAngle(current, target, t)
	local diff = (target - current) % (math.pi * 2)
	if diff > math.pi then diff -= math.pi * 2 end
	if diff < -math.pi then diff += math.pi * 2 end
	return current + diff * t
end

local function lockCharacter(player, lock)
	local char = player.Character
	if not char then return end
	local humanoid = char:FindFirstChildOfClass("Humanoid")
	if not humanoid then return end
	if lock then
		humanoid.WalkSpeed = 0
		humanoid.JumpPower = 0
		humanoid.JumpHeight = 0
		humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, false)
		local root = char:FindFirstChild("HumanoidRootPart")
		if root then
			root.AssemblyLinearVelocity = Vector3.zero
		end
	else
		humanoid.WalkSpeed = 16
		humanoid.JumpPower = 50
		humanoid.JumpHeight = 7.2
		humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, true)
	end
end

local function launch(player)
	if activeBlades[player.UserId] then return end

	local char = player.Character
	if not char then return end
	local root = char:FindFirstChild("HumanoidRootPart")
	if not root then return end

	local blade = template:Clone()
	blade.Name = "Beyblade"
	blade.Parent = workspace

	for _, p in ipairs(blade:GetDescendants()) do
		if p:IsA("BasePart") then
			p.Anchored = true
			p.CanCollide = false
		end
	end

	local spawnPos = root.Position + root.CFrame.LookVector * 6
	spawnPos = Vector3.new(spawnPos.X, TIP_Y_OFFSET, spawnPos.Z)
	blade:PivotTo(CFrame.new(spawnPos) * CFrame.Angles(math.rad(90), 0, 0))

	blade:SetAttribute("Stamina", MAX_STAMINA)
	blade:SetAttribute("Owner", player.UserId)
	blade:SetAttribute("Phase", "active")

	activeBlades[player.UserId] = {
		model = blade,
		stamina = MAX_STAMINA,
		spin = 0,
		wobble = 0,
		death = 0,
		phase = "active",
		deathCenter = spawnPos,
		moveDir = Vector3.zero,
		pos = spawnPos,
		yaw = 0,
		velocity = Vector3.zero,
		targetYaw = 0,
	}

	lockCharacter(player, true)
end

local function updateInput(player, dir)
	local state = activeBlades[player.UserId]
	if not state or state.phase == "stopped" then return end
	state.moveDir = dir
end

beybladeEvent.OnServerEvent:Connect(function(player, action, ...)
	if action == "launch" then
		launch(player)
	elseif action == "input" then
		updateInput(player, ...)
	end
end)

Players.PlayerRemoving:Connect(function(player)
	local state = activeBlades[player.UserId]
	if state and state.model then
		state.model:Destroy()
	end
	lockCharacter(player, false)
	activeBlades[player.UserId] = nil
end)

RunService.Heartbeat:Connect(function(dt)
	for _, state in pairs(activeBlades) do
		local blade = state.model
		if not blade or not blade.Parent then continue end

		if state.phase == "stopped" then continue end

		state.stamina = math.max(0, state.stamina - DRAIN_RATE * dt)
		blade:SetAttribute("Stamina", math.floor(state.stamina))

		local ratio = state.stamina / MAX_STAMINA

		if state.stamina > 0 then
			state.phase = ratio > 0.3 and "active" or "wobble"
			blade:SetAttribute("Phase", state.phase)
		elseif state.phase ~= "dying" and state.phase ~= "stopped" then
			state.phase = "dying"
			state.deathCenter = state.pos
			blade:SetAttribute("Phase", "dying")
		end

		if state.phase == "active" then
			state.spin += BASE_SPIN * dt

			local dir = state.moveDir

			if dir.Magnitude > 0 then
				state.velocity += dir * ACCEL * dt
				state.targetYaw = math.atan2(-dir.X, -dir.Z)
			else
				local speed = state.velocity.Magnitude
				if speed > 0 then
					local newSpeed = math.max(0, speed - FRICTION * dt)
					state.velocity = (state.velocity / speed) * newSpeed
				end
			end

			local maxSpeed = MOVE_SPEED
			if state.velocity.Magnitude > maxSpeed then
				state.velocity = state.velocity.Unit * maxSpeed
			end

			state.yaw = lerpAngle(state.yaw, state.targetYaw, dt * 8)
			state.pos += state.velocity * dt

			local speed = state.velocity.Magnitude
			local tiltCF = CFrame.identity
			if speed > 0.5 then
				local dir2D = state.velocity.Unit
				local perp = Vector3.new(dir2D.Z, 0, -dir2D.X)
				local tiltAngle = math.min(speed / maxSpeed, 1) * 0.22
				tiltCF = CFrame.fromAxisAngle(perp, tiltAngle)
			end

			blade:PivotTo(CFrame.new(state.pos) * tiltCF * CFrame.Angles(0, state.yaw, 0) * CFrame.Angles(math.rad(90), 0, 0) * CFrame.Angles(0, 0, state.spin))

		elseif state.phase == "wobble" then
			local f = ratio / 0.3

			state.spin += BASE_SPIN * f * 0.6 * dt
			state.wobble += dt

			local dir = state.moveDir

			if dir.Magnitude > 0 then
				state.velocity += dir * ACCEL * f * dt
				state.targetYaw = math.atan2(-dir.X, -dir.Z)
			else
				local speed = state.velocity.Magnitude
				if speed > 0 then
					local newSpeed = math.max(0, speed - FRICTION * dt)
					state.velocity = (state.velocity / speed) * newSpeed
				end
			end

			local maxSpeed = MOVE_SPEED * f
			if state.velocity.Magnitude > maxSpeed then
				state.velocity = state.velocity.Unit * maxSpeed
			end

			state.yaw = lerpAngle(state.yaw, state.targetYaw, dt * 6)
			state.pos += state.velocity * dt

			local wobbleTilt = (1 - f) * 0.22
			local precess = state.wobble * 7
			local wobbleX = math.sin(precess) * wobbleTilt
			local wobbleZ = math.cos(precess) * wobbleTilt
			local wobbleCF = CFrame.Angles(wobbleX, 0, wobbleZ)

			local speed = state.velocity.Magnitude
			local moveTiltCF = CFrame.identity
			if speed > 0.5 then
				local dir2D = state.velocity.Unit
				local perp = Vector3.new(dir2D.Z, 0, -dir2D.X)
				local tiltAngle = math.min(speed / MOVE_SPEED, 1) * 0.15 * f
				moveTiltCF = CFrame.fromAxisAngle(perp, tiltAngle)
			end

			blade:PivotTo(CFrame.new(state.pos) * moveTiltCF * CFrame.Angles(0, state.yaw, 0) * wobbleCF * CFrame.Angles(math.rad(90), 0, 0) * CFrame.Angles(0, 0, state.spin))

		elseif state.phase == "dying" then
			state.death += dt
			local p = math.min(1, state.death / DEATH_TIME)

			local cx = state.deathCenter.X
			local cz = state.deathCenter.Z

			if p < 0.6 then
				local wobbleP = p / 0.6
				state.spin += BASE_SPIN * (1 - wobbleP * 0.7) * 0.3 * dt

				local tiltAngle = wobbleP * 0.7
				local precess = state.death * (8 - wobbleP * 4)
				local tx = math.sin(precess) * tiltAngle
				local tz = math.cos(precess) * tiltAngle

				blade:PivotTo(CFrame.new(cx, TIP_Y_OFFSET, cz) * CFrame.Angles(tx, 0, tz) * CFrame.Angles(math.rad(90), 0, 0) * CFrame.Angles(0, 0, state.spin))

			elseif p < 1 then
				local fallP = (p - 0.6) / 0.4
				local fallEase = fallP * fallP * (3 - 2 * fallP)

				state.spin += BASE_SPIN * (1 - p) * 0.1 * dt

				local standAngle = math.rad(90) * (1 - fallEase)

				local tiltAngle = 0.7 * (1 - fallP)
				local precess = state.death * 4
				local tx = math.sin(precess) * tiltAngle
				local tz = math.cos(precess) * tiltAngle

				local y = TIP_Y_OFFSET + (FLAT_Y_OFFSET - TIP_Y_OFFSET) * fallEase

				blade:PivotTo(CFrame.new(cx, y, cz) * CFrame.Angles(tx, 0, tz) * CFrame.Angles(standAngle, 0, 0) * CFrame.Angles(0, 0, state.spin))

			else
				state.phase = "stopped"
				blade:SetAttribute("Phase", "stopped")
				blade:PivotTo(CFrame.new(cx, FLAT_Y_OFFSET, cz))

				local userId = blade:GetAttribute("Owner")
				local owner = Players:GetPlayerByUserId(userId)
				if owner then
					lockCharacter(owner, false)
					beybladeEvent:FireClient(owner, "stopped")
				end
			end
		end
	end
end)
