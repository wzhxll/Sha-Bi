-- 独立飞行脚本 - 优化版（完整UI，原样提取）
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local lp = Players.LocalPlayer
local camera = workspace.CurrentCamera
local pgui = lp:WaitForChild("PlayerGui")
local CoreGui = game:GetService("CoreGui")
local ControlModule = require(lp.PlayerScripts:WaitForChild("PlayerModule")):GetControls()

-- ========== 原样复制 L 中的飞行变量和函数 ==========
local Fly = {
    bv = nil,
    bg = nil,
    animCache = nil,
    hrp = nil,
    hum = nil,
    isFlying = false,
    flySpeed = 40,
    isWallhack = false,
    originalCollisions = {}
}

local function getBodyParts(character)
    local parts = {}
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        local success, rigParts = pcall(function() return humanoid:GetRigParts() end)
        if success and rigParts then
            for _, part in ipairs(rigParts) do
                if part:IsA("BasePart") then
                    table.insert(parts, part)
                end
            end
        end
    end
    if #parts == 0 then
        local bodyNames = {"Head", "Torso", "UpperTorso", "LowerTorso", "HumanoidRootPart",
                           "Left Arm", "Right Arm", "Left Leg", "Right Leg",
                           "LeftUpperArm", "LeftLowerArm", "RightUpperArm", "RightLowerArm",
                           "LeftUpperLeg", "LeftLowerLeg", "RightUpperLeg", "RightLowerLeg"}
        for _, name in ipairs(bodyNames) do
            local part = character:FindFirstChild(name)
            if part and part:IsA("BasePart") then
                table.insert(parts, part)
            end
        end
    end
    return parts
end

local SmoothTurner = {}
SmoothTurner.__index = SmoothTurner

function SmoothTurner.new(rootPart, camera, options)
    options = options or {}
    local self = setmetatable({}, SmoothTurner)
    self.RootPart = rootPart
    self.Camera = camera or workspace.CurrentCamera
    self.Enabled = false
    self.BodyGyro = nil
    self.P = options.P or 10000
    self.D = options.D or 50
    self.MaxTorque = options.MaxTorque or Vector3.new(math.huge, math.huge, math.huge)
    return self
end

function SmoothTurner:Start()
    if self.Enabled then return end
    if not self.RootPart or not self.RootPart.Parent then return end
    local gyro = Instance.new("BodyGyro")
    gyro.MaxTorque = self.MaxTorque
    gyro.P = self.P
    gyro.D = self.D
    gyro.CFrame = self.RootPart.CFrame
    gyro.Parent = self.RootPart
    self.BodyGyro = gyro
    self.Enabled = true
    self:_startHeartbeat()
end

function SmoothTurner:Stop()
    if self.BodyGyro then
        self.BodyGyro:Destroy()
        self.BodyGyro = nil
    end
    self.Enabled = false
    if self.HeartbeatConn then
        self.HeartbeatConn:Disconnect()
        self.HeartbeatConn = nil
    end
end

function SmoothTurner:SetDirection(direction)
    if not self.Enabled or not self.BodyGyro or not self.RootPart then return end
    local newCFrame = CFrame.lookAt(self.RootPart.Position, self.RootPart.Position + direction.Unit)
    self.BodyGyro.CFrame = newCFrame
end

function SmoothTurner:_startHeartbeat()
    if self.HeartbeatConn then self.HeartbeatConn:Disconnect() end
    self.HeartbeatConn = RunService.Heartbeat:Connect(function()
        if not self.Enabled or not self.BodyGyro or not self.RootPart or not self.Camera then return end
        local look = self.Camera.CFrame.LookVector
        self:SetDirection(look)
    end)
end

function SmoothTurner:Destroy()
    self:Stop()
    self.RootPart = nil
    self.Camera = nil
end

local flyTurner = nil

function Fly.clearFlyRes()
    local char = lp.Character
    if char then
        local bodyParts = getBodyParts(char)
        for part, originalState in pairs(Fly.originalCollisions) do
            if part and part.Parent then
                for _, bp in ipairs(bodyParts) do
                    if bp == part then
                        part.CanCollide = originalState
                        break
                    end
                end
            end
        end
        Fly.originalCollisions = {}
    end
    if Fly.animCache and lp.Character then Fly.animCache.Parent = lp.Character end
    if Fly.bv then Fly.bv:Destroy() end
    if Fly.bg then Fly.bg:Destroy() end
    Fly.bv, Fly.bg = nil, nil
    if flyTurner then flyTurner:Destroy(); flyTurner = nil end
    if Fly.hum and Fly.hum.Parent then Fly.hum:ChangeState(Enum.HumanoidStateType.Running) end
end

