-- ==========================================
-- 🗡️ [ORCS] Solo Hunters - Smart Hub v51 (Blacklist & Chest Separated)
-- ==========================================
local Players = game:GetService("Players")
local ts = game:GetService("TeleportService")
local vu = game:GetService("VirtualUser")
local tweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local vim = game:GetService("VirtualInputManager")
local player = Players.LocalPlayer

-- ==========================================
-- ⚙️ Global Settings
-- ==========================================
getgenv().SmartAura = false
getgenv().AutoFarm = false 
getgenv().AutoStart = false 
getgenv().AutoSkip = false
getgenv().AutoCollect = false
getgenv().AutoChest = false -- 🌟 ตัวแปรใหม่ แยกเปิดกล่อง
getgenv().StickyMode = false
getgenv().FastWalk = false
getgenv().AutoSkill = false
getgenv().AutoRetreat = false 
getgenv().RetreatHP = 25 

getgenv().WalkSpeedValue = 50
getgenv().GodModeHeight = 6 
getgenv().RetreatHeight = 50 
getgenv().tweenSpeed = 60 

local attackLimit = 20
local isTweening = false
local safetyLimit = 800
local isRetreating = false 

local function getChar() 
    return player.Character or player.CharacterAdded:Wait() 
end

local function getHRP() 
    return getChar():WaitForChild("HumanoidRootPart", 5) 
end

local function getWeapon()
    local char = getChar()
    for _, v in pairs(char:GetChildren()) do
        if v:IsA("Tool") and not string.find(string.lower(v.Name), "health") and not string.find(string.lower(v.Name), "potion") then return v end
    end
    for _, v in pairs(player.Backpack:GetChildren()) do
        if v:IsA("Tool") and not string.find(string.lower(v.Name), "health") and not string.find(string.lower(v.Name), "potion") then return v end
    end
    return nil
end

-- ==========================================
-- 🛡️ ระบบ Auto Retreat & Heal (Force Click)
-- ==========================================
task.spawn(function()
    while task.wait(0.2) do 
        if getgenv().AutoRetreat then
            pcall(function()
                local char = getChar()
                local hum = char:FindFirstChild("Humanoid")
                if hum and hum.Health > 0 then
                    local hpPercent = (hum.Health / hum.MaxHealth) * 100
                    
                    if hpPercent <= getgenv().RetreatHP then
                        isRetreating = true
                    elseif hpPercent >= 99 then 
                        isRetreating = false
                    end

                    if isRetreating then
                        local potion = nil
                        for _, v in pairs(player.Backpack:GetChildren()) do
                            if v:IsA("Tool") and (string.find(string.lower(v.Name), "health") or string.find(string.lower(v.Name), "potion")) then
                                potion = v
                                break
                            end
                        end
                        if not potion then
                            for _, v in pairs(char:GetChildren()) do
                                if v:IsA("Tool") and (string.find(string.lower(v.Name), "health") or string.find(string.lower(v.Name), "potion")) then
                                    potion = v
                                    break
                                end
                            end
                        end
                        
                        if potion then
                            hum:EquipTool(potion)
                            task.wait(0.1) 
                            potion:Activate() 
                            
                            vu:Button1Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                            task.wait(0.05)
                            vu:Button1Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                        end
                    end
                end
            end)
        else
            isRetreating = false
        end
    end
end)

-- ==========================================
-- 👻 ระบบ Noclip
-- ==========================================
RunService.Stepped:Connect(function()
    if getgenv().AutoFarm or getgenv().StickyMode or isRetreating then
        local char = player.Character
        if char then
            for _, v in pairs(char:GetDescendants()) do
                if v:IsA("BasePart") and v.CanCollide then
                    v.CanCollide = false
                end
            end
        end
    end
end)

