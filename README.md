function fly_ ()
	local Players = game:GetService("Players")
	local UserInputService = game:GetService("UserInputService")
	local RunService = game:GetService("RunService")

	local CurrentCamera = workspace.CurrentCamera

	local LocalPlayer = Players.LocalPlayer

	local Boolean = true
	local Speed = 100
	local MovementTable = {
		0,
		0,
		0,
		0,
		0,
		0
	}
	local KeyCodeTable = {
		[Enum.KeyCode.W] = 1,
		[Enum.KeyCode.A] = 2,
		[Enum.KeyCode.S] = 3,
		[Enum.KeyCode.D] = 4,
		[Enum.KeyCode.LeftControl] = 5,
		[Enum.KeyCode.Space] = 6
	}

	UserInputService.InputBegan:Connect(function(Input, ...)
		if Input.KeyCode == Enum.KeyCode.F then
			if Boolean then
				Boolean = false
			else
				Boolean = true
			end
		elseif Input.KeyCode == Enum.KeyCode.LeftShift then
			Speed = 200
		elseif KeyCodeTable[Input.KeyCode] then
			MovementTable[KeyCodeTable[Input.KeyCode]] = 1
		end
	end)

	UserInputService.InputEnded:Connect(function(Input, ...)
		if Input.KeyCode == Enum.KeyCode.LeftShift then
			Speed = 50
		elseif KeyCodeTable[Input.KeyCode] then
			MovementTable[KeyCodeTable[Input.KeyCode]] = 0
		end
	end)

	local GetMass = function(Model)
		local Mass = 0
		for _, Value in pairs(Model:GetDescendants()) do
			if Value:IsA("BasePart") then
				Mass = Mass + Value:GetMass()
			end
		end
		return Mass * workspace.Gravity
	end

	RunService.RenderStepped:Connect(function(...)
		local Character = LocalPlayer.Character
		if Character then
			local HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
			local Mass = GetMass(Character)
			if HumanoidRootPart then
				local BodyVelocity = HumanoidRootPart:FindFirstChildOfClass("BodyVelocity")
				if BodyVelocity then
					if Boolean then
						BodyVelocity.MaxForce = Vector3.new(Mass * Speed, Mass * Speed, Mass * Speed)
						BodyVelocity.Velocity = CurrentCamera.CFrame.LookVector * Speed * (MovementTable[1] - MovementTable[3]) + CurrentCamera.CFrame.RightVector * Speed * (MovementTable[4] - MovementTable[2]) + CurrentCamera.CFrame.UpVector * Speed * (MovementTable[6] - MovementTable[5])
					else
						BodyVelocity.MaxForce = Vector3.new(0, 0, 0)
						BodyVelocity.Velocity = Vector3.new(0, 2, 0)
					end
				end
			end
		end
	end)
end

function Auto_wallbang ()
	local plrs = game:GetService("Players")
	local ws = game:GetService("Workspace")
	local run = game:GetService("RunService")

	-- localize some functions we will be using
	local tick = tick -- time including milleseconds
	local delay = task.delay -- delays code to run after a certain amount of time in "parallel" without stopping the main thread
	local wait = task.wait -- waits x amount of seconds (OMG MIND EXPLODED)
	local v3New = Vector3.new -- creates a new vector

	-- some constants and other stuff that wont change
	local heartbeat = run.Heartbeat -- a signal that fires its connections every frame
	local plr = plrs.LocalPlayer -- the user's player instance
	local camera = ws:FindFirstChildOfClass("Camera") -- the user's camera - think of it as if its recording a video and ur watching that video on ur monitor (analogy)
	local gravity = v3New(0,-ws.Gravity,0) -- gravity but expressed in a vector


	-- pre declare the stuff we r going to get from the game
	local network,plrList,charTbl,HUD,clientData,bulletCheck,trajectory

	for i, v in next, getgc(true) do -- loop through garbage collection, (basically) every table and function
		if type(v) == "table" then -- if its a table
			if rawget(v,"send") then
				network = v
			elseif rawget(v,"getbodyparts") then
				plrList = getupvalue(v.getbodyparts,1)
            --[[
            a table of players and their limbs
            plrList = {
                [playerInstance] = {
                    head = head,
                    lleg = leftLeg,
                    etc...
                }
            }
            --]]
			elseif rawget(v,"setbasewalkspeed") then
				charTbl = v
			elseif rawget(v,"gammo") then
				clientData = v
			elseif rawget(v,"isplayeralive") then
				HUD = v
			end
		elseif type(v) == "function" then -- if its a function
			local name = getinfo(v).name -- gets the name of the function

			if name == "bulletcheck" then
				bulletCheck = v
				-- bulletCheck(origin,targetPos,velocity,gravity,penetration)
			elseif name == "trajectory" then
				trajectory = v
				-- trajectory(origin,gravity,targetPos,bulletSpeed)
			end
		end
	end



	-- delcaring some values that will change throughout the process
	-- mostly for hit registration and other quirks

	local bulletNum = 9e9
	-- a super lazy bypass for bullet ids
	-- bullet ids may only be used once
	-- this will probably be detected in the future, but for the moment it works
	-- super large number so its unlikely the user has already fired this many bullets
	-- will increment this every time our ragebot shoots!

	local firerateDebounce = false
	-- the server has checks to make sure you arent shooting too many bullets
	-- this means that shooting 5+ people in the same frame wont register
	-- not very ideal, so we will stop the ragebot with respect to the guns firerate

	local debounce = {}
	-- a table of players that are currently on cooldown
	-- basically, the ragebot wont be able to shoot them again for a while
	-- this is to account for lag and ping, dont want to shoot a dead body



	heartbeat:Connect(function() -- the following code will run every frame
		if not charTbl.alive or firerateDebounce then
			-- will go here if the user has not spawned
			-- or when we have recently shot, so we dont go over the limit
			return -- prevent the following code from running until next frame
		end

		local plrTeam = plr.Team -- the user's team
		local origin = camera.CFrame.p -- the position of the users camera
		local gunData = clientData.currentgun.data -- contains information on the gun the user is using
		if not gunData then
			-- will go here if the user isnt holding a gun
			-- ex. if they were holding a knife, we obviously cant shoot with a knife
			return -- prevent the following code from running until next frame
		end

		local bulletSpeed = gunData.bulletspeed -- how fast the bullet travels
		local penetrationDepth = gunData.penetrationdepth -- maximum thickness of a wall of which the bullet can shoot through
		local firerate = gunData.firerate -- firerate of the gun
		if type(firerate) == "table" then
			-- certain guns have variable firerates
			-- we are going to just pick whatever first value is assigned
			firerate = firerate[1]
		end

		for i, v in next, plrList do -- loop through the players in the game
			if i.Team ~= plrTeam and not debounce[i] and HUD:isplayeralive(i) then
				-- going to break down this huge if statement
				-- will go here ONLY if all of the following conditions are met:
				-- 1. the players are on different teams
				-- 2. they are not on cooldown (were they recently shot?)
				-- 3. make sure they are alive (sometimes dead people get in the list somehow)

				local target = v.head -- what we will be shooting at, we want all headshots so
				local targetPos = target.Position -- position of the enemy
				local velocity,travelTime = trajectory(origin,gravity,targetPos,bulletSpeed)
				-- calculates the required velocity of the bullet for it to hit the enemy
				-- this accounts for bullet drop, which is vital when the enemy is far away
				-- a straight line would lead to hit registration issues
				-- also calculates how long the bullet would be in the air for

				if bulletCheck(origin,targetPos,velocity,gravity,penetrationDepth) then
					-- will only go here if we are able to shoot the enemy
					-- essentially, every gun has a different penetration depth
					-- this is the maximum relative thickness of a wall the gun can shoot through
					-- if we have a penetration depth of 4, we cant shoot through a 5 stud wall or a 4.000001 stud wall
					-- however, we can shoot through a 1 stud wall, 3 or 3.9999 stud wall
					-- this is important so we dont shoot at people we wouldnt be able to hit anyways

					firerateDebounce = true -- do not shoot again until the time for the firerate passes
					debounce[i] = true -- do not shoot this specific player again for a while to prevent wasted ammo
					bulletNum = bulletNum + 1 -- increment the bullet id

					network:send("newbullets",{ -- tell the server we are shooting a bullet
						camerapos = origin, -- the user's position
						firepos = origin, -- the user's position

						bullets = {{velocity,bulletNum}},
						-- this is a bit of a generalization
						-- shotguns are able to shoot more than one bullet, but i have excluded that
					},tick()) -- tick() is the current time including milleseconds

					delay(travelTime,function()
						-- the game has checks to make sure the bullet was in the air for long enough
						-- essentially, did enough time pass where the bullet could have traveled to the enemies position
						-- if not enough time was passed, you may be flagged and banned
						-- this is a SUPER lazy bypass, there are better ways to do it but you can figure that out yourself :)
						network:send("bullethit",i,targetPos,target.Name,bulletNum) -- let the server know we did damage to this enemy
					end)

					delay(1000 / firerate / 60,function()
						-- allow the ragebot to shoot again after a time based on the gun's firerate
						-- the firerate is in RPM, or rounds per minute,we want to get the time between bullets
						-- where did these numbers come from? i forgor
						firerateDebounce = false
					end)
					delay(.1,function()
						-- allow the ragebot to shoot this user again in .1 seconds
						debounce[i] = nil
					end)

					return -- we shot already, prevent ragebot from even considering any other enemies
				end
			end
		end
	end)
end

function Rage_aim ()
	local Players = game:GetService("Players")
	local Camera = workspace.CurrentCamera
	local LocalPlayer = Players.LocalPlayer
	local Mouse = LocalPlayer:GetMouse()
	local GameLogic, CharTable, Trajectory
	for I,V in pairs(getgc(true)) do
		if type(V) == "table" then
			if rawget(V, "gammo") then
				GameLogic = V
			end
		elseif type(V) == "function" then
			if debug.getinfo(V).name == "getbodyparts" then
				CharTable = debug.getupvalue(V, 1)
			elseif debug.getinfo(V).name == "trajectory" then
				Trajectory = V
			end
		end
		if GameLogic and CharTable and Trajectory then break end
	end

	local function Closest()
		local Max, Close = math.huge
		for I,V in pairs(Players:GetPlayers()) do
			if V ~= LocalPlayer and V.Team ~= LocalPlayer.Team and CharTable[V] then
				local Pos, OnScreen = Camera:WorldToScreenPoint(CharTable[V].head.Position)
				if OnScreen then
					local Dist = (Vector2.new(Pos.X, Pos.Y) - Vector2.new(Mouse.X, Mouse.Y)).Magnitude
					if Dist < Max then
						Max = Dist
						Close = V
					end
				end
			end
		end
		return Close
	end
	local MT = getrawmetatable(game)
	local __index = MT.__index
	setreadonly(MT, false)
	MT.__index = newcclosure(function(self, K)
		if not checkcaller() and GameLogic.currentgun and GameLogic.currentgun.data and (self == GameLogic.currentgun.barrel or tostring(self) == "SightMark") and K == "CFrame" then
			local CharChosen = (CharTable[Closest()] and CharTable[Closest()].head)
			if CharChosen then
				local _, Time = Trajectory(self.Position, Vector3.new(0, -workspace.Gravity, 0), CharChosen.Position, GameLogic.currentgun.data.bulletspeed)
				return CFrame.new(self.Position, CharChosen.Position + (Vector3.new(0, -workspace.Gravity / 2, 0)) * (Time ^ 2) + (CharChosen.Velocity * Time))
			end
		end
		return __index(self, K)
	end)
	setreadonly(MT, true)
end

function Rejoin_when_kick ()
	local PlaceId = 292439477
	if game.PlaceId ~= PlaceId then
		return
	elseif not game:IsLoaded() then
		game.Loaded:Wait()
	end

	local Players = game:GetService("Players")
	local HttpService = game:GetService("HttpService")
	local TeleportService = game:GetService("TeleportService")

	local LocalPlayer = Players.LocalPlayer

	local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

	local ChatGame = PlayerGui:WaitForChild("ChatGame")

	local Votekick = ChatGame:WaitForChild("Votekick")

	local Title = Votekick:WaitForChild("Title")

	local Boolean, JobIds = pcall(readfile, "JobIds.json")
	if not Boolean then
		writefile("JobIds.json", HttpService:JSONEncode({}))
		JobIds = HttpService:JSONDecode(readfile("JobIds.json"))
	else
		JobIds = HttpService:JSONDecode(JobIds)
	end

	local GetRandomJobId = function()
		local JSONDecode = HttpService:JSONDecode(syn.request({
			Url = string.format("https://games.roblox.com/v1/games/%s/servers/Public?sortOrder=Asc&limit=100", tostring(PlaceId)),
			Method = "GET"
		}).Body)
		local Number = 0
		local JobId = JSONDecode.data[math.random(1, table.getn(JSONDecode.data))].id
		if table.find(JobIds, JobId) then
			repeat
				Number = math.random(1, table.getn(JSONDecode.data))
				JobId = JSONDecode.data[Number].id
				task.wait()
			until not table.find(JobIds, JobId) and JSONDecode.data[Number].playing < JSONDecode.data[Number].maxPlayers
			return JobId
		else
			return JobId
		end
	end

	Title:GetPropertyChangedSignal("Text"):Connect(function()
		if string.find(Title.Text, LocalPlayer.Name) then
			table.insert(JobIds, game.JobId)
			writefile("JobIds.json", HttpService:JSONEncode(JobIds))
			TeleportService:TeleportToPlaceInstance(PlaceId, GetRandomJobId())
		end
	end)
	wait(2.5)

	--TODO: This is where you put Loadstrings and Modules.
end

function Silent_aim ()
	local Players = game:GetService("Players")
	local Camera = workspace.CurrentCamera
	local LocalPlayer = Players.LocalPlayer
	local Mouse = LocalPlayer:GetMouse()
	local GameLogic, CharTable, Trajectory
	for I,V in pairs(getgc(true)) do
		if type(V) == "table" then
			if rawget(V, "gammo") then
				GameLogic = V
			end
		elseif type(V) == "function" then
			if debug.getinfo(V).name == "getbodyparts" then
				CharTable = debug.getupvalue(V, 1)
			elseif debug.getinfo(V).name == "trajectory" then
				Trajectory = V
			end
		end
		if GameLogic and CharTable and Trajectory then break end
	end

	local function Closest()
		local Max, Close = math.huge
		for I,V in pairs(Players:GetPlayers()) do
			if V ~= LocalPlayer and V.Team ~= LocalPlayer.Team and CharTable[V] then
				local Pos, OnScreen = Camera:WorldToScreenPoint(CharTable[V].torso.Position)
				if OnScreen then
					local Dist = (Vector2.new(Pos.X, Pos.Y) - Vector2.new(Mouse.X, Mouse.Y)).Magnitude
					if Dist < Max then
						Max = Dist
						Close = V
					end
				end
			end
		end
		return Close
	end
	local MT = getrawmetatable(game)
	local __index = MT.__index
	setreadonly(MT, false)
	MT.__index = newcclosure(function(self, K)
		if not checkcaller() and GameLogic.currentgun and GameLogic.currentgun.data and (self == GameLogic.currentgun.barrel or tostring(self) == "SightMark") and K == "CFrame" then
			local CharChosen = (CharTable[Closest()] and CharTable[Closest()].torso)
			if CharChosen then
				local _, Time = Trajectory(self.Position, Vector3.new(0, -workspace.Gravity, 0), CharChosen.Position, GameLogic.currentgun.data.bulletspeed)
				return CFrame.new(self.Position, CharChosen.Position + (Vector3.new(0, -workspace.Gravity / 2, 0)) * (Time ^ 2) + (CharChosen.Velocity * Time))
			end
		end
		return __index(self, K)
	end)
	setreadonly(MT, true)
end

function Hit_box_extender_Torso ()
	Size = 4 -- Setting higher than 8 or so will screw with the server hit detection and prevent your guns from damaging people. 8 is still easy to "rage" with. I recommend 2-5 if you want to look legit.
	Transparency = 0.3 -- Leave it at 0.5 if you want the torsos/left legs to be visible. Set to 1 to make them invisible.

	game:GetService("RunService").Stepped:Connect(function()
		for i,v in next, workspace.Players:GetDescendants() do
			if v:FindFirstChild("Left Leg") and not v:FindFirstChildWhichIsA("MeshPart") then
				sethiddenproperty(v["Left Leg"], "Massless", true)
				v["Left Leg"].CanCollide = false
				v["Left Leg"].Transparency = Transparency
				if v["Left Leg"].Size ~= Vector3.new(Size, Size, Size) and v["Left Leg"].Mesh.Scale ~= Vector3.new(Size, Size, Size) then
					v["Left Leg"].Size = Vector3.new(Size, Size, Size)
					v["Left Leg"].Mesh.Scale = Vector3.new(Size, Size, Size)
				end
				if v["Left Leg"].Parent.Parent.Name == "Bright blue" then
					v["Left Leg"].BrickColor = BrickColor.new("Bright blue")
				end
				if v["Left Leg"].Parent.Parent.Name == "Bright orange" then
					v["Left Leg"].BrickColor = BrickColor.new("Bright orange")
				end
			end
		end
	end)

	while wait() do
		for i,v in next, workspace.Ignore.DeadBody:GetChildren() do
			v:Destroy()
		end
	end
end

function Hit_box_extender_head ()
	Size = 3 -- Setting higher than 8 or so will screw with the server hit detection and prevent your guns from damaging people. 8 is still easy to "rage" with. I recommend 2-5 if you want to look legit.
	Transparency = 0.5 -- Leave it at 0.5 if you want the heads to be visible. Set to 1 to make them invisible.

	game:GetService("RunService").Stepped:Connect(function()
		for i,v in next, workspace.Players:GetDescendants() do
			if v:FindFirstChild("Head") and not v:FindFirstChildWhichIsA("MeshPart") then
				sethiddenproperty(v.Head, "Massless", true)
				v.Head.CanCollide = false
				v.Head.Transparency = Transparency
				if v.Head.Size ~= Vector3.new(Size, Size, Size) and v.Head.Mesh.Scale ~= Vector3.new(Size, Size, Size) then
					v.Head.Size = Vector3.new(Size, Size, Size)
					v.Head.Mesh.Scale = Vector3.new(Size, Size, Size)
				end
				if v.Head.Parent.Parent.Name == "Bright blue" then
					v.Head.BrickColor = BrickColor.new("Bright blue")
				end
				if v.Head.Parent.Parent.Name == "Bright orange" then
					v.Head.BrickColor = BrickColor.new("Bright orange")
				end
			end
		end
	end)

	while wait() do
		for i,v in next, workspace.Ignore.DeadBody:GetChildren() do
			v:Destroy()
		end
	end
end

function Aim_lock ()
   -- aim lock 
local client = game.Players.LocalPlayer
local camera = workspace.CurrentCamera
local mouse = client:GetMouse()
local players = game:GetService("Players")
local rs = game:GetService("RunService")
local uis = game:GetService("UserInputService")


for _, v in pairs(getgc(true)) do
    if type(v) == "table" and type(rawget(v, 'getbodyparts')) == 'function' then
        getbody = v
    end
end

for _, v in pairs(getgc(true)) do
    if type(v) == "table" and rawget(v, "sensitivity") then
        if v.sensitivity > 0.7 then
            v.sensitivity = 0.5
        end
    end
end

for _, v in pairs(getgc(true)) do
    if type(v) == "table" and rawget(v, "aimsensitivity") then
        if v.aimsensitivity > 0.7 then
            v.aimsensitivity = 0.5
        end
    end
end



if not getgenv().aim_smooth then
    getgenv().aim_smooth = 2
end

if not getgenv().aim_at then
    getgenv().aim_at = "head"
end

if not getgenv().fov then
    getgenv().fov = 400
end

local aimParts = {"head","torso"}
local function randomAimPart(table)
    local value = math.random(1,#table) -- Get random number with 1 to length of table.
    return table[value]
end


local Rayparams = RaycastParams.new();
Rayparams.FilterType = Enum.RaycastFilterType.Blacklist;



local function CheckRay(from,to)
    local CF = CFrame.new(from.Position, to.Position);
    local Hit = game.Workspace:Raycast(CF.p, CF.LookVector * (from.Position - to.Position).magnitude, Rayparams);
    if Hit.Instance.Name == "Head" then
        return true
    else
        return false
    end
end




local function closestPlayer(fov)
    local target = nil
    local closest = fov or math.huge
    for i,v in ipairs(players:GetPlayers()) do
        local character = getbody.getbodyparts(v)
        if character and client.Character and v ~= client and v.TeamColor ~= client.TeamColor  then
            local _, onscreen = camera:WorldToScreenPoint(character.head.Position)
            if onscreen then
                local targetPos = camera:WorldToViewportPoint(character.head.Position)
                local mousePos = camera:WorldToViewportPoint(mouse.Hit.p)
                local dist = (Vector2.new(mousePos.X, mousePos.Y) - Vector2.new(targetPos.X, targetPos.Y)).magnitude
                Rayparams.FilterDescendantsInstances = {client.Character}
                if dist < closest and CheckRay(game.Players.LocalPlayer.Character.Head,character.head) then
                    closest = dist
                    target = v
                end
            end
        end
    end
    return target
end

local function aimAt(pos,smooth)
    local targetPos = camera:WorldToScreenPoint(pos)
    local mousePos = camera:WorldToScreenPoint(mouse.Hit.p)
    mousemoverel((targetPos.X-mousePos.X)/smooth,(targetPos.Y-mousePos.Y)/smooth)
end
getgenv().random_aim = true
local isAiming = false
uis.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = true
        if getgenv().random_aim then
            getgenv().aim_at = randomAimPart(aimParts)
        end
    end
end)
uis.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then isAiming = false end
end)

rs.RenderStepped:connect(function()
    local t = closestPlayer(getgenv().fov)
    if isAiming and t and getgenv().aim_at ~= "random" and CheckRay(game.Players.LocalPlayer.Character.Head,getbody.getbodyparts(t).head)then
        aimAt(getbody.getbodyparts(t)[getgenv().aim_at].Position, getgenv().aim_smooth)

    elseif isAiming and t and getgenv().aim_at == "random" and CheckRay(game.Players.LocalPlayer.Character.Head,getbody.getbodyparts(t).head) then
        aimAt(getbody.getbodyparts(t)[randomAimPart(aimParts)].Position, getgenv().aim_smooth)
    end
end)
end