function Fly.ensurePhysics(hrp, useGyro)
    if hrp:FindFirstChild("LeipzigBV_new") then hrp.LeipzigBV_new:Destroy() end
    if hrp:FindFirstChild("LeipzigBG_new") then hrp.LeipzigBG_new:Destroy() end
    Fly.bv = Instance.new("BodyVelocity", hrp)
    Fly.bv.Name = "LeipzigBV_new"
    Fly.bv.MaxForce = Vector3.new(1e6, 1e6, 1e6)
    if useGyro then
        if flyTurner then flyTurner:Destroy() end
        flyTurner = SmoothTurner.new(hrp, workspace.CurrentCamera)
        flyTurner:Start()
    end
end

function Fly.applyWallhackState()
    local char = lp.Character
    if not char then return end
    if Fly.isWallhack then
        local bodyParts = getBodyParts(char)
        Fly.originalCollisions = {}
        for _, part in ipairs(bodyParts) do
            Fly.originalCollisions[part] = part.CanCollide
            part.CanCollide = false
        end
    else
        for part, originalState in pairs(Fly.originalCollisions) do
            if part and part.Parent then
                part.CanCollide = originalState
            end
        end
        Fly.originalCollisions = {}
    end
end

function Fly.startFlyNormal()
    local char = lp.Character
    if not char then return end
    Fly.hrp = char:WaitForChild("HumanoidRootPart")
    Fly.hum = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then Fly.animCache = ani; ani.Parent = nil end
    Fly.ensurePhysics(Fly.hrp, true)
    task.spawn(function()
        while Fly.isFlying and char.Parent do
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)
            if mv.Magnitude > 0 then
                Fly.bv.Velocity = dir.Unit * Fly.flySpeed
            else
                Fly.bv.Velocity = Vector3.new(0,0.01,0)
            end
            Fly.hum:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
        end
        Fly.clearFlyRes()
    end)
end

function Fly.startFlyWallhack()
    local char = lp.Character
    if not char then return end
    Fly.hrp = char:WaitForChild("HumanoidRootPart")
    Fly.hum = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then Fly.animCache = ani; ani.Parent = nil end
    Fly.applyWallhackState()
    Fly.ensurePhysics(Fly.hrp, true)
    task.spawn(function()
        local lastPos = Fly.hrp.Position
        local lastTime = tick()
        while Fly.isFlying and char.Parent do
            local dt = tick() - lastTime
            lastTime = tick()
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)
            local targetVelocity
            if mv.Magnitude > 0 then
                targetVelocity = dir.Unit * Fly.flySpeed
                Fly.bv.Velocity = targetVelocity
            else
                Fly.bv.Velocity = Vector3.new(0,0.01,0)
                targetVelocity = Vector3.new(0,0.01,0)
            end
            Fly.hum:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
            local expectedPos = lastPos + targetVelocity * dt
            local actualPos = Fly.hrp.Position
            local deviation = actualPos - expectedPos
            if deviation.Magnitude > 0.00001 then
                Fly.hrp.CFrame = CFrame.new(expectedPos) * Fly.hrp.CFrame.Rotation
                Fly.bv.Velocity = targetVelocity
                lastPos = expectedPos
            else
                lastPos = actualPos
            end
        end
        Fly.clearFlyRes()
    end)
end

function Fly.startFly()
    if Fly.isFlying then return end
    Fly.isFlying = true
    if Fly.isWallhack then
        Fly.startFlyWallhack()
    else
        Fly.startFlyNormal()
    end
end

function Fly.stopFly()
    if not Fly.isFlying then return end
    Fly.isFlying = false
    Fly.clearFlyRes()
end

function Fly.bindCharacter()
    local char = lp.Character or lp.CharacterAdded:Wait()
    Fly.hrp = char:WaitForChild("HumanoidRootPart")
    Fly.hum = char:WaitForChild("Humanoid")
    Fly.clearFlyRes()
    char.AncestryChanged:Connect(function(_, parent)
        if not parent then
            Fly.clearFlyRes()
            Fly.bindCharacter()
        end
    end)
end
Fly.bindCharacter()

-- ========== 创建UI（原样从 FlyTab:Button 回调中复制） ==========
if pgui:FindFirstChild("NewFlightUI") then pgui.NewFlightUI:Destroy() end
task.wait(0.1)