-- ==========================================
-- 🛡️ ระบบ Auto Start & Skip
-- ==========================================
task.spawn(function()
    while task.wait(0.5) do
        if getgenv().AutoStart or getgenv().AutoSkip then
            pcall(function()
                for _, v in pairs(player.PlayerGui:GetDescendants()) do
                    if v:IsA("GuiButton") and v.Visible and v.AbsoluteSize.X > 0 then
                        local textObj = v:FindFirstChildOfClass("TextLabel")
                        local t = string.lower(v.ClassName == "TextButton" and v.Text or (textObj and textObj.Text or ""))
                        if (getgenv().AutoStart and (string.find(t, "start") or string.find(t, "dungeon"))) or 
                           (getgenv().AutoSkip and string.find(t, "skip")) then
                            v:Activate()
                        end
                    end
                end
            end)
        end
    end
end)

-- ==========================================
-- 🧲 ระบบ Item Magnet & Auto Chest (Blacklist Edition)
-- ==========================================
-- 🌟 ตัวแปรหน่วยความจำสำหรับกันบอทสแปมรัวๆ
local dropAttempts = {}
local blacklistedUUIDs = {}
local openedChests = {}

task.spawn(function()
    local rs = game:GetService("ReplicatedStorage")
    local remoteServices = rs:WaitForChild("RemoteServices", 5)
    
    -- โหลด Remote
    local dropsRF = remoteServices and remoteServices:WaitForChild("DropsService", 5) and remoteServices.DropsService:WaitForChild("RF", 5) and remoteServices.DropsService.RF:FindFirstChild("CollectDrop")
    local chestRF = remoteServices and remoteServices:WaitForChild("BossDropsService", 5) and remoteServices.BossDropsService:WaitForChild("RF", 5) and remoteServices.BossDropsService.RF:FindFirstChild("OpenChest")
    
    local uuidPattern = "%x%x%x%x%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%x%x%x%x%x%x%x%x"

    -- ฟังก์ชันดึง UUID
    local function getUUID(obj)
        local u = string.match(obj.Name, uuidPattern)
        if u then return u end
        for _, v in pairs(obj:GetAttributes()) do
            if type(v) == "string" and string.match(v, uuidPattern) then return v end
        end
        return nil
    end

    while task.wait(0.2) do
        pcall(function()
            local hrp = getHRP()
            if not hrp then return end
            
            -- 🌟 1. ระบบ Auto Chest (แยกอิสระ)
            if getgenv().AutoChest and chestRF then
                local chestFolders = {workspace:FindFirstChild("Chests"), workspace:FindFirstChild("BossChests")}
                for _, f in pairs(chestFolders) do
                    if f then
                        for _, obj in pairs(f:GetDescendants()) do
                            local uuid = getUUID(obj)
                            if uuid and not openedChests[uuid] then
                                openedChests[uuid] = true
                                task.spawn(function()
                                    pcall(function() chestRF:InvokeServer(uuid) end)
                                end)
                            end
                        end
                    end
                end
            end

            -- 🌟 2. ระบบ Auto Collect (แบบมีบัญชีดำ + Local Destroy)
            if getgenv().AutoCollect then
                local pulledSomething = false
                local dropFolders = {workspace.CurrentCamera:FindFirstChild("Drops"), workspace:FindFirstChild("Drops")}

                for _, f in pairs(dropFolders) do
                    if f then
                        for _, obj in pairs(f:GetDescendants()) do
                            local uuid = getUUID(obj)
                            
                            -- ถ้ารู้ UUID จะพยายามยิงหลังบ้าน
                            if uuid and dropsRF then
                                if not blacklistedUUIDs[uuid] then
                                    -- บวกจำนวนครั้งที่พยายามเก็บ
                                    dropAttempts[uuid] = (dropAttempts[uuid] or 0) + 1
                                    
                                    if dropAttempts[uuid] > 3 then
                                        -- 🚨 พยายามเก็บ 3 รอบแล้วยังอยู่ แปลว่ากระเป๋าเต็ม -> แบน & ลบทิ้ง!
                                        blacklistedUUIDs[uuid] = true
                                        pcall(function() obj:Destroy() end)
                                    else
                                        task.spawn(function() 
                                            pcall(function() dropsRF:InvokeServer(uuid, false) end)
                                        end)
                                    end
                                end
                            
                            -- ถ้าหา UUID ไม่เจอจริงๆ หรือเซิร์ฟหน่วง ให้ใช้ระบบแม่เหล็กและกด E สำรอง
                            elseif obj:IsA("ProximityPrompt") then
                                local part = obj.Parent
                                if part and part:IsA("BasePart") then
                                    part.CFrame = hrp.CFrame
                                end
                                obj.RequiresLineOfSight = false
                                obj.MaxActivationDistance = math.huge
                                obj.HoldDuration = 0
                                if fireproximityprompt then 
                                    fireproximityprompt(obj, 1, true) 
                                end
                                pulledSomething = true
                            end
                        end
                    end
                end

                -- กด E สำรอง
                if pulledSomething then
                    vim:SendKeyEvent(true, Enum.KeyCode.E, false, game)
                    task.wait(0.05)
                    vim:SendKeyEvent(false, Enum.KeyCode.E, false, game)
                end
            end
        end)
    end
end)