function TracersG ()
	local client = {}; do
		-- Tables
		client.esp = {}

		-- Modules
		for i,v in pairs(getgc(true)) do
			if (type(v) == "table") then
				if (rawget(v, "getplayerhealth")) then
					client.hud = v
				elseif (rawget(v, "getplayerhit")) then
					client.replication = v
				end
			end
		end

		client.chartable = debug.getupvalue(client.replication.getbodyparts, 1)
	end

	client.esp.Options = {
		Enable = true,
		TeamCheck = true,
		TeamColor = false,
		VisibleOnly = false,
		Color = Color3.fromRGB(0, 255, 255),
		Name = false,
		Box = false,
		Health = false,
		Distance = false,
		Tracer = true
	}

	client.esp.Services = setmetatable({}, {
		__index = function(Self, Index)
			local GetService = game.GetService
			local Service = GetService(game, Index)

			if Service then
				Self[Index] = Service
			end

			return Service
		end
	})

	local function GetDrawingObjects()
		return {
			Name = Drawing.new("Text"),
			Box = Drawing.new("Quad"),
			Tracer = Drawing.new("Line"),
		}
	end

	local function CreateEsp(Player)
		local Objects = GetDrawingObjects()
		local Character = client.chartable[Player].head.Parent
		local Head = Character.Head
		local HeadPosition = Head.Position
		local Head2dPosition, OnScreen = workspace.CurrentCamera:WorldToScreenPoint(HeadPosition)
		local Origin = workspace.CurrentCamera.CFrame.p
		local HeadPos = Head.Position
		local IgnoreList = { Character, client.esp.Services.Players.LocalPlayer.Character, workspace.CurrentCamera, workspace.Ignore }
		local PlayerRay = Ray.new(Origin, HeadPos - Origin)
		local Hit = workspace:FindPartOnRayWithIgnoreList(PlayerRay, IgnoreList)

		local function Create()
			if (OnScreen) then
				local Name = ""
				local Health = ""
				local Distance = ""

				if (client.esp.Options.Name) then
					Name = Player.Name
				end

				if (client.esp.Options.Health) then
					local Characters = debug.getupvalue(client.replication.getplayerhit, 1)
					Health = " [ " .. client.hud:getplayerhealth(Characters[Character]) .. "% ]"
				end

				if (client.esp.Options.Distance) then
					Distance = " [ " .. math.round((HeadPosition - workspace.CurrentCamera.CFrame.p).Magnitude) .. " studs ]"
				end

				Objects.Name.Visible = true
				Objects.Name.Transparency = 1
				Objects.Name.Text = string.format("%s%s%s", Name, Health, Distance)
				Objects.Name.Size = 18
				Objects.Name.Center = true
				Objects.Name.Outline = true
				Objects.Name.OutlineColor = Color3.fromRGB(0, 0, 0)
				Objects.Name.Position = Vector2.new(Head2dPosition.X, Head2dPosition.Y)

				if (client.esp.Options.TeamColor) then
					Objects.Name.Color = Player.Team.TeamColor.Color
				else
					Objects.Name.Color = Color3.fromRGB(255, 255, 255)
				end

				if (client.esp.Options.Box) then
					local Part = Character.HumanoidRootPart
					local Size = Part.Size * Vector3.new(1, 1.5)
					local Sizes = {
						TopRight = (Part.CFrame * CFrame.new(-Size.X, -Size.Y, 0)).Position,
						BottomRight = (Part.CFrame * CFrame.new(-Size.X, Size.Y, 0)).Position,
						TopLeft = (Part.CFrame * CFrame.new(Size.X, -Size.Y, 0)).Position,
						BottomLeft = (Part.CFrame * CFrame.new(Size.X, Size.Y, 0)).Position,
					}

					local TL, OnScreenTL = workspace.CurrentCamera:WorldToScreenPoint(Sizes.TopLeft)
					local TR, OnScreenTR = workspace.CurrentCamera:WorldToScreenPoint(Sizes.TopRight)
					local BL, OnScreenBL = workspace.CurrentCamera:WorldToScreenPoint(Sizes.BottomLeft)
					local BR, OnScreenBR = workspace.CurrentCamera:WorldToScreenPoint(Sizes.BottomRight)

					if (OnScreenTL and OnScreenTR and OnScreenBL and OnScreenBR) then
						Objects.Box.Visible = true
						Objects.Box.Transparency = 1
						Objects.Box.Thickness = 2
						Objects.Box.Filled = false
						Objects.Box.PointA = Vector2.new(TL.X, TL.Y + 36)
						Objects.Box.PointB = Vector2.new(TR.X, TR.Y + 36)
						Objects.Box.PointC = Vector2.new(BR.X, BR.Y + 36)
						Objects.Box.PointD = Vector2.new(BL.X, BL.Y + 36)

						if (client.esp.Options.TeamColor) then
							Objects.Box.Color = Player.Team.TeamColor.Color
						else
							Objects.Box.Color = client.esp.Options.Color
						end
					end
				end

				if (client.esp.Options.Tracer) then
					local CharTorso = Character:FindFirstChild("Torso") or Character:FindFirstChild("UpperTorso")
					local Torso, OnScreen = workspace.CurrentCamera:WorldToScreenPoint(CharTorso.Position)

					if (OnScreen) then
						Objects.Tracer.Visible = true
						Objects.Tracer.Transparency = 1
						Objects.Tracer.Thickness = 1
						Objects.Tracer.From = Vector2.new(workspace.CurrentCamera.ViewportSize.X / 2, workspace.CurrentCamera.ViewportSize.Y / 1)
						Objects.Tracer.To = Vector2.new(Torso.X, Torso.Y + 36)

						if (client.esp.Options.TeamColor) then
							Objects.Tracer.Color = Player.Team.TeamColor.Color
						else
							Objects.Tracer.Color = client.esp.Options.Color
						end
					end
				end
			end
		end

		if (client.esp.Options.VisibleOnly) then
			if (Hit == nil) then
				Create()
			end
		else
			Create()
		end

		client.esp.Services.RunService.Heartbeat:Wait()
		client.esp.Services.RunService.Heartbeat:Wait()

		Objects.Name:Remove()
		Objects.Box:Remove()
		Objects.Tracer:Remove()
	end

	client.esp.Services.RunService.RenderStepped:Connect(function()
		local LocalPlayer = client.esp.Services.Players.LocalPlayer

		for i,v in pairs(client.esp.Services.Players:GetPlayers()) do
			if (v and client.chartable[v] and v.Name ~= LocalPlayer.Name) then
				if (client.esp.Options.Enable) then
					if (client.esp.Options.TeamCheck) then
						if (v.Team ~= LocalPlayer.Team) then
							CreateEsp(v)
						end
					else
						CreateEsp(v)
					end
				end
			end
		end
	end)
end

function Name ()
	local client = {}; do
		-- Tables
		client.esp = {}

		-- Modules
		for i,v in pairs(getgc(true)) do
			if (type(v) == "table") then
				if (rawget(v, "getplayerhealth")) then
					client.hud = v
				elseif (rawget(v, "getplayerhit")) then
					client.replication = v
				end
			end
		end

		client.chartable = debug.getupvalue(client.replication.getbodyparts, 1)
	end

	client.esp.Options = {
		Enable = true,
		TeamCheck = true,
		TeamColor = false,
		VisibleOnly = false,
		Color = Color3.fromRGB(0, 255, 255),
		Name = true,
		Box = false,
		Health = false,
		Distance = false,
		Tracer = false
	}

	client.esp.Services = setmetatable({}, {
		__index = function(Self, Index)
			local GetService = game.GetService
			local Service = GetService(game, Index)

			if Service then
				Self[Index] = Service
			end

			return Service
		end
	})

	local function GetDrawingObjects()
		return {
			Name = Drawing.new("Text"),
			Box = Drawing.new("Quad"),
			Tracer = Drawing.new("Line"),
		}
	end

	local function CreateEsp(Player)
		local Objects = GetDrawingObjects()
		local Character = client.chartable[Player].head.Parent
		local Head = Character.Head
		local HeadPosition = Head.Position
		local Head2dPosition, OnScreen = workspace.CurrentCamera:WorldToScreenPoint(HeadPosition)
		local Origin = workspace.CurrentCamera.CFrame.p
		local HeadPos = Head.Position
		local IgnoreList = { Character, client.esp.Services.Players.LocalPlayer.Character, workspace.CurrentCamera, workspace.Ignore }
		local PlayerRay = Ray.new(Origin, HeadPos - Origin)
		local Hit = workspace:FindPartOnRayWithIgnoreList(PlayerRay, IgnoreList)

		local function Create()
			if (OnScreen) then
				local Name = ""
				local Health = ""
				local Distance = ""

				if (client.esp.Options.Name) then
					Name = Player.Name
				end

				if (client.esp.Options.Health) then
					local Characters = debug.getupvalue(client.replication.getplayerhit, 1)
					Health = " [ " .. client.hud:getplayerhealth(Characters[Character]) .. "% ]"
				end

				if (client.esp.Options.Distance) then
					Distance = " [ " .. math.round((HeadPosition - workspace.CurrentCamera.CFrame.p).Magnitude) .. " studs ]"
				end

				Objects.Name.Visible = true
				Objects.Name.Transparency = 1
				Objects.Name.Text = string.format("%s%s%s", Name, Health, Distance)
				Objects.Name.Size = 18
				Objects.Name.Center = true
				Objects.Name.Outline = true
				Objects.Name.OutlineColor = Color3.fromRGB(0, 0, 0)
				Objects.Name.Position = Vector2.new(Head2dPosition.X, Head2dPosition.Y)

				if (client.esp.Options.TeamColor) then
					Objects.Name.Color = Player.Team.TeamColor.Color
				else
					Objects.Name.Color = Color3.fromRGB(255, 255, 255)
				end

				if (client.esp.Options.Box) then
					local Part = Character.HumanoidRootPart
					local Size = Part.Size * Vector3.new(1, 1.5)
					local Sizes = {
						TopRight = (Part.CFrame * CFrame.new(-Size.X, -Size.Y, 0)).Position,
						BottomRight = (Part.CFrame * CFrame.new(-Size.X, Size.Y, 0)).Position,
						TopLeft = (Part.CFrame * CFrame.new(Size.X, -Size.Y, 0)).Position,
						BottomLeft = (Part.CFrame * CFrame.new(Size.X, Size.Y, 0)).Position,
					}

					local TL, OnScreenTL = workspace.CurrentCamera:WorldToScreenPoint(Sizes.TopLeft)
					local TR, OnScreenTR = workspace.CurrentCamera:WorldToScreenPoint(Sizes.TopRight)
					local BL, OnScreenBL = workspace.CurrentCamera:WorldToScreenPoint(Sizes.BottomLeft)
					local BR, OnScreenBR = workspace.CurrentCamera:WorldToScreenPoint(Sizes.BottomRight)

					if (OnScreenTL and OnScreenTR and OnScreenBL and OnScreenBR) then
						Objects.Box.Visible = true
						Objects.Box.Transparency = 1
						Objects.Box.Thickness = 2
						Objects.Box.Filled = false
						Objects.Box.PointA = Vector2.new(TL.X, TL.Y + 36)
						Objects.Box.PointB = Vector2.new(TR.X, TR.Y + 36)
						Objects.Box.PointC = Vector2.new(BR.X, BR.Y + 36)
						Objects.Box.PointD = Vector2.new(BL.X, BL.Y + 36)

						if (client.esp.Options.TeamColor) then
							Objects.Box.Color = Player.Team.TeamColor.Color
						else
							Objects.Box.Color = client.esp.Options.Color
						end
					end
				end

				if (client.esp.Options.Tracer) then
					local CharTorso = Character:FindFirstChild("Torso") or Character:FindFirstChild("UpperTorso")
					local Torso, OnScreen = workspace.CurrentCamera:WorldToScreenPoint(CharTorso.Position)

					if (OnScreen) then
						Objects.Tracer.Visible = true
						Objects.Tracer.Transparency = 1
						Objects.Tracer.Thickness = 1
						Objects.Tracer.From = Vector2.new(workspace.CurrentCamera.ViewportSize.X / 2, workspace.CurrentCamera.ViewportSize.Y / 1)
						Objects.Tracer.To = Vector2.new(Torso.X, Torso.Y + 36)

						if (client.esp.Options.TeamColor) then
							Objects.Tracer.Color = Player.Team.TeamColor.Color
						else
							Objects.Tracer.Color = client.esp.Options.Color
						end
					end
				end
			end
		end

		if (client.esp.Options.VisibleOnly) then
			if (Hit == nil) then
				Create()
			end
		else
			Create()
		end

		client.esp.Services.RunService.Heartbeat:Wait()
		client.esp.Services.RunService.Heartbeat:Wait()

		Objects.Name:Remove()
		Objects.Box:Remove()
		Objects.Tracer:Remove()
	end

	client.esp.Services.RunService.RenderStepped:Connect(function()
		local LocalPlayer = client.esp.Services.Players.LocalPlayer

		for i,v in pairs(client.esp.Services.Players:GetPlayers()) do
			if (v and client.chartable[v] and v.Name ~= LocalPlayer.Name) then
				if (client.esp.Options.Enable) then
					if (client.esp.Options.TeamCheck) then
						if (v.Team ~= LocalPlayer.Team) then
							CreateEsp(v)
						end
					else
						CreateEsp(v)
					end
				end
			end
		end
	end)
end

