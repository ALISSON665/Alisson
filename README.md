-- LocalScript em StarterPlayerScripts

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local hrp = character:WaitForChild("HumanoidRootPart")

local FLY_TOGGLE_KEY = Enum.KeyCode.F
local UP_KEY = Enum.KeyCode.Space
local DOWN_KEY = Enum.KeyCode.LeftControl

local BASE_SPEED = 60           -- velocidade horizontal
local VERTICAL_SPEED = 40       -- velocidade vertical (subir/descer)

local flying = false
local conn -- ligação do RenderStepped
local pressed = {} -- teclas pressionadas

-- Atualiza o dicionário de teclas
local function onInputBegan(input, gpe)
	if gpe then return end
	if input.UserInputType == Enum.UserInputType.Keyboard then
		pressed[input.KeyCode] = true

		-- alternar voo
		if input.KeyCode == FLY_TOGGLE_KEY then
			flying = not flying

			if flying then
				-- Entrar em "voo"
				humanoid:ChangeState(Enum.HumanoidStateType.Physics)
				humanoid.AutoRotate = false

				conn = RunService.RenderStepped:Connect(function(dt)
					-- direção baseada na câmera
					local camCF = Camera.CFrame
					local forward = camCF.LookVector
					local right = camCF.RightVector

					-- Zera componente vertical de forward/right para movimento plano
					forward = Vector3.new(forward.X, 0, forward.Z).Unit
					right   = Vector3.new(right.X,   0, right.Z).Unit

					local moveDir = Vector3.zero

					if pressed[Enum.KeyCode.W] then moveDir += forward end
					if pressed[Enum.KeyCode.S] then moveDir -= forward end
					if pressed[Enum.KeyCode.D] then moveDir += right end
					if pressed[Enum.KeyCode.A] then moveDir -= right end

					-- vertical
					local y = 0
					if pressed[UP_KEY] then
						y = VERTICAL_SPEED
					elseif pressed[DOWN_KEY] then
						y = -VERTICAL_SPEED
					end

					if moveDir.Magnitude > 0 then
						moveDir = moveDir.Unit * BASE_SPEED
					end

					local desiredVelocity = Vector3.new(moveDir.X, y, moveDir.Z)

					-- aplica a velocidade diretamente ao HRP
					hrp.AssemblyLinearVelocity = desiredVelocity

					-- Mantém a orientação acompanhando a câmera (opcional)
					local _, yAngle, _ = camCF:ToOrientation()
					hrp.CFrame = CFrame.new(hrp.Position) * CFrame.Angles(0, yAngle, 0)
				end)
			else
				-- Sair do "voo"
				if conn then conn:Disconnect() conn = nil end
				humanoid.AutoRotate = true
				humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
			end
		end
	end
end

local function onInputEnded(input, gpe)
	if input.UserInputType == Enum.UserInputType.Keyboard then
		pressed[input.KeyCode] = nil
	end
end

UserInputService.InputBegan:Connect(onInputBegan)
UserInputService.InputEnded:Connect(onInputEnded)

-- Garanta que, se o personagem respawnar, tudo continue funcionando
player.CharacterAdded:Connect(function(newChar)
	character = newChar
	humanoid = character:WaitForChild("Humanoid")
	hrp = character:WaitForChild("HumanoidRootPart")
	flying = false
	if conn then conn:Disconnect() conn = nil end
end)