-- ==========================================
-- 🎯 ฟังก์ชันสแกนเป้าหมาย
-- ==========================================
local function getClosestMonster()
    local closest, dist = nil, math.huge
    local hrp = getHRP()
    if not hrp then return nil end

    local folders = {
        workspace:FindFirstChild("Mobs"), 
        workspace:FindFirstChild("Bosses"), 
        workspace:FindFirstChild("Enemies"), 
        workspace:FindFirstChild("Map")
    }
    
    for _, f in pairs(folders) do
        if f then
            for _, obj in pairs(f:GetDescendants()) do
                if obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj.Humanoid.Health > 0 and obj ~= getChar() then
                    local mhrp = obj:FindFirstChild("HumanoidRootPart") or obj.PrimaryPart
                    if mhrp then
                        local d = (mhrp.Position - hrp.Position).Magnitude
                        if d < dist and d <= safetyLimit then
                            dist = d
                            closest = obj
                        end
                    end
                end
            end
        end
    end
    return closest
end

local function getActualPortal()
    local closest, dist = nil, math.huge
    local hrp = getHRP()
    local cameraPortals = workspace.CurrentCamera:FindFirstChild("PortalModel")
    local portalFolders = {cameraPortals, workspace:FindFirstChild("DungeonPortals"), workspace:FindFirstChild("PortalSpawns")}
    
    for _, f in pairs(portalFolders) do
        if f then
            if f:IsA("Model") and (string.find(f.Name, "Portal") or string.find(f.Name, "Exit")) then
                local pPart = f:FindFirstChild("ProximityPart") or f.PrimaryPart
                if pPart then
                    local d = (pPart.Position - hrp.Position).Magnitude
                    if d < dist and d < 300 then dist = d; closest = pPart end
                end
            else
                for _, obj in pairs(f:GetDescendants()) do
                    if obj:IsA("BasePart") and (string.find(obj.Name, "Portal") or string.find(obj.Name, "Exit") or obj.Name == "ProximityPart") then
                        if not string.find(string.lower(obj.Name), "torch") and not string.find(string.lower(obj.Name), "pillar") then
                            local d = (obj.Position - hrp.Position).Magnitude
                            if d < dist and d < 300 then dist = d; closest = obj end
                        end
                    end
                end
            end
        end
    end
    return closest
end

-- ==========================================
-- ⚔️ ระบบโจมตี (Kill Aura) & 🏃 Fast Walk
-- ==========================================
task.spawn(function()
    while task.wait(0.2) do 
        pcall(function()
            local char = getChar()
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                if getgenv().FastWalk then
                    hum.WalkSpeed = getgenv().WalkSpeedValue
                elseif hum.WalkSpeed == getgenv().WalkSpeedValue then
                    hum.WalkSpeed = 16
                end
            end

            if getgenv().SmartAura and not isRetreating then
                local target = getClosestMonster()
                if target then
                    local mhrp = target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart
                    if mhrp and (mhrp.Position - getHRP().Position).Magnitude <= attackLimit + 10 then
                        local weapon = getWeapon()
                        if weapon and weapon.Parent ~= char then 
                            char.Humanoid:EquipTool(weapon) 
                        end
                        if weapon then weapon:Activate() end
                        vu:Button1Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                    end
                end
            end
        end)
    end
end)