function Grenade_1 ()
	local runService                = game:GetService("RunService")
	local replicatedStorage         = game:GetService("ReplicatedStorage")
	local players                   = game:GetService("Players")
	local localPlayer               = players.LocalPlayer
	local camera                    = workspace.CurrentCamera

	local screenSize                = camera.ViewportSize
	local cameraFrustumHeightScale  = math.tan(math.deg(camera.FieldOfView / 2))
	local cameraFrustumWidthScale   = cameraFrustumHeightScale * screenSize.x/screenSize.y

	camera.Changed:Connect(function()
		task.wait()
		screenSize = camera.ViewportSize
		cameraFrustumHeightScale  = math.tan(math.rad(camera.FieldOfView / 2))
		cameraFrustumWidthScale   = cameraFrustumHeightScale * screenSize.x/screenSize.y
	end)

	local drawingPool               = {}
	local maxNadePerPlayer          = 3

	local char, netFuncs
	local newgrenadeFunc, newgrenadeIdx

	local Maid      = loadstring(game:HttpGet("https://raw.githubusercontent.com/Quenty/NevermoreEngine/version2/Modules/Shared/Events/Maid.lua"))()
	local imageUrl  = game:HttpGet("https://i.imgur.com/3HGuyVa.png") --thanks bbot

	for i,v in next, getgc(true) do
		local t = type(v)
		if t == "table" then
			if rawget(v, "jump") then
				char = v
			end
		elseif t == "function" then
			local info = debug.getinfo(v)
			if info.name == "call" and info.source:find("network") then
				netFuncs = debug.getupvalue(v, 1)
				for i2,v2 in next, netFuncs do
					local const = debug.getconstants(v2)
					if table.find(const, "Indicator") and table.find(const, "Ticking") then
						newgrenadeFunc, newgrenadeIdx = v2, i2
					end
				end
			end
		end
	end

	local function translateToScreenSpace(worldPos)
		local relativePos = camera.CFrame:PointToObjectSpace(worldPos)
		local depth = -relativePos.z
		local rX, rY = relativePos.x, relativePos.y
		local pX, pY = 0.5 + 0.5*rX/(cameraFrustumWidthScale*depth), 0.5 - 0.5*rY/(cameraFrustumHeightScale*depth)
		return Vector3.new(screenSize.x*pX, screenSize.y*pY, depth), depth > 0 and pX >= 0 and pX <= 1 and pY >= 0 and pY <= 1
	end


	local function lerp(a, b, t)
		if t < 0.5 then
			return a + (b - a) * t
		else
			return b - (b - a) * (1 - t)
		end
	end

	local function map(x, a, b, c, d)
		return lerp(c, d, (x - a)/(b - a))
	end

	local function newDrawing(type, prop)
		local obj = Drawing.new(type)
		if prop then
			for i,v in next, prop do
				obj[i] = v
			end
		end
		return obj
	end

	for i = 1, maxNadePerPlayer*players.MaxPlayers do
		drawingPool[i] = {
			maid = Maid.new(),
			used = false,
			objects = {
				mainSquareOutline = newDrawing("Square", {
					Color = Color3.fromRGB(30, 30, 30),
					Transparency = 1,
					Filled = true,
					Thickness = 1,
					Visible = false
				}),
				upperBorder = newDrawing("Square", {
					Color = Color3.fromRGB(255, 255, 255),
					Transparency = 1,
					Filled = true,
					Thickness = 1,
					Visible = false
				}),
				nadeLogo = newDrawing("Image", {
					Data = imageUrl,
					Rounding = 0,
					Size = Vector2.new(30, 35),
					Visible = false
				}),
				nadeTypeText = newDrawing("Text", {
					Font = 2,
					Size = 13,
					Text = "HE",
					Outline = true,
					Color = Color3.fromRGB(255, 255, 255),
					Center = false,
					Visible = false
				}),
				damageText = newDrawing("Text", {
					Font = 2,
					Size = 13,
					Text = "LETHAL",
					Outline = true,
					Color = Color3.fromRGB(255, 255, 255),
					Center = false,
					Visible = false
				}),
				secondsRemainingText = newDrawing("Text", {
					Font = 2,
					Size = 13,
					Text = "5.0s",
					Outline = true,
					Color = Color3.fromRGB(255, 255, 255),
					Center = false,
					Visible = false
				}),
				timerOutline = newDrawing("Square", {
					Color = Color3.fromRGB(0, 0, 0),
					Transparency = 1,
					Filled = true,
					Visible = false,
					Thickness = 1
				}),
				timerSquare = newDrawing("Square", {
					Color = Color3.fromRGB(0, 255, 0),
					Transparency = 1,
					Filled = true,
					Visible = false,
					Thickness = 1
				})
			}
		}
	end


	netFuncs[newgrenadeIdx] = function(plr, fragName, data)
		newgrenadeFunc(plr, fragName, data)
		local nadeData = require(replicatedStorage.GunModules:FindFirstChild(fragName))
		local d0, d1, r0, r1 = nadeData.damage0, nadeData.damage1, nadeData.range0, nadeData.range1

		local blastRadius = nadeData.blastradius

		local startTick = data.time
		local lastFrame = data.frames[#data.frames]
		local endPos = lastFrame.p0 + lastFrame.offset

		local drawing
		for i,v in next, drawingPool do
			if not v.used then
				v.used = true
				drawing = v
				break
			end
		end
		if not drawing then
			drawing = {
				maid = Maid.new(),
				used = true,
				objects = {
					mainSquareOutline = newDrawing("Square", {
						Color = Color3.fromRGB(30, 30, 30),
						Transparency = 1,
						Filled = true,
						Thickness = 1,
						Visible = false
					}),
					upperBorder = newDrawing("Square", {
						Color = Color3.fromRGB(255, 255, 255),
						Transparency = 1,
						Filled = true,
						Thickness = 1,
						Visible = false
					}),
					nadeLogo = newDrawing("Image", {
						Data = imageUrl,
						Rounding = 0,
						Size = Vector2.new(30, 35),
						Visible = false
					}),
					nadeTypeText = newDrawing("Text", {
						Font = 2,
						Size = 13,
						Text = "HE",
						Outline = true,
						Color = Color3.fromRGB(255, 255, 255),
						Center = false,
						Visible = false
					}),
					damageText = newDrawing("Text", {
						Font = 2,
						Size = 13,
						Text = "LETHAL",
						Outline = true,
						Color = Color3.fromRGB(255, 255, 255),
						Center = false,
						Visible = false
					}),
					secondsRemainingText = newDrawing("Text", {
						Font = 2,
						Size = 13,
						Text = "5.0s",
						Outline = true,
						Color = Color3.fromRGB(255, 255, 255),
						Center = false,
						Visible = false
					}),
					timerOutline = newDrawing("Square", {
						Color = Color3.fromRGB(0, 0, 0),
						Transparency = 1,
						Filled = true,
						Visible = false,
						Thickness = 1
					}),
					timerSquare = newDrawing("Square", {
						Color = Color3.fromRGB(0, 255, 0),
						Transparency = 1,
						Filled = true,
						Visible = false,
						Thickness = 1
					})
				}
			}
			table.insert(drawingPool, drawing)
			print("THIS SHOULD NEVER HAPPEN, HOLY FUCK, CONTACT int#2996 IF IT DOES")
		end


		drawing.maid:GiveTask(runService.RenderStepped:Connect(function(deltaTime)
			local percentage = (tick() - startTick)/data.blowuptime
			if percentage >= 1 then
				return drawing.maid:DoCleaning()
			end

			if char.rootpart then
				local nadeTrans = 1
				local direction = endPos - char.rootpart.Position
				local distance = direction.Magnitude

				if distance > blastRadius then
					nadeTrans = 1 - map(distance, blastRadius, blastRadius + 40, 0, 1)
				end

				local objects = drawing.objects
				local mainSquareOutline = objects.mainSquareOutline
				local upperBorder = objects.upperBorder
				local nadeLogo = objects.nadeLogo
				local nadeTypeText = objects.nadeTypeText
				local damageText = objects.damageText
				local secondsRemainingText = objects.secondsRemainingText
				local timerOutline = objects.timerOutline
				local timerSquare = objects.timerSquare

				local pos, onScreen = translateToScreenSpace(endPos)


				local secondsRemaining = math.floor((1 - percentage)*data.blowuptime*10)/10
				local nadeDamage = "Safe"
				if not workspace:FindPartOnRayWithWhitelist(Ray.new(char.rootpart.Position, direction), {workspace.Map}, true) then
					if distance < r1 then
						nadeDamage = distance < r0 and d0 or (distance < r1 and (d1 - d0) / (r1 - r0) * (distance - r0) + d0 or d1)
						if nadeDamage >= char.gethealth() then
							nadeDamage = "LETHAL!"
						else
							nadeDamage = "-" .. tostring(math.floor(nadeDamage)) .. "HP"
						end
					end
				end

				nadeTypeText.Text = fragName
				damageText.Text = nadeDamage
				secondsRemainingText.Text = tostring(secondsRemaining)

				timerSquare.Size = Vector2.new(40 * (1 - percentage), 2)
				timerSquare.Color = Color3.fromRGB(0, 255, 0):Lerp(Color3.fromRGB(255, 0, 0), percentage)

				local size = Vector2.new(nadeTypeText.TextBounds.x + damageText.TextBounds.x + secondsRemainingText.TextBounds.x, nadeTypeText.TextBounds.y + damageText.TextBounds.y + secondsRemainingText.TextBounds.y - 4)
				local ultimateSize = size + Vector2.new(nadeLogo.Size.x, 11)

				if not onScreen or math.floor(pos.x) >= ((screenSize.x - 10) - ultimateSize.x) or math.floor(pos.y) >= ((screenSize.y - 10) - ultimateSize.y) then
					local rX = cameraFrustumWidthScale * ((2*pos.x)/screenSize.x - 1) * pos.z
					local rY = cameraFrustumHeightScale * (1 - (2*pos.y)/(screenSize.y)) * pos.z

					--t0ny math begin
					local widthEdge, heightEdge = rX < 0 and 10 or ((screenSize.x - 10) - ultimateSize.x), -rY < 0 and 10 or ((screenSize.y - 10) - ultimateSize.y)
					local m = 0.5*(rY*screenSize.x + rX*screenSize.y)
					local newY = (m - rY*widthEdge)/rX
					if newY > 0 and newY < ((screenSize.y - 10) - ultimateSize.y) then
						pos = Vector2.new(widthEdge, newY)
					else
						pos = Vector2.new((m - rX*heightEdge)/rY, heightEdge)
					end
					--t0ny math end
				end

				pos = Vector2.new(math.floor(pos.x), math.floor(pos.y))

				mainSquareOutline.Position = pos
				upperBorder.Position = mainSquareOutline.Position + Vector2.new(1, 1)

				mainSquareOutline.Size = ultimateSize
				upperBorder.Size = Vector2.new(mainSquareOutline.Size.x - 2, 2)

				nadeLogo.Position = mainSquareOutline.Position + Vector2.new(0, 4)

				local startingPos = nadeLogo.Position + Vector2.new(nadeLogo.Size.x + 1, 0)

				nadeTypeText.Position = startingPos
				damageText.Position = nadeTypeText.Position + Vector2.new(0, nadeTypeText.TextBounds.y - 2)
				secondsRemainingText.Position = damageText.Position + Vector2.new(0, damageText.TextBounds.y - 2)
				timerOutline.Position = secondsRemainingText.Position + Vector2.new(0, secondsRemainingText.TextBounds.y)
				timerSquare.Position = timerOutline.Position + Vector2.new(1, 1)


				timerOutline.Size = Vector2.new(42, 4)
				nadeLogo.Position = Vector2.new(startingPos.x - nadeLogo.Size.x, startingPos.y + math.floor((size.y - nadeLogo.Size.y + timerOutline.Size.y)/2))
				for i,v in next, objects do
					v.Visible = true
					v.Transparency = nadeTrans
				end
			else
				for i,v in next, drawing.objects do
					v.Visible = false
				end
			end
		end))

		drawing.maid:GiveTask(function()
			drawing.used = false
			for i,v in next, drawing.objects do
				v.Visible = false
			end
		end)

	end
end

function Charms_1 ()
	local plr = game.Players.LocalPlayer

	local l_character = plr.Character or plr.CharacterAdded:wait()
	local f_team
	local e_team
	local e_plrlist
	local rs = game:GetService("RunService")
	local camera = workspace.CurrentCamera
	local pfId = 292439477
	local pId = game.PlaceId
	local is_pf = pfId == pId


	local function geteplrlist()
		local t = {}
		if is_pf then
			local team_color_to_string = tostring(game.Players.LocalPlayer.TeamColor)
			if team_color_to_string == "Bright orange" then
				t = workspace.Players["Bright blue"]:GetChildren()
			else
				t = workspace.Players["Bright orange"]:GetChildren()
			end
		elseif not is_pf then
			if #game.Teams:GetChildren() > 0 then
				for i,v in next, game.Players:GetPlayers() do
					if v.Team~=game.Players.LocalPlayer.Team then
						table.insert(t,v)
					end
				end
			else
				for i,v in next, game.Players:GetPlayers() do
					if v ~= game.Players.LocalPlayer then
						table.insert(t,v)
					end
				end
			end
		end
		return t
	end

	rs.Stepped:Connect(function()
		e_plrlist = geteplrlist()
	end)

	local function check_for_esp(c_model)
		if not c_model then
			return
		else
			returnv = false
			for i,v in next, c_model:GetDescendants() do
				if v:IsA("BoxHandleAdornment") then
					returnv = true
					break
				end
			end
			return returnv
		end
	end

	local function remove_esp(c_model)
		for i,v in next, c_model:GetDescendants() do
			if v:IsA("BoxHandleAdornment") then
				v:Destroy()
			end
		end
	end




	local function cast_ray(body_part)
		local rp = RaycastParams.new()
		rp.FilterDescendantsInstances = l_character:GetDescendants()
		rp.FilterType = Enum.RaycastFilterType.Blacklist

		local rcr = workspace:Raycast(camera.CFrame.Position, (body_part.Position - camera.CFrame.Position).Unit * 15000,rp)
		if rcr and rcr.Instance:IsDescendantOf(body_part.Parent) then
			return true
		else
			return false
		end
	end

	local function create_esp(c_model)
		if not c_model then
			return
		else
			if check_for_esp(c_model) then
				for i,v in next, c_model:GetChildren() do
					if v:IsA("BasePart") and v:FindFirstChild("BoxHandleAdornment") then
						local walt = v:FindFirstChild("BoxHandleAdornment")
						if cast_ray(v) then
							walt.Color3 = Color3.fromRGB(255,0,255)
						else
							walt.Color3 = Color3.fromRGB(0,255,255)
						end
					end
				end
			else
				for i,v in next, c_model:GetChildren() do
					if v:IsA("BasePart") then
						local b = Instance.new("BoxHandleAdornment")
						b.Parent = v
						b.Adornee = v
						b.AlwaysOnTop = true
						b.Size = v.Size
						b.ZIndex = 2
						b.Transparency = 0.5
					end
				end
			end
		end
	end

	setfpscap(10000)

	rs.RenderStepped:Connect(function()
		for i,v in next, e_plrlist do
			if is_pf then
				create_esp(v)
			else
				create_esp(v.Character)
			end
		end
	end)
end

function Arms_Charms ()
	getgenv().Color = Color3.fromRGB(255, 0, 255)
	getgenv().Material = Enum.Material.ForceField

	game.RunService.Heartbeat:Connect(function()
		if (workspace.CurrentCamera["Left Arm"] ~= nil) then
			for i, v in pairs(workspace.CurrentCamera:GetChildren()) do
				if (v:IsA("Model") and v.Name:find("Arm")) then
					for i2, v2 in pairs(v:GetChildren()) do
						if (v2:IsA("MeshPart") or v2:IsA("BasePart")) then
							v2.Color = getgenv().Color
							v2.Material = getgenv().Material
						end
					end
				end
			end

			for i, v in pairs(workspace.CurrentCamera:GetChildren()) do
				if (v.Name ~= "Left Arm" or v.Name ~= "Right Arm") then
					if (v:IsA("Model")) then
						for i2, v2 in pairs(v:GetChildren()) do
							if (v2:IsA("MeshPart") or v2:IsA("BasePart")) then
								v2.Color = getgenv().Color
								v2.Material = getgenv().Material
							end
						end
					end
				end
			end
		end
	end)
end

function Fullbright_1 ()
	local L = game:GetService("Lighting")

	L:GetPropertyChangedSignal("Brightness"):connect(function()
		L.Brightness = math.huge;
	end)

	L:GetPropertyChangedSignal("Ambient"):connect(function()
		L.Ambient = Color3.fromRGB(255,255,255)
	end)

	L:GetPropertyChangedSignal("GlobalShadows"):connect(function()
		L.GlobalShadows = false;
	end)

	L.Brightness = math.huge;
	L.Ambient = Color3.fromRGB(255,255,255)
	L.GlobalShadows = false;
	L.MapSaturation:Destroy()
	L.SkyBox:Destroy()
	L.BlackWhite:Destroy()
	sethiddenproperty(game.Workspace.Lighting,"Technology",2)
end

function Grenade_prediction ()
	local pfCamera;
	local cfModule;
	local char;
	local oldLoadGrenade;
	local network;
	local oldSend;
	local roundsystem;
	local beamStorage = Instance.new("Part", workspace.Terrain);
	beamStorage.Anchored = true;
	beamStorage.Transparency = 1;
	beamStorage.CanCollide = false;
	local runService = game:GetService("RunService");


	--grenade constants
	local castResolution = 1/60;
	local nadeAcceleration = Vector3.new(0, -80, 0);
	local bounceElasticity = 0.2;
	local frictionConstant = 0.08;

	local function isBadNumber(x)
		return x ~= x or math.abs(x) == 1 / 0;
	end


	local function bezier(origin, velocity, acceleration, time)
		local p3 = origin + time*(0.5*acceleration*time + velocity);
		local p2 = p3 - time*(acceleration*time + velocity)/3;
		local p1 = (origin*7/3) - p2 - p3/3 + (acceleration*time*time)/3 + (time*velocity*4/3);


		local curve0 = (p1 - origin).Magnitude;
		local curve1 = -((p2 - p3).Magnitude);


		local b = (origin - p3).Unit;
		local r1 = (p1 - origin).Unit;
		local u1 = r1:Cross(b).Unit;
		local r2 = (p2 - p3).Unit;
		local u2 = r2:Cross(b).Unit;
		b = u1:Cross(r1);

		return curve0, curve1, CFrame.fromMatrix(origin, r1, u1, b), CFrame.fromMatrix(p3, r2, u2, b);
	end

	local function nadeBeam(origin, velocity, acceleration, time)
		local curve0, curve1, cfA, cfB = bezier(origin, velocity, acceleration, time);

		local beam = Instance.new("Beam");
		local a0 = Instance.new("Attachment");
		local a1 = Instance.new("Attachment");

		a0.CFrame = cfA;
		a1.CFrame = cfB;

		a0.Parent = beamStorage;
		a1.Parent = beamStorage;

		beam.Attachment0 = a0;
		beam.Attachment1 = a1;
		beam.CurveSize0 = curve0;
		beam.CurveSize1 = curve1;
		beam.Segments = 10000;
		beam.Color = ColorSequence.new(Color3.new(1, 1, 1));
		beam.Transparency = NumberSequence.new(0);
		beam.Width0 = 0.05;
		beam.Width1 = 0.05;
		beam.FaceCamera = true;
		beam.Parent = beamStorage;
	end


	local function generateNadeArguments(flyingNade, nadeData, remainingSecond)
		local aimCf = pfCamera.cframe * CFrame.Angles(math.rad(nadeData.throwangle or 0), 0, 0);
		local position = flyingNade.Position;
		local velocity = aimCf.LookVector * nadeData.throwspeed + char.rootpart.Velocity;
		do
			--do a check incase nade clips inside a wall
			local p = pfCamera.basecframe.p;
			local _, pos, norm = workspace:FindPartOnRayWithWhitelist(Ray.new(p, position - p), roundsystem.raycastwhitelist);
			position = pos + 0.01*norm;
		end
		local av0 = (pfCamera.cframe - pfCamera.cframe.p) * Vector3.new(19.539, -5, 0);
		local rot0 = flyingNade.CFrame - flyingNade.CFrame.p;
		local offset = Vector3.new();
		local lastbounce = false;
		local glassCount = 0;
		local frames = {
			{
				t0 = 0,
				p0 = position,
				v0 = velocity,
				offset = offset,
				rot0 = rot0,
				rotv = av0,
				glassbreaks = {}
			}
		};

		for i = 1, remainingSecond/castResolution + 1 do
			local newPosition = position + castResolution*velocity + castResolution*castResolution/2*nadeAcceleration;
			local hit, pos, norm = workspace:FindPartOnRayWithWhitelist(Ray.new(position, newPosition - position - 0.05 * offset), roundsystem.raycastwhitelist);
			local time = i*castResolution;
			if hit and hit.Name ~= "Window" and hit.Name ~= "Col" then
				rot0 = flyingNade.CFrame - flyingNade.CFrame.p;
				offset = 0.2*norm;
				av0 = norm:Cross(velocity) / 0.2;
				local delta = pos - position;
				local fixpls = 1 - 0.001 / delta.Magnitude;
				fixpls = fixpls < 0 and 0 or fixpls;
				position = position + fixpls*delta + 0.05*norm;
				local normvel = norm:Dot(velocity)*norm;
				local tanvel = velocity - normvel;
				local geometricdeceleration;
				local d1 = -norm:Dot(nadeAcceleration);
				local d2 = -(1 + bounceElasticity) * norm:Dot(velocity);
				geometricdeceleration = 1 - frictionConstant * (10 * (d1 < 0 and 0 or d1) * castResolution + (d2 < 0 and 0 or d2)) / tanvel.Magnitude;
				geometricdeceleration = geometricdeceleration < 0 and 0 or geometricdeceleration;
				velocity = geometricdeceleration * tanvel - bounceElasticity * normvel;
				if velocity.Magnitude < 1 then
					frames[#frames + 1] = {
						t0 = time - castResolution*newPosition:Dot(pos)/newPosition:Dot(position), --magnitude/magnitude = dot/dot
						p0 = position,
						v0 = Vector3.new(),
						rot0 = cfModule.fromaxisangle(time * av0) * rot0,
						b = true,
						offset = 0.2 * norm,
						rotv = Vector3.new(),
						glassbreaks = {}
					};
					break;
				end
				frames[#frames + 1] = {
					t0 = time - castResolution*newPosition:Dot(pos)/newPosition:Dot(position),
					p0 = position,
					v0 = velocity,
					rot0 = cfModule.fromaxisangle(time * av0) * rot0,
					b = lastbounce,
					offset = 0.2 * norm,
					rotv = av0,
					glassbreaks = {}
				};
				lastbounce = true;
			else
				position = newPosition;
				velocity = velocity + castResolution*nadeAcceleration;
				lastbounce = false;
				if hit and hit.Name == "Window" and glassCount < 5 then
					glassCount = glassCount + 1;
					frames[#frames].glassbreaks[#frames[#frames].glassbreaks + 1] = {
						t = time,
						part = hit
					};
				end
			end
		end
		return frames;
	end




	for i,v in ipairs(getgc(true)) do
		if type(v) == "table" then
			if rawget(v, "basecframe") then
				pfCamera = v;
			elseif rawget(v, "fromaxisangle") then
				cfModule = v;
			elseif rawget(v, "jump") then
				char = v;
				oldLoadGrenade = v.loadgrenade;
			elseif rawget(v, "send") then
				network = v;
				oldSend = v.send;
			elseif rawget(v, "raycastwhitelist") then
				roundsystem = v;
			end
		end
	end


	char.loadgrenade = function(...)
		local _, nadeName = ...;
		local self = oldLoadGrenade(...);
		local oldEquip = self.setequipped;
		local oldThrow = self.throw;
		local oldCreatenade;
		local nadeData;
		local start, blowUp;
		local connection;
		local frames;
		local appliedConnection = false;
		for i,v in next, debug.getupvalues(oldThrow) do
			if type(v) == "function" and debug.getinfo(v).name == "createnade" then
				oldCreatenade = v;
				print("createnade found");
				debug.setupvalue(oldThrow, i, function(...)
					v(...);
					for j,k in next, debug.getupvalues(v) do
						if type(k) == "table" and rawget(k, "frames") then
							k.frames = frames;
						end
					end
				end);
				break;
			end
		end
		for i,v in next, debug.getupvalues(oldEquip) do
			if type(v) == "table" and rawget(v, "type") and rawget(v, "crosssize") then
				nadeData = v;
				print("nade data found");
				break;
			end
		end



		self.setequipped = function(this, on)
			if on then
				local lastRan = tick();
				start = lastRan;
				blowUp = start + 5;
				connection = runService.RenderStepped:Connect(function()
					if not char.alive then
						if connection then
							connection:Disconnect();
							beamStorage:ClearAllChildren();
						end
					end
					local now = tick();
					if lastRan <= now then
						lastRan = now + castResolution - (now - lastRan)%castResolution;
						local remaining = blowUp - now;
						local flyingNade = debug.getupvalue(oldCreatenade, 2);
						if flyingNade then
							if not appliedConnection then
								--gay but kys
								appliedConnection = true;
								flyingNade.AncestryChanged:Connect(function(_, new)
									if new == nil then
										beamStorage:ClearAllChildren();
									end
								end);
							end
							frames = generateNadeArguments(flyingNade, nadeData, remaining);
							beamStorage:ClearAllChildren();
							for j,currentFrame in next, frames do
								local nextFrame = frames[j + 1];
								if nextFrame then
									local origin = currentFrame.p0 + currentFrame.offset;
									local time = nextFrame.t0 - currentFrame.t0;
									local acceleration = currentFrame.b and Vector3.new() or nadeAcceleration;
									local velocity = ((nextFrame.p0 + nextFrame.offset) - origin - 0.5*acceleration*time*time)/time;
									nadeBeam(origin, velocity, acceleration, time);
								end
							end
						end
					end
				end);
			end
			return oldEquip(this, on);
		end

		self.throw = function(...)
			appliedConnection = false;
			if connection then
				connection:Disconnect();
				--beamStorage:ClearAllChildren();
			end
			if frames then
				oldSend(network, "newgrenade", nadeName, {
					time = tick(),
					blowuptime = blowUp - tick(),
					frames = frames
				});
			end
			return oldThrow(...);
		end
		return self;
	end

	network.send = function(self, name, ...)
		if name == "newgrenade" then
			return;
		end
		return oldSend(self, name, ...);
	end
end

function Droped_guns ()
	loadstring(game:HttpGet("https://gist.githubusercontent.com/Mick-gordon/40f30b722da085fcd526139f150763b4/raw/3a836d7e1b376f0f81dd17baa34858a4c6237c52/gistfile1.txt", true))()
end

function fire_rate ()
	for i, a in pairs(getgc(true)) do
		if type(a) == 'table' and rawget(a, "aimrotkickmin") then
			a.firerate = 1000 --Change to your own
		end
	end
end

function No_Recoil ()
	for i, a in pairs(getgc(true)) do
		if type(a) == 'table' and rawget(a, "aimrotkickmin") then
			a.aimrotkickmin = Vector3.new(0,0,0)
			a.aimrotkickmax = Vector3.new(0,0,0)
			a.aimtranskickmin = Vector3.new(0,0,0)
			a.aimtranskickmax = Vector3.new(0,0,0)
			a.aimcamkickmin = Vector3.new(0,0,0)
			a.aimcamkickmax = Vector3.new(0,0,0)
			a.rotkickmin = Vector3.new(0,0,0)
			a.rotkickmax = Vector3.new(0,0,0)
			a.transkickmin = Vector3.new(0,0,0)
			a.transkickmax = Vector3.new(0,0,0)
			a.camkickmin = Vector3.new(0,0,0)
			a.camkickmax = Vector3.new(0,0,0)
			a.aimcamkickspeed = 99999
			a.modelkickspeed = 9999
			a.modelrecoverspeed = 9999
		end
	end
end

function Conbine_Mags ()
	local GunHandler
	for i,v in next, getgc(true) do
		if type(v) == "table" then
			if rawget(v, "setsprintdisable", "setequipped") then
				GunHandler = v
			end
		end
		if GunHandler then break end
	end

	if not GunHandler then return game.Players.LocalPlayer:Kick("Failed to find GunHandler") end

	game:GetService("RunService").RenderStepped:Connect(function()
		local Gun = GunHandler.currentgun
		if Gun and Gun.setequipped then
			local UpValues = debug.getupvalues(Gun.setequipped)
			local Magazine, CurrentAmmo = UpValues[11], UpValues[10]
			if Magazine ~= 0 then
				debug.setupvalue(Gun.setequipped, 10, (Magazine+CurrentAmmo))
				debug.setupvalue(Gun.setequipped, 11, 0)
			end
		end
	end)
end


function Rage_Config ()
		Auto_wallbang()
		fly_()
		Charms_1()
		TracersG()
		Hit_box_extender_head()
		
end

function Rage_With_Visuals ()
		Auto_wallbang()
		fly_()
		Charms_1()
		TracersG()
		Hit_box_extender_head()
		Grenade_1()
		Gray_hhhp ()
		Concrete_hhhp ()
		Arms_Charms()
end

function Legit_Config ()
		Gray_hhhp ()
		Concrete_hhhp ()
		Hit_box_extender_Torso()
		Grenade_prediction()
	    Grenade_1()
        Charms_1()
end

function Beta_Rage ()
	--[[
        * Line 1247 for name and health 
        * Line 1418 for boxes
        * Line 774 rexcute 
        --------------------
        * No Fall Damage
        * WalkSpeed
        * JumpPower
        * SilentAim
        * Headshot Percentage
        * Wallhacks
    
]]

	-- Default launch settings
	local settings = {
		silentaim = true,
		nofalldamage = true,

		setwalkspeed = true,
		walkspeed = 55,

		setjumppower = true,
		jumppower = 45,


		headshotchanceenabled = true,
		headshotchance = 100,

		--

		gunVisuals = false,
		deadVisuals = true,
		colorEffect = false,

		--

		fov = 1000,
		fovcircle = true,
		fovsides = 12,
		fovthickness = 1
	}

	local main = {running = true}
	local end_funcs = {}
	local __SETTINGS__ = {}
	function main:End()
		main.running = false
		for _,func in pairs(end_funcs) do
			func()
		end
		if (main == shared.__main) then
			shared.__main = nil -- k
		end

		main = nil
		spawn(function()
			for i,v in pairs(connections) do
				pcall(function() v:Disconnect() end)
			end
		end)
		shared.main = nil -- k
		__SETTINGS__:Save()
	end
	if shared.__main then
		pcall(shared.__main.End, shared.__main)
		shared.__main = nil
	end
	shared.__main = main

	local connections = {}
	local function bindEvent(event, callback) -- Let me disconnect in peace
		local con = event:Connect(callback)
		table.insert(connections, con)
		return con
	end
	table.insert(end_funcs, function()
		for i,v in pairs(connections) do
			v:Disconnect()
			connections[i] = nil
		end
	end)

	local servs
	servs = setmetatable(
		{
			Get = function(self, serv)
				if servs[serv] then return servs[serv] end
				local s = game:GetService(serv)
				if s then servs[serv] = s end
				return s
			end;
		}, {
			__index = function(self, index)
				local s = game:GetService(index)
				if s then servs[index] = s end
				return s
			end;
		})

	local players = servs.Players
	local runservice = servs.RunService
	local http = servs.HttpService
	local uis = servs.UserInputService

	local function jsonEncode(t)
		return http:JSONEncode(t)
	end
	local function jsonDecode(t)
		return http:JSONDecode(t)
	end

	local function existsFile(name)
		return pcall(function()
			return readfile(name)
		end)
	end

	local function mergetab(a,b)
		local c = a or {}
		for i,v in pairs(b or {}) do 
			c[i] = v 
		end
		return c
	end

	local serialize
	local deserialize
	do
		--/ Serializer : garbage : slow as fuck

		local function hex_encode(IN, len)
			local B,K,OUT,I,D=16,"0123456789ABCDEF","",0,nil
			while IN>0 do
				I=I+1
				IN,D=math.floor(IN/B), IN%B+1
				OUT=string.sub(K,D,D)..OUT
			end
			if len then
				OUT = ('0'):rep(len - #OUT) .. OUT
			end
			return OUT
		end
		local function hex_decode(IN) 
			return tonumber(IN, 16) 
		end

		local types = {
			["nil"] = "0";
			["boolean"] = "1";
			["number"] = "2";
			["string"] = "3";
			["table"] = "4";

			["Vector3"] = "5";
			["CFrame"] = "6";
			["Instance"] = "7";

			["Color3"] = "8";
		}
		local rtypes = (function()
			local a = {}
			for i,v in pairs(types) do
				a[v] = i
			end
			return a
		end)()

		local typeof = typeof or type
		local function encode(t, ...)
			local type = typeof(t)
			local s = types[type]
			local c = ''
			if type == "nil" then
				c = types[type] .. "0"
			elseif type == "boolean" then
				local t = t == true and '1' or '0'
				c = s .. t
			elseif type == "number" then
				local new = tostring(t)
				local len = #new
				c = s .. len .. "." .. new
			elseif type == "string" then
				local new = t
				local len = #new
				c = s .. len .. "." .. new
			elseif type == "Vector3" then
				local x,y,z = tostring(t.X), tostring(t.Y), tostring(t.Z)
				local new = hex_encode(#x, 2) .. x .. hex_encode(#y, 2) .. y .. hex_encode(#z, 2) .. z
				c = s .. new
			elseif type == "CFrame" then
				local a = {t:GetComponents()}
				local new = ''
				for i,v in pairs(a) do
					local l = tostring(v)
					new = new .. hex_encode(#l, 2) .. l
				end
				c = s .. new
			elseif type == "Color3" then
				local a = {t.R, t.G, t.B}
				local new = ''
				for i,v in pairs(a) do
					local l = tostring(v)
					new = new .. hex_encode(#l, 2) .. l
				end
				c = s .. new
			elseif type == "table" then
				return serialize(t, ...)
			end
			return c
		end
		local function decode(t, extra)
			local p = 0
			local function read(l)
				l = l or 1
				p = p + l
				return t:sub(p-l + 1, p)
			end
			local function get(a)
				local k = ""
				while p < #t do
					if t:sub(p+1,p+1) == a then
						break
					else
						k = k .. read()
					end
				end
				return k
			end
			local type = rtypes[read()]
			local c

			if type == "nil" then
				read()
			elseif type == "boolean" then
				local d = read()
				c = d == "1" and true or false
			elseif type == "number" then
				local length = tonumber(get("."))
				local d = read(length+1):sub(2,-1)
				c = tonumber(d)
			elseif type == "string" then
				local length = tonumber(get(".")) --read()
				local d = read(length+1):sub(2,-1)
				c = d
			elseif type == "Vector3" then
				local function getnext()
					local length = hex_decode(read(2))
					local a = read(tonumber(length))
					return tonumber(a)
				end
				local x,y,z = getnext(),getnext(),getnext()
				c = Vector3.new(x, y, z)
			elseif type == "CFrame" then
				local a = {}
				for i = 1,12 do
					local l = hex_decode(read(2))
					local b = read(tonumber(l))
					a[i] = tonumber(b)
				end
				c = CFrame.new(unpack(a))
			elseif type == "Instance" then
				local pos = hex_decode(read(2))
				c = extra[tonumber(pos)]
			elseif type == "Color3" then
				local a = {}
				for i = 1,3 do
					local l = hex_decode(read(2))
					local b = read(tonumber(l))
					a[i] = tonumber(b)
				end
				c = Color3.new(unpack(a))
			end
			return c
		end

		function serialize(data, p)
			if data == nil then return end
			local type = typeof(data)
			if type == "table" then
				local extra = {}
				local s = types[type]
				local new = ""
				local p = p or 0
				for i,v in pairs(data) do
					local i1,camera
					local t0,t1 = typeof(i), typeof(v)

					local a,b
					if t0 == "Instance" then
						p = p + 1
						extra[p] = i
						i1 = types[t0] .. hex_encode(p, 2)
					else
						i1, a = encode(i, p)
						if a then
							for i,v in pairs(a) do
								extra[i] = v
							end
						end
					end

					if t1 == "Instance" then
						p = p + 1
						extra[p] = v
						camera = types[t1] .. hex_encode(p, 2)
					else
						camera, b = encode(v, p)
						if b then
							for i,v in pairs(b) do
								extra[i] = v
							end
						end
					end
					new = new .. i1 .. camera
				end
				return s .. #new .. "." .. new, extra
			elseif type == "Instance" then
				return types[type] .. hex_encode(1, 2), {data}
			else
				return encode(data), {}
			end
		end

		function deserialize(data, extra)
			if data == nil then return end
			extra = extra or {}

			local type = rtypes[data:sub(1,1)]
			if type == "table" then

				local p = 0
				local function read(l)
					l = l or 1
					p = p + l
					return data:sub(p-l + 1, p)
				end
				local function get(a)
					local k = ""
					while p < #data do
						if data:sub(p+1,p+1) == a then
							break
						else
							k = k .. read()
						end
					end
					return k
				end

				local length = tonumber(get("."):sub(2, -1))
				read()

				local new = {}

				local l = 0
				while p <= length do
					l = l + 1

					local function getnext()
						local i
						local t = read()
						local type = rtypes[t]

						if type == "nil" then
							i = decode(t .. read())
						elseif type == "boolean" then
							i = decode(t .. read())
						elseif type == "number" then
							local l = get(".")

							local dc = t .. l .. read()
							local a = read(tonumber(l))
							dc = dc .. a

							i = decode(dc)
						elseif type == "string" then
							local l = get(".")
							local dc = t .. l .. read()
							local a = read(tonumber(l))
							dc = dc .. a

							i = decode(dc)
						elseif type == "Vector3" then
							local function getnext()
								local length = hex_decode(read(2))
								local a = read(tonumber(length))
								return tonumber(a)
							end
							local x,y,z = getnext(),getnext(),getnext()
							i = Vector3.new(x, y, z)
						elseif type == "CFrame" then
							local a = {}
							for i = 1,12 do
								local l = hex_decode(read(2))
								local b = read(tonumber(l)) -- why did I decide to do this
								a[i] = tonumber(b)
							end
							i = CFrame.new(unpack(a))
						elseif type == "Instance" then
							local pos = hex_decode(read(2))
							i = extra[tonumber(pos)]
						elseif type == "Color3" then
							local a = {}
							for i = 1,3 do
								local l = hex_decode(read(2))
								local b = read(tonumber(l))
								a[i] = tonumber(b)
							end
							i = Color3.new(unpack(a))
						elseif type == "table" then
							local l = get(".")
							local dc = t .. l .. read() .. read(tonumber(l))
							i = deserialize(dc, extra)
						end
						return i
					end
					local i = getnext()
					local v = getnext()

					new[(typeof(i) ~= "nil" and i or l)] =  v
				end


				return new
			elseif type == "Instance" then
				local pos = tonumber(hex_decode(data:sub(2,3)))
				return extra[pos]
			else
				return decode(data, extra)
			end
		end
	end


	local utility do
		-- shit
		utility = {}

		local servs
		servs = setmetatable({}, {
			__index = function(self, index)
				local s = game:GetService(index)
				if s then servs[index] = s end
				return s
			end;
		})

		local game, workspace = game, workspace

		local v2 = Vector2
		local math, table = math, table

		local v2new = v2.new

		local players = servs.Players
		local locpl = players.LocalPlayer
		local mouse = locpl:GetMouse()
		local currentcamera = workspace.CurrentCamera

		local findFirstChildOfClass = game.FindFirstChildOfClass
		local isDescendantOf = game.IsDescendantOf    

		local getPlayers = players.GetPlayers
		local getPartsObscuringTarget = currentcamera.GetPartsObscuringTarget
		local worldToViewportPoint = currentcamera.WorldToViewportPoint
		local raynew = Ray.new
		local findPartOnRayWithIgnoreList = workspace.FindPartOnRayWithIgnoreList
		local findFirstChild = game.FindFirstChild

		local function raycast(ray, ignore, callback)
			local ignore = ignore or {}

			local hit, pos, normal, material = findPartOnRayWithIgnoreList(workspace, ray, ignore)
			while hit and callback do
				local Continue, _ignore = callback(hit)
				if not Continue then
					break
				end
				if _ignore then
					table.insert(ignore, _ignore)
				else
					table.insert(ignore, hit)
				end
				hit, pos, normal, material = findPartOnRayWithIgnoreList(workspace, ray, ignore)
			end
			return hit, pos, normal, material
		end

		local function badraycastnotevensure(pos, ignore) -- 1 ray > 1 obscuringthing | 100 rays < 1 obscuring thing
			local hitparts = getPartsObscuringTarget(currentcamera, {pos}, ignore or {})
			return hitparts
		end

		local charshit = {}
		function utility.getcharacter(player) -- Change this or something if you want to add support for other games.
			if (player == nil) then return end
			if (charshit[player]) then return charshit[player] end

			local char = player.Character
			if (char == nil or isDescendantOf(char, game) == false) then
				char = findFirstChild(workspace, player.Name)
			end

			return char
		end

		utility.mychar = nil
		utility.myroot = nil

		local rootshit = {}
		function utility.getroot(player)
			if (player == nil) then return end
			if (rootshit[player]) then return rootshit[player] end

			local char
			if (player:IsA("Player")) then
				char = utility.getcharacter(player)
			else
				char = player
			end

			if (char ~= nil) then
				return (findFirstChild(char, "Torso") or char.PrimaryPart)
			end

			return
		end

		function utility.isalive(_1, _2)
			if _1 == nil then return end
			local Char, RootPart
			if _2 ~= nil then
				Char, RootPart = _1,_2
			else
				Char = utility.getcharacter(_1)
				RootPart = Char and (Char:FindFirstChild("Torso") or Char.PrimaryPart)
			end

			if Char and RootPart then
				local Human = findFirstChildOfClass(Char, "Humanoid")
				if RootPart and Human then
					if Human.Health > 0 then
						return true
					end
				elseif RootPart and isDescendantOf(Char, game) then
					return true
				end
			end

			return false
		end

		local shit = false
		function utility.isvisible(char, root, max, ...)
			local pos = root.Position
			if shit or max > 4 then
				local parts = badraycastnotevensure(pos, {utility.mychar, ..., workspace.CurrentCamera, char, root, workspace:FindFirstChild("Ignore"), })

				return parts <= max
			else
				local camp = currentcamera.CFrame.p
				local dist = (camp - pos).Magnitude

				local hitt = 0
				local hit = raycast(raynew(camp, (pos - camp).unit * dist), {utility.mychar, ..., currentcamera}, function(hit)
					if hit.Name == "Window" then
						return true
					end

					if hit.CanCollide == true then-- hit.Transparency ~= 1 then
						hitt = hitt + 1
						return hitt < max
					end

					if isDescendantOf(hit, char) then
						return
					end

					return true
				end)

				return hit == nil or isDescendantOf(hit, char) or hitt <= max, hitt
			end
		end
		function utility.sameteam(player, p1)
			local p0 = p1 or locpl
			return (player.Team~=nil and player.Team==p0.Team) and player.Neutral == false or false
		end
		function utility.getDistanceFromMouse(position)
			local screenpos, vis = worldToViewportPoint(currentcamera, position)
			if vis and screenpos.Z > 0 then
				return (v2new(mouse.X, mouse.Y) - v2new(screenpos.X, screenpos.Y)).Magnitude
			end
			return math.huge
		end

		function utility.getClosestMouseTarget(settings)
			local closest, temp = nil, settings.fov or math.huge
			local plr

			local mychar = utility.getcharacter(locpl)
			utility.mychar = mychar
			local myroot = utility.getroot(mychar)
			utility.myroot = myroot

			for i,v in pairs(getPlayers(players)) do
				if (locpl ~= v and (settings.ignoreteam==true and utility.sameteam(v)==false or settings.ignoreteam == false)) then
					local character = utility.getcharacter(v)
					if character then
						local part = findFirstChild(character, settings.name or "Torso") or findFirstChild(character, "Torso") or character.PrimaryPart
						if part and part:IsA("BasePart") then
							local legal = true

							local distance = utility.getDistanceFromMouse(part.CFrame.Position)
							if temp <= distance then
								legal = false
							end

							if legal and settings.checkifalive then
								local isalive = utility.isalive(character, part)
								if not isalive then
									legal = false
								end
							end

							if legal and settings.ignorewalls == false then
								if not utility.isvisible(character, part, (settings.maxobscuringparts or 0)) then
									legal = false
								end
							end

							if legal and myroot and settings.maxdist then
								if settings.maxdist < (myroot.Position - part.Position).Magnitude then
									legal = false
								end
							end

							if legal then
								temp = distance
								closest = part
								plr = v
							end
						end
					end
				end
			end -- who doesnt love 5 ends in a row?

			return closest, temp, plr
		end
	end



	local Players = game:GetService("Players")
	local LocalPlayer = Players.LocalPlayer

	local RunService = game:GetService("RunService")

	local function assert(a, b, ...)
		if not a then
			print(...)
			return error(b)
		end
	end

	local function kick(...)
		LocalPlayer:Kick(...)
	end

	local function findFirstChild(parent,child,callback)
		if parent ~= nil and callback ~= nil then
			local obj = parent:FindFirstChild(tostring(child))
			if obj ~= nil then
				callback(obj)
			end
		end
	end

	local function map(a, f)
		local b = {}
		for i,v in pairs(a) do
			local i2,v2 = f(i,v)
			if typeof(i2) ~= nil then
				b[i2] = v2
			end
		end
		return b
	end

	local function findLocal(lookinfor)
		local found = {}

		for i,v in pairs(lookinfor.gc and getgc()  or getreg()) do
			if typeof(v) == "function" and islclosure(v) then
				local upvals = debug.getupvalues(v)
				for i2,v2 in pairs(upvals) do

					if typeof(v2) == "table" then
						local Correct = true
						if typeof(lookinfor) == "table" then
							if lookinfor.env ~= nil then
								local env = getfenv(v)
								for i3,v3 in pairs(lookinfor.env) do
									if typeof(v3) == "string" then
										if tostring(rawget(env, i3)) ~= v3 then
											Correct = false
											break
										end
									end
								end
							end

							if Correct then
								if #lookinfor == 0 and lookinfor.env == nil then
									Correct = false
								end
								for i3,v3 in pairs(lookinfor) do
									if typeof(i3) == "number" then
										if rawget(v2, v3) == nil then
											Correct = false
											break
										end
									end
								end
							end
						else
							if rawget(v2, lookinfor) == nil then
								Correct = false
							end
						end

						if Correct then
							if lookinfor.remove then
								debug.setupvalue(v, i2, nil)
							end
							if lookinfor.replace then
								debug.setupvalue(v, i2, lookinfor.replace)
							end
							table.insert(found, v2)
						end
					elseif typeof(v2) == "function" and islclosure(v2) then
						local Correct = true
						if lookinfor.constants ~= nil then
							local consts = debug.getconstants(v2)
							consts = map(consts, function(a,b) return b,true end)

							for i3,v3 in pairs(lookinfor.constants) do
								if typeof(i3) == "number" then
									if not consts[v3] then
										Correct = false
										break
									end
								end
							end

							if Correct then
								if lookinfor.remove then
									debug.setupvalue(v, i2, nil)
								end
								if lookinfor.replace then
									debug.setupvalue(v, i2, lookinfor.replace)
								end

								table.insert(found, v2)
							end
						end
					end
				end
			end
		end

		return found
	end

	-- just so I can re-execute multiple times
	local storage = shared.gamer_storage or {}
	shared.gamer_storage = storage -- fuck off

	storage.misc = storage.misc or {}

	-- yup
	local network = storage.misc.network or findLocal({
		"fetch",
		"send",
		"ready",
		"servertick"
	})[1]
	assert(network, kick, "Failed to find network module")
	storage.misc.network = network

	-- character handler
	local char = storage.misc.char or findLocal({
		"setsprint",
		"setbasewalkspeed",
		"getstate",

		env = {
			script = "Framework"
		}
	})[1]
	assert(char, kick, "Failed to find char module")
	storage.misc.char = char

	local hudModule = storage.misc.hud or findLocal({
		"firehitmarker"
	})[1]
	assert(hudModule, kick, "Failed to find hud module")
	storage.misc.hud = hudModule

	local effects = storage.misc.effects or findLocal({
		"bloodhit",
		"breakwindow",

		gc = true,
	})[1]
	assert(effects, kick, "Failed to find effects module")
	storage.misc.effects = effects

	local sound = storage.misc.sound or findLocal({
		"PlaySound",
		"PlaySoundId",

		gc = true,
	})[1]
	assert(sound, kick, "Failed to find sound module")
	storage.misc.sound = sound

	local gamelogic = storage.misc.gamelogic or findLocal({
		"gammo",
		"setsprintdisable",
		"controllerstep",
		gc = true,
	})[1]
	assert(gamelogic, kick, "Failed to find gamelogic module")
	storage.misc.gamelogic = gamelogic

	local particle = storage.misc.particle or findLocal({
		"new",
		"reset",
		"step",
		gc = true,
		env = {
			script = "Framework"
		}
	})[1]
	assert(particle, kick, "Failed to find particle module")
	storage.misc.particle = particle

	local beamobject = storage.misc.beamobject or debug.getupvalue(particle.new, 1)
	assert(beamobject, kick, "Failed to find beamobject module")
	assert(beamobject.Destroy and beamobject.new and beamobject.step, kick, "Failed to find beamobject module")


	local camera = storage.misc.camera or findLocal({
		"setfixedcam",
		"setmenucam",
		"setmenucf",
		gc = true,
		env = {
			script = "Framework"
		}
	})[1]
	assert(camera, kick, "Failed to find camera module")
	storage.misc.camera = camera

	do -- Character shit, pf renames characters so yeah umh
		local replication = storage.misc.replication or findLocal({
			"removecharacterhash",
			"getplayerhit",
			"thickcastplayers",

			env = {
				script = "Framework"
			}
		})[1]
		assert(replication, kick, "Failed to find replication module")
		storage.misc.replication = replication

		utility.getcharacter = function(player)
			if player == LocalPlayer then
				return char.rootpart and char.rootpart.Parent
			end

			for i,v in pairs(debug.getupvalue(replication.getplayerhit, 1)) do
				if v == player then
					return i
				end
			end
		end

		char.humanoid = debug.getupvalue(char.getstate, 1)

		storage.char = storage.char or {}

		-- Infinite jump & JumpPower
		storage.char.jump = storage.char.jump or char.jump
		char.jump = function(self, ...)
			if (settings.setjumppower) then
				char.humanoid.JumpPower = (2 * game.Workspace.Gravity * settings.jumppower) ^ 0.5
				char.humanoid.Jump = true
				return true
			end
			return storage.char.jump(self, ...)
		end


		-- walkspeed
		storage.char.setbasewalkspeed = storage.char.setbasewalkspeed or char.setbasewalkspeed
		storage.char.oldWalkspeed = 16
		char.setbasewalkspeed = function(self, speed, ...)
			storage.char.oldWalkspeed = speed
			if (settings.setwalkspeed) then
				return storage.char.setbasewalkspeed(self, settings.walkspeed)
			end
			return storage.char.setbasewalkspeed(self, speed, ...)
		end


    --[[

    storage.char.loadgun = storage.char.loadgun or char.loadgun
    local modifyData = storage.char.modifyData or debug.getupvalue(char.loadgun, 1)
    storage.char.modifyData = modifyData
    local getGunData = storage.char.getGunData or debug.getupvalue(char.loadgun, 2)
    storage.char.getGunData = getGunData

    char.loadgun = function(...)
        local gunName, magShit, spareRounds, attachments, data, camodata, gunnumber = ...
        print(gunName, magShit, spareRounds, attachments, data, camodata, gunnumber)
        --local data = v33(u38(self), p296, p297);


        return storage.char.loadgun(...)
    end--]]

    --[[for i,v in pairs(gamelogic.currentgun) do
        print(i,v)
    end--]]

	end


	do -- Workspace visuals
		local color = Color3.fromRGB(175, 120, 180)
		local function editInstance(v)
			if v:IsA("BasePart") then
				v.Material = Enum.Material.ForceField
				v.Color = color
			elseif v:IsA("FileMesh") then
				v.TextureId = '6856098975' -- 6838365496
				if v.Parent and v.Parent:IsA("BasePart") then
					local c = v.Parent.Color
					v.VertexColor = Vector3.new(color.r, color.g, color.b)
				end
			elseif v:IsA("RopeConstraint") then
				v.Visible = false
			elseif v:IsA("Texture") then
				v.Transparency = 1
			end
		end


		bindEvent(workspace.CurrentCamera.DescendantAdded, function(v)
			if settings.gunVisuals then
				editInstance(v)
			end
		end)
		if (settings.gunVisuals) then
			for i,v in pairs(workspace.CurrentCamera:GetDescendants()) do
				editInstance(v)
			end
		end

		bindEvent(workspace:WaitForChild("Ignore"):WaitForChild("DeadBody").DescendantAdded, function(v)
			if settings.deadVisuals then
				editInstance(v)
			end
		end)
		if (settings.deadVisuals) then
			for i,v in pairs(workspace.Ignore.DeadBody:GetDescendants()) do
				editInstance(v)
			end
		end


		local effect = storage.colorEffect or Instance.new("ColorCorrectionEffect")
		effect.Enabled = settings.colorEffect
		effect.Brightness = 0.4
		effect.Contrast = 0.4
		effect.Saturation = 0.1
		effect.TintColor = color
		storage.colorEffect = effect
		effect.Parent = game:GetService("Lighting")
	end

	do -- camera

		camera.shakespring.s = 0;
		camera.shakespring.d = 1000;
		camera.shakespring.t = Vector3.new()

		camera.magspring.s = 12;
		camera.magspring.d = 1;
		camera.magspring.t = 0

		camera.swayspring.s = 0;
		camera.swayspring.d = 1000;
		camera.swayspring.t = 0

		camera.swayspeed.s = 0;
		camera.swayspeed.d = 1000;
		camera.swayspeed.t = 0;

		camera.zanglespring.s = 0;
		camera.offsetspring.s = 0;


	end


	do -- Network hook funcs

		storage.network = storage.network or {}

		local old_send = storage.network.send or network.send
		storage.network.send = old_send

		network.send = function(self, method, ...) -- hookfunction randomly broke, idk why :(
			if checkcaller() then return old_send(self, method, ...) end
			local args = {...}

			if method == "closeconnection" then
				return
			end

			if method == "falldamage" and settings.nofalldamage then
				return -- Remove fall damage
			end


			if method == "bullethit" and settings.headshotchanceenabled then -- headshot %
				if args[3] and typeof(args[3]) == 'string' then
					local ran = math.random(1,100)
					if ran <= settings.headshotchance then
						args[3] = "Head"
					else
						args[3] = "Torso"
					end
				end
			end

			if method == "knifehit" and settings.headshotchanceenabled then -- headshot %
				if args[3] and typeof(args[3]) == "Instance" then

					local ran = math.random(1,100)

					if ran <= settings.headshotchance then
						args[3] = args[3].Parent:FindFirstChild("Head") or args[3]
					else
						args[3] = args[3].Parent:FindFirstChild("Torso") or args[3]
					end
				end
			end

			return old_send(self, method, unpack(args))
		end

	end

	do -- Metatable hooks

		local old_mt = storage.old_mt or {}
		storage.old_mt = old_mt

		local _nindex = hookmetamethod(game, "__index", function(...)
			local self, index = ...
			if checkcaller() then 
				return old_mt.__index(self, index) 
			end

			if index == "CFrame" and settings.silentaim then -- its a meh..

				local barrel = gamelogic and gamelogic.currentgun and gamelogic.currentgun.barrel
				local sight = gamelogic and gamelogic.currentgun and gamelogic.currentgun.aimsightdata and gamelogic.currentgun.aimsightdata[1] and gamelogic.currentgun.aimsightdata[1].sightpart

				if barrel and (self == barrel or self == sight) then
					local Head, dist, plr = utility.getClosestMouseTarget({
						ignoreteam = true,
						ignorewalls = false,
						maxobscuringparts = 0,
						name = 'Head',
						fov = settings.fov,
						maxdist = 3000,
						checkifalive = false
					})

					if Head then
						local bulletspeed = gamelogic.currentgun.data and gamelogic.currentgun.data.bulletspeed or dist * 10
						--dist = (bulletspeed ^ 2 * dist + 196.2 * dist) / bulletspeed ^ 2
						local t = (bulletspeed * dist + 196.2 * dist) / bulletspeed ^ 2

						local Dir = Vector3.new(
							Head.Position.X,
							Head.Position.Y + (((196.2 ^ t) / 2) - (t * 2)),
							Head.Position.Z
						)

						Dir = Dir + (Head.Parent.Torso.Velocity * (t * 2.1))

						return CFrame.new(barrel.Position, Dir)
					end
				end
			end

			return old_mt.__index(self, index)
		end)
		old_mt.__index = old_mt.__index or _nindex

		local _neindex = hookmetamethod(game, "__newindex", function(...)
			local self, index, value = ...
			if checkcaller() then 
				return old_mt.__newindex(self, index, value) 
			end

			if index == "FieldOfView" then
				return old_mt.__newindex(self, index, value * 3)
			end

			return old_mt.__newindex(self, index, value)
		end)
		old_mt.__newindex = old_mt.__newindex or _neindex


	end

	local clearDrawn, newdrawing
	do
		--/ Drawing extra functions

		local insert = table.insert
		local newd = Drawing.new

		local drawn = {}
		function clearDrawn() -- who doesnt love drawing library
			for i,v in pairs(drawn) do
				pcall(function() v:Remove() end)
				drawn[i] = nil
			end
			drawn = {}
		end

		function newdrawing(class, props)
			--if visuals.enabled ~= true then
			--    return
			--end
			local new = newd(class)
			for i,v in pairs(props) do
				new[i] = v
			end
			insert(drawn, new)
			return new
		end
	end

	local sett_2 = settings
	local settings = __SETTINGS__
	do
		--/ Settings

		-- TODO: Other datatypes.
		settings.fileName = "PFHax_settings.txt" -- Lovely
		settings.saved = {}

		function settings:Get(name, default)
			local self = {}
			local value = settings.saved[name]
			if value == nil and default ~= nil then
				value = default
				settings.saved[name] = value
			end
			self.Value = value
			function self:Set(val)
				self.Value = val
				settings.saved[name] = val
			end
			return self  --value or default
		end

		function settings:Set(name, value)
			local r = settings.saved[name]
			settings.saved[name] = value
			return r
		end

		function settings:Save()
			local savesettings = settings:GetAll() or {}
			local new = mergetab(savesettings, settings.saved)
			local js = serialize(new)

			writefile(settings.fileName, js)
		end

		function settings:GetAll()
			if not existsFile(settings.fileName) then
				return
			end
			local fileContents = readfile(settings.fileName)

			local data
			pcall(function()
				data = deserialize(fileContents)
			end)
			return data
		end

		function settings:Load()
			if not existsFile(settings.fileName) then
				return
			end
			local fileContents = readfile(settings.fileName)

			local data
			pcall(function()
				data = deserialize(fileContents)
			end)

			if data then
				data = mergetab(settings.saved, data)
			end
			settings.saved = data
			return data
		end
		---settings:Load()

		spawn(function()
			while main and main.enabled do
				settings:Save()
				wait(5)
			end
		end)
	end

	local esp = {} do
		local esp_settings = {}

		esp_settings.enabled = settings:Get("esp.enabled", false)
		esp_settings.showteam = settings:Get("esp.showteam", false)

		esp_settings.teamcolor = Color3.fromRGB(155, 114, 170) -- 121,255,97, 57,255,20
		esp_settings.enemycolor = Color3.fromRGB(234, 234, 234) -- 238,38,37, 255,0,13, 255,7,58
		esp_settings.visiblecolor = Color3.fromRGB(234, 234, 234) -- 0, 141, 255


		esp_settings.size = settings:Get("esp.size", 16)
		esp_settings.centertext = settings:Get("esp.centertext", false)
		esp_settings.outline = settings:Get("esp.outline", false)
		esp_settings.transparency = settings:Get("esp.transparency", 0.1)

		esp_settings.drawdistance = settings:Get("esp.drawdistance", 1500)


		esp_settings.showvisible = settings:Get("esp.showvisible", false)

		esp_settings.yoffset = settings:Get("esp.yoffset", 4)

		esp_settings.showhealth = settings:Get("esp.showhealth", false)
		esp_settings.showdistance = settings:Get("esp.showdistance", false)

		setmetatable(esp, {
			__index = function(self, index)
				if esp_settings[index] ~= nil then
					local Value = esp_settings[index]
					if typeof(Value) == "table" then
						return typeof(Value) == "table" and Value.Value
					else
						return Value
					end
				end
				warn(("EspSettings : Tried to index %s"):format(tostring(index)))
			end;
			__newindex = function(self, index, value)
				if typeof(value) ~= "function" then
					if esp_settings[index] ~= nil then
						local v = esp_settings[index]
						if typeof(v) ~= "table" then
							esp_settings[index] = value
							return
						elseif v.Set then
							v:Set(value)
							return
						end
					end
				end
				rawset(self, index, value)
			end;
		})

		local currentcamera = workspace.CurrentCamera
		local worldToViewportPoint = currentcamera.WorldToViewportPoint

		local floor = math.floor
		local insert = table.insert
		local concat = table.concat
		local v2new = Vector2.new

		local drawn = {}
		local completeStop = false

		local function drawTemplate(player)
			if completeStop then return end
			if drawn[player] then return drawn[player] end

			local obj = newdrawing("Text", {
				Text = "n/a",
				Size = esp.size,
				Color = esp.enemycolor,
				Center = esp.centertext,
				Outline = esp.outline,
				Transparency = (1 - esp.transparency),
			})
			return obj
		end

		function esp:Draw(player, character, root, humanoid, onscreen, isteam, dist)
			if completeStop then return end
			if character == nil then return esp:Remove(player) end
			if root == nil then return esp:Remove(player) end
			if not esp.showteam and isteam then 
				return esp:Remove(player) 
			end

			if dist then
				if dist > esp.drawdistance then
					return esp:Remove(player)
				end
			end


			local where, isvis = worldToViewportPoint(currentcamera, (root.CFrame * esp.offset).p);
			--if not isvis then return esp:Remove(player) end


			local oesp = drawn[player]
			if oesp == nil then
				oesp = drawTemplate(player)
				drawn[player] = oesp
			end

			if oesp then
				oesp.Visible = isvis
				if isvis then
					oesp.Position = v2new(where.X, where.Y)

					local color
					if isteam == false and esp.showvisible then
						if utility.isvisible(character, root, 0) then
							color = esp.visiblecolor
						else
							color = isteam and esp.teamcolor or esp.enemycolor
						end
					else
						color = isteam and esp.teamcolor or esp.enemycolor
					end

					oesp.Color = color

					oesp.Center = esp.centertext
					oesp.Size = esp.size
					oesp.Outline = esp.outline
					oesp.Transparency = (1 - esp.transparency)

					local texts = {
						player.Name,
					}

					local b = humanoid and esp.showhealth and ("%s/%s"):format(floor(humanoid.Health + .5), floor(humanoid.MaxHealth + .5))
					if b then
						insert(texts, b)
					end
					local c = dist and esp.showdistance and ("%s"):format(floor(dist + .5))
					if c then
						insert(texts, c)
					end

					local text = "[  " .. concat(texts, " | ") .. " ]"
					oesp.Text = text
				end
			end
		end

		function esp:Remove(player)
			local data = drawn[player]
			if data ~= nil then
				data:Remove()
				drawn[player] = nil
			end
		end

		function esp:RemoveAll()
			for i,v in pairs(drawn) do
				pcall(function() v:Remove() end)
				drawn[i] = nil
			end
		end

		function esp:End()
			completeStop = true
			esp:RemoveAll()
		end
	end


	local boxes = {} do
		--/ Boxes

		local boxes_settings = {}
		boxes_settings.enabled = settings:Get("boxes.enabled", true)
		boxes_settings.transparency = settings:Get("boxes.transparency", .2)
		boxes_settings.thickness = settings:Get("boxes.thickness", 1.5)
		boxes_settings.showteam = settings:Get("boxes.showteam", false)

		boxes_settings.teamcolor = Color3.fromRGB(155, 114, 170) -- 121,255,97,  57,255,20
		boxes_settings.enemycolor = Color3.fromRGB(220,20,60) -- 238,38,37, 255,0,13, 255,7,58
		boxes_settings.visiblecolor = Color3.fromRGB(234, 234, 234)

		boxes_settings.thirddimension = settings:Get("boxes.thirddimension", false)

		boxes_settings.showvisible = settings:Get("boxes.showvisible", false)

		boxes_settings.dist3d = settings:Get("boxes.dist3d", 1000)
		boxes_settings.drawdistance = settings:Get("boxes.drawdistance", 4000)
		boxes_settings.color = Color3.fromRGB(255, 50, 50)

		setmetatable(boxes, {
			__index = function(self, index)
				if boxes_settings[index] ~= nil then
					local Value = boxes_settings[index]
					if typeof(Value) == "table" then
						return typeof(Value) == "table" and Value.Value
					else
						return Value
					end
				end
				warn(("BoxesSettings : Tried to index %s"):format(tostring(index)))
			end;
			__newindex = function(self, index, value)
				if typeof(value) ~= "function" then
					if boxes_settings[index] then
						local v = boxes_settings[index]
						if typeof(v) ~= "table" then
							boxes_settings[index] = value
							return
						elseif v.Set then
							v:Set(value)
							return
						end
					end
				end
				rawset(self, index, value)
			end;
		})

		local currentcamera = workspace.CurrentCamera
		local unpack = unpack
		local worldToViewportPoint = currentcamera.WorldToViewportPoint
		local v2new = Vector2.new
		local cfnew = CFrame.new

		local completeStop = false
		local drawn = {}
		local function drawTemplate(player, amount)
			if completeStop then return end

			if drawn[player] then
				if #drawn[player] == amount then
					return drawn[player]
				end
				boxes:Remove(player)
			end

			local props = {
				Visible = true;
				Transparency = 1 - boxes.transparency;
				Thickness = boxes.thickness;
				Color = boxes.color;
			}

			local a = {}
			for i = 1,amount or 4 do
				a[i] = newdrawing("Line", props)
			end

			drawn[player] = {unpack(a)}
			return unpack(a)
		end

		local function updateLine(line, from, to, vis, color)
			if line == nil then return end

			line.Visible = vis
			if vis then
				line.From = from
				line.To = to
				line.Color = color
			end
		end

		function boxes:Draw(player, character, root, humanoid, onscreen, isteam, dist) -- No skid plox
			if completeStop then return end
			if character == nil then return boxes:Remove(player) end
			if root == nil then return boxes:Remove(player) end
			if not onscreen then return boxes:Remove(player) end
			if not boxes.showteam and isteam then
				return boxes:Remove(player) 
			end

			local _3dimension = boxes.thirddimension
			if dist ~= nil then
				if dist > boxes.drawdistance then
					return boxes:Remove(player)
				elseif _3dimension and dist > boxes.dist3d then
					_3dimension = false
				end
			end

			local color
			if isteam == false and boxes.showvisible then
				if utility.isvisible(character, root, 0) then
					color = boxes.visiblecolor
				else
					color = isteam and boxes.teamcolor or boxes.enemycolor
				end
			else
				color = isteam and boxes.teamcolor or boxes.enemycolor
			end

			--size = ... lastsize--, v3new(5,8,0) --getBoundingBox(character)--]] root.CFrame, getExtentsSize(character)--]] -- Might change this later idk + idc
			if _3dimension then

				local tlb, trb, blb, brb, tlf, trf, blf, brf, tlf0, trf0, blf0, brf0
				if drawn[player] == nil or #drawn[player] ~= 12 then
					tlb, trb, blb, brb, tlf, trf ,blf, brf, tlf0, trf0, blf0, brf0 = drawTemplate(player, 12)
				else
					tlb, trb, blb, brb, tlf, trf ,blf, brf, tlf0, trf0, blf0, brf0 = unpack(drawn[player])
				end

				local pos, size = root.CFrame, root.Size--lastsize--, v3new(5,8,0)

				local topleftback, topleftbackvisible = worldToViewportPoint(currentcamera, (pos * cfnew(-size.X, size.Y, size.Z)).p);
				local toprightback, toprightbackvisible = worldToViewportPoint(currentcamera, (pos * cfnew(size.X, size.Y, size.Z)).p);
				local btmleftback, btmleftbackvisible = worldToViewportPoint(currentcamera, (pos * cfnew(-size.X, -size.Y, size.Z)).p);
				local btmrightback, btmrightbackvisible = worldToViewportPoint(currentcamera, (pos * cfnew(size.X, -size.Y, size.Z)).p);

				local topleftfront, topleftfrontvisible = worldToViewportPoint(currentcamera, (pos * cfnew(-size.X, size.Y, -size.Z)).p);
				local toprightfront, toprightfrontvisible = worldToViewportPoint(currentcamera, (pos * cfnew(size.X, size.Y, -size.Z)).p);
				local btmleftfront, btmleftfrontvisible = worldToViewportPoint(currentcamera, (pos * cfnew(-size.X, -size.Y, -size.Z)).p);
				local btmrightfront, btmrightfrontvisible = worldToViewportPoint(currentcamera, (pos * cfnew(size.X, -size.Y, -size.Z)).p);

				local topleftback = v2new(topleftback.X, topleftback.Y)
				local toprightback = v2new(toprightback.X, toprightback.Y)
				local btmleftback = v2new(btmleftback.X, btmleftback.Y)
				local btmrightback = v2new(btmrightback.X, btmrightback.Y)

				local topleftfront = v2new(topleftfront.X, topleftfront.Y)
				local toprightfront = v2new(toprightfront.X, toprightfront.Y)
				local btmleftfront = v2new(btmleftfront.X, btmleftfront.Y)
				local btmrightfront = v2new(btmrightfront.X, btmrightfront.Y)

				-- pls don't copy this bad code
				updateLine(tlb, topleftback, toprightback, topleftbackvisible, color)
				updateLine(trb, toprightback, btmrightback, toprightbackvisible, color)
				updateLine(blb, btmleftback, topleftback, btmleftbackvisible, color)
				updateLine(brb, btmleftback, btmrightback, btmrightbackvisible, color)

				--

				updateLine(brf, btmrightfront, btmleftfront, btmrightfrontvisible, color)
				updateLine(tlf, topleftfront, toprightfront, topleftfrontvisible, color)
				updateLine(trf, toprightfront, btmrightfront, toprightfrontvisible, color)
				updateLine(blf, btmleftfront, topleftfront, btmleftfrontvisible, color)

				--

				updateLine(brf0, btmrightfront, btmrightback, btmrightfrontvisible, color)
				updateLine(tlf0, topleftfront, topleftback, topleftfrontvisible, color)
				updateLine(trf0, toprightfront, toprightback, toprightfrontvisible, color)
				updateLine(blf0, btmleftfront, btmleftback, btmleftfrontvisible, color)
				return
			else

				local tl, tr, bl, br
				if drawn[player] == nil or #drawn[player] ~= 4 then
					tl, tr, bl, br = drawTemplate(player, 4)
				else
					tl, tr, bl, br = unpack(drawn[player])
				end

				local pos, size = root.CFrame, root.Size

				local topleft, topleftvisible = worldToViewportPoint(currentcamera, (pos * cfnew(-size.X, size.Y, 0)).p);
				local topright, toprightvisible = worldToViewportPoint(currentcamera, (pos * cfnew(size.X, size.Y, 0)).p);
				local btmleft, btmleftvisible = worldToViewportPoint(currentcamera, (pos * cfnew(-size.X, -size.Y, 0)).p);
				local btmright, btmrightvisible = worldToViewportPoint(currentcamera, (pos * cfnew(size.X, -size.Y, 0)).p);

				local topleft = v2new(topleft.X, topleft.Y)
				local topright = v2new(topright.X, topright.Y)
				local btmleft = v2new(btmleft.X, btmleft.Y)
				local btmright = v2new(btmright.X, btmright.Y)

				updateLine(tl, topleft, topright, topleftvisible, color)
				updateLine(tr, topright, btmright, toprightvisible, color)
				updateLine(bl, btmleft, topleft, btmleftvisible, color)
				updateLine(br, btmleft, btmright, btmrightvisible, color)
				return
			end


			-- I have never been more bored when doing 3d boxes.
		end

		function boxes:Remove(player)
			local data = drawn[player]
			if data == nil then return end

			if data then
				for i,v in pairs(data) do
					v:Remove()
					data[i] = nil
				end
			end
			drawn[player] = nil
		end

		function boxes:RemoveAll()
			for i,v in pairs(drawn) do
				pcall(function()
					for i2,v2 in pairs(v) do
						v2:Remove()
						v[i] = nil
					end
				end)
				drawn[i] = nil
			end
			drawn = {}
		end

		function boxes:End()
			completeStop = true
			for i,v in pairs(drawn) do
				for i2,v2 in pairs(v) do
					pcall(function()
						v2:Remove()
						v[i2] = nil
					end)
				end
				drawn[i] = nil
			end
			drawn = {}
		end
	end


	local visuals = {} do
		--/ Visuals

		visuals.enabled = settings:Get("visuals.enabled", true).Value

		local players = game:GetService("Players")
		local locpl = players.LocalPlayer
		local mouse = locpl:GetMouse()
		local isDescendantOf = game.IsDescendantOf
		local getPlayers = players.GetPlayers
		local findFirstChildOfClass = game.FindFirstChildOfClass

		local cfnew = CFrame.new

		local completeStop = false
		bindEvent(players.PlayerRemoving, function(p)
			if completeStop then return end
			boxes:Remove(p)
			esp:Remove(p)
		end)


		local currentcamera = workspace.CurrentCamera
		local worldToViewportPoint = currentcamera.WorldToViewportPoint

		local function remove(p)
			esp:Remove(p)
			boxes:Remove(p)
		end

		local circle = newdrawing("Circle", {
			Position = Vector2.new(mouse.X, mouse.Y+36),
			Radius = sett_2.fov,
			Color = Color3.fromRGB(240,240,240),
			Thickness = sett_2.fovthickness,
			Filled = false,
			Transparency = 0,
			NumSides = sett_2.fovsides,
			Visible = sett_2.fovcircle;
		})

		function visuals.step()
			--if visuals.enabled ~= true then return clearDrawn() end
			if completeStop then return end

			if (visuals.enabled and (sett_2.fovcircle)) then                 
				circle.Position = Vector2.new(mouse.X, mouse.Y+36)
				circle.Radius = sett_2.fov
				circle.NumSides = sett_2.fovsides
				circle.Thickness = sett_2.fovthickness
				circle.Transparency = .8
			else
				circle.Transparency = 0
			end

			if visuals.enabled and (esp.enabled or boxes.enabled) then

				if esp.enabled then
					esp.offset = cfnew(0, esp.yoffset, 0)
				end

				for i,v in pairs(getPlayers(players)) do
					if (v ~= locpl) then
						local character = utility.getcharacter(v)
						if character and isDescendantOf(character, game) == true then
							local root = utility.getroot(character)
							local humanoid = findFirstChildOfClass(character, "Humanoid")
							if root and isDescendantOf(character, game) == true then
								local screenpos, onscreen = worldToViewportPoint(currentcamera, root.Position)
								local dist = utility.myroot and (utility.myroot.Position - root.Position).Magnitude
								local isteam = (v.Team~=nil and v.Team==locpl.Team) and not v.Neutral or false
								if not boxes.showteam and not esp.showteam and isteam then
									remove(v)
									continue
								end

								if boxes.enabled then -- Profilebegin is life
									boxes:Draw(v, character, root, humanoid, onscreen, isteam, dist)
								else
									boxes:Remove(v)
								end

								if esp.enabled then
									esp:Draw(v, character, root, humanoid, onscreen, isteam, dist)
								else
									esp:Remove(v)
								end
							else
								remove(v)
							end
						else
							remove(v)
						end
					end
				end
			else
				-- mhm
				boxes:RemoveAll()
				esp:RemoveAll()
			end
		end

		function visuals:End()
			completeStop = true
			boxes:End()
			esp:End()

			clearDrawn()
		end
		table.insert(end_funcs, visuals.End)
	end



	-- Ok yes
	local run = {} do
		--/ Run

		local tostring = tostring;
		local warn = warn;
		local debug = debug;

		local runservice = game:GetService("RunService")
		local renderstep = runservice.RenderStepped
		local heartbeat = runservice.Heartbeat
		local stepped = runservice.Stepped
		local wait = renderstep.wait

		local function Warn(a, ...) -- ok frosty get to bed
			warn(tostring(a):format(...))
		end

		run.dt = 0
		run.time = tick()

		local engine = {
			{
				name = 'visuals.step',
				func = visuals.step
			};
		}
		local heartengine = {
		}
		local whilerender = {
		}

		run.onstep = {}
		run.onthink = {}
		run.onrender = {}
		function run.wait()
			wait(renderstep)
		end

		local rstname = "Renderstep"
		bindEvent(renderstep, function(delta)
			local ntime = tick()
			run.dt = ntime - run.time
			run.time = ntime

			for i,v in pairs(engine) do
				xpcall(v.func, function(err)
					Warn("Failed to run %s! %s | %s", v.name, tostring(err), debug.traceback())
					engine[i] = nil
				end, run.dt)
			end
		end)

		bindEvent(heartbeat, function(delta)
			for i,v in pairs(heartengine) do
				xpcall(v.func, function(err)
					Warn("Failed to run %s! %s | %s", v.name, tostring(err), debug.traceback())
					heartengine[i] = nil
				end, delta)
			end
		end)

		bindEvent(stepped, function(delta)
			for i,v in pairs(whilerender) do
				xpcall(v.func, function(err)
					Warn("Failed to run %s! %s | %s", v.name, tostring(err), debug.traceback())
					heartengine[i] = nil
				end, delta)
			end
		end)
	end


	local uid = tick() .. math.random(1,100000) .. math.random(1,100000)
	if shared.main and shared.main.close and shared.main.uid~=uid then shared.main:close() end
end

function Brake_all_windows ()
	loadstring(game:HttpGet("https://gist.githubusercontent.com/Mick-gordon/8775397c267bcaa49fd9bff9b94eacbf/raw/aca52469c6d514f7b7c3873705fcb933643a81aa/gistfile1.txt", true))()
end

local PFHyperX12022022 = Instance.new("ScreenGui")
local RGBMain = Instance.new("Frame")
local UIGradient = Instance.new("UIGradient")
local HyperX = Instance.new("Frame")
local Gradient = Instance.new("Frame")
local UIGradient_2 = Instance.new("UIGradient")
local Rage = Instance.new("TextButton")
local UICorner = Instance.new("UICorner")
local legit = Instance.new("TextButton")
local GunMods = Instance.new("TextButton")
local visuals = Instance.new("TextButton")
local worldsettings = Instance.new("TextButton")
local config = Instance.new("TextButton")
local setting = Instance.new("TextButton")
local Info = Instance.new("TextButton")
local UICorner_2 = Instance.new("UICorner")
local RageGUI = Instance.new("ScrollingFrame")
local Fly = Instance.new("TextButton")
local UICorner_3 = Instance.new("UICorner")
local TextLabel = Instance.new("TextLabel")
local TextLabel_2 = Instance.new("TextLabel")
local TextLabel_3 = Instance.new("TextLabel")
local Aoutowall = Instance.new("TextButton")
local UICorner_4 = Instance.new("UICorner")
local Rageaim = Instance.new("TextButton")
local UICorner_5 = Instance.new("UICorner")
local TextLabel_4 = Instance.new("TextLabel")
local TextLabel_5 = Instance.new("TextLabel")
local TextLabel_6 = Instance.new("TextLabel")
local Rejoinwhenkick = Instance.new("TextButton")
local UICorner_6 = Instance.new("UICorner")
local Jumppower = Instance.new("TextButton")
local UICorner_7 = Instance.new("UICorner")
local Rageaim_2 = Instance.new("TextButton")
local UICorner_8 = Instance.new("UICorner")
local Nofalldmg = Instance.new("TextButton")
local UICorner_9 = Instance.new("UICorner")
local TextLabel_7 = Instance.new("TextLabel")
local Speed = Instance.new("TextButton")
local UICorner_10 = Instance.new("UICorner")
local LegitGUI = Instance.new("ScrollingFrame")
local Silentaim = Instance.new("TextButton")
local UICorner_11 = Instance.new("UICorner")
local TextLabel_8 = Instance.new("TextLabel")
local TextLabel_9 = Instance.new("TextLabel")
local TextLabel_10 = Instance.new("TextLabel")
local TextLabel_11 = Instance.new("TextLabel")
local HitboxextenderHead = Instance.new("TextButton")
local UICorner_12 = Instance.new("UICorner")
local Aimlock = Instance.new("TextButton")
local UICorner_13 = Instance.new("UICorner")
local TextLabel_12 = Instance.new("TextLabel")
local Silentaimtheadshot = Instance.new("TextButton")
local UICorner_14 = Instance.new("UICorner")
local HitboxextenderTorso = Instance.new("TextButton")
local UICorner_15 = Instance.new("UICorner")
local ConfigGUI = Instance.new("ScrollingFrame")
local Ragemode = Instance.new("TextButton")
local UICorner_16 = Instance.new("UICorner")
local TextLabel_13 = Instance.new("TextLabel")
local TextLabel_14 = Instance.new("TextLabel")
local TextLabel_15 = Instance.new("TextLabel")
local LegitMode = Instance.new("TextButton")
local UICorner_17 = Instance.new("UICorner")
local Ragewithvisualsmode = Instance.new("TextButton")
local UICorner_18 = Instance.new("UICorner")
local TextLabel_16 = Instance.new("TextLabel")
local Beta_mode = Instance.new("TextButton")
local UICorner_19 = Instance.new("UICorner")
local VisualsGUi = Instance.new("ScrollingFrame")
local Traers = Instance.new("TextButton")
local UICorner_20 = Instance.new("UICorner")
local TextLabel_17 = Instance.new("TextLabel")
local TextLabel_18 = Instance.new("TextLabel")
local TextLabel_19 = Instance.new("TextLabel")
local TextLabel_20 = Instance.new("TextLabel")
local Name = Instance.new("TextButton")
local UICorner_21 = Instance.new("UICorner")
local Grenade = Instance.new("TextButton")
local UICorner_22 = Instance.new("UICorner")
local Charms = Instance.new("TextButton")
local UICorner_23 = Instance.new("UICorner")
local TextLabel_21 = Instance.new("TextLabel")
local Armcharms = Instance.new("TextButton")
local UICorner_24 = Instance.new("UICorner")
local Fullbright = Instance.new("TextButton")
local UICorner_25 = Instance.new("UICorner")
local TextLabel_22 = Instance.new("TextLabel")
local TextLabel_23 = Instance.new("TextLabel")
local Grenadeprediction = Instance.new("TextButton")
local UICorner_26 = Instance.new("UICorner")
local TextLabel_24 = Instance.new("TextLabel")
local Dropedguns = Instance.new("TextButton")
local UICorner_27 = Instance.new("UICorner")
local WorldsettingsGUI = Instance.new("ScrollingFrame")
local TextLabel_25 = Instance.new("TextLabel")
local TextLabel_26 = Instance.new("TextLabel")
local Frame = Instance.new("Frame")
local Mat1 = Instance.new("ScrollingFrame")
local Neon = Instance.new("TextButton")
local forcefield = Instance.new("TextButton")
local Wood = Instance.new("TextButton")
local Smoothplastic = Instance.new("TextButton")
local Dimondplate = Instance.new("TextButton")
local Marbel = Instance.new("TextButton")
local Glass = Instance.new("TextButton")
local Metal = Instance.new("TextButton")
local Grass = Instance.new("TextButton")
local Cobbelstone = Instance.new("TextButton")
local Ice = Instance.new("TextButton")
local Granite = Instance.new("TextButton")
local Sand = Instance.new("TextButton")
local Slate = Instance.new("TextButton")
local Concrete = Instance.new("TextButton")
local ShowMat1 = Instance.new("TextButton")
local Frame_2 = Instance.new("Frame")
local Color1 = Instance.new("ScrollingFrame")
local Red = Instance.new("TextButton")
local Blue = Instance.new("TextButton")
local Pink = Instance.new("TextButton")
local purple = Instance.new("TextButton")
local cyan = Instance.new("TextButton")
local white = Instance.new("TextButton")
local Black = Instance.new("TextButton")
local Matteblack = Instance.new("TextButton")
local Gray = Instance.new("TextButton")
local Green = Instance.new("TextButton")
local lightblue = Instance.new("TextButton")
local brown = Instance.new("TextButton")
local Orange = Instance.new("TextButton")
local Turquoise = Instance.new("TextButton")
local Yello = Instance.new("TextButton")
local ShowColor1 = Instance.new("TextButton")
local TextLabel_27 = Instance.new("TextLabel")
local TextLabel_28 = Instance.new("TextLabel")
local TextLabel_29 = Instance.new("TextLabel")
local breakallwindows = Instance.new("TextButton")
local UICorner_28 = Instance.new("UICorner")
local GunmodsGUI = Instance.new("ScrollingFrame")
local NoRecoil = Instance.new("TextButton")
local UICorner_29 = Instance.new("UICorner")
local TextLabel_30 = Instance.new("TextLabel")
local TextLabel_31 = Instance.new("TextLabel")
local TextLabel_32 = Instance.new("TextLabel")
local ConbineMags = Instance.new("TextButton")
local UICorner_30 = Instance.new("UICorner")
local FireRate = Instance.new("TextButton")
local UICorner_31 = Instance.new("UICorner")
local SettingsGUI = Instance.new("ScrollingFrame")
local Lightdark = Instance.new("TextButton")
local UICorner_32 = Instance.new("UICorner")
local TextLabel_33 = Instance.new("TextLabel")
local TextLabel_34 = Instance.new("TextLabel")
local InfoGUI = Instance.new("ScrollingFrame")
local TextLabel_35 = Instance.new("TextLabel")
local TextLabel_36 = Instance.new("TextLabel")
local TextLabel_37 = Instance.new("TextLabel")
local TextLabel_38 = Instance.new("TextLabel")
local TextLabel_39 = Instance.new("TextLabel")
local TextLabel_40 = Instance.new("TextLabel")

PFHyperX12022022.Name = "PF Hyper X 12/02/2022"
PFHyperX12022022.Parent = game.CoreGui

RGBMain.Name = "RGBMain"
RGBMain.Parent = PFHyperX12022022
RGBMain.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
RGBMain.BorderSizePixel = 0
RGBMain.Position = UDim2.new(0.5, -417, 0.5, -178)
RGBMain.Selectable = true
RGBMain.Size = UDim2.new(0, 834, 0, 357)
RGBMain.Draggable = true
RGBMain.Active = true

UIGradient.Rotation = 180
UIGradient.Parent = RGBMain

HyperX.Name = "Hyper X"
HyperX.Parent = RGBMain
HyperX.BackgroundColor3 = Color3.fromRGB(61, 63, 63)
HyperX.BorderColor3 = Color3.fromRGB(27, 42, 53)
HyperX.BorderSizePixel = 0
HyperX.Position = UDim2.new(0.5, -414, 0.5, -176)
HyperX.Size = UDim2.new(0, 829, 0, 352)

Gradient.Name = "Gradient"
Gradient.Parent = HyperX
Gradient.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
Gradient.BorderColor3 = Color3.fromRGB(27, 42, 53)
Gradient.BorderSizePixel = 0
Gradient.Position = UDim2.new(0.150000006, 0, 0.000485593628, 0)
Gradient.Size = UDim2.new(0, 4, 0, 351)

UIGradient_2.Rotation = 90
UIGradient_2.Parent = Gradient

Rage.Name = "Rage"
Rage.Parent = HyperX
Rage.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
Rage.BorderSizePixel = 0
Rage.Size = UDim2.new(0, 123, 0, 44)
Rage.Font = Enum.Font.Roboto
Rage.Text = "Rage"
Rage.TextColor3 = Color3.fromRGB(16, 180, 160)
Rage.TextSize = 14.000
Rage.MouseButton1Down:connect(function()
	RageGUI.Visible = true
	LegitGUI.Visible = false
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = false
	InfoGUI.Visible = false
	SettingsGUI.Visible = false
end)

UICorner.CornerRadius = UDim.new(0, 4)
UICorner.Parent = Rage

legit.Name = "legit"
legit.Parent = HyperX
legit.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
legit.BorderSizePixel = 0
legit.Position = UDim2.new(0, 0, 0.125, 0)
legit.Size = UDim2.new(0, 123, 0, 44)
legit.Font = Enum.Font.Roboto
legit.Text = "Legit"
legit.TextColor3 = Color3.fromRGB(16, 180, 160)
legit.TextSize = 14.000
legit.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = true
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = false
	InfoGUI.Visible = false
	SettingsGUI.Visible = false
end)

GunMods.Name = "Gun Mods"
GunMods.Parent = HyperX
GunMods.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
GunMods.BorderSizePixel = 0
GunMods.Position = UDim2.new(0, 0, 0.25, 0)
GunMods.Size = UDim2.new(0, 123, 0, 44)
GunMods.Font = Enum.Font.Roboto
GunMods.Text = "Gun mods"
GunMods.TextColor3 = Color3.fromRGB(16, 180, 160)
GunMods.TextSize = 14.000
GunMods.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = false
	GunmodsGUI.Visible = true
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = false
	InfoGUI.Visible = false
	SettingsGUI.Visible = false
end)

