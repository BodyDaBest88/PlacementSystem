local PlacementController = {}
local player = game:GetService("Players").LocalPlayer
local UserInputServices = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")
local RunService = game:GetService("RunService")
 
local ghostModel = nil
PlacementController.selectedComputer = nil
local rotationAngle = 0 -- Keep track of rotation of ghost part
local gridSize = 2
local connection
 
local Mouse = player:GetMouse()
 
local PlotsFolder = workspace:WaitForChild("Plots")
local serverPlacedComputers = workspace:WaitForChild("PlacedComputers"):FindFirstChild("Server")
 
local PlaceEvent = game.ReplicatedStorage.Remotes.Place
local PS = require(game.ReplicatedStorage.Modules.PlotServices)
local SoundHandler = require(game.ReplicatedStorage.Modules.Utilities.SoundHandler)
 
-- Round position to nearest grid point
local function snapToGrid(position)
    return Vector3.new(
        math.floor(position.X / gridSize + 0.5) * gridSize,
        math.floor(position.Y),
        math.floor(position.Z / gridSize + 0.5) * gridSize
    )
end
 
function PlacementController.RayCast()
    local camera = workspace.CurrentCamera
    local origin = camera.CFrame.Position
    local direction = (Mouse.Hit.Position - origin).Unit * 100
 
    local plot = PS.GetPlot(player)
    if not plot then return end
    local floor = plot:FindFirstChild("Floor")
    if not floor then return end
 
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {floor}
    rayParams.FilterType = Enum.RaycastFilterType.Include
 
    local raycastResult = workspace:Raycast(origin, direction, rayParams)
    if raycastResult then
        if raycastResult.Normal:Dot(Vector3.new(0, 1, 0)) > 0.9 then
            return raycastResult.Position
        end
    end
end
 
-- Make model exactly on the Y surface of the floor
local function GetPlacementOffset(model, targetPart)
    if not model.PrimaryPart then
        local firstPart = model:FindFirstChildWhichIsA("BasePart")
        if firstPart then
            model.PrimaryPart = firstPart
        else
            error("Model has no PrimaryPart or BasePart")
        end
    end
 
    local floorTopY = targetPart.Position.Y + (targetPart.Size.Y / 2)
    local lowestY = math.huge
    for _, part in ipairs(model:GetDescendants()) do
        if part:IsA("BasePart") then
            local partBottom = part.Position.Y - (part.Size.Y / 2)
            if partBottom < lowestY then
                lowestY = partBottom
            end
        end
    end
 
    local primaryPartY = model.PrimaryPart.Position.Y
    local offsetNeeded = floorTopY - lowestY
    local finalY = primaryPartY + offsetNeeded
 
    return Vector3.new(0, finalY, 0)
end
 
-- Clamp position to floor boundaries
local function clampToFloorBounds(position, model, floor)
    local _, modelSize = model:GetBoundingBox()
    local floorSize = floor.Size
    local floorPos = floor.Position
 
    local minX = floorPos.X - (floorSize.X / 2) + (modelSize.X / 2)
    local maxX = floorPos.X + (floorSize.X / 2) - (modelSize.X / 2)
    local minZ = floorPos.Z - (floorSize.Z / 2) + (modelSize.Z / 2)
    local maxZ = floorPos.Z + (floorSize.Z / 2) - (modelSize.Z / 2)
 
    if minX > maxX then minX, maxX = floorPos.X, floorPos.X end
    if minZ > maxZ then minZ, maxZ = floorPos.Z, floorPos.Z end
 
    return Vector3.new(
        math.clamp(position.X, minX, maxX),
        position.Y,
        math.clamp(position.Z, minZ, maxZ)
    )
end
 