local UI_BG = Color3.fromRGB(200, 230, 255)
local BTN_OFF = Color3.fromRGB(150, 200, 255)
local BTN_ON = Color3.fromRGB(70, 150, 255)
local DESTROY_BTN = Color3.fromRGB(110, 180, 255)
local TEXT_COLOR = Color3.fromRGB(0, 60, 120)
local SPEED_BG = Color3.fromRGB(180, 220, 255)

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NewFlightUI"
ScreenGui.Parent = pgui
ScreenGui.ResetOnSpawn = false
ScreenGui.DisplayOrder = 999

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 150, 0, 160)
MainFrame.Position = UDim2.new(0.5, -75, 0.3, 0)
MainFrame.BackgroundColor3 = UI_BG
MainFrame.BackgroundTransparency = 0.4
MainFrame.Draggable = true
MainFrame.Active = true
MainFrame.Parent = ScreenGui

Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 10)
local stroke = Instance.new("UIStroke", MainFrame)
stroke.Color = Color3.fromRGB(120, 200, 255)
stroke.Thickness = 3
stroke.Transparency = 0.1

local Title = Instance.new("TextLabel", MainFrame)
Title.Size = UDim2.new(1,0,0,20)
Title.BackgroundTransparency = 1
Title.Text = "飞行-优化"
Title.TextColor3 = TEXT_COLOR
Title.TextSize = 12
Title.Font = Enum.Font.GothamBold

local SpeedInput = Instance.new("TextBox", MainFrame)
SpeedInput.Size = UDim2.new(0,120,0,24)
SpeedInput.Position = UDim2.new(0.5,-60,0, 30)
SpeedInput.BackgroundColor3 = SPEED_BG
SpeedInput.BackgroundTransparency = 0.3
SpeedInput.Text = "40"
SpeedInput.TextColor3 = TEXT_COLOR
SpeedInput.TextSize = 11
Instance.new("UICorner", SpeedInput).CornerRadius = UDim.new(0,7)

local WallhackBtn = Instance.new("TextButton", MainFrame)
WallhackBtn.Size = UDim2.new(0,120,0,26)
WallhackBtn.Position = UDim2.new(0.5,-60,0, 64)
WallhackBtn.BackgroundColor3 = BTN_OFF
WallhackBtn.BackgroundTransparency = 0.3
WallhackBtn.Text = "穿墙模式: 关闭"
WallhackBtn.TextColor3 = TEXT_COLOR
WallhackBtn.TextSize = 11
Instance.new("UICorner", WallhackBtn).CornerRadius = UDim.new(0,8)

local FlyBtn = Instance.new("TextButton", MainFrame)
FlyBtn.Size = UDim2.new(0,120,0,26)
FlyBtn.Position = UDim2.new(0.5,-60,0, 98)
FlyBtn.BackgroundColor3 = BTN_OFF
FlyBtn.BackgroundTransparency = 0.3
FlyBtn.Text = "飞行"
FlyBtn.TextColor3 = TEXT_COLOR
FlyBtn.TextSize = 11
Instance.new("UICorner", FlyBtn).CornerRadius = UDim.new(0,8)

local DestroyUI = Instance.new("TextButton", MainFrame)
DestroyUI.Size = UDim2.new(0,120,0,26)
DestroyUI.Position = UDim2.new(0.5,-60,0, 132)
DestroyUI.BackgroundColor3 = DESTROY_BTN
DestroyUI.BackgroundTransparency = 0.3
DestroyUI.Text = "销毁UI"
DestroyUI.TextColor3 = TEXT_COLOR
DestroyUI.TextSize = 11
Instance.new("UICorner", DestroyUI).CornerRadius = UDim.new(0,8)

local dragging, dragStart, startPos
MainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
end)

SpeedInput.FocusLost:Connect(function()
    local val = tonumber(SpeedInput.Text)
    if val then Fly.flySpeed = math.clamp(val, 10, 100) else Fly.flySpeed = 40 end
    SpeedInput.Text = tostring(Fly.flySpeed)
end)

WallhackBtn.MouseButton1Click:Connect(function()
    Fly.isWallhack = not Fly.isWallhack
    WallhackBtn.Text = "穿墙模式: " .. (Fly.isWallhack and "开启" or "关闭")
    WallhackBtn.BackgroundColor3 = Fly.isWallhack and BTN_ON or BTN_OFF
    if Fly.isFlying then
        Fly.stopFly()
        task.wait(0.05)
        Fly.startFly()
    else
        Fly.applyWallhackState()
    end
end)

FlyBtn.MouseButton1Click:Connect(function()
    if Fly.isFlying then
        Fly.stopFly()
        FlyBtn.Text = "飞行"
        FlyBtn.BackgroundColor3 = BTN_OFF
    else
        Fly.startFly()
        FlyBtn.Text = "飞行开"
        FlyBtn.BackgroundColor3 = BTN_ON
    end
end)

DestroyUI.MouseButton1Click:Connect(function()
    Fly.stopFly()
    Fly.applyWallhackState()
    ScreenGui:Destroy()
end)

MainFrame.Size = UDim2.new(0,0,0,0)
MainFrame:TweenSize(UDim2.new(0,150,0,160), Enum.EasingDirection.Out, Enum.EasingStyle.Back, 0.4, true)