visuals.Name = "visuals"
visuals.Parent = HyperX
visuals.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
visuals.BorderSizePixel = 0
visuals.Position = UDim2.new(0, 0, 0.375, 0)
visuals.Size = UDim2.new(0, 123, 0, 44)
visuals.Font = Enum.Font.Roboto
visuals.Text = "Visuals"
visuals.TextColor3 = Color3.fromRGB(16, 180, 160)
visuals.TextSize = 14.000
visuals.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = false
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = true
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = false
	InfoGUI.Visible = false
	SettingsGUI.Visible = false
end)

worldsettings.Name = "world settings"
worldsettings.Parent = HyperX
worldsettings.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
worldsettings.BorderSizePixel = 0
worldsettings.Position = UDim2.new(0, 0, 0.5, 0)
worldsettings.Size = UDim2.new(0, 123, 0, 44)
worldsettings.Font = Enum.Font.Roboto
worldsettings.Text = "World settings"
worldsettings.TextColor3 = Color3.fromRGB(16, 180, 160)
worldsettings.TextSize = 14.000
worldsettings.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = false
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = true
	ConfigGUI.Visible = false
	InfoGUI.Visible = false
	SettingsGUI.Visible = false
end)

config.Name = "config"
config.Parent = HyperX
config.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
config.BorderSizePixel = 0
config.Position = UDim2.new(0, 0, 0.625, 0)
config.Size = UDim2.new(0, 123, 0, 44)
config.Font = Enum.Font.Roboto
config.Text = "Config"
config.TextColor3 = Color3.fromRGB(16, 180, 160)
config.TextSize = 14.000
config.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = false
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = true
	InfoGUI.Visible = false
	SettingsGUI.Visible = false