-- Check for collisions excluding floor
local function isOverlapping(model)
    if not model.PrimaryPart then return false end
 
    local overlapParams = OverlapParams.new()
    overlapParams.FilterType = Enum.RaycastFilterType.Exclude
    overlapParams.FilterDescendantsInstances = {model, player.Character}
 
    local touchingParts = workspace:GetPartBoundsInBox(model.PrimaryPart.CFrame, model:GetExtentsSize(), overlapParams)
 
    for _, part in ipairs(touchingParts) do
        if part.Name == "Baseplate" then continue end
        if part:IsDescendantOf(PS.GetPlot(player)) then
            if part.Name:lower():find("floor") or part.Name == "Part" then continue end
        end
        return true
    end
 
    return false
end
 
local function updatepos()
    if not ghostModel or not ghostModel.PrimaryPart then return end
    if connection then connection:Disconnect() end
 
    local floor = PS.GetPlot(player):FindFirstChild("Floor")
 
    connection = RunService.RenderStepped:Connect(function()
        local hitPosition = PlacementController.RayCast()
        if hitPosition then
            local snappedPosition = snapToGrid(hitPosition)
            local offsetPosition = GetPlacementOffset(ghostModel, floor)
            local rotationCFrame = CFrame.Angles(0, math.rad(rotationAngle), 0)
 
            local finalPosition = clampToFloorBounds(Vector3.new(snappedPosition.X, offsetPosition.Y + 0.1, snappedPosition.Z), ghostModel, floor)
            ghostModel:SetPrimaryPartCFrame(CFrame.new(finalPosition) * rotationCFrame)
 
            local selectionBox = ghostModel:FindFirstChild("SelectionBox")
            if not selectionBox then
                selectionBox = Instance.new("SelectionBox")
                selectionBox.Name = "SelectionBox"
                selectionBox.Parent = ghostModel
                selectionBox.Adornee = ghostModel
                selectionBox.Color3 = Color3.fromRGB(0, 255, 0)
                selectionBox.Visible = true
                selectionBox.Transparency = 0.2
                selectionBox.LineThickness = 0.2
            end
 
            local overlap = isOverlapping(ghostModel)
            selectionBox.Color3 = overlap and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(0, 255, 0)
        end
    end)
end
 
function PlacementController.Ghost(selectedComputer: string)
    if ghostModel then ghostModel:Destroy() end
    if connection then connection:Disconnect() end
 
    local selectedTemplate = game.ReplicatedStorage.Assets.Computers.ClientModels:FindFirstChild(selectedComputer)
    if not selectedTemplate then return end
 
    PlacementController.selectedComputer = selectedComputer
    ghostModel = selectedTemplate:Clone()
 
    if not ghostModel.PrimaryPart then
        local firstPart = ghostModel:FindFirstChildWhichIsA("BasePart")
        if firstPart then ghostModel.PrimaryPart = firstPart end
    end
 
    -- Make all ghost parts uncollidable
    for _, part in ipairs(ghostModel:GetDescendants()) do
        if part:IsA("BasePart") and part.Name ~= "RegionBox" then
            part.Transparency = 0.5
            part.CanCollide = false
            part.CanTouch = false
        end
    end
 
    ghostModel.Parent = workspace
    updatepos()
end
 
function PlacementController.place()
    if not ghostModel then return end
    local finalPosition = ghostModel.PrimaryPart and ghostModel.PrimaryPart.CFrame
    if connection then connection:Disconnect() connection = nil end
    if finalPosition then
        if isOverlapping(ghostModel) then
            PlacementController.cancelPlacement()
            player.Character:FindFirstChildWhichIsA("Tool").Parent = player.Backpack
            return
        end
        PlaceEvent:FireServer(PlacementController.selectedComputer, finalPosition)
    end
    ghostModel:Destroy()
    ghostModel = nil
    PlacementController.selectedComputer = nil
end
 
function PlacementController.cancelPlacement()
    if ghostModel then
        if connection then connection:Disconnect() connection = nil end
        ghostModel:Destroy()
        ghostModel = nil
        PlacementController.selectedComputer = nil
    end
end
 
