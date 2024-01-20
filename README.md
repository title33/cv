local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Player = Players.LocalPlayer or Players.PlayerAdded:Wait()
local Remotes = ReplicatedStorage:WaitForChild("Remotes")
local Balls = workspace:WaitForChild("Balls")

local DEBUG = false -- Use this flag to enable/disable debug printing

local function print(...)
    if DEBUG then
        warn(...)
    end
end

local function verifyBall(ball)
    return typeof(ball) == "Instance" and ball:IsA("BasePart") and ball:IsDescendantOf(Balls) and ball:GetAttribute("realBall") == true
end

local function isTarget()
    return Player.Character and Player.Character:FindFirstChild("Highlight")
end

local function parry()
    Remotes:WaitForChild("ParryButtonPress"):FireServer() -- Use FireServer for remote events
end

local function trackBallData(ball)
    local ballData = {
        distance = 0,
        velocity = 0,
        lastPositionUpdate = tick(),
    }

    local connection = ball:GetPropertyChangedSignal("Position"):Connect(function()
        if isTarget() then
            local currentPosition = ball.Position
            local timeDelta = tick() - ballData.lastPositionUpdate
            ballData.distance = (currentPosition - workspace.CurrentCamera.Focus.Position).Magnitude
            ballData.velocity = (currentPosition - ballData.lastPosition) / timeDelta

            print("Distance:", ballData.distance)
            print("Velocity:", ballData.velocity)
            print("Time to reach:", ballData.distance / ballData.velocity)

            if ballData.distance / ballData.velocity <= 10 then
                parry()
            end

            ballData.lastPosition = currentPosition
            ballData.lastPositionUpdate = tick()
        end
    end)

    return connection
end

local trackedBalls = {}

Balls.ChildAdded:Connect(function(ball)
    if verifyBall(ball) then
        print("Ball Spawned:", ball)
        local trackConnection = trackBallData(ball)
        table.insert(trackedBalls, trackConnection)
    end
end)

-- Cleanup connections when the script ends
Players.PlayerRemoving:Connect(function()
    for _, connection in pairs(trackedBalls) do
        connection:Disconnect()
    end
end)