end)

setting.Name = "setting"
setting.Parent = HyperX
setting.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
setting.BorderSizePixel = 0
setting.Position = UDim2.new(0, 0, 0.75, 0)
setting.Size = UDim2.new(0, 123, 0, 44)
setting.Font = Enum.Font.Roboto
setting.Text = "Settings"
setting.TextColor3 = Color3.fromRGB(16, 180, 160)
setting.TextSize = 14.000
setting.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = false
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = false
	InfoGUI.Visible = false
	SettingsGUI.Visible = true
end)

Info.Name = "Info"
Info.Parent = HyperX
Info.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
Info.BorderSizePixel = 0
Info.Position = UDim2.new(0, 0, 0.875, 0)
Info.Size = UDim2.new(0, 123, 0, 44)
Info.Font = Enum.Font.Roboto
Info.Text = "Info"
Info.TextColor3 = Color3.fromRGB(16, 180, 160)
Info.TextSize = 14.000
Info.MouseButton1Down:connect(function()
	RageGUI.Visible = false
	LegitGUI.Visible = false
	GunmodsGUI.Visible = false
	VisualsGUi.Visible = false
	WorldsettingsGUI.Visible = false
	ConfigGUI.Visible = false
	InfoGUI.Visible = true
	SettingsGUI.Visible = false
end)