function PlacementController.init()
    local smokeParticleTemplate = game.ReplicatedStorage.Particles:FindFirstChild("Smoke-01")
 
    local function addClientModel(child: Instance)
        if child:IsA("Model") then
            local computerName = child.Name
            local computerTrueName = computerName:split(":")[1] or computerName
            local computerNameID = child:FindFirstChild("Configuration"):FindFirstChild("NameID")
            local clientModelTemplate = game.ReplicatedStorage.Assets.Computers.ClientModels:FindFirstChild(computerTrueName)
            if not clientModelTemplate then return end
 
            local clientModel = clientModelTemplate:Clone()
            clientModel.Name = computerName
            clientModel.Parent = workspace.PlacedComputers.Client
 
            -- Make all client model parts uncollidable
            for _, part in ipairs(clientModel:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                    part.CanTouch = true
                end
            end
 
            if child.PrimaryPart then
                clientModel:SetPrimaryPartCFrame(child.PrimaryPart.CFrame)
                
                --Place Effects
                local originalCFrames = {}
                for _, part in ipairs(clientModel:GetDescendants()) do
                    if part:IsA("BasePart") then
                        originalCFrames[part] = part.CFrame
                        part.Size = part.Size * 0.7
                    end
                end
 
                for part, _ in pairs(originalCFrames) do
                    local tween = TweenService:Create(part, TweenInfo.new(0.6, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = part.Size / 0.7})
                    tween:Play()
                end
 
                if smokeParticleTemplate then
                    local smoke = smokeParticleTemplate:Clone()
                    smoke.CFrame = clientModel.PrimaryPart.CFrame
                    smoke.Parent = workspace.Temp
                    smoke:FindFirstChild("Main"):FindFirstChild("Smoke1"):Emit()
                    Debris:AddItem(smoke, 1)
                end
                
                --Touch Effects
                clientModel.PrimaryPart.Touched:Connect(function(hitPart)
                    local character = hitPart:FindFirstAncestorOfClass("Model")
                    if not character then return end
                    local player = game.Players:GetPlayerFromCharacter(character)
                    if not player then return end
 
                    -- Create debounce folder if missing
                    if not player:FindFirstChild("ComputerDebounce") then
                        local folder = Instance.new("Folder")
                        folder.Name = "ComputerDebounce"
                        folder.Parent = player
                    end
                    local debounceFolder = player:FindFirstChild("ComputerDebounce")
 
                    if debounceFolder:FindFirstChild(computerNameID.Value) then
                        return
                    end
 
                    local debounceObj = Instance.new("BoolValue")
                    debounceObj.Name = computerNameID.Value
                    debounceObj.Value = true
                    debounceObj.Parent = debounceFolder
 
                    game:GetService("Debris"):AddItem(debounceObj, 1)
 
                    local highlight = clientModel:FindFirstChild("Highlight")
                    if not highlight then return end
 
                    local collectParticle = game.ReplicatedStorage.Particles.Money:Clone()
                    collectParticle.CFrame = clientModel.PrimaryPart.CFrame
                    collectParticle.Parent = workspace.Temp
 
                    SoundHandler:PlayLocal("CashGain", workspace.Garbage)
 
                    for _, v in collectParticle:GetDescendants() do
                        if v:IsA("ParticleEmitter") then
                            v:Emit(8)
                        end
                    end
 
                    highlight.FillTransparency = 1 
                    local fadeIn = TweenService:Create(highlight, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {FillTransparency = 0})
                    local hold = TweenService:Create(highlight, TweenInfo.new(0.15), {FillTransparency = 0})
                    local fadeOut = TweenService:Create(highlight, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {FillTransparency = 1})
 
                    fadeIn:Play()
                    fadeIn.Completed:Connect(function()
                        hold:Play()
                        hold.Completed:Connect(function()
                            fadeOut:Play()
                            Debris:AddItem(collectParticle, 1)
                        end)
                    end)
                end)
            else
                warn(child.Name .. " has no primary part")
            end
        end
    end
 
    local function removeClientModel(child: Instance)
        -- Optional removal effects (destoryed)
    end
 
 
    serverPlacedComputers.ChildAdded:Connect(addClientModel)
    serverPlacedComputers.ChildRemoved:Connect(removeClientModel)
 
    UserInputServices.InputBegan:Connect(function(input, gpe)
        if not ghostModel or gpe then return end
        if input.KeyCode == Enum.KeyCode.R then
            rotationAngle = (rotationAngle + 90) % 360
        end
    end)
end
 
return PlacementController
 
local placeRemote = game.ReplicatedStorage.Remotes.Place
local PS = require(game.ReplicatedStorage.Modules.PlotServices)
local Datastore = require(game.ServerScriptService.Modules.DataStoreManager)
local HP = require(game.ReplicatedStorage.Modules.Utilities.HelperFunctions)
local Computer = require(game.ServerScriptService.Modules.Computer)
 
local PlacementServer = {}
 
--Checking if player has the tool (computer)
local function hasTool(player: Player, computerName: string)
    local character = player.Character
    if not character then return false end
 
    for _, tool in pairs(character:GetChildren()) do
        if tool:IsA("Tool") and tool.Name == computerName then
            local quantity = tool:GetAttribute("Quantity")
 
            if quantity then
                if quantity > 1 then
                    quantity -= 1
                    tool.Name = computerName .. " x" .. quantity
                else
                    -- Defer destruction to next frame to avoid FPS drop
                    task.defer(function()
                        if tool and tool.Parent then
                            tool:Destroy()
                        end
                    end)
                end
                return true
            else
                -- Defer destruction to next frame
                task.defer(function()
                    if tool and tool.Parent then
                        tool:Destroy()
                    end
                end)
                return true
            end
        end
    end
    return false
end
 
--Raycast to check if location is within plot
local function isWithinPlot(player: Player, location: Vector3)
    local plot = PS.GetPlot(player)
    local floor = plot and plot:FindFirstChild("Floor")
    if not floor then return false end
 
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {floor}
    rayParams.FilterType = Enum.RaycastFilterType.Include
 
    local origin = location + Vector3.new(0, 5, 0)
    local direction = Vector3.new(0, -99999, 0)
 
    local raycastResult = workspace:Raycast(origin, direction, rayParams)
    return raycastResult ~= nil
end
 
local function isTooClose(player: Player, location: Vector3)
    -- TODO: Implement real distance checks
    return true
end
 
function PlacementServer.init()
    placeRemote.OnServerEvent:Connect(function(player: Player, computerName: string, location: CFrame)
        local position = location.Position -- Extract Vector3 from CFrame
        local plot = PS.GetPlot(player)
        local refrencePoint = plot:FindFirstChild("RefrencePoint")
        
        --Validation
        if hasTool(player, computerName) then
            if isWithinPlot(player, position) then
                if isTooClose(player, position) then
                    
                    --Save to datastore
                    local plotData = PS.GetPlotData(player)
                    local computerID = HP.generateKey(computerName)
                    local LocationToSave = refrencePoint.CFrame:ToObjectSpace(location)
                    
                    plotData[computerID] = {
                        location = {LocationToSave:GetComponents()} ,
                        Name = computerID,
                        DatePlaced = os.time(),
                        cashGenerated = 0
                    }
                    
                    --Make a new class which clones and creates a new ocmputer object
                    Computer.new(plotData[computerID], player)
                    
                    warn(plotData)
                end
            end
        end
    end)
end
 
return PlacementServer
 
local Computer = {}
Computer.__index = Computer
 
local PS = require(game.ReplicatedStorage.Modules.PlotServices)
local Config = require(game.ReplicatedStorage.Modules.Config.Computers)
local MoneyServices = require(game.ServerScriptService.Modules.MoneyServices)
 
-- Constructor
function Computer.new(data: any, player: Player)
    local self = setmetatable({}, Computer)
 
    self.Name = data.Name or nil
    self.TrueName = self.Name:split(":")[1]
    self.DatePlaced = data.DatePlaced or os.time()
    self.Location = data.location or nil
    self.CashGenerated = data.cashGenerated or 0
    self.CashPerSecond = Config[self.TrueName].Income or 10
 
    self.Plot = PS.GetPlot(player) or nil
 
    -- Clone server model
    self.model = game.ReplicatedStorage.Assets.Computers.ServerModels:FindFirstChild(self.TrueName):Clone()
    self.model.Name = self.TrueName
 
    -- Set configuration values
    local config = self.model:FindFirstChild("Configuration")
    config:FindFirstChild("NameID").Value = self.Name
    config:FindFirstChild("CashStorage").Value = self.CashGenerated
    config:FindFirstChild("Owner").Value = player.UserId
 
    self:SetUp()
 
    return self
end
 
function Computer:SetUp()
    self:GenerateMoney()
    self:createServerModel()
 
    local regionBox = self.model:FindFirstChild("RegionBox")
    if not regionBox then return end
 
    regionBox.Touched:Connect(function(hitPart)
        local character = hitPart:FindFirstAncestorOfClass("Model")
        if not character then return end
        local player = game.Players:GetPlayerFromCharacter(character)
        if not player then return end
 
        -- Create debounce folder if missing
        if not player:FindFirstChild("ComputerDebounce") then
            local folder = Instance.new("Folder")
            folder.Name = "ComputerDebounce"
            folder.Parent = player
        end
        local debounceFolder = player:FindFirstChild("ComputerDebounce")
 
        if debounceFolder:FindFirstChild(self.Name) then
            return
        end
 
        -- Create debounce for this computer
        local debounceObj = Instance.new("BoolValue")
        debounceObj.Name = self.Name
        debounceObj.Value = true
        debounceObj.Parent = debounceFolder
 
        game:GetService("Debris"):AddItem(debounceObj, 1)
 
        local humanoid = character:FindFirstChild("Humanoid")
        if not humanoid then return end
 
        self:CollectMoney(player)
    end)
end
 
-- Creating server model
function Computer:createServerModel()
    if not self.model.PrimaryPart then
        self.model.PrimaryPart = self.model:FindFirstChildWhichIsA("Part")
    end
 
    -- Pivot to stored location
    local deserializedCframe = CFrame.new(table.unpack(self.Location))
    deserializedCframe = self.Plot:FindFirstChild("RefrencePoint").CFrame:ToWorldSpace(deserializedCframe)
 
    self.model:SetPrimaryPartCFrame(deserializedCframe)
 
    -- Parent to workspace
    self.model.Parent = workspace.PlacedComputers.Server
end
 
function Computer:GenerateMoney()
    local infoBillboard = self.model:FindFirstChild("Info")
    local collectedMoneyText = infoBillboard and infoBillboard:FindFirstChild("Collected Money")
 
    local config = self.model:FindFirstChild("Configuration")
    local cashStorage = config and config:FindFirstChild("CashStorage")
 
    if cashStorage then
        cashStorage.Changed:Connect(function(value)
            self.CashGenerated = value
            if collectedMoneyText and collectedMoneyText:IsA("TextLabel") then
                collectedMoneyText.Text = tostring(self.CashGenerated) .. "$"
            end
        end)
    end
 
    task.spawn(function()
        while self.model and cashStorage do
            cashStorage.Value += self.CashPerSecond
            if collectedMoneyText and collectedMoneyText:IsA("TextLabel") then
                collectedMoneyText.Text = tostring(self.CashGenerated) .. "$"
            end
            task.wait(1)
        end
    end)
end
 
function Computer:CollectMoney(player: Player)
    if not player then return end
 
    local config = self.model:FindFirstChild("Configuration")
    local cashStorage = config and config:FindFirstChild("CashStorage")
    if not cashStorage then return end
 
    local amount = self.CashGenerated
    cashStorage.Value = 0
 
    MoneyServices.GiveMoney(player, amount)
end
 
return Computer
