--[[
    PROFESSIONAL BUILD ASSIST SUITE (STUDIO LITE COMPATIBLE)
    Version: 2.0
    Author: Senior Lua Engineer
    
    Architecture:
    1. Services & Constants
    2. State Management (Settings, Selection, History)
    3. Math & Snapping Utils
    4. Core Logic (Tools Implementation)
    5. UI Engine (Procedural GUI)
    6. Main Bootstrapper
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")

-- Protect GUI from game (attempt to put in CoreGui, fallback to PlayerGui)
local Player = Players.LocalPlayer
local PlayerMouse = Player:GetMouse()
local Camera = Workspace.CurrentCamera

local TARGET_PARENT = CoreGui:FindFirstChild("RobloxGui") or Player:WaitForChild("PlayerGui")

--=============================================================================
-- 1. CONFIGURATION & STYLES
--=============================================================================

local Config = {
    UI = {
        Theme = Color3.fromRGB(35, 35, 35),
        Accent = Color3.fromRGB(0, 120, 215),
        Danger = Color3.fromRGB(200, 50, 50),
        Text = Color3.fromRGB(240, 240, 240),
        SubText = Color3.fromRGB(150, 150, 150),
        CornerRadius = UDim.new(0, 6),
        Padding = 6,
        Width = 220,
    },
    Defaults = {
        GridSize = 1,
        RotStep = 15,
        SnapEnabled = true,
        AxisMode = "World", -- "World" or "Local"
        MaxHistory = 50,
    }
}

--=============================================================================
-- 2. STATE MANAGEMENT
--=============================================================================

local State = {
    Selection = {}, -- Array of BaseParts
    Clipboard = {}, -- Array of Data
    History = {Stack = {}, Index = 0},
    ActiveTool = "Select", -- Select, Move, Rotate, Scale
    Settings = {
        Grid = Config.Defaults.GridSize,
        Rot = Config.Defaults.RotStep,
        Snapping = Config.Defaults.SnapEnabled,
        Axis = Config.Defaults.AxisMode
    },
    IsDragging = false
}

-- Events for loose coupling
local Events = {
    SelectionChanged = Instance.new("BindableEvent"),
    ToolChanged = Instance.new("BindableEvent"),
    HistoryChanged = Instance.new("BindableEvent")
}

--=============================================================================
-- 3. UTILITY & MATH
--=============================================================================

local Utils = {}

function Utils.Snap(number, step)
    if not State.Settings.Snapping or step == 0 then return number end
    return math.round(number / step) * step
end

function Utils.SnapV3(vector, step)
    return Vector3.new(
        Utils.Snap(vector.X, step),
        Utils.Snap(vector.Y, step),
        Utils.Snap(vector.Z, step)
    )
end

function Utils.GetSelectionCenter()
    if #State.Selection == 0 then return nil end
    local cf = State.Selection[#State.Selection].CFrame -- Pivot around last selected
    return cf
end

function Utils.Create(className, props)
    local inst = Instance.new(className)
    for k, v in pairs(props) do inst[k] = v end
    return inst
end

function Utils.DeepCopy(t)
    local copy = {}
    for k, v in pairs(t) do
        if type(v) == "table" then copy[k] = Utils.DeepCopy(v) else copy[k] = v end
    end
    return copy
end

-- Safe execution to prevent tool breaking
function Utils.Safe(func)
    local s, e = pcall(func)
    if not s then warn("BuildAssist Error:", e) end
end

--=============================================================================
-- 4. HISTORY SYSTEM (UNDO/REDO)
--=============================================================================

local History = {}

function History.RecordState()
    -- Captures the state of current selection BEFORE a change
    local state = {}
    for _, part in pairs(State.Selection) do
        if part and part.Parent then
            table.insert(state, {
                Part = part,
                CFrame = part.CFrame,
                Size = part.Size,
                Color = part.Color,
                Parent = part.Parent
            })
        end
    end
    return state
end

function History.AddAction(actionType, beforeState, afterState)
    -- Remove redo steps if we do a new action
    if State.History.Index < #State.History.Stack then
        for i = #State.History.Stack, State.History.Index + 1, -1 do
            table.remove(State.History.Stack, i)
        end
    end

    table.insert(State.History.Stack, {
        Type = actionType,
        Before = beforeState,
        After = afterState
    })

    -- Cap size
    if #State.History.Stack > Config.Defaults.MaxHistory then
        table.remove(State.History.Stack, 1)
    else
        State.History.Index = State.History.Index + 1
    end
    
    Events.HistoryChanged:Fire()
end

function History.ApplyState(stateData)
    local newSelection = {}
    for _, item in pairs(stateData) do
        if item.Part then
            -- Handle deletion restoration
            if not item.Part.Parent and item.Parent then
                item.Part.Parent = item.Parent
            end
            
            -- Apply Properties
            if item.CFrame then item.Part.CFrame = item.CFrame end
            if item.Size then item.Part.Size = item.Size end
            if item.Color then item.Part.Color = item.Color end
            
            table.insert(newSelection, item.Part)
        end
    end
    -- Update selection to match what was undone/redone
    State.Selection = newSelection
    Events.SelectionChanged:Fire()
end

function History.Undo()
    if State.History.Index > 0 then
        local action = State.History.Stack[State.History.Index]
        History.ApplyState(action.Before)
        State.History.Index = State.History.Index - 1
        Events.HistoryChanged:Fire()
    end
end

function History.Redo()
    if State.History.Index < #State.History.Stack then
        State.History.Index = State.History.Index + 1
        local action = State.History.Stack[State.History.Index]
        History.ApplyState(action.After)
        Events.HistoryChanged:Fire()
    end
end

--=============================================================================
-- 5. SELECTION & VISUALS SYSTEM
--=============================================================================

local Selector = {}
local VisualsContainer = Utils.Create("Folder", {Name = "BuildAssist_Visuals", Parent = CoreGui})
local HandlesContainer = Utils.Create("ScreenGui", {Name = "BuildAssist_Handles", Parent = CoreGui, ResetOnSpawn = false})

function Selector.Clear()
    for _, v in pairs(State.Selection) do
        if v:FindFirstChild("BuildAssist_Highlight") then
            v.BuildAssist_Highlight:Destroy()
        end
    end
    State.Selection = {}
    Events.SelectionChanged:Fire()
end

function Selector.Add(part)
    if not part or not part:IsA("BasePart") or part:IsLocked() then return end
    if table.find(State.Selection, part) then return end
    
    table.insert(State.Selection, part)
    
    -- Visual Feedback
    local h = Utils.Create("Highlight", {
        Name = "BuildAssist_Highlight",
        Adornee = part,
        Parent = part,
        FillColor = Config.UI.Accent,
        FillTransparency = 0.8,
        OutlineColor = Config.UI.Accent,
        OutlineTransparency = 0
    })
    
    Events.SelectionChanged:Fire()
end

function Selector.Remove(part)
    local idx = table.find(State.Selection, part)
    if idx then
        if part:FindFirstChild("BuildAssist_Highlight") then
            part.BuildAssist_Highlight:Destroy()
        end
        table.remove(State.Selection, idx)
        Events.SelectionChanged:Fire()
    end
end

function Selector.Raycast()
    local unitRay = Camera:ScreenPointToRay(UserInputService:GetMouseLocation().X, UserInputService:GetMouseLocation().Y)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    -- Filter out handles and current selection to avoid self-clicking issues
    local filter = {}
    for _, p in pairs(State.Selection) do table.insert(filter, p) end
    if Player.Character then table.insert(filter, Player.Character) end
    params.FilterDescendantsInstances = filter

    return Workspace:Raycast(unitRay.Origin, unitRay.Direction * 1000, params)
end

--=============================================================================
-- 6. CORE TOOLS LOGIC (HANDLES)
--=============================================================================

local Tools = {}
local ActiveHandles = nil
local ActiveArcHandles = nil

function Tools.UpdateHandles()
    -- Clear existing
    if ActiveHandles then ActiveHandles:Destroy() ActiveHandles = nil end
    if ActiveArcHandles then ActiveArcHandles:Destroy() ActiveArcHandles = nil end

    if #State.Selection == 0 then return end
    
    local target = State.Selection[#State.Selection] -- Main target is last selected
    
    local function SetupCommon(h)
        h.Adornee = target
        h.Color3 = Config.UI.Accent
        h.Parent = HandlesContainer
        return h
    end

    if State.ActiveTool == "Move" or State.ActiveTool == "Scale" then
        ActiveHandles = SetupCommon(Instance.new("Handles"))
        ActiveHandles.Style = (State.ActiveTool == "Move") and Enum.HandlesStyle.Movement or Enum.HandlesStyle.Resize
        
        local startState = {}
        local originalCFrame = {}
        local originalSize = {}
        
        ActiveHandles.MouseButton1Down:Connect(function()
            State.IsDragging = true
            startState = History.RecordState()
            
            -- Cache relative offsets for multi-select
            for _, p in pairs(State.Selection) do
                originalCFrame[p] = p.CFrame
                originalSize[p] = p.Size
            end
        end)
        
        ActiveHandles.MouseDrag:Connect(function(face, distance)
            if not State.IsDragging then return end
            
            local snapDist = Utils.Snap(distance, State.Settings.Grid)
            
            if State.ActiveTool == "Move" then
                -- MOVE LOGIC
                for _, part in pairs(State.Selection) do
                    local cf = originalCFrame[part]
                    local delta = Vector3.FromNormalId(face) * snapDist
                    
                    if State.Settings.Axis == "Local" then
                        part.CFrame = cf * CFrame.new(delta)
                    else
                        -- World Axis logic requires transforming normal to world space relative to part
                        -- Simplified: Just use CFrame.new for local, and manual math for World
                        -- For robustness in this script, we'll stick to Local/Object space movement for Handles
                        -- or apply translation in World Space:
                        part.CFrame = CFrame.new(cf.Position + delta) * cf.Rotation
                    end
                end
            elseif State.ActiveTool == "Scale" then
                -- SCALE LOGIC
                for _, part in pairs(State.Selection) do
                    local os = originalSize[part]
                    local ocf = originalCFrame[part]
                    
                    -- Calculate size change
                    local deltaSize = math.max(0.05, os[face.Name] + snapDist) - os[face.Name]
                    local vecChange = Vector3.FromNormalId(face) * deltaSize
                    
                    -- Apply Size
                    local newSize = os + Vector3.new(math.abs(vecChange.X), math.abs(vecChange.Y), math.abs(vecChange.Z))
                    part.Size = newSize
                    
                    -- Adjust Position to scale from face (not center)
                    -- Pivot logic: Move center by half delta in direction of face
                    local posOffset = Vector3.FromNormalId(face) * (deltaSize / 2)
                    part.CFrame = ocf * CFrame.new(posOffset)
                end
            end
        end)
        
        ActiveHandles.MouseButton1Up:Connect(function()
            State.IsDragging = false
            local endState = History.RecordState()
            History.AddAction(State.ActiveTool, startState, endState)
        end)
        
    elseif State.ActiveTool == "Rotate" then
        ActiveArcHandles = SetupCommon(Instance.new("ArcHandles"))
        
        local startState = {}
        local originalCFrame = {}
        
        ActiveArcHandles.MouseButton1Down:Connect(function()
            State.IsDragging = true
            startState = History.RecordState()
            for _, p in pairs(State.Selection) do originalCFrame[p] = p.CFrame end
        end)
        
        ActiveArcHandles.MouseDrag:Connect(function(axis, relativeAngle)
            if not State.IsDragging then return end
            
            local snapAngle = Utils.Snap(math.deg(relativeAngle), State.Settings.Rot)
            local rad = math.rad(snapAngle)
            
            for _, part in pairs(State.Selection) do
                local ocf = originalCFrame[part]
                local axisVec = Vector3.new(
                    axis == Enum.Axis.X and 1 or 0,
                    axis == Enum.Axis.Y and 1 or 0,
                    axis == Enum.Axis.Z and 1 or 0
                )
                part.CFrame = ocf * CFrame.Angles(axisVec.X * rad, axisVec.Y * rad, axisVec.Z * rad)
            end
        end)
        
        ActiveArcHandles.MouseButton1Up:Connect(function()
            State.IsDragging = false
            local endState = History.RecordState()
            History.AddAction("Rotate", startState, endState)
        end)
    end
end

function Tools.Delete()
    if #State.Selection == 0 then return end
    
    local startState = History.RecordState()
    local endState = {} -- Empty because they are gone
    
    -- We need to store parents before destroying for Undo to work
    for _, part in pairs(State.Selection) do
        part.Parent = nil -- Don't destroy, just parent nil for Undo capability. 
        -- *Note: In a real game, Destroy is better, but for Undo, nil parenting is safer.*
    end
    
    History.AddAction("Delete", startState, endState)
    Selector.Clear()
end

function Tools.Copy()
    State.Clipboard = {}
    for _, part in pairs(State.Selection) do
        table.insert(State.Clipboard, {
            Size = part.Size,
            Color = part.Color,
            Material = part.Material,
            CFrame = part.CFrame,
            Name = part.Name,
            Shape = part.Shape,
            Ref = part -- Keep reference to clone from if possible
        })
    end
    -- Visual Feedback
    local hint = Instance.new("Hint", Workspace)
    hint.Text = "Copied " .. #State.Clipboard .. " parts."
    game.Debris:AddItem(hint, 1)
end

function Tools.Paste()
    if #State.Clipboard == 0 then return end
    
    Selector.Clear() -- Select newly pasted items
    local startState = {} -- Nothing before
    local createdParts = {}
    
    local center = State.Clipboard[1].CFrame.Position
    local targetPos = PlayerMouse.Hit.Position
    local offset = targetPos - center
    
    for _, data in pairs(State.Clipboard) do
        local part = Utils.Create("Part", {
            Parent = Workspace,
            Size = data.Size,
            Color = data.Color,
            Material = data.Material,
            CFrame = data.CFrame + Vector3.new(0, 5, 0), -- Paste slightly above or at mouse
            Name = data.Name,
            Anchored = true,
            TopSurface = Enum.SurfaceType.Smooth,
            BottomSurface = Enum.SurfaceType.Smooth
        })
        
        -- Move to mouse
        part.Position = data.CFrame.Position + offset + Vector3.new(0, State.Settings.Grid, 0)
        
        table.insert(createdParts, part)
        Selector.Add(part)
    end
    
    local endState = History.RecordState()
    History.AddAction("Paste", startState, endState)
end

function Tools.Duplicate()
    Tools.Copy()
    Tools.Paste()
end

--=============================================================================
-- 7. UI ENGINE (Pure Lua)
--=============================================================================

local UI = {}
local MainFrame = nil

function UI.CreateRound(parent, radius)
    local uic = Instance.new("UICorner")
    uic.CornerRadius = radius or Config.UI.CornerRadius
    uic.Parent = parent
    return uic
end

function UI.CreateButton(name, parent, text, color, callback)
    local btn = Utils.Create("TextButton", {
        Name = name,
        Parent = parent,
        BackgroundColor3 = color or Config.UI.Theme,
        Text = text,
        TextColor3 = Config.UI.Text,
        Font = Enum.Font.GothamMedium,
        TextSize = 14,
        AutoButtonColor = true
    })
    UI.CreateRound(btn)
    
    if callback then
        btn.MouseButton1Click:Connect(callback)
    end
    return btn
end

function UI.Build()
    -- Main ScreenGui
    local sg = Utils.Create("ScreenGui", {
        Name = "BuildAssistUI",
        Parent = TARGET_PARENT,
        ResetOnSpawn = false,
        IgnoreGuiInset = true
    })
    
    -- Main Panel
    MainFrame = Utils.Create("Frame", {
        Name = "MainPanel",
        Parent = sg,
        BackgroundColor3 = Color3.fromRGB(25, 25, 25),
        Position = UDim2.new(0, 20, 0.5, -200),
        Size = UDim2.new(0, Config.UI.Width, 0, 400),
        BorderSizePixel = 0,
        Active = true,
        Draggable = true
    })
    UI.CreateRound(MainFrame)
    
    -- Title Bar
    local title = Utils.Create("TextLabel", {
        Parent = MainFrame,
        Size = UDim2.new(1, -12, 0, 30),
        Position = UDim2.new(0, 6, 0, 4),
        BackgroundTransparency = 1,
        Text = "BUILD ASSIST PRO",
        Font = Enum.Font.GothamBold,
        TextColor3 = Config.UI.Accent,
        TextXAlignment = Enum.TextXAlignment.Left
    })
    
    -- Close Button
    UI.CreateButton("Close", MainFrame, "X", Config.UI.Danger, function()
        sg:Destroy()
        Selector.Clear()
        HandlesContainer:Destroy()
        script:Destroy()
    end).Size = UDim2.new(0, 25, 0, 25)
    MainFrame.Close.Position = UDim2.new(1, -30, 0, 4)
    
    -- Tool Grid Container
    local toolContainer = Utils.Create("ScrollingFrame", {
        Parent = MainFrame,
        Position = UDim2.new(0, 6, 0, 40),
        Size = UDim2.new(1, -12, 0.5, 0),
        BackgroundTransparency = 1,
        ScrollBarThickness = 2
    })
    local grid = Utils.Create("UIGridLayout", {
        Parent = toolContainer,
        CellSize = UDim2.new(0.48, 0, 0, 35),
        CellPadding = UDim2.new(0.04, 0, 0, 6)
    })
    
    -- Tool Buttons Generator
    local function AddTool(id, name, isAction)
        local btn = UI.CreateButton(id, toolContainer, name, Config.UI.Theme, function()
            if isAction then
                if Tools[id] then Tools[id]() end
            else
                State.ActiveTool = id
                Events.ToolChanged:Fire()
            end
        end)
        
        -- Visual state updater
        Events.ToolChanged:Event:Connect(function()
            if State.ActiveTool == id then
                TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = Config.UI.Accent, TextColor3 = Color3.new(1,1,1)}):Play()
            else
                TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = Config.UI.Theme, TextColor3 = Config.UI.Text}):Play()
            end
        end)
    end
    
    AddTool("Select", "Select", false)
    AddTool("Move", "Move", false)
    AddTool("Scale", "Scale", false)
    AddTool("Rotate", "Rotate", false)
    
    -- Separator
    local sep = Utils.Create("Frame", {
        Parent = toolContainer,
        BackgroundColor3 = Config.UI.SubText,
        BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,1) -- Just to take up a slot logically, handled by layout
    })
    
    AddTool("Duplicate", "Duplicate", true)
    AddTool("Delete", "Delete (Del)", true)
    AddTool("Copy", "Copy (C)", true)
    AddTool("Paste", "Paste (V)", true)
    
    -- Undo/Redo Row
    local undoFrame = Utils.Create("Frame", {
        Parent = MainFrame,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 6, 0.52, 0),
        Size = UDim2.new(1, -12, 0, 30)
    })
    
    UI.CreateButton("Undo", undoFrame, "< Undo", Config.UI.Theme, History.Undo).Size = UDim2.new(0.48, 0, 1, 0)
    local redo = UI.CreateButton("Redo", undoFrame, "Redo >", Config.UI.Theme, History.Redo)
    redo.Size = UDim2.new(0.48, 0, 1, 0)
    redo.Position = UDim2.new(0.52, 0, 0, 0)
    
    -- Properties Panel
    local propFrame = Utils.Create("Frame", {
        Parent = MainFrame,
        BackgroundColor3 = Color3.fromRGB(30, 30, 30),
        Position = UDim2.new(0, 6, 0.62, 0),
        Size = UDim2.new(1, -12, 0.36, 0)
    })
    UI.CreateRound(propFrame)
    
    local function AddSlider(name, min, max, default, key)
        local c = Utils.Create("Frame", {
            Parent = propFrame,
            BackgroundTransparency = 1,
            Size = UDim2.new(1, -10, 0, 35)
        })
        local list = Utils.Create("UIListLayout", {Parent = propFrame, Padding = UDim.new(0,4), HorizontalAlignment = Enum.HorizontalAlignment.Center})
        
        local label = Utils.Create("TextLabel", {
            Parent = c,
            Size = UDim2.new(1, 0, 0, 15),
            BackgroundTransparency = 1,
            Text = name .. ": " .. default,
            TextColor3 = Config.UI.SubText,
            Font = Enum.Font.Gotham,
            TextSize = 12,
            TextXAlignment = Enum.TextXAlignment.Left
        })
        
        local input = Utils.Create("TextBox", {
            Parent = c,
            Position = UDim2.new(0,0,0,18),
            Size = UDim2.new(1,0,0,16),
            BackgroundColor3 = Config.UI.Theme,
            Text = tostring(default),
            TextColor3 = Config.UI.Text
        })
        UI.CreateRound(input, UDim.new(0,4))
        
        input.FocusLost:Connect(function()
            local n = tonumber(input.Text)
            if n then
                n = math.clamp(n, min, max)
                State.Settings[key] = n
                label.Text = name .. ": " .. n
                input.Text = tostring(n)
            end
        end)
    end
    
    AddSlider("Grid Size", 0.1, 50, Config.Defaults.GridSize, "Grid")
    AddSlider("Rotate Step", 1, 90, Config.Defaults.RotStep, "Rot")
    
    -- Toggles
    local toggleFrame = Utils.Create("Frame", {
        Parent = propFrame,
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -10, 0, 30)
    })
    
    local snapBtn = UI.CreateButton("SnapToggle", toggleFrame, "Snap: ON", Config.UI.Accent, function()
        State.Settings.Snapping = not State.Settings.Snapping
        local btn = toggleFrame.SnapToggle
        btn.Text = "Snap: " .. (State.Settings.Snapping and "ON" or "OFF")
        btn.BackgroundColor3 = State.Settings.Snapping and Config.UI.Accent or Config.UI.Theme
    end)
    snapBtn.Size = UDim2.new(0.48, 0, 1, 0)
    
    local axisBtn = UI.CreateButton("AxisToggle", toggleFrame, "Axis: World", Config.UI.Theme, function()
        State.Settings.Axis = (State.Settings.Axis == "World") and "Local" or "World"
        toggleFrame.AxisToggle.Text = "Axis: " .. State.Settings.Axis
    end)
    axisBtn.Size = UDim2.new(0.48, 0, 1, 0)
    axisBtn.Position = UDim2.new(0.52, 0, 0, 0)
    
    -- Init Events
    Events.ToolChanged:Fire() -- Set initial visuals