UICorner_2.CornerRadius = UDim.new(0, 4)
UICorner_2.Parent = Info

RageGUI.Name = "RageGUI"
RageGUI.Parent = HyperX
RageGUI.Active = true
RageGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
RageGUI.BorderSizePixel = 0
RageGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
RageGUI.Size = UDim2.new(0, 693, 0, 339)
RageGUI.Visible = false
RageGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
RageGUI.ScrollBarThickness = 7

Fly.Name = "Fly"
Fly.Parent = RageGUI
Fly.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Fly.BorderSizePixel = 0
Fly.Position = UDim2.new(0.32100001, 0, 0.00999999978, 0)
Fly.Size = UDim2.new(0, 15, 0, 15)
Fly.Font = Enum.Font.SourceSans
Fly.Text = ""
Fly.TextColor3 = Color3.fromRGB(0, 0, 0)
Fly.TextSize = 14.000
Fly.MouseButton1Down:connect(function()
	fly_()
end)

UICorner_3.CornerRadius = UDim.new(0, 4)
UICorner_3.Parent = Fly

TextLabel.Parent = RageGUI
TextLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel.BackgroundTransparency = 1.000
TextLabel.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel.Size = UDim2.new(0, 72, 0, 50)
TextLabel.Font = Enum.Font.SourceSans
TextLabel.Text = "Fly"
TextLabel.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel.TextSize = 14.000

TextLabel_2.Parent = RageGUI
TextLabel_2.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_2.BackgroundTransparency = 1.000
TextLabel_2.Position = UDim2.new(0.0140724014, 0, 0.0224702358, 0)
TextLabel_2.Size = UDim2.new(0, 72, 0, 50)
TextLabel_2.Font = Enum.Font.SourceSans
TextLabel_2.Text = "Auto wallbang"
TextLabel_2.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_2.TextSize = 14.000

TextLabel_3.Parent = RageGUI
TextLabel_3.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_3.BackgroundTransparency = 1.000
TextLabel_3.Position = UDim2.new(0.0140724014, 0, 0.0508793294, 0)
TextLabel_3.Size = UDim2.new(0, 72, 0, 50)
TextLabel_3.Font = Enum.Font.SourceSans
TextLabel_3.Text = "Rage aim"
TextLabel_3.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_3.TextSize = 14.000

Aoutowall.Name = "Aoutowall"
Aoutowall.Parent = RageGUI
Aoutowall.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Aoutowall.BorderSizePixel = 0
Aoutowall.Position = UDim2.new(0.32100001, 0, 0.0399999991, 0)
Aoutowall.Size = UDim2.new(0, 15, 0, 15)
Aoutowall.Font = Enum.Font.SourceSans
Aoutowall.Text = ""
Aoutowall.TextColor3 = Color3.fromRGB(0, 0, 0)
Aoutowall.TextSize = 14.000
Aoutowall.MouseButton1Down:connect(function()
	Auto_wallbang()
end)

UICorner_4.CornerRadius = UDim.new(0, 4)
UICorner_4.Parent = Aoutowall

Rageaim.Name = "Rageaim"
Rageaim.Parent = RageGUI
Rageaim.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Rageaim.BorderSizePixel = 0
Rageaim.Position = UDim2.new(0.32100001, 0, 0.0689999983, 0)
Rageaim.Size = UDim2.new(0, 15, 0, 15)
Rageaim.Font = Enum.Font.SourceSans
Rageaim.Text = ""
Rageaim.TextColor3 = Color3.fromRGB(0, 0, 0)
Rageaim.TextSize = 14.000
Rageaim.MouseButton1Down:connect(function()
	Rage_aim()
end)

UICorner_5.CornerRadius = UDim.new(0, 4)
UICorner_5.Parent = Rageaim

TextLabel_4.Parent = RageGUI
TextLabel_4.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_4.BackgroundTransparency = 1.000
TextLabel_4.Position = UDim2.new(0.0140724014, 0, 0.173985392, 0)
TextLabel_4.Size = UDim2.new(0, 72, 0, 50)
TextLabel_4.Visible = false
TextLabel_4.Font = Enum.Font.SourceSans
TextLabel_4.Text = "Speed"
TextLabel_4.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_4.TextSize = 14.000

TextLabel_5.Parent = RageGUI
TextLabel_5.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_5.BackgroundTransparency = 1.000
TextLabel_5.Position = UDim2.new(0.0140724014, 0, 0.112432361, 0)
TextLabel_5.Size = UDim2.new(0, 72, 0, 50)
TextLabel_5.Visible = false
TextLabel_5.Font = Enum.Font.SourceSans
TextLabel_5.Text = "Jump power"
TextLabel_5.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_5.TextSize = 14.000

TextLabel_6.Parent = RageGUI
TextLabel_6.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_6.BackgroundTransparency = 1.000
TextLabel_6.Position = UDim2.new(0.0140724014, 0, 0.142735392, 0)
TextLabel_6.Size = UDim2.new(0, 72, 0, 50)
TextLabel_6.Visible = false
TextLabel_6.Font = Enum.Font.SourceSans
TextLabel_6.Text = "No fall dmg"
TextLabel_6.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_6.TextSize = 14.000

Rejoinwhenkick.Name = "Rejoin when kick"
Rejoinwhenkick.Parent = RageGUI
Rejoinwhenkick.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Rejoinwhenkick.BorderSizePixel = 0
Rejoinwhenkick.Position = UDim2.new(0.32100001, 0, 0.0993030295, 0)
Rejoinwhenkick.Size = UDim2.new(0, 15, 0, 15)
Rejoinwhenkick.Font = Enum.Font.SourceSans
Rejoinwhenkick.Text = ""
Rejoinwhenkick.TextColor3 = Color3.fromRGB(0, 0, 0)
Rejoinwhenkick.TextSize = 14.000
Rejoinwhenkick.MouseButton1Down:connect(function()
	Rejoin_when_kick()
end)

UICorner_6.CornerRadius = UDim.new(0, 4)
UICorner_6.Parent = Rejoinwhenkick

Jumppower.Name = "Jump power"
Jumppower.Parent = RageGUI
Jumppower.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Jumppower.BorderSizePixel = 0
Jumppower.Position = UDim2.new(0.32100001, 0, 0.131500006, 0)
Jumppower.Size = UDim2.new(0, 15, 0, 15)
Jumppower.Visible = false
Jumppower.Font = Enum.Font.SourceSans
Jumppower.Text = ""
Jumppower.TextColor3 = Color3.fromRGB(0, 0, 0)
Jumppower.TextSize = 14.000

UICorner_7.CornerRadius = UDim.new(0, 4)
UICorner_7.Parent = Jumppower

Rageaim_2.Name = "Rageaim"
Rageaim_2.Parent = RageGUI
Rageaim_2.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Rageaim_2.BorderSizePixel = 0
Rageaim_2.Position = UDim2.new(0.32100001, 0, 0.409628361, 0)
Rageaim_2.Size = UDim2.new(0, 15, 0, 15)
Rageaim_2.Font = Enum.Font.SourceSans
Rageaim_2.Text = ""
Rageaim_2.TextColor3 = Color3.fromRGB(0, 0, 0)
Rageaim_2.TextSize = 14.000

UICorner_8.CornerRadius = UDim.new(0, 4)
UICorner_8.Parent = Rageaim_2

Nofalldmg.Name = "No fall dmg"
Nofalldmg.Parent = RageGUI
Nofalldmg.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Nofalldmg.BorderSizePixel = 0
Nofalldmg.Position = UDim2.new(0.32100001, 0, 0.159909099, 0)
Nofalldmg.Size = UDim2.new(0, 15, 0, 15)
Nofalldmg.Visible = false
Nofalldmg.Font = Enum.Font.SourceSans
Nofalldmg.Text = ""
Nofalldmg.TextColor3 = Color3.fromRGB(0, 0, 0)
Nofalldmg.TextSize = 14.000

UICorner_9.CornerRadius = UDim.new(0, 4)
UICorner_9.Parent = Nofalldmg

TextLabel_7.Parent = RageGUI
TextLabel_7.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_7.BackgroundTransparency = 1.000
TextLabel_7.Position = UDim2.new(0.0140724014, 0, 0.0830762982, 0)
TextLabel_7.Size = UDim2.new(0, 72, 0, 50)
TextLabel_7.Font = Enum.Font.SourceSans
TextLabel_7.Text = "Rejoin when kick"
TextLabel_7.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_7.TextSize = 14.000

Speed.Name = "Speed"
Speed.Parent = RageGUI
Speed.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Speed.BorderSizePixel = 0
Speed.Position = UDim2.new(0.32100001, 0, 0.190212131, 0)
Speed.Size = UDim2.new(0, 15, 0, 15)
Speed.Visible = false
Speed.Font = Enum.Font.SourceSans
Speed.Text = ""
Speed.TextColor3 = Color3.fromRGB(0, 0, 0)
Speed.TextSize = 14.000

UICorner_10.CornerRadius = UDim.new(0, 4)
UICorner_10.Parent = Speed

LegitGUI.Name = "Legit GUI"
LegitGUI.Parent = HyperX
LegitGUI.Active = true
LegitGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
LegitGUI.BorderSizePixel = 0
LegitGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
LegitGUI.Size = UDim2.new(0, 693, 0, 339)
LegitGUI.Visible = false
LegitGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
LegitGUI.ScrollBarThickness = 7

Silentaim.Name = "Silent aim"
Silentaim.Parent = LegitGUI
Silentaim.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Silentaim.BorderSizePixel = 0
Silentaim.Position = UDim2.new(0.32100001, 0, 0.00999999978, 0)
Silentaim.Size = UDim2.new(0, 15, 0, 15)
Silentaim.Font = Enum.Font.SourceSans
Silentaim.Text = ""
Silentaim.TextColor3 = Color3.fromRGB(0, 0, 0)
Silentaim.TextSize = 14.000
Silentaim.MouseButton1Down:connect(function()
	Silent_aim()
end)

UICorner_11.CornerRadius = UDim.new(0, 4)
UICorner_11.Parent = Silentaim

TextLabel_8.Parent = LegitGUI
TextLabel_8.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_8.BackgroundTransparency = 1.000
TextLabel_8.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel_8.Size = UDim2.new(0, 72, 0, 50)
TextLabel_8.Font = Enum.Font.SourceSans
TextLabel_8.Text = "Silent aim"
TextLabel_8.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_8.TextSize = 14.000

TextLabel_9.Parent = LegitGUI
TextLabel_9.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_9.BackgroundTransparency = 1.000
TextLabel_9.Position = UDim2.new(0.0299454182, 0, 0.0234172046, 0)
TextLabel_9.Size = UDim2.new(0, 72, 0, 50)
TextLabel_9.Font = Enum.Font.SourceSans
TextLabel_9.Text = "Hit box extender (Torso)"
TextLabel_9.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_9.TextSize = 14.000

TextLabel_10.Parent = LegitGUI
TextLabel_10.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_10.BackgroundTransparency = 1.000
TextLabel_10.Position = UDim2.new(0.0400464274, 0, 0.052773267, 0)
TextLabel_10.Size = UDim2.new(0, 72, 0, 50)
TextLabel_10.Font = Enum.Font.SourceSans
TextLabel_10.Text = "Hit box extender (Head)"
TextLabel_10.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_10.TextSize = 14.000

TextLabel_11.Parent = LegitGUI
TextLabel_11.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_11.BackgroundTransparency = 1.000
TextLabel_11.Position = UDim2.new(0.0400464274, 0, 0.084023267, 0)
TextLabel_11.Size = UDim2.new(0, 72, 0, 50)
TextLabel_11.Font = Enum.Font.SourceSans
TextLabel_11.Text = "Aim lock"
TextLabel_11.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_11.TextSize = 14.000

HitboxextenderHead.Name = "Hit box extender (Head)"
HitboxextenderHead.Parent = LegitGUI
HitboxextenderHead.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
HitboxextenderHead.BorderSizePixel = 0
HitboxextenderHead.Position = UDim2.new(0.32100001, 0, 0.0687121227, 0)
HitboxextenderHead.Size = UDim2.new(0, 15, 0, 15)
HitboxextenderHead.Font = Enum.Font.SourceSans
HitboxextenderHead.Text = ""
HitboxextenderHead.TextColor3 = Color3.fromRGB(0, 0, 0)
HitboxextenderHead.TextSize = 14.000
HitboxextenderHead.MouseButton1Down:connect(function()
	Hit_box_extender_head()
end)

UICorner_12.CornerRadius = UDim.new(0, 4)
UICorner_12.Parent = HitboxextenderHead

Aimlock.Name = "Aim lock"
Aimlock.Parent = LegitGUI
Aimlock.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Aimlock.BorderSizePixel = 0
Aimlock.Position = UDim2.new(0.32100001, 0, 0.0999621227, 0)
Aimlock.Size = UDim2.new(0, 15, 0, 15)
Aimlock.Font = Enum.Font.SourceSans
Aimlock.Text = ""
Aimlock.TextColor3 = Color3.fromRGB(0, 0, 0)
Aimlock.TextSize = 14.000
Aimlock.MouseButton1Down:connect(function()
	Aim_lock()
end)

UICorner_13.CornerRadius = UDim.new(0, 4)
UICorner_13.Parent = Aimlock

TextLabel_12.Parent = LegitGUI
TextLabel_12.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_12.BackgroundTransparency = 1.000
TextLabel_12.Position = UDim2.new(0.0400464274, 0, 0.115273267, 0)
TextLabel_12.Size = UDim2.new(0, 72, 0, 50)
TextLabel_12.Visible = false
TextLabel_12.Font = Enum.Font.SourceSans
TextLabel_12.Text = "Silent aimt head shot"
TextLabel_12.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_12.TextSize = 14.000

Silentaimtheadshot.Name = "Silent aimt head shot"
Silentaimtheadshot.Parent = LegitGUI
Silentaimtheadshot.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Silentaimtheadshot.BorderSizePixel = 0
Silentaimtheadshot.Position = UDim2.new(0.32100001, 0, 0.131212115, 0)
Silentaimtheadshot.Size = UDim2.new(0, 15, 0, 15)
Silentaimtheadshot.Visible = false
Silentaimtheadshot.Font = Enum.Font.SourceSans
Silentaimtheadshot.Text = ""
Silentaimtheadshot.TextColor3 = Color3.fromRGB(0, 0, 0)
Silentaimtheadshot.TextSize = 14.000

UICorner_14.CornerRadius = UDim.new(0, 4)
UICorner_14.Parent = Silentaimtheadshot

HitboxextenderTorso.Name = "Hit box extender (Torso)"
HitboxextenderTorso.Parent = LegitGUI
HitboxextenderTorso.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
HitboxextenderTorso.BorderSizePixel = 0
HitboxextenderTorso.Position = UDim2.new(0.32100001, 0, 0.0393560603, 0)
HitboxextenderTorso.Size = UDim2.new(0, 15, 0, 15)
HitboxextenderTorso.Font = Enum.Font.SourceSans
HitboxextenderTorso.Text = ""
HitboxextenderTorso.TextColor3 = Color3.fromRGB(0, 0, 0)
HitboxextenderTorso.TextSize = 14.000
HitboxextenderTorso.MouseButton1Down:connect(function()
	Hit_box_extender_Torso()
end)

UICorner_15.CornerRadius = UDim.new(0, 4)
UICorner_15.Parent = HitboxextenderTorso

ConfigGUI.Name = "Config GUI"
ConfigGUI.Parent = HyperX
ConfigGUI.Active = true
ConfigGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
ConfigGUI.BorderSizePixel = 0
ConfigGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
ConfigGUI.Size = UDim2.new(0, 693, 0, 339)
ConfigGUI.Visible = false
ConfigGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
ConfigGUI.ScrollBarThickness = 7

Ragemode.Name = "Rage mode"
Ragemode.Parent = ConfigGUI
Ragemode.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Ragemode.BorderSizePixel = 0
Ragemode.Position = UDim2.new(0.32100001, 0, 0.00999999978, 0)
Ragemode.Size = UDim2.new(0, 15, 0, 15)
Ragemode.Font = Enum.Font.SourceSans
Ragemode.Text = ""
Ragemode.TextColor3 = Color3.fromRGB(0, 0, 0)
Ragemode.TextSize = 14.000
Ragemode.MouseButton1Down:connect(function()
	Rage_Config()
end)

UICorner_16.CornerRadius = UDim.new(0, 4)
UICorner_16.Parent = Ragemode

TextLabel_13.Parent = ConfigGUI
TextLabel_13.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_13.BackgroundTransparency = 1.000
TextLabel_13.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel_13.Size = UDim2.new(0, 72, 0, 50)
TextLabel_13.Font = Enum.Font.SourceSans
TextLabel_13.Text = "Rage"
TextLabel_13.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_13.TextSize = 14.000

TextLabel_14.Parent = ConfigGUI
TextLabel_14.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_14.BackgroundTransparency = 1.000
TextLabel_14.Position = UDim2.new(0.0140724014, 0, 0.0224702358, 0)
TextLabel_14.Size = UDim2.new(0, 72, 0, 50)
TextLabel_14.Font = Enum.Font.SourceSans
TextLabel_14.Text = "Rage with visuals"
TextLabel_14.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_14.TextSize = 14.000

TextLabel_15.Parent = ConfigGUI
TextLabel_15.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_15.BackgroundTransparency = 1.000
TextLabel_15.Position = UDim2.new(0.0140724014, 0, 0.0508793294, 0)
TextLabel_15.Size = UDim2.new(0, 72, 0, 50)
TextLabel_15.Font = Enum.Font.SourceSans
TextLabel_15.Text = "Legit "
TextLabel_15.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_15.TextSize = 14.000