-- ==========================================
-- 💥 ระบบ Auto Skill (F, R, C Combo)
-- ==========================================
task.spawn(function()
    while task.wait(0.1) do 
        if getgenv().AutoSkill and not isRetreating then
            pcall(function()
                local target = getClosestMonster()
                if target then
                    local mhrp = target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart
                    if mhrp and (mhrp.Position - getHRP().Position).Magnitude <= attackLimit + 15 then
                        local skillKeys = {Enum.KeyCode.F, Enum.KeyCode.R, Enum.KeyCode.C}
                        for _, key in ipairs(skillKeys) do
                            vim:SendKeyEvent(true, key, false, game)
                            task.wait(0.02)
                            vim:SendKeyEvent(false, key, false, game)
                        end
                    end
                end
            end)
        end
    end
end)

-- ==========================================
-- 🚀 ระบบเคลื่อนที่
-- ==========================================
task.spawn(function()
    while true do 
        task.wait(getgenv().StickyMode and 0.01 or 0.1) 
        
        if getgenv().AutoFarm or isRetreating then
            pcall(function()
                local hrp = getHRP()
                local target = getClosestMonster()
                
                if target then
                    local mhrp = target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart
                    
                    local currentHeight = isRetreating and getgenv().RetreatHeight or getgenv().GodModeHeight
                    local targetPos = mhrp.Position + Vector3.new(0, currentHeight, 0)
                    local d = (targetPos - hrp.Position).Magnitude
                    
                    if getgenv().StickyMode or isRetreating then
                        isTweening = false 
                        hrp.Velocity = Vector3.new(0,0,0) 
                        if d > 60 then
                            hrp.CFrame = hrp.CFrame:Lerp(CFrame.new(targetPos, Vector3.new(mhrp.Position.X, mhrp.Position.Y, mhrp.Position.Z)), 0.15)
                        else
                            hrp.CFrame = CFrame.new(targetPos, Vector3.new(mhrp.Position.X, mhrp.Position.Y, mhrp.Position.Z))
                        end
                    else
                        if not isTweening then
                            if d > 3 then
                                isTweening = true
                                local tInfo = TweenInfo.new(d / getgenv().tweenSpeed, Enum.EasingStyle.Linear)
                                tweenService:Create(hrp, tInfo, {CFrame = CFrame.new(targetPos, Vector3.new(mhrp.Position.X, mhrp.Position.Y, mhrp.Position.Z))}):Play()
                                task.wait(d / getgenv().tweenSpeed)
                                isTweening = false
                            end
                        end
                    end
                else
                    if not isTweening and getgenv().AutoFarm and not isRetreating then
                        local portal = getActualPortal()
                        if portal then
                            local d = (portal.Position - hrp.Position).Magnitude
                            isTweening = true
                            tweenService:Create(hrp, TweenInfo.new(d / getgenv().tweenSpeed, Enum.EasingStyle.Linear), {
                                CFrame = portal.CFrame * CFrame.new(0,5,0)
                            }):Play()
                            task.wait(d / getgenv().tweenSpeed)
                            isTweening = false
                            task.wait(1)
                        end
                    end
                end
            end)
        end
    end
end)

-- Anti-AFK
player.Idled:Connect(function()
    vu:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    task.wait(1)
    vu:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
end)

-- ==========================================
-- 🎨 Orion GUI Setup
-- ==========================================
local OrionLib = loadstring(game:HttpGet(('https://raw.githubusercontent.com/jensonhirst/Orion/main/source')))()

local Window = OrionLib:MakeWindow({Name = " SOLO HUNTERS ", HidePremium = true, SaveConfig = false, IntroText = "Solo Hunters Hub", IntroEnabled = true})