end

--=============================================================================
-- 8. INPUT HANDLING
--=============================================================================

local Input = {}

function Input.Init()
    -- Selection Logic
    UserInputService.InputBegan:Connect(function(input, gpe)
        if gpe then return end -- Clicked on UI
        
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            -- Raycast
            local result = Selector.Raycast()
            
            local isMulti = UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) or UserInputService:IsKeyDown(Enum.KeyCode.LeftControl)
            
            if result and result.Instance then
                local part = result.Instance
                if part:IsA("BasePart") then
                    if isMulti then
                        if table.find(State.Selection, part) then
                            Selector.Remove(part)
                        else
                            Selector.Add(part)
                        end
                    else
                        -- If clicking a new part and not multi, clear others
                        if not table.find(State.Selection, part) then
                            Selector.Clear()
                            Selector.Add(part)
                        end
                        -- If clicking an already selected part, do nothing (ready for drag)
                    end
                elseif part.Parent:IsA("Model") then
                    -- Select Model PrimaryPart or Children (Basic logic: select clicked part)
                    if isMulti then Selector.Add(part) else Selector.Clear() Selector.Add(part) end
                end
            else
                -- Clicked void
                if not isMulti then Selector.Clear() end
            end
        end
        
        -- Shortcuts
        if input.KeyCode == Enum.KeyCode.Delete then Tools.Delete() end
        if input.KeyCode == Enum.KeyCode.Z and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then History.Undo() end
        if input.KeyCode == Enum.KeyCode.Y and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then History.Redo() end
        if input.KeyCode == Enum.KeyCode.C and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then Tools.Copy() end
        if input.KeyCode == Enum.KeyCode.V and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then Tools.Paste() end
        if input.KeyCode == Enum.KeyCode.D and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then Tools.Duplicate() end
        
        -- Tool Shortcuts
        if input.KeyCode == Enum.KeyCode.One then State.ActiveTool = "Select" Events.ToolChanged:Fire() end
        if input.KeyCode == Enum.KeyCode.Two then State.ActiveTool = "Move" Events.ToolChanged:Fire() end
        if input.KeyCode == Enum.KeyCode.Three then State.ActiveTool = "Scale" Events.ToolChanged:Fire() end
        if input.KeyCode == Enum.KeyCode.Four then State.ActiveTool = "Rotate" Events.ToolChanged:Fire() end
    end)
end

-- Hook Events
Events.SelectionChanged:Event:Connect(Tools.UpdateHandles)
Events.ToolChanged:Event:Connect(Tools.UpdateHandles)

--=============================================================================
-- 9. MAIN BOOTSTRAP
--=============================================================================

local function Init()
    -- cleanup old instances if re-running
    if TARGET_PARENT:FindFirstChild("BuildAssistUI") then TARGET_PARENT.BuildAssistUI:Destroy() end
    if CoreGui:FindFirstChild("BuildAssist_Visuals") then CoreGui.BuildAssist_Visuals:Destroy() end
    if CoreGui:FindFirstChild("BuildAssist_Handles") then CoreGui.BuildAssist_Handles:Destroy() end

    UI.Build()
    Input.Init()
    
    print("Build Assist Pro Loaded.")
end

Utils.Safe(Init)