LegitMode.Name = "Legit Mode"
LegitMode.Parent = ConfigGUI
LegitMode.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
LegitMode.BorderSizePixel = 0
LegitMode.Position = UDim2.new(0.32100001, 0, 0.0689999983, 0)
LegitMode.Size = UDim2.new(0, 15, 0, 15)
LegitMode.Font = Enum.Font.SourceSans
LegitMode.Text = ""
LegitMode.TextColor3 = Color3.fromRGB(0, 0, 0)
LegitMode.TextSize = 14.000
LegitMode.MouseButton1Down:connect(function()
	Legit_Config()
end)

UICorner_17.CornerRadius = UDim.new(0, 4)
UICorner_17.Parent = LegitMode

Ragewithvisualsmode.Name = "Rage with visualsmode"
Ragewithvisualsmode.Parent = ConfigGUI
Ragewithvisualsmode.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Ragewithvisualsmode.BorderSizePixel = 0
Ragewithvisualsmode.Position = UDim2.new(0.32100001, 0, 0.0399999991, 0)
Ragewithvisualsmode.Size = UDim2.new(0, 15, 0, 15)
Ragewithvisualsmode.Font = Enum.Font.SourceSans
Ragewithvisualsmode.Text = ""
Ragewithvisualsmode.TextColor3 = Color3.fromRGB(0, 0, 0)
Ragewithvisualsmode.TextSize = 14.000
Ragewithvisualsmode.MouseButton1Down:connect(function()
	Rage_With_Visuals()
end)

UICorner_18.CornerRadius = UDim.new(0, 4)
UICorner_18.Parent = Ragewithvisualsmode

TextLabel_16.Parent = ConfigGUI
TextLabel_16.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_16.BackgroundTransparency = 1.000
TextLabel_16.Position = UDim2.new(0.0140724014, 0, 0.0830762982, 0)
TextLabel_16.Size = UDim2.new(0, 72, 0, 50)
TextLabel_16.Font = Enum.Font.SourceSans
TextLabel_16.Text = "Beta"
TextLabel_16.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_16.TextSize = 14.000

Beta_mode.Name = "Beta_mode"
Beta_mode.Parent = ConfigGUI
Beta_mode.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Beta_mode.BorderSizePixel = 0
Beta_mode.Position = UDim2.new(0.32100001, 0, 0.0993030295, 0)
Beta_mode.Size = UDim2.new(0, 15, 0, 15)
Beta_mode.Font = Enum.Font.SourceSans
Beta_mode.Text = ""
Beta_mode.TextColor3 = Color3.fromRGB(0, 0, 0)
Beta_mode.TextSize = 14.000
Beta_mode.MouseButton1Down:connect(function()
	Beta_Rage()
end)

UICorner_19.CornerRadius = UDim.new(0, 4)
UICorner_19.Parent = Beta_mode

VisualsGUi.Name = "Visuals GUi"
VisualsGUi.Parent = HyperX
VisualsGUi.Active = true
VisualsGUi.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
VisualsGUi.BorderSizePixel = 0
VisualsGUi.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
VisualsGUi.Size = UDim2.new(0, 693, 0, 339)
VisualsGUi.Visible = false
VisualsGUi.CanvasSize = UDim2.new(0, 0, 3, 0)
VisualsGUi.ScrollBarThickness = 7

Traers.Name = "Traers"
Traers.Parent = VisualsGUi
Traers.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Traers.BorderSizePixel = 0
Traers.Position = UDim2.new(0.32100001, 0, 0.00999999978, 0)
Traers.Size = UDim2.new(0, 15, 0, 15)
Traers.Font = Enum.Font.SourceSans
Traers.Text = ""
Traers.TextColor3 = Color3.fromRGB(0, 0, 0)
Traers.TextSize = 14.000
Traers.MouseButton1Down:connect(function()
	TracersG()
end)

UICorner_20.CornerRadius = UDim.new(0, 4)
UICorner_20.Parent = Traers

TextLabel_17.Parent = VisualsGUi
TextLabel_17.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_17.BackgroundTransparency = 1.000
TextLabel_17.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel_17.Size = UDim2.new(0, 72, 0, 50)
TextLabel_17.Font = Enum.Font.SourceSans
TextLabel_17.Text = "Traers"
TextLabel_17.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_17.TextSize = 14.000

TextLabel_18.Parent = VisualsGUi
TextLabel_18.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_18.BackgroundTransparency = 1.000
TextLabel_18.Position = UDim2.new(0.0140724014, 0, 0.0224702358, 0)
TextLabel_18.Size = UDim2.new(0, 72, 0, 50)
TextLabel_18.Font = Enum.Font.SourceSans
TextLabel_18.Text = "Name"
TextLabel_18.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_18.TextSize = 14.000

TextLabel_19.Parent = VisualsGUi
TextLabel_19.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_19.BackgroundTransparency = 1.000
TextLabel_19.Position = UDim2.new(0.0140724014, 0, 0.0508793294, 0)
TextLabel_19.Size = UDim2.new(0, 72, 0, 50)
TextLabel_19.Font = Enum.Font.SourceSans
TextLabel_19.Text = "Grenade"
TextLabel_19.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_19.TextSize = 14.000

TextLabel_20.Parent = VisualsGUi
TextLabel_20.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_20.BackgroundTransparency = 1.000
TextLabel_20.Position = UDim2.new(0.0140724014, 0, 0.0811823606, 0)
TextLabel_20.Size = UDim2.new(0, 72, 0, 50)
TextLabel_20.Font = Enum.Font.SourceSans
TextLabel_20.Text = "Charms"
TextLabel_20.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_20.TextSize = 14.000

Name.Name = "Name"
Name.Parent = VisualsGUi
Name.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Name.BorderSizePixel = 0
Name.Position = UDim2.new(0.32100001, 0, 0.0399999991, 0)
Name.Size = UDim2.new(0, 15, 0, 15)
Name.Font = Enum.Font.SourceSans
Name.Text = ""
Name.TextColor3 = Color3.fromRGB(0, 0, 0)
Name.TextSize = 14.000
Name.MouseButton1Down:connect(function()
	Name()
end)

UICorner_21.CornerRadius = UDim.new(0, 4)
UICorner_21.Parent = Name

Grenade.Name = "Grenade"
Grenade.Parent = VisualsGUi
Grenade.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Grenade.BorderSizePixel = 0
Grenade.Position = UDim2.new(0.32100001, 0, 0.0689999983, 0)
Grenade.Size = UDim2.new(0, 15, 0, 15)
Grenade.Font = Enum.Font.SourceSans
Grenade.Text = ""
Grenade.TextColor3 = Color3.fromRGB(0, 0, 0)
Grenade.TextSize = 14.000
Grenade.MouseButton1Down:connect(function()
	Grenade_1()
end)

UICorner_22.CornerRadius = UDim.new(0, 4)
UICorner_22.Parent = Grenade

Charms.Name = "Charms"
Charms.Parent = VisualsGUi
Charms.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Charms.BorderSizePixel = 0
Charms.Position = UDim2.new(0.32100001, 0, 0.0979999974, 0)
Charms.Size = UDim2.new(0, 15, 0, 15)
Charms.Font = Enum.Font.SourceSans
Charms.Text = ""
Charms.TextColor3 = Color3.fromRGB(0, 0, 0)
Charms.TextSize = 14.000
Charms.MouseButton1Down:connect(function()
	Charms_1()
end)

UICorner_23.CornerRadius = UDim.new(0, 4)
UICorner_23.Parent = Charms

TextLabel_21.Parent = VisualsGUi
TextLabel_21.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_21.BackgroundTransparency = 1.000
TextLabel_21.Position = UDim2.new(0.0140724014, 0, 0.111485392, 0)
TextLabel_21.Size = UDim2.new(0, 72, 0, 50)
TextLabel_21.Font = Enum.Font.SourceSans
TextLabel_21.Text = "Arm charms"
TextLabel_21.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_21.TextSize = 14.000

Armcharms.Name = "Arm charms"
Armcharms.Parent = VisualsGUi
Armcharms.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Armcharms.BorderSizePixel = 0
Armcharms.Position = UDim2.new(0.32100001, 0, 0.128999993, 0)
Armcharms.Size = UDim2.new(0, 15, 0, 15)
Armcharms.Font = Enum.Font.SourceSans
Armcharms.Text = ""
Armcharms.TextColor3 = Color3.fromRGB(0, 0, 0)
Armcharms.TextSize = 14.000
Armcharms.MouseButton1Down:connect(function()
	Arms_Charms()
end)

UICorner_24.CornerRadius = UDim.new(0, 4)
UICorner_24.Parent = Armcharms

Fullbright.Name = "Fullbright"
Fullbright.Parent = VisualsGUi
Fullbright.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Fullbright.BorderSizePixel = 0
Fullbright.Position = UDim2.new(0.32100001, 0, 0.160999998, 0)
Fullbright.Size = UDim2.new(0, 15, 0, 15)
Fullbright.Font = Enum.Font.SourceSans
Fullbright.Text = ""
Fullbright.TextColor3 = Color3.fromRGB(0, 0, 0)
Fullbright.TextSize = 14.000
Fullbright.MouseButton1Down:connect(function()
	Fullbright_1()
end)

UICorner_25.CornerRadius = UDim.new(0, 4)
UICorner_25.Parent = Fullbright

TextLabel_22.Parent = VisualsGUi
TextLabel_22.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_22.BackgroundTransparency = 1.000
TextLabel_22.Position = UDim2.new(0.0140724014, 0, 0.143682361, 0)
TextLabel_22.Size = UDim2.new(0, 72, 0, 50)
TextLabel_22.Font = Enum.Font.SourceSans
TextLabel_22.Text = "Fullbright"
TextLabel_22.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_22.TextSize = 14.000

TextLabel_23.Parent = VisualsGUi
TextLabel_23.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_23.BackgroundTransparency = 1.000
TextLabel_23.Position = UDim2.new(0.0299454182, 0, 0.174932346, 0)
TextLabel_23.Size = UDim2.new(0, 72, 0, 50)
TextLabel_23.Font = Enum.Font.SourceSans
TextLabel_23.Text = "Grenade prediction"
TextLabel_23.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_23.TextSize = 14.000

Grenadeprediction.Name = "Grenade prediction"
Grenadeprediction.Parent = VisualsGUi
Grenadeprediction.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Grenadeprediction.BorderSizePixel = 0
Grenadeprediction.Position = UDim2.new(0.32100001, 0, 0.19130303, 0)
Grenadeprediction.Size = UDim2.new(0, 15, 0, 15)
Grenadeprediction.Font = Enum.Font.SourceSans
Grenadeprediction.Text = ""
Grenadeprediction.TextColor3 = Color3.fromRGB(0, 0, 0)
Grenadeprediction.TextSize = 14.000
Grenadeprediction.MouseButton1Down:connect(function()
	Grenade_prediction()
end)

UICorner_26.CornerRadius = UDim.new(0, 4)
UICorner_26.Parent = Grenadeprediction

TextLabel_24.Parent = VisualsGUi
TextLabel_24.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_24.BackgroundTransparency = 1.000
TextLabel_24.Position = UDim2.new(0.0299454182, 0, 0.205235377, 0)
TextLabel_24.Size = UDim2.new(0, 72, 0, 50)
TextLabel_24.Font = Enum.Font.SourceSans
TextLabel_24.Text = "Droped guns"
TextLabel_24.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_24.TextSize = 14.000

Dropedguns.Name = "Droped guns"
Dropedguns.Parent = VisualsGUi
Dropedguns.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Dropedguns.BorderSizePixel = 0
Dropedguns.Position = UDim2.new(0.32100001, 0, 0.221606061, 0)
Dropedguns.Size = UDim2.new(0, 15, 0, 15)
Dropedguns.Font = Enum.Font.SourceSans
Dropedguns.Text = ""
Dropedguns.TextColor3 = Color3.fromRGB(0, 0, 0)
Dropedguns.TextSize = 14.000
Dropedguns.MouseButton1Down:connect(function()
	Droped_guns()
end)

UICorner_27.CornerRadius = UDim.new(0, 4)
UICorner_27.Parent = Dropedguns

WorldsettingsGUI.Name = "World settings GUI"
WorldsettingsGUI.Parent = HyperX
WorldsettingsGUI.Active = true
WorldsettingsGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
WorldsettingsGUI.BorderSizePixel = 0
WorldsettingsGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
WorldsettingsGUI.Size = UDim2.new(0, 693, 0, 339)
WorldsettingsGUI.Visible = false
WorldsettingsGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
WorldsettingsGUI.ScrollBarThickness = 7

TextLabel_25.Parent = WorldsettingsGUI
TextLabel_25.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_25.BackgroundTransparency = 1.000
TextLabel_25.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel_25.Size = UDim2.new(0, 72, 0, 50)
TextLabel_25.Font = Enum.Font.SourceSans
TextLabel_25.Text = "World material"
TextLabel_25.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_25.TextSize = 14.000

TextLabel_26.Parent = WorldsettingsGUI
TextLabel_26.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_26.BackgroundTransparency = 1.000
TextLabel_26.Position = UDim2.new(0.0140724014, 0, 0.0224702358, 0)
TextLabel_26.Size = UDim2.new(0, 72, 0, 50)
TextLabel_26.Font = Enum.Font.SourceSans
TextLabel_26.Text = "World Color"
TextLabel_26.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_26.TextSize = 14.000

Frame.Parent = WorldsettingsGUI
Frame.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Frame.BorderSizePixel = 0
Frame.Position = UDim2.new(0.144300148, 0, 0.0066948887, 0)
Frame.Size = UDim2.new(0, 122, 0, 21)

Mat1.Name = "Mat1"
Mat1.Parent = Frame
Mat1.Active = true
Mat1.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Mat1.BorderSizePixel = 0
Mat1.Position = UDim2.new(1, 0, 0, 0)
Mat1.Size = UDim2.new(0, 406, 0, 60)
Mat1.Visible = false

Neon.Name = "Neon"
Neon.Parent = Mat1
Neon.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Neon.BorderSizePixel = 0
Neon.Size = UDim2.new(0, 81, 0, 16)
Neon.Font = Enum.Font.SourceSans
Neon.Text = "Neon"
Neon.TextColor3 = Color3.fromRGB(16, 180, 160)
Neon.TextSize = 14.000
Neon.MouseButton1Down:connect(function()
	Neon_hhhp ()
end)

forcefield.Name = "forcefield"
forcefield.Parent = Mat1
forcefield.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
forcefield.BorderSizePixel = 0
forcefield.Position = UDim2.new(0.199507385, 0, 0, 0)
forcefield.Size = UDim2.new(0, 81, 0, 16)
forcefield.Font = Enum.Font.SourceSans
forcefield.Text = "Forcefield"
forcefield.TextColor3 = Color3.fromRGB(16, 180, 160)
forcefield.TextSize = 14.000
forcefield.MouseButton1Down:connect(function()
	forcefield_hhhp ()
end)

Wood.Name = "Wood"
Wood.Parent = Mat1
Wood.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Wood.BorderSizePixel = 0
Wood.Position = UDim2.new(0.399014771, 0, 0, 0)
Wood.Size = UDim2.new(0, 81, 0, 16)
Wood.Font = Enum.Font.SourceSans
Wood.Text = "Wood"
Wood.TextColor3 = Color3.fromRGB(16, 180, 160)
Wood.TextSize = 14.000
Wood.MouseButton1Down:connect(function()
	Wood_hhhp ()
end)

Smoothplastic.Name = "Smooth plastic"
Smoothplastic.Parent = Mat1
Smoothplastic.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Smoothplastic.BorderSizePixel = 0
Smoothplastic.Position = UDim2.new(0.598522186, 0, 0.0333333351, 0)
Smoothplastic.Size = UDim2.new(0, 81, 0, 16)
Smoothplastic.Font = Enum.Font.SourceSans
Smoothplastic.Text = "Smooth plastic"
Smoothplastic.TextColor3 = Color3.fromRGB(16, 180, 160)
Smoothplastic.TextSize = 14.000
Smoothplastic.MouseButton1Down:connect(function()
	Smoothplastic_hhhp ()
end)

Dimondplate.Name = "Dimond plate"
Dimondplate.Parent = Mat1
Dimondplate.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Dimondplate.BorderSizePixel = 0
Dimondplate.Position = UDim2.new(0.798029542, 0, 0.0333333351, 0)
Dimondplate.Size = UDim2.new(0, 81, 0, 16)
Dimondplate.Font = Enum.Font.SourceSans
Dimondplate.Text = "Diamond plate"
Dimondplate.TextColor3 = Color3.fromRGB(16, 180, 160)
Dimondplate.TextSize = 14.000
Dimondplate.MouseButton1Down:connect(function()
	Dimondplate_hhhp ()
end)

Marbel.Name = "Marbel"
Marbel.Parent = Mat1
Marbel.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Marbel.BorderSizePixel = 0
Marbel.Position = UDim2.new(0, 0, 0.316666663, 0)
Marbel.Size = UDim2.new(0, 81, 0, 16)
Marbel.Font = Enum.Font.SourceSans
Marbel.Text = "Marbel"
Marbel.TextColor3 = Color3.fromRGB(16, 180, 160)
Marbel.TextSize = 14.000
Marbel.MouseButton1Down:connect(function()
	Marbel_hhhp ()
end)

Glass.Name = "Glass"
Glass.Parent = Mat1
Glass.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Glass.BorderSizePixel = 0
Glass.Position = UDim2.new(0.1995074, 0, 0.316666663, 0)
Glass.Size = UDim2.new(0, 81, 0, 16)
Glass.Font = Enum.Font.SourceSans
Glass.Text = "Glass"
Glass.TextColor3 = Color3.fromRGB(16, 180, 160)
Glass.TextSize = 14.000
Glass.MouseButton1Down:connect(function()
	Glass_hhhp ()
end)

Metal.Name = "Metal"
Metal.Parent = Mat1
Metal.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Metal.BorderSizePixel = 0
Metal.Position = UDim2.new(0.399014801, 0, 0.366666675, 0)
Metal.Size = UDim2.new(0, 81, 0, 16)
Metal.Font = Enum.Font.SourceSans
Metal.Text = "Metal"
Metal.TextColor3 = Color3.fromRGB(16, 180, 160)
Metal.TextSize = 14.000
Metal.MouseButton1Down:connect(function()
	Metal_hhhp ()
end)

Grass.Name = "Grass"
Grass.Parent = Mat1
Grass.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Grass.BorderSizePixel = 0
Grass.Position = UDim2.new(0.598522186, 0, 0.316666663, 0)
Grass.Size = UDim2.new(0, 81, 0, 16)
Grass.Font = Enum.Font.SourceSans
Grass.Text = "Grass"
Grass.TextColor3 = Color3.fromRGB(16, 180, 160)
Grass.TextSize = 14.000
Grass.MouseButton1Down:connect(function()
	Grass_hhhp ()
end)

Cobbelstone.Name = "Cobbel stone"
Cobbelstone.Parent = Mat1
Cobbelstone.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Cobbelstone.BorderSizePixel = 0
Cobbelstone.Position = UDim2.new(0.798029542, 0, 0.316666663, 0)
Cobbelstone.Size = UDim2.new(0, 81, 0, 16)
Cobbelstone.Font = Enum.Font.SourceSans
Cobbelstone.Text = "Cobblestone"
Cobbelstone.TextColor3 = Color3.fromRGB(16, 180, 160)
Cobbelstone.TextSize = 14.000
Cobbelstone.MouseButton1Down:connect(function()
	Cobbelstone_hhhp ()
end)

Ice.Name = "Ice"
Ice.Parent = Mat1
Ice.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Ice.BorderSizePixel = 0
Ice.Position = UDim2.new(0.206896558, 0, 0.633333325, 0)
Ice.Size = UDim2.new(0, 81, 0, 16)
Ice.Font = Enum.Font.SourceSans
Ice.Text = "Ice"
Ice.TextColor3 = Color3.fromRGB(16, 180, 160)
Ice.TextSize = 14.000
Ice.MouseButton1Down:connect(function()
	Ice_hhp ()
end)

Granite.Name = "Granite "
Granite.Parent = Mat1
Granite.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Granite.BorderSizePixel = 0
Granite.Position = UDim2.new(0.399014771, 0, 0.633333325, 0)
Granite.Size = UDim2.new(0, 81, 0, 16)
Granite.Font = Enum.Font.SourceSans
Granite.Text = "Granite "
Granite.TextColor3 = Color3.fromRGB(16, 180, 160)
Granite.TextSize = 14.000
Granite.MouseButton1Down:connect(function()
	Granite_hhhp ()
end)

Sand.Name = "Sand"
Sand.Parent = Mat1
Sand.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Sand.BorderSizePixel = 0
Sand.Position = UDim2.new(0.598522186, 0, 0.633333325, 0)
Sand.Size = UDim2.new(0, 81, 0, 16)
Sand.Font = Enum.Font.SourceSans
Sand.Text = "Sand"
Sand.TextColor3 = Color3.fromRGB(16, 180, 160)
Sand.TextSize = 14.000
Sand.MouseButton1Down:connect(function()
	Sand_hhhp ()
end)

Slate.Name = "Slate"
Slate.Parent = Mat1
Slate.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Slate.BorderSizePixel = 0
Slate.Position = UDim2.new(0.798029542, 0, 0.633333325, 0)
Slate.Size = UDim2.new(0, 81, 0, 16)
Slate.Font = Enum.Font.SourceSans
Slate.Text = "Slate"
Slate.TextColor3 = Color3.fromRGB(16, 180, 160)
Slate.TextSize = 14.000
Slate.MouseButton1Down:connect(function()
	Slate_hhhp ()
end)

Concrete.Name = "Concrete"
Concrete.Parent = Mat1
Concrete.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Concrete.BorderSizePixel = 0
Concrete.Position = UDim2.new(0, 0, 0.633333325, 0)
Concrete.Size = UDim2.new(0, 81, 0, 16)
Concrete.Font = Enum.Font.SourceSans
Concrete.Text = "Concrete"
Concrete.TextColor3 = Color3.fromRGB(16, 180, 160)
Concrete.TextSize = 14.000
Concrete.MouseButton1Down:connect(function()
	Concrete_hhhp ()
end)

ShowMat1.Name = "Show Mat1"
ShowMat1.Parent = Frame
ShowMat1.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
ShowMat1.BorderSizePixel = 0
ShowMat1.Position = UDim2.new(0, 0, 0.095238097, 0)
ShowMat1.Size = UDim2.new(0, 122, 0, 19)
ShowMat1.Font = Enum.Font.SourceSans
ShowMat1.Text = "Show options"
ShowMat1.TextColor3 = Color3.fromRGB(21, 168, 150)
ShowMat1.TextSize = 14.000
ShowMat1.MouseButton1Down:connect(function()
	if Mat1.Visible == false then
		Mat1.Visible = true
	else
		Mat1.Visible = false
	end
	if TextLabel_27.Visible == false then
		TextLabel_27.Visible = true 
	else
		TextLabel_27.Visible = false
	end
end)

Frame_2.Parent = WorldsettingsGUI
Frame_2.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Frame_2.BorderSizePixel = 0
Frame_2.Position = UDim2.new(0.144300148, 0, 0.0360509492, 0)
Frame_2.Size = UDim2.new(0, 122, 0, 21)

Color1.Name = "Color1"
Color1.Parent = Frame_2
Color1.Active = true
Color1.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Color1.BorderSizePixel = 0
Color1.Position = UDim2.new(1, 0, 0, 0)
Color1.Size = UDim2.new(0, 406, 0, 60)
Color1.Visible = false

Red.Name = "Red"
Red.Parent = Color1
Red.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Red.BorderSizePixel = 0
Red.Size = UDim2.new(0, 81, 0, 16)
Red.Font = Enum.Font.SourceSans
Red.Text = "Red"
Red.TextColor3 = Color3.fromRGB(16, 180, 160)
Red.TextSize = 14.000
Red.MouseButton1Down:connect(function()
	Red_hhhp ()
end)

Blue.Name = "Blue"
Blue.Parent = Color1
Blue.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Blue.BorderSizePixel = 0
Blue.Position = UDim2.new(0.199507385, 0, 0, 0)
Blue.Size = UDim2.new(0, 81, 0, 16)
Blue.Font = Enum.Font.SourceSans
Blue.Text = "Blue"
Blue.TextColor3 = Color3.fromRGB(16, 180, 160)
Blue.TextSize = 14.000
Blue.MouseButton1Down:connect(function()
	Blue_hhhp ()
end)