local MainTab = Window:MakeTab({ Name = "Main Features", Icon = "rbxassetid://4483345998", PremiumOnly = false })
MainTab:AddToggle({ Name = "Smart Aura (ฟันออโต้)", Default = false, Callback = function(val) getgenv().SmartAura = val end })
MainTab:AddToggle({ Name = "Auto Skill (F, R, C)", Default = false, Callback = function(val) getgenv().AutoSkill = val end })
MainTab:AddToggle({ Name = "Auto Farm (ลุยดันเจี้ยน)", Default = false, Callback = function(val) getgenv().AutoFarm = val end })
MainTab:AddToggle({ Name = "Sticky Mode (เกาะติดมอน)", Default = false, Callback = function(val) getgenv().StickyMode = val end })
MainTab:AddToggle({ Name = "Auto Collect (ดูดไอเทมดรอป)", Default = false, Callback = function(val) getgenv().AutoCollect = val end })
MainTab:AddToggle({ Name = "Auto Chest (เปิดกล่องอัตโนมัติ)", Default = false, Callback = function(val) getgenv().AutoChest = val end }) -- 🌟 ปุ่มใหม่

local SurviveTab = Window:MakeTab({ Name = "Survival", Icon = "rbxassetid://4483345998", PremiumOnly = false })
SurviveTab:AddToggle({ Name = "Auto Retreat & Heal (หนีตาย+กดยา)", Default = false, Callback = function(val) getgenv().AutoRetreat = val end })
SurviveTab:AddSlider({ Name = "Retreat HP % (เลือดต่ำกว่านี้ให้หนี)", Min = 10, Max = 90, Default = 25, Color = Color3.fromRGB(255, 50, 50), Increment = 5, ValueName = "%", Callback = function(Value) getgenv().RetreatHP = Value end })
SurviveTab:AddTextbox({ Name = "Retreat Height (ความสูงตอนหนีตาย)", Default = "50", TextDisappear = false, Callback = function(val) local num = tonumber(val:match("-?%d+")); if num then getgenv().RetreatHeight = num end end })

local ConfigTab = Window:MakeTab({ Name = "Settings & Config", Icon = "rbxassetid://4483345998", PremiumOnly = false })
ConfigTab:AddToggle({ Name = "Auto Start", Default = false, Callback = function(val) getgenv().AutoStart = val end })
ConfigTab:AddToggle({ Name = "Auto Skip", Default = false, Callback = function(val) getgenv().AutoSkip = val end })
ConfigTab:AddToggle({ Name = "Fast Walk (วิ่งไว)", Default = false, Callback = function(val) getgenv().FastWalk = val end })
ConfigTab:AddTextbox({ Name = "Walk Speed Value", Default = "50", TextDisappear = false, Callback = function(val) local num = tonumber(val:match("%d+")); if num then getgenv().WalkSpeedValue = num end end })
ConfigTab:AddTextbox({ Name = "God Mode Height (ความสูงตอนสู้)", Default = "6", TextDisappear = false, Callback = function(val) local num = tonumber(val:match("-?%d+")); if num then getgenv().GodModeHeight = num end end })
ConfigTab:AddTextbox({ Name = "Fly Speed (Tween)", Default = "60", TextDisappear = false, Callback = function(val) local num = tonumber(val:match("%d+")); if num then getgenv().tweenSpeed = num end end })

local MiscTab = Window:MakeTab({ Name = "UI Controls", Icon = "rbxassetid://4483345998", PremiumOnly = false })
MiscTab:AddButton({ Name = "🔄 Rejoin Server", Callback = function() ts:TeleportToPlaceInstance(game.PlaceId, game.JobId, player) end })
MiscTab:AddButton({ Name = "❌ Destroy GUI", Callback = function()
    getgenv().AutoFarm = false; getgenv().AutoCollect = false; getgenv().AutoChest = false; getgenv().SmartAura = false; getgenv().AutoSkill = false; getgenv().StickyMode = false; getgenv().AutoRetreat = false; OrionLib:Destroy()
end})

OrionLib:Init()
