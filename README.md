local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

-- Create UI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "GlitchUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 200, 0, 280)
frame.Position = UDim2.new(0, 20, 0.5, -140)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 30)
title.BackgroundTransparency = 1
title.Text = "Glitch Controls"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 18
title.Font = Enum.Font.GothamBold
title.Parent = frame

-- Glitch state
local glitching = false
local undergroundGlitching = false
local noclip = false
local glitchConnection = nil
local undergroundConnection = nil
local startY = nil
local loopHeight = 15

-- Create button function
local function createButton(text, position, callback)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0.9, 0, 0, 35)
    button.Position = UDim2.new(0.05, 0, 0, position)
    button.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    button.Text = text
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.TextSize = 14
    button.Font = Enum.Font.Gotham
    button.Parent = frame
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = button
    
    button.MouseButton1Click:Connect(callback)
    
    button.MouseEnter:Connect(function()
        button.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    end)
    
    button.MouseLeave:Connect(function()
        button.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    end)
    
    return button
end

-- Noclip functionality
local noclipConnection = nil

local function toggleNoclip()
    noclip = not noclip
    
    if noclip then
        noclipConnection = RunService.Stepped:Connect(function()
            if character then
                for _, part in pairs(character:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = false
                    end
                end
            end
        end)
    else
        if noclipConnection then
            noclipConnection:Disconnect()
            noclipConnection = nil
        end
        if character then
            for _, part in pairs(character:GetDescendants()) do
                if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                    part.CanCollide = true
                end
            end
        end
    end
end

-- Underground loop glitch - goes down 15 studs then resets above
local function startUndergroundGlitch()
    if undergroundGlitching then return end
    undergroundGlitching = true
    
    if not noclip then
        toggleNoclip()
    end
    
    -- Store starting Y position
    startY = rootPart.Position.Y
    
    undergroundConnection = RunService.Heartbeat:Connect(function()
        if not character or not rootPart or not humanoid then return end
        
        local currentY = rootPart.Position.Y
        local currentX = rootPart.Position.X
        local currentZ = rootPart.Position.Z
        
        -- Check if we've gone 15 studs below start
        if startY - currentY >= loopHeight then
            -- Reset to 15 studs above original position
            local resetY = startY + loopHeight
            rootPart.CFrame = CFrame.new(currentX, resetY, currentZ)
            
            -- Re-enable noclip after teleport (sometimes gets disabled)
            for _, part in pairs(character:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        else
            -- Continue sinking down
            rootPart.CFrame = rootPart.CFrame - Vector3.new(0, 0.5, 0)
        end
        
        -- Move forward constantly
        local lookVector = rootPart.CFrame.LookVector
        rootPart.Velocity = Vector3.new(lookVector.X * 30, rootPart.Velocity.Y, lookVector.Z * 30)
        
        -- Tilt upward slightly while underground
        local rightVector = rootPart.CFrame.RightVector
        local upVector = Vector3.new(0, 1, 0)
        local tiltedLook = (upVector * 0.9 + lookVector * 0.1).Unit
        rootPart.CFrame = CFrame.fromMatrix(rootPart.Position, rightVector, tiltedLook)
    end)
end

local function stopUndergroundGlitch()
    undergroundGlitching = false
    startY = nil
    if undergroundConnection then
        undergroundConnection:Disconnect()
        undergroundConnection = nil
    end
    if rootPart then
        rootPart.Velocity = Vector3.new(0, 0, 0)
    end
end

-- Original ground glitch (tilted forward)
local function startGroundGlitch()
    if glitching then return end
    glitching = true
    
    if not noclip then
        toggleNoclip()
    end
    
    glitchConnection = RunService.Heartbeat:Connect(function()
        if not character or not rootPart or not humanoid then return end
        
        local currentCFrame = rootPart.CFrame
        local upVector = Vector3.new(0, 1, 0)
        local lookVector = currentCFrame.LookVector
        local rightVector = currentCFrame.RightVector
        
        local tiltedLook = (upVector * 0.98 + lookVector * 0.02).Unit
        local newCFrame = CFrame.fromMatrix(currentCFrame.Position, rightVector, tiltedLook)
        
        rootPart.CFrame = newCFrame
        rootPart.Velocity = tiltedLook * 50
        rootPart.CFrame = rootPart.CFrame - Vector3.new(0, 2, 0)
    end)
end

local function stopGroundGlitch()
    glitching = false
    if glitchConnection then
        glitchConnection:Disconnect()
        glitchConnection = nil
    end
    if rootPart then
        rootPart.Velocity = Vector3.new(0, 0, 0)
    end
end

local function stopAll()
    stopGroundGlitch()
    stopUndergroundGlitch()
end

-- Wall clip
local function startWallClip()
    if not character or not rootPart then return end
    if not noclip then
        toggleNoclip()
    end
    
    local currentPos = rootPart.Position
    rootPart.CFrame = CFrame.new(currentPos.X, currentPos.Y - 5, currentPos.Z)
    local lookVector = rootPart.CFrame.LookVector
    rootPart.Velocity = lookVector * 100
    
    local rightVector = rootPart.CFrame.RightVector
    local upVector = Vector3.new(0, 1, 0)
    rootPart.CFrame = CFrame.fromMatrix(rootPart.Position, rightVector, upVector)
end

-- Create buttons
createButton("Underground Loop", 40, function()
    startUndergroundGlitch()
end)

createButton("Stop Underground", 80, function()
    stopUndergroundGlitch()
end)

createButton("Sky Glitch", 120, function()
    startGroundGlitch()
end)

createButton("Stop Sky Glitch", 160, function()
    stopGroundGlitch()
end)

createButton("Toggle Noclip", 200, function()
    toggleNoclip()
end)

createButton("Wall Clip", 240, function()
    startWallClip()
end)

-- Cleanup
humanoid.Died:Connect(function()
    stopAll()
    if noclip then
        toggleNoclip()
    end
end)

player.CharacterAdded:Connect(function(newChar)
    character = newChar
    humanoid = character:WaitForChild("Humanoid")
    rootPart = character:WaitForChild("HumanoidRootPart")
    
    humanoid.Died:Connect(function()
        stopAll()
        if noclip then
            toggleNoclip()
        end
    end)
end)