Pink.Name = "Pink"
Pink.Parent = Color1
Pink.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Pink.BorderSizePixel = 0
Pink.Position = UDim2.new(0.399014771, 0, 0, 0)
Pink.Size = UDim2.new(0, 81, 0, 16)
Pink.Font = Enum.Font.SourceSans
Pink.Text = "Pink"
Pink.TextColor3 = Color3.fromRGB(16, 180, 160)
Pink.TextSize = 14.000
Pink.MouseButton1Down:connect(function()
	Pink_hhhp ()
end)

purple.Name = "purple"
purple.Parent = Color1
purple.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
purple.BorderSizePixel = 0
purple.Position = UDim2.new(0.598522186, 0, 0.0333333351, 0)
purple.Size = UDim2.new(0, 81, 0, 16)
purple.Font = Enum.Font.SourceSans
purple.Text = "purple"
purple.TextColor3 = Color3.fromRGB(16, 180, 160)
purple.TextSize = 14.000
purple.MouseButton1Down:connect(function()
	purple_hhhp ()
end)

cyan.Name = "cyan"
cyan.Parent = Color1
cyan.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
cyan.BorderSizePixel = 0
cyan.Position = UDim2.new(0.798029542, 0, 0.0333333351, 0)
cyan.Size = UDim2.new(0, 81, 0, 16)
cyan.Font = Enum.Font.SourceSans
cyan.Text = "cyan"
cyan.TextColor3 = Color3.fromRGB(16, 180, 160)
cyan.TextSize = 14.000
cyan.MouseButton1Down:connect(function()
	cyan_hhhp ()
end)

white.Name = "white"
white.Parent = Color1
white.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
white.BorderSizePixel = 0
white.Position = UDim2.new(0, 0, 0.316666663, 0)
white.Size = UDim2.new(0, 81, 0, 16)
white.Font = Enum.Font.SourceSans
white.Text = "white"
white.TextColor3 = Color3.fromRGB(16, 180, 160)
white.TextSize = 14.000
white.MouseButton1Down:connect(function()
	white_hhhp ()
end)

Black.Name = "Black"
Black.Parent = Color1
Black.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Black.BorderSizePixel = 0
Black.Position = UDim2.new(0.1995074, 0, 0.316666663, 0)
Black.Size = UDim2.new(0, 81, 0, 16)
Black.Font = Enum.Font.SourceSans
Black.Text = "Black"
Black.TextColor3 = Color3.fromRGB(16, 180, 160)
Black.TextSize = 14.000
Black.MouseButton1Down:connect(function()
	Black_hhhp ()
end)

Matteblack.Name = "Matte black"
Matteblack.Parent = Color1
Matteblack.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Matteblack.BorderSizePixel = 0
Matteblack.Position = UDim2.new(0.399014801, 0, 0.366666675, 0)
Matteblack.Size = UDim2.new(0, 81, 0, 16)
Matteblack.Font = Enum.Font.SourceSans
Matteblack.Text = "Matte black"
Matteblack.TextColor3 = Color3.fromRGB(16, 180, 160)
Matteblack.TextSize = 14.000
Matteblack.MouseButton1Down:connect(function()
	Matteblack_hhhp ()
end)

Gray.Name = "Gray"
Gray.Parent = Color1
Gray.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Gray.BorderSizePixel = 0
Gray.Position = UDim2.new(0.598522186, 0, 0.316666663, 0)
Gray.Size = UDim2.new(0, 81, 0, 16)
Gray.Font = Enum.Font.SourceSans
Gray.Text = "Gray"
Gray.TextColor3 = Color3.fromRGB(16, 180, 160)
Gray.TextSize = 14.000
Gray.MouseButton1Down:connect(function()
	Gray_hhhp ()
end)

Green.Name = "Green"
Green.Parent = Color1
Green.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Green.BorderSizePixel = 0
Green.Position = UDim2.new(0.798029542, 0, 0.316666663, 0)
Green.Size = UDim2.new(0, 81, 0, 16)
Green.Font = Enum.Font.SourceSans
Green.Text = "Green"
Green.TextColor3 = Color3.fromRGB(16, 180, 160)
Green.TextSize = 14.000
Green.MouseButton1Down:connect(function()
	Green_hhhp ()
end)

lightblue.Name = "light blue"
lightblue.Parent = Color1
lightblue.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
lightblue.BorderSizePixel = 0
lightblue.Position = UDim2.new(0.206896558, 0, 0.633333325, 0)
lightblue.Size = UDim2.new(0, 81, 0, 16)
lightblue.Font = Enum.Font.SourceSans
lightblue.Text = "light blue"
lightblue.TextColor3 = Color3.fromRGB(16, 180, 160)
lightblue.TextSize = 14.000
lightblue.MouseButton1Down:connect(function()
	lightblue_hhhp ()
end)

brown.Name = "brown"
brown.Parent = Color1
brown.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
brown.BorderSizePixel = 0
brown.Position = UDim2.new(0.399014771, 0, 0.633333325, 0)
brown.Size = UDim2.new(0, 81, 0, 16)
brown.Font = Enum.Font.SourceSans
brown.Text = "brown"
brown.TextColor3 = Color3.fromRGB(16, 180, 160)
brown.TextSize = 14.000
brown.MouseButton1Down:connect(function()
	brown_hhhp_ ()
end)

Orange.Name = "Orange"
Orange.Parent = Color1
Orange.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Orange.BorderSizePixel = 0
Orange.Position = UDim2.new(0.598522186, 0, 0.633333325, 0)
Orange.Size = UDim2.new(0, 81, 0, 16)
Orange.Font = Enum.Font.SourceSans
Orange.Text = "Orange"
Orange.TextColor3 = Color3.fromRGB(16, 180, 160)
Orange.TextSize = 14.000
Orange.MouseButton1Down:connect(function()
	Orange_hhhp ()
end)

Turquoise.Name = "Turquoise "
Turquoise.Parent = Color1
Turquoise.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Turquoise.BorderSizePixel = 0
Turquoise.Position = UDim2.new(0.798029542, 0, 0.633333325, 0)
Turquoise.Size = UDim2.new(0, 81, 0, 16)
Turquoise.Font = Enum.Font.SourceSans
Turquoise.Text = "Turquoise "
Turquoise.TextColor3 = Color3.fromRGB(16, 180, 160)
Turquoise.TextSize = 14.000
Turquoise.MouseButton1Down:connect(function()
	Turquoise_hhhp ()
end)

Yello.Name = "Yello"
Yello.Parent = Color1
Yello.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
Yello.BorderSizePixel = 0
Yello.Position = UDim2.new(0, 0, 0.633333325, 0)
Yello.Size = UDim2.new(0, 81, 0, 16)
Yello.Font = Enum.Font.SourceSans
Yello.Text = "Yello"
Yello.TextColor3 = Color3.fromRGB(16, 180, 160)
Yello.TextSize = 14.000
Yello.MouseButton1Down:connect(function()
	Yello_hhhp ()
end)

ShowColor1.Name = "Show Color1"
ShowColor1.Parent = Frame_2
ShowColor1.BackgroundColor3 = Color3.fromRGB(38, 39, 39)
ShowColor1.BorderSizePixel = 0
ShowColor1.Position = UDim2.new(0, 0, 0.0476191044, 0)
ShowColor1.Size = UDim2.new(0, 122, 0, 19)
ShowColor1.Font = Enum.Font.SourceSans
ShowColor1.Text = "Show options"
ShowColor1.TextColor3 = Color3.fromRGB(21, 168, 150)
ShowColor1.TextSize = 14.000
ShowColor1.MouseButton1Down:connect(function()
	if Color1.Visible == false then
		Color1.Visible = true
	else
		Color1.Visible = false
	end
end)

TextLabel_27.Parent = WorldsettingsGUI
TextLabel_27.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_27.BackgroundTransparency = 1.000
TextLabel_27.Position = UDim2.new(0.462845802, 0, -0.00972673576, 0)
TextLabel_27.Size = UDim2.new(0, 72, 0, 50)
TextLabel_27.Font = Enum.Font.SourceSans
TextLabel_27.Text = "<----- Smooth plastic will make fps better"
TextLabel_27.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_27.TextSize = 14.000

TextLabel_28.Parent = WorldsettingsGUI
TextLabel_28.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_28.BackgroundTransparency = 1.000
TextLabel_28.Position = UDim2.new(0.0140724033, 0, 0.0556141734, 0)
TextLabel_28.Size = UDim2.new(0, 72, 0, 50)
TextLabel_28.Font = Enum.Font.SourceSans
TextLabel_28.Text = "Break all windows"
TextLabel_28.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_28.TextSize = 14.000

TextLabel_29.Parent = WorldsettingsGUI
TextLabel_29.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_29.BackgroundTransparency = 1.000
TextLabel_29.Position = UDim2.new(0.0790074691, 0, 0.0963338763, 0)
TextLabel_29.Size = UDim2.new(0, 72, 0, 50)
TextLabel_29.Font = Enum.Font.SourceSans
TextLabel_29.Text = "More options in later updates !!!"
TextLabel_29.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_29.TextSize = 14.000

breakallwindows.Name = "break all windows"
breakallwindows.Parent = WorldsettingsGUI
breakallwindows.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
breakallwindows.BorderSizePixel = 0
breakallwindows.Position = UDim2.new(0.319557011, 0, 0.0712499991, 0)
breakallwindows.Size = UDim2.new(0, 15, 0, 15)
breakallwindows.Font = Enum.Font.SourceSans
breakallwindows.Text = ""
breakallwindows.TextColor3 = Color3.fromRGB(0, 0, 0)
breakallwindows.TextSize = 14.000
breakallwindows.MouseButton1Down:connect(function()
	Brake_all_windows()
end)

UICorner_28.CornerRadius = UDim.new(0, 4)
UICorner_28.Parent = breakallwindows

GunmodsGUI.Name = "Gun mods GUI"
GunmodsGUI.Parent = HyperX
GunmodsGUI.Active = true
GunmodsGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
GunmodsGUI.BorderSizePixel = 0
GunmodsGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
GunmodsGUI.Size = UDim2.new(0, 693, 0, 339)
GunmodsGUI.Visible = false
GunmodsGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
GunmodsGUI.ScrollBarThickness = 7

NoRecoil.Name = "No Recoil"
NoRecoil.Parent = GunmodsGUI
NoRecoil.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
NoRecoil.BorderSizePixel = 0
NoRecoil.Position = UDim2.new(0.32100001, 0, 0.00999999978, 0)
NoRecoil.Size = UDim2.new(0, 15, 0, 15)
NoRecoil.Font = Enum.Font.SourceSans
NoRecoil.Text = ""
NoRecoil.TextColor3 = Color3.fromRGB(0, 0, 0)
NoRecoil.TextSize = 14.000
NoRecoil.MouseButton1Down:connect(function()
	No_Recoil()
end)

UICorner_29.CornerRadius = UDim.new(0, 4)
UICorner_29.Parent = NoRecoil

TextLabel_30.Parent = GunmodsGUI
TextLabel_30.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_30.BackgroundTransparency = 1.000
TextLabel_30.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel_30.Size = UDim2.new(0, 72, 0, 50)
TextLabel_30.Font = Enum.Font.SourceSans
TextLabel_30.Text = "No Recoil"
TextLabel_30.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_30.TextSize = 14.000

TextLabel_31.Parent = GunmodsGUI
TextLabel_31.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_31.BackgroundTransparency = 1.000
TextLabel_31.Position = UDim2.new(0.0140724014, 0, 0.0224702358, 0)
TextLabel_31.Size = UDim2.new(0, 72, 0, 50)
TextLabel_31.Font = Enum.Font.SourceSans
TextLabel_31.Text = "Conbine Mags"
TextLabel_31.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_31.TextSize = 14.000

TextLabel_32.Parent = GunmodsGUI
TextLabel_32.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_32.BackgroundTransparency = 1.000
TextLabel_32.Position = UDim2.new(0.0140724014, 0, 0.0508793294, 0)
TextLabel_32.Size = UDim2.new(0, 72, 0, 50)
TextLabel_32.Font = Enum.Font.SourceSans
TextLabel_32.Text = "Fire Rate"
TextLabel_32.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_32.TextSize = 14.000

ConbineMags.Name = "Conbine Mags"
ConbineMags.Parent = GunmodsGUI
ConbineMags.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
ConbineMags.BorderSizePixel = 0
ConbineMags.Position = UDim2.new(0.32100001, 0, 0.0399999991, 0)
ConbineMags.Size = UDim2.new(0, 15, 0, 15)
ConbineMags.Font = Enum.Font.SourceSans
ConbineMags.Text = ""
ConbineMags.TextColor3 = Color3.fromRGB(0, 0, 0)
ConbineMags.TextSize = 14.000
ConbineMags.MouseButton1Down:connect(function()
	Conbine_Mags()
end)

UICorner_30.CornerRadius = UDim.new(0, 4)
UICorner_30.Parent = ConbineMags

FireRate.Name = "Fire Rate"
FireRate.Parent = GunmodsGUI
FireRate.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
FireRate.BorderSizePixel = 0
FireRate.Position = UDim2.new(0.32100001, 0, 0.0689999983, 0)
FireRate.Size = UDim2.new(0, 15, 0, 15)
FireRate.Font = Enum.Font.SourceSans
FireRate.Text = ""
FireRate.TextColor3 = Color3.fromRGB(0, 0, 0)
FireRate.TextSize = 14.000
FireRate.MouseButton1Down:connect(function()
	fire_rate()
end)

UICorner_31.CornerRadius = UDim.new(0, 4)
UICorner_31.Parent = FireRate

SettingsGUI.Name = "Settings GUI"
SettingsGUI.Parent = HyperX
SettingsGUI.Active = true
SettingsGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
SettingsGUI.BorderSizePixel = 0
SettingsGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
SettingsGUI.Size = UDim2.new(0, 693, 0, 339)
SettingsGUI.Visible = false
SettingsGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
SettingsGUI.ScrollBarThickness = 7

Lightdark.Name = "Light/dark"
Lightdark.Parent = SettingsGUI
Lightdark.BackgroundColor3 = Color3.fromRGB(39, 39, 39)
Lightdark.BorderSizePixel = 0
Lightdark.Position = UDim2.new(0.32100001, 0, 0.00999999978, 0)
Lightdark.Size = UDim2.new(0, 15, 0, 15)
Lightdark.Font = Enum.Font.SourceSans
Lightdark.Text = ""
Lightdark.TextColor3 = Color3.fromRGB(0, 0, 0)
Lightdark.TextSize = 14.000

UICorner_32.CornerRadius = UDim.new(0, 4)
UICorner_32.Parent = Lightdark

TextLabel_33.Parent = SettingsGUI
TextLabel_33.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_33.BackgroundTransparency = 1.000
TextLabel_33.Position = UDim2.new(0.0140724014, 0, -0.00688582659, 0)
TextLabel_33.Size = UDim2.new(0, 72, 0, 50)
TextLabel_33.Font = Enum.Font.SourceSans
TextLabel_33.Text = "Light/Dark"
TextLabel_33.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_33.TextSize = 14.000

TextLabel_34.Parent = SettingsGUI
TextLabel_34.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_34.BackgroundTransparency = 1.000
TextLabel_34.Position = UDim2.new(0.446972847, 0, -0.00783279352, 0)
TextLabel_34.Size = UDim2.new(0, 72, 0, 50)
TextLabel_34.Font = Enum.Font.SourceSans
TextLabel_34.Text = "<--- Dont work will be working soon"
TextLabel_34.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_34.TextSize = 14.000

InfoGUI.Name = "InfoGUI"
InfoGUI.Parent = HyperX
InfoGUI.Active = true
InfoGUI.BackgroundColor3 = Color3.fromRGB(61, 62, 62)
InfoGUI.BorderSizePixel = 0
InfoGUI.Position = UDim2.new(0.15487805, 0, 0.0198863633, 0)
InfoGUI.Size = UDim2.new(0, 693, 0, 339)
InfoGUI.CanvasSize = UDim2.new(0, 0, 3, 0)
InfoGUI.ScrollBarThickness = 7

TextLabel_35.Parent = InfoGUI
TextLabel_35.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_35.BackgroundTransparency = 1.000
TextLabel_35.Position = UDim2.new(0.0631344467, 0, -0.00688582659, 0)
TextLabel_35.Size = UDim2.new(0, 72, 0, 50)
TextLabel_35.Font = Enum.Font.SourceSans
TextLabel_35.Text = "Made by Mick Gordon#2226"
TextLabel_35.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_35.TextSize = 14.000

TextLabel_36.Parent = InfoGUI
TextLabel_36.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_36.BackgroundTransparency = 1.000
TextLabel_36.Position = UDim2.new(0.061691463, 0, 0.0224702358, 0)
TextLabel_36.Size = UDim2.new(0, 72, 0, 50)
TextLabel_36.Font = Enum.Font.SourceSans
TextLabel_36.Text = "Last update 0/4/2022 UK"
TextLabel_36.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_36.TextSize = 14.000

TextLabel_37.Parent = InfoGUI
TextLabel_37.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_37.BackgroundTransparency = 1.000
TextLabel_37.Position = UDim2.new(0.061691463, 0, 0.0546672046, 0)
TextLabel_37.Size = UDim2.new(0, 72, 0, 50)
TextLabel_37.Font = Enum.Font.SourceSans
TextLabel_37.Text = "Dm me discord for sever "
TextLabel_37.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_37.TextSize = 14.000

TextLabel_38.Parent = InfoGUI
TextLabel_38.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_38.BackgroundTransparency = 1.000
TextLabel_38.Position = UDim2.new(0.061691463, 0, 0.0849702358, 0)
TextLabel_38.Size = UDim2.new(0, 72, 0, 50)
TextLabel_38.Font = Enum.Font.SourceSans
TextLabel_38.Text = "Looking for coders to help me"
TextLabel_38.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_38.TextSize = 14.000

TextLabel_39.Parent = InfoGUI
TextLabel_39.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_39.BackgroundTransparency = 1.000
TextLabel_39.Position = UDim2.new(0.807723165, 0, 0.28478083, 0)
TextLabel_39.Size = UDim2.new(0, 72, 0, 50)
TextLabel_39.Font = Enum.Font.SourceSans
TextLabel_39.Text = "Thanks for yousing this script "
TextLabel_39.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_39.TextSize = 14.000

TextLabel_40.Parent = HyperX
TextLabel_40.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextLabel_40.BackgroundTransparency = 1.000
TextLabel_40.Position = UDim2.new(0.853378356, 0, -0.00120401382, 0)
TextLabel_40.Size = UDim2.new(0, 72, 0, 25)
TextLabel_40.Font = Enum.Font.Roboto
TextLabel_40.Text = "Hyper X │Free beta version"
TextLabel_40.TextColor3 = Color3.fromRGB(16, 180, 160)
TextLabel_40.TextSize = 12.000

-- Scripts:

local function SXKS_fake_script() -- PFHyperX12022022.LocalScript 
	local script = Instance.new('LocalScript', PFHyperX12022022)

	local Player = game.Players.LocalPlayer
	local Mouse = Player:GetMouse()
	local CoolFrame = script.Parent["RGBMain"]
	
	Mouse.KeyDown:Connect(function(Key)
		Key = Key:lower()
		if Key == 'j' then
			if CoolFrame.Visible == false then
				CoolFrame.Visible = true 
			else
				CoolFrame.Visible = false
			end
		end
	end)
	
end
coroutine.wrap(SXKS_fake_script)()
local function NLAHE_fake_script() -- RGBMain.LocalScript 
	local script = Instance.new('LocalScript', RGBMain)

	local RS = game:GetService("RunService")
	
	local rainbow = script.Parent  -- GUI object
	local grad = rainbow.UIGradient
	
	local counter = 0       -- phase shift 
	local w = math.pi / 12  -- frequency 
	local CS = {}           -- colorsequence table
	local num = 15 			-- number of colorsequence points (maxed at 15 or 16 I believe)
	local frames = 0		-- frame counter, used to buffer if you want lower framerate updates
	
	RS.Heartbeat:Connect(function()
		if math.fmod(frames, 2) == 0 then
			for i = 0, num do
				local c = Color3.fromRGB(127 * math.sin(w*i + counter) + 128, 127 * math.sin(w*i + 2 * math.pi/3 + counter) + 128, 127*math.sin(w*i + 4*math.pi/3 + counter) + 128)
				table.insert(CS, i+1, ColorSequenceKeypoint.new(i/num, c))
			end
			grad.Color = ColorSequence.new(CS)
			CS = {}
			counter = counter + math.pi/40
			if (counter >= math.pi * 2) then
				counter = 0
			end
		end
		if frames >= 1000 then
			frames = 0
		end
		frames = frames + 1
	end)
	
end
coroutine.wrap(NLAHE_fake_script)()
local function FKWPYUR_fake_script() -- Gradient.LocalScript 
	local script = Instance.new('LocalScript', Gradient)

	local RS = game:GetService("RunService")
	
	local rainbow = script.Parent  -- GUI object
	local grad = rainbow.UIGradient
	
	local counter = 0       -- phase shift 
	local w = math.pi / 12  -- frequency 
	local CS = {}           -- colorsequence table
	local num = 15 			-- number of colorsequence points (maxed at 15 or 16 I believe)
	local frames = 0		-- frame counter, used to buffer if you want lower framerate updates
	
	RS.Heartbeat:Connect(function()
		if math.fmod(frames, 2) == 0 then
			for i = 0, num do
				local c = Color3.fromRGB(127 * math.sin(w*i + counter) + 128, 127 * math.sin(w*i + 2 * math.pi/3 + counter) + 128, 127*math.sin(w*i + 4*math.pi/3 + counter) + 128)
				table.insert(CS, i+1, ColorSequenceKeypoint.new(i/num, c))
			end
			grad.Color = ColorSequence.new(CS)
			CS = {}
			counter = counter + math.pi/40
			if (counter >= math.pi * 2) then
				counter = 0
			end
		end
		if frames >= 1000 then
			frames = 0
		end
		frames = frames + 1
	end)
	
end
coroutine.wrap(FKWPYUR_fake_script)()


------------------------------------------------------------------------------------------------------------

-- Textures for the word gui

function Neon_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Neon"
		end
	end
end

function forcefield_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "ForceField"
		end
	end
end

function Wood_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Wood"
		end
	end
end

function Smoothplastic_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "SmoothPlastic"
		end
	end
end

function Dimondplate_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "DiamondPlate"
		end
	end
end

function Marbel_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Marble"
		end
	end
end

function Glass_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Glass"
		end
	end
end

function Metal_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Metal"
		end
	end
end

function Grass_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Grass"
		end
	end
end

function Cobbelstone_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Cobblestone"
		end
	end
end

function Ice_hhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Ice"
		end
	end
end

function Granite_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Granite"
		end
	end
end

function Sand_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Sand"
		end
	end
end

function Slate_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Slate"
		end
	end
end

function Concrete_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Material = "Concrete"
		end
	end
end

------------------------------------------------------------------------------------------------------------------

--This is the colors for word editing

function Red_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(255, 0, 0)
		end
	end
end

function Blue_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(0, 0, 255)
		end
	end
end

function Pink_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(255, 53, 184)
		end
	end
end

function purple_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(120, 81, 169)
		end
	end
end

function cyan_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(0, 255, 255)
		end
	end
end

function white_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(255, 255, 255)
		end
	end
end

function Black_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(0, 0, 0)
		end
	end
end

function Matteblack_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(23, 23, 23)
		end
	end
end

function Gray_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(127, 127, 127)
		end
	end
end

function Green_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(119, 221, 119)
		end
	end
end

function lightblue_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(173, 216, 230)
		end
	end
end

function brown_hhhp_ ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(115, 103, 60)
		end
	end
end

function Orange_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(255, 153, 0)
		end
	end
end

function Turquoise_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(64, 224, 208)
		end
	end
end

function Yello_hhhp ()
	for I,V in pairs(workspace:GetDescendants()) do
		if (V.ClassName == "Part" or V.ClassName == "WedgePart") then
			V.Color = Color3.fromRGB(255, 255, 0)
		end
	end
end

-- End of the world 
----------------------------------------------------------------------------------------
-- Settings
