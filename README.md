--[[
    TSB M1 打击特效脚本 v23.0 —— 可打假人 + 语言菜单
    ------------------------------------------------------------
    ★ v23.1 紧急修复：脚本加载就崩（一扫到假人就挂）
      原因：scanDummies 定义在 setupTarget 之前，里面调用 setupTarget 时
            local setupTarget 还没声明 → Lua 当成全局变量找 → 是 nil
            → attempt to call a nil value  → 整个脚本挂掉
      修法：在假人模块开头加前向声明 `local setupTarget`，
            并把 `local function setupTarget(...)` 改成 `setupTarget = function(...)`
      （上一版测试没抓到，是因为测试环境的 GetDescendants 返回空表，
        循环体没跑到；这次补了"返回真实假人"的用例才暴露出来）

    ★ v23.2：Ctrl + 任意键 重绑定 + 选完语言的按键提示弹幕 + 夜神月大笑播放器
      · Shift+任意键 改成 Ctrl+任意键（你说 Shift 没反应）
        现在可重绑定的是「切换风格」键，默认 G
      · 选完语言后 0.55s 弹一条所选语言的提示弹幕：
        默认按键是什么 + 怎么绑定；弹完就没了
        中文：默认按键 G 切换特效 ｜ 绑定：Ctrl + 任意键
        EN  ：Default key G to switch effects | Rebind: Ctrl + any key
        VI  ：Phím mặc định G để đổi hiệu ứng | Gán lại: Ctrl + phím bất kỳ
      · 只保留夜神月大笑。搜到 6 个 ID，做成播放器：
          H = 依次试听（第1个 → 第2个 → …… 循环）
          J = 把刚听到的那个「锁定」为正式音效
        选完语言的彩蛋优先播放锁定的那个，没锁定就用第 1 个
        ⚠️ Roblox 音频随时会失效/私密化，所以要挨个试
      · 数字键会显示成 1/2/3 而不是 One/Two/Three；Ctrl 显示 L-Ctrl

    ★ v23 改动一：可打假人（默认开，CONFIG.HIT_DUMMY）
      · TSB 的假人不在 Players 里，是 workspace 下的普通 Model（Weakest Dummy 等）
      · 从 workspace 扫描，命中/击杀判定完全复用现有的 setupTarget
        （它只依赖 Humanoid + HumanoidRootPart，对假人一样适用，不用另写一套）
      · 一个独立心跳每 1 秒重扫一次：假人被打死会重生，重生后是新 Model 要重新挂
      · workspace.DescendantAdded：后来生成 / 重生的假人立刻挂上，不用等扫描
      · 按你的说明，不加"站着不动 10~20 秒自动回满血"的冷却——
        正常打的时候一直在输出，不存在干等回血的情况

    ★ v23 改动二：开场弹幕 + 语言选择菜单
      · 执行时先滚一条弹幕：script by golden
      · 接着一个面板从屏幕下方平滑升到正中（0.9s，Quart 缓出）
      · 菜单标题：Choose Language（英文菜单）
      · 选项按你的要求用英文标注：Chinese / English / Vietnamese
        （按钮主文字是英文，后面括号里带一小行母语：中文 / English / Tiếng Việt）
      · 选完后弹一条对应语言的确认弹幕，菜单再平滑滑回去
      · 选中的语言会作用于之后所有提示（风格切换、三种音效试听、假人扫描数量）

    ★ v24：移动端适配（PC 体验完全不变）
      · 平台检测：IS_MOBILE = 有触摸 且 无键盘 → 手机 / 平板
      · 手机没有键盘，G/H/R/T/Y 全按不了，所以给一套屏幕虚拟按钮：
        风格 / 夜神月 / 普攻音 / 终结音 / 技能音 / 假人开关
        面板可拖动，避免挡住战斗视野
      · UI 自适应：弹幕宽 0.94 屏、语言菜单 0.86 屏、字号 ×0.85（PC 保持像素原值）
      · 性能降级：特效尺寸 ×0.8、线条 ×0.8、震动 ×0.7、FOV ×0.7
      · PC 上 IS_MOBILE=false，虚拟按钮那段根本不执行，一点影响都没有

    v22 内容（保留）：取消普攻黑闪+音效、ShockRing 替换廉价实心圆盘 Ring
    v21 内容（保留）：击杀归因 classifyKill 三条路径、pcall 错误不再静默吞掉
    v20 内容（保留）：虎杖线条 thick=1.9、黑闪六段演出
    ------------------------------------------------------------
    键位（PC）：G 切风格 / R 听普攻音 / T 听终结音 / Y 听一技能击杀音
                H 试听夜神月大笑
                Ctrl + 任意键 = 重新绑定「切换风格」键（默认 G）
    手机：自动启用屏幕虚拟按钮面板（可拖动），无需键盘
    开关：CONFIG.HIT_DUMMY = false 可关掉打假人
]]

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RS = game:GetService("RunService")
local TS = game:GetService("TweenService")
local SS = game:GetService("SoundService")
local Camera = workspace.CurrentCamera
local player = Players.LocalPlayer

local CONFIG = {
	EFFECT_SIZE = 3,
	LINE_THICK = 2.8,        -- 全局线条粗细倍率
	MIN_LINE_WIDTH = 0.07,   -- 最细线条保底宽度
	BURST_SIZE_SCALE = 1.35, -- 粒子体积倍率

	LINE_LIFE = 1,
	ATTACK_LOCK_TIME = 0.55,
	SWING_GRACE = 0.15,
	HIT_THROTTLE = 0.05,
	LOCK_RANGE = 30,
	LOCK_ANGLE_DOT = 0.3,
	SOUND_VOLUME = 0.7,
	SOUND_COOLDOWN = 0.05,
	COMBO_RESET_TIME = 1.5,
	NOTICE_DURATION = 1.2,
	SHAKE_ENABLED = true,
	SHAKE_SCALE = 0.7,
	FOV_KICK = 1.2,

	-- ===== 击杀判定（v21 修复：原来只认 M1 挥击，用技能打死完全不触发）=====
	SKILL1_KEY = Enum.KeyCode.One,  -- 一技能键（TSB 一般是 1；不是就改这里）
	SKILL1_WINDOW = 2.0,           -- 按了 1 之后多少秒内击杀算「一技能击杀」
	KILL_CREDIT_TIME = 2.5,        -- 我最近打过他多少秒内他死了 → 算我杀的
	SKILL_LOCK_RANGE = 45,         -- 技能击杀的判定距离（比普攻远）

	-- ===== 夜神月大笑 ID 播放器（v23 新增）=====
	-- 按 H 依次试听下面每一个：播第 1 个 → 再按 H 播第 2 个 → ……循环
	-- 找到能响的那个后，按 J 把它「锁定」为正式使用的音效。
	KIRA_VOLUME = 1.0,

	-- ===== 可打假人（v23 新增）=====
	HIT_DUMMY        = true,   -- 是否把假人也当成命中/击杀目标（默认开）
	DUMMY_SCAN_EVERY = 1.0,    -- 每隔几秒重扫一次假人（假人被打死会重生）
	-- 注：按你说的，不用处理「站着不动 10~20 秒自动回满血」的冷却——
	--     正常打的时候一直在输出，不存在干等回血的情况，故不加额外冷却逻辑。
}

-- ===================== 移动端适配（v24 新增）=====================
-- 平台检测：有触摸且没键盘 = 手机 / 平板（PC 模拟器通常仍带键盘，会判成 PC，属正常）
local IS_MOBILE = UIS.TouchEnabled and not UIS.KeyboardEnabled

-- 手机性能有限，特效密度自动降一档（PC 完全不受影响）
if IS_MOBILE then
	CONFIG.EFFECT_SIZE   = CONFIG.EFFECT_SIZE   * 0.8
	CONFIG.LINE_THICK    = CONFIG.LINE_THICK    * 0.8
	CONFIG.SHAKE_ENABLED = true
	CONFIG.SHAKE_SCALE   = CONFIG.SHAKE_SCALE   * 0.7
	CONFIG.FOV_KICK      = CONFIG.FOV_KICK      * 0.7
end

-- 手机屏窄，UI 尺寸走比例；PC 保持原像素尺寸
local function uiW(px)
	if IS_MOBILE then return UDim2.new(0.92, 0, 0, px) end
	return UDim2.new(0, px, 0, 44)
end
local function uiText(px)
	if IS_MOBILE then return math.floor(px * 0.85) end
	return px
end

-- 风格级粗细倍率：每个风格可在 EFFECTS 里写 thick = N，单独再加粗/收细
-- fireEffect 触发时会写入这里；特效函数（含 task.delay 的异步部分）都读它
local styleThick = 1
local function withThick(n)
	local old = styleThick
	styleThick = n or 1
	return old
end

--[[
    音效字段说明：
    sound          = M1~M3 普攻音效
    soundCut       = 普攻音效播放多少秒后淡出（nil = 放完）
    soundFinisher  = M4 / 击杀 才播放的音效
    finisherCut    = 终结音效截断时间
]]
----------------------------------------------------------------------
-- UWU 音效备胎表（Roblox 音频随时可能被私密化/下架，哑了就换一个）
-- 想改音效：把下面 UWU_IDS 里的数字填到 cat_girl 的 sound / soundFinisher 即可
----------------------------------------------------------------------
local UWU_IDS = {
	main      = "8323804973", -- UwU 语音（短促，打击感最好，首选）
	alt1      = "8679659744", -- uwu 语音（另一版）
	alt2      = "8516511704", -- UwU Sound effect（boombox 向，稍长）
	song_uwu  = "6460148774", -- UwU（chevy 的歌，整首，太长只适合当终结/彩蛋）
	song_meme = "5979060667", -- Submerged Meme UwU（meme 歌版）
	-- 注：senzawa 那版 "UwU (memes)" 8391724462 已因版权下架，不要再用
}

local EFFECTS = {
	{
		name = "宿傩斩特效",
		key = "sukuna_slash",
		sound = "7122613461",
	},
	{
		name = "TF2暴击特效",
		key = "tf2_crit",
		sound = "8255306220",
	},
	{
		name = "边狱巴士破碎",
		key = "limbus_shatter",
		sound = "17731296797",
	},
	{
		name = "猫娘打击",
		key = "cat_girl",
		-- 普攻 M1~M4：用原来终结那条 UwU（0.5 秒截断，一声就完，不拖尾）
		sound = UWU_IDS.alt1, -- "8679659744"；想换回另一条就写 UWU_IDS.main
		soundCut = 0.5,
		-- 终结（击杀）：不要音效了，保持 nil 即可
		soundFinisher = nil,
		finisherCut = nil,
		volume = 1, -- UwU 原声偏小，单独拉满
	},
	{
		name = "爱心打击",
		key = "love_heart",
		-- 朋友指定、确认有效的音效
		sound = "7851649484",
		soundCut = nil, -- 完整播放；想让它短一点就改成 0.5
		soundFinisher = nil,
		finisherCut = nil,
		volume = 1,
	},
	{
		name = "虎杖打击",
		key = "yuji",
		-- 虎杖线条单独再加粗（在全局 2.8 的基础上再乘 1.9 ≈ 原版的 5.3 倍）
		thick = 1.9,
		-- 每个段位各配一个音效（虎杖五段都不同，取自 JJS Vessel 的招式音）
		perDirection = {
			"4571259077",  -- M1 直拳 Fist
			"8595975878",  -- M2 上勾拳 M1 Hit
			"9125615451",  -- M3 卍踢 Manji Kick Startup
			"5795505380",  -- M4 径庭拳 Divergent Fist
		},
		-- 普攻击杀：按需求已取消（不放黑闪、不放音效）
		soundFinisher = nil,
		-- （原日语「黒閃」18466149472 已停用；要恢复就把它填回这一行）
		-- 一技能击杀：另一条虎杖喊「黒閃」的日语语音（更激烈那版）
		-- ⚠️ 公开资料查不到明确标注"愤怒版"的独立 ID，这里先用日语版 81528757439622
		--    你自己找到更合适的直接改这一行数字即可（按 Y 可试听）
		soundSkill1 = "81528757439622",
		-- 备胎：12764933067（黑闪火花）/ 9114314398（黑闪）/ 13792648554
		finisherCut = nil,
		volume = 1,
	},
}

local effectFolder = workspace:FindFirstChild("_M1Effects")
local soundFolder = SS:FindFirstChild("_M1Sounds")

local function ensureFolders()
	if not effectFolder or not effectFolder.Parent then
		effectFolder = Instance.new("Folder")
		effectFolder.Name = "_M1Effects"
		effectFolder.Parent = workspace
	end
	if not soundFolder or not soundFolder.Parent then
		soundFolder = Instance.new("Folder")
		soundFolder.Name = "_M1Sounds"
		soundFolder.Parent = SS
	end
end
ensureFolders()

local soundPool, POOL_SIZE, poolIndex = {}, 5, 1
for i = 1, POOL_SIZE do
	local s = Instance.new("Sound")
	s.Name = "M1Sound_" .. i
	s.Volume = CONFIG.SOUND_VOLUME
	s.Parent = soundFolder
	soundPool[i] = s
end

local lastSoundTime = 0
-- cutAfter：多少秒后淡出并停止（用来让"喵"一声就干净结束）
local function playSound(id, volume, cutAfter)
	if not id then return end
	if tick() - lastSoundTime < CONFIG.SOUND_COOLDOWN then return end
	lastSoundTime = tick()
	ensureFolders()
	local s = soundPool[poolIndex]
	poolIndex = (poolIndex % POOL_SIZE) + 1
	if not s then return end
	local vol = volume or CONFIG.SOUND_VOLUME
	s.SoundId = "rbxassetid://" .. id
	s.Volume = vol
	s:Play()
	if cutAfter and cutAfter > 0 then
		local token = tostring(tick()) .. tostring(math.random(1, 1e6))
		s:SetAttribute("_cutToken", token)
		task.delay(cutAfter, function()
			if s:GetAttribute("_cutToken") ~= token then return end
			for i = 1, 5 do
				s.Volume = vol * (1 - i / 5)
				task.wait(0.03)
			end
			if s:GetAttribute("_cutToken") == token then
				s:Stop()
				s.Volume = vol
			end
		end)
	end
end

local function pickSound(v)
	if not v then return nil end
	if type(v) == "string" then return v end
	if #v == 0 then return nil end
	return v[math.random(1, #v)]
end

local Ease = {}
Ease.linear = function(t) return t end
Ease.inQuad = function(t) return t * t end
Ease.outQuad = function(t) return 1 - (1 - t) * (1 - t) end
Ease.inCubic = function(t) return t * t * t end
Ease.outCubic = function(t) return 1 - (1 - t) ^ 3 end
Ease.inOutQuad = function(t) return (t < 0.5) and (2 * t * t) or (1 - ((-2 * t + 2) ^ 2) / 2) end
Ease.outExpo = function(t) return (t >= 1) and 1 or (1 - 2 ^ (-9 * t)) end
Ease.outBack = function(t) local c = 1.9; return 1 + (c + 1) * (t - 1) ^ 3 + c * (t - 1) ^ 2 end

local anims = {}
local function Anim(dur, ease, update, onDone)
	local a = { t = 0, d = math.max(dur, 0.016), e = Ease[ease] or Ease.outQuad, u = update, c = onDone }
	anims[#anims + 1] = a
	return a
end
RS.Heartbeat:Connect(function(dt)
	for i = #anims, 1, -1 do
		local a = anims[i]
		a.t = a.t + dt
		if a.t >= a.d then
			if a.u then a.u(1) end
			table.remove(anims, i)
			if a.c then a.c() end
		else
			if a.u then a.u(a.e(a.t / a.d)) end
		end
	end
end)

local pool = {}
local function getPart()
	local p = table.remove(pool)
	if not p then
		p = Instance.new("Part")
		p.Anchored = true
		p.CanCollide = false
		p.CastShadow = false
	end
	p.Parent = effectFolder
	p.Shape = Enum.PartType.Block
	p.Material = Enum.Material.Neon
	p.Color = Color3.new(1, 1, 1)
	p.Transparency = 1
	p.Size = Vector3.new(0.2, 0.2, 0.2)
	p.CFrame = CFrame.new()
	return p
end
local function freePart(p)
	if not p or p.Parent == nil then return end
	for _, c in ipairs(p:GetChildren()) do c:Destroy() end
	p.Parent = nil
	if #pool < 400 then pool[#pool + 1] = p end
end

local function faceCamCF(pos)
	local dir = Camera.CFrame.Position - pos
	if dir.Magnitude < 0.01 then dir = Vector3.new(0, 0, 1) end
	return CFrame.lookAt(pos, pos + dir.Unit)
end

-- 基础线条（其他三种风格用）
local function Slash(o)
	local p = getPart()
	p.Material = o.material or Enum.Material.Neon
	p.Color = o.color
	local L = o.len or (CONFIG.EFFECT_SIZE * 3.5)
	local th = (o.thick or 0.14) * CONFIG.LINE_THICK * styleThick
	local depth = o.depth or (th * 0.85)
	local life = (o.life or 0.24) * CONFIG.LINE_LIFE
	local sweep = o.sweep or 0
	local sDir = o.sweepDir or 1
	local t0 = o.transparency or 0
	local baseCF = faceCamCF(o.pos) * CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0) * CFrame.Angles(0, 0, math.rad(90 + (o.angle or 0)))
	local minW = CONFIG.MIN_LINE_WIDTH
	local function run()
		Anim(life, o.ease or "outExpo", function(e)
			local g = math.min(e / 0.2, 1)
			local len = L * (0.28 + 0.72 * (1 - (1 - g) ^ 3))
			local off = sweep * Ease.outQuad(e) * sDir
			local w = math.max(th, minW) * (1 - e) ^ 1.35
			local d = math.max(depth, minW * 0.8) * (1 - e) ^ 1.35
			p.Size = Vector3.new(math.max(w, 0.012), len, math.max(d, 0.012))
			p.CFrame = baseCF * CFrame.new(0, off, 0)
			p.Transparency = t0 + (1 - t0) * (e ^ 1.25)
		end, function() freePart(p) end)
	end
	if (o.delay or 0) > 0 then task.delay(o.delay, run) else run() end
end

local function Orb(o)
	local p = getPart()
	p.Shape = Enum.PartType.Ball
	p.Material = o.material or Enum.Material.Neon
	p.Color = o.color
	local s0, s1 = o.size or 0.8, o.sizeEnd or 2.2
	local t0, t1 = o.transparency or 0.05, o.transparencyEnd or 1
	local pos, drift = o.pos, o.drift or Vector3.zero
	local function run()
		Anim((o.life or 0.2) * CONFIG.LINE_LIFE, o.ease or "outQuad", function(e)
			local s = s0 + (s1 - s0) * e
			p.Size = Vector3.new(s, s, s)
			p.CFrame = CFrame.new(pos + drift * e)
			p.Transparency = t0 + (t1 - t0) * e
		end, function() freePart(p) end)
	end
	if (o.delay or 0) > 0 then task.delay(o.delay, run) else run() end
	return p
end

local function Ring(o)
	local p = getPart()
	p.Shape = Enum.PartType.Cylinder
	p.Material = o.material or Enum.Material.Neon
	p.Color = o.color
	local r0, r1 = o.radius or 0.5, o.radiusEnd or 4
	local th = (o.thickness or 0.3) * CONFIG.LINE_THICK * styleThick
	local t0, t1 = o.transparency or 0.2, 1
	local baseCF = faceCamCF(o.pos) * CFrame.Angles(math.rad(90), 0, 0)
	local function run()
		Anim((o.life or 0.3) * CONFIG.LINE_LIFE, o.ease or "outCubic", function(e)
			local r = r0 + (r1 - r0) * e
			local w = math.max(th, CONFIG.MIN_LINE_WIDTH * 1.5) * (1 - e * 0.9)
			p.Size = Vector3.new(math.max(w, 0.012), r, r)
			p.CFrame = baseCF
			p.Transparency = t0 + (t1 - t0) * e
		end, function() freePart(p) end)
	end
	if (o.delay or 0) > 0 then task.delay(o.delay, run) else run() end
end

local function Rays(o)
	local n = o.count or 6
	for i = 1, n do
		Slash{
			pos = o.pos,
			angle = (o.startAngle or 0) + (i - 1) * (360 / n),
			len = (o.len or CONFIG.EFFECT_SIZE * 2.4) * (o.rand and (0.75 + math.random() * 0.5) or 1),
			thick = o.thick or 0.1,
			color = (o.color2 and (i % 2 == 0)) and o.color2 or o.color,
			transparency = o.transparency or 0,
			life = o.life or 0.22,
			delay = o.delay or 0,
			ease = o.ease or "outExpo",
		}
	end
end

local function Burst(o)
	local host = getPart()
	host.Size = Vector3.new(0.1, 0.1, 0.1)
	host.Transparency = 1
	host.CFrame = CFrame.new(o.pos)
	local life = o.life or 0.25
	local spd = o.speed or 18
	local pe = Instance.new("ParticleEmitter")
	pe.Texture = o.texture or "rbxasset://textures/particles/sparkles_main.dds"
	pe.Color = o.color2 and ColorSequence.new(o.color, o.color2) or ColorSequence.new(o.color)
	pe.Lifetime = NumberRange.new(life, life * 1.7)
	pe.Speed = NumberRange.new(spd, spd * (o.speedVar or 2))
	pe.SpreadAngle = Vector2.new(180, 180)
	pe.Size = NumberSequence.new((o.size or 0.35) * CONFIG.BURST_SIZE_SCALE, 0)
	pe.LightEmission = o.emission or 1
	pe.LightInfluence = 0
	pe.Rate = 0
	pe.Parent = host
	pe:Emit(o.count or 20)
	task.delay(life * 1.8 + 0.3, function() freePart(host) end)
end

local function Shards(o)
	local count = o.count or 12
	local baseCF = faceCamCF(o.pos)
	for i = 1, count do
		local p = getPart()
		p.Material = o.material or Enum.Material.Slate
		p.Color = (o.color2 and (i % 2 == 0)) and o.color2 or o.color
		local ang = math.rad((i - 1) * (360 / count) + math.random(-12, 12))
		local dirV = baseCF * Vector3.new(math.cos(ang), math.sin(ang), (math.random() - 0.5) * 0.6)
		local dir = (dirV - o.pos).Unit
		local dist = (o.dist or 4) * (0.6 + math.random() * 0.7)
		local sz = o.size or 0.45
		local l = sz * (1.4 + math.random() * 0.8)
		local function run()
			Anim(o.life or 0.4, o.ease or "outQuad", function(e)
				p.Size = Vector3.new(sz * (1 - e * 0.5), sz * (1 - e * 0.5), l)
				p.CFrame = CFrame.new(o.pos + dir * (o.inDist or 1.2) + dir * (dist - (o.inDist or 1.2)) * e) * CFrame.Angles(e * 3, e * 3, e * 2)
				p.Transparency = (o.transparency or 0.1) + (1 - (o.transparency or 0.1)) * e
			end, function() freePart(p) end)
		end
		if (o.delay or 0) > 0 then task.delay(o.delay, run) else run() end
	end
end

local function Impact(o)
	local R = CONFIG.EFFECT_SIZE
	local c1, c2 = o.color, o.color2 or o.color
	local n = o.rays or 4
	for i = 1, n do
		local ang = (o.rayAngle or 0) + (i - 1) * (360 / n) + math.random(-10, 10)
		Slash{
			pos = o.pos, angle = ang, len = (o.rayLen or R * 1.1) * (0.7 + math.random() * 0.6),
			thick = o.rayThick or 0.09, color = (i % 2 == 0) and c2 or c1,
			life = (o.life or 0.18), transparency = 0, delay = o.delay or 0,
		}
	end
	if o.core ~= false then
		Orb{
			pos = o.pos, size = o.coreSize or 0.45, sizeEnd = (o.coreSize or 0.45) * 5.5,
			color = o.coreColor or c2, transparency = 0, life = o.coreLife or 0.16, delay = o.delay or 0,
		}
	end
end

-- ============================================================
--                    猫娘专用构件（原创）
-- ============================================================

-- 弧形猫爪痕：一条抛物线拱形的爪痕，由若干短段拼成，逐段延迟出现
-- 模拟爪子从一端划到另一端的动态感（不是一根直线）
local function ClawArc(o)
	local n = o.claws or 3
	local L = o.len or CONFIG.EFFECT_SIZE * 2.6
	local curve = o.curve or 0.42          -- 拱起程度（占长度比例）
	local segs = o.segs or 9               -- 每道爪痕的分段数
	local th = (o.thick or 0.13) * CONFIG.LINE_THICK * styleThick
	local gap = o.gap or 0.55              -- 爪与爪的间距
	local spread = o.spread or 5           -- 爪之间的扇形张角
	local life = o.life or 0.28
	local sweepT = o.sweepTime or 0.08     -- 划过整条痕所需时间
	local sd = o.sweepDir or 1
	local t0 = o.transparency or 0
	local d0 = o.delay or 0
	local minW = CONFIG.MIN_LINE_WIDTH
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))

	for c = 1, n do
		local k = c - (n + 1) / 2
		local cLen = L * (1 - math.abs(k) * (o.lenFalloff or 0.1))
		local yOff = gap * k
		local tilt = math.rad(k * spread)
		local segLen = cLen / segs * 1.45
		for s = 1, segs do
			local t = (s - 1) / (segs - 1)
			local u = (t - 0.5) * 2 * sd
			local x = u * cLen * 0.5
			local y = curve * cLen * (1 - u * u) * 0.5 + yOff
			local tang = math.atan2(-1, -2 * curve * u)   -- 该点切线角
			local w = math.max(th, minW) * (1 - t * 0.5)  -- 起端粗、尾端细
			local seg = getPart()
			seg.Material = o.material or Enum.Material.Neon
			seg.Color = o.color
			local cf = base * CFrame.new(x, y, 0) * CFrame.Angles(0, 0, tang + tilt)
			local function run()
				Anim(life, o.ease or "outQuad", function(e)
					local ww = w * (1 - e) ^ 1.1
					seg.Size = Vector3.new(math.max(ww, 0.012), segLen, math.max(ww * 0.9, 0.012))
					seg.CFrame = cf
					seg.Transparency = t0 + (1 - t0) * (e ^ 1.2)
				end, function() freePart(seg) end)
			end
			local dly = d0 + t * sweepT
			if dly > 0 then task.delay(dly, run) else run() end
		end
	end
end

-- 爪尖亮点：沿弧线扫过的光点，是"挥爪"的动感来源
local function ClawTip(o)
	local L = o.len or CONFIG.EFFECT_SIZE * 2.6
	local curve = o.curve or 0.42
	local sd = o.sweepDir or 1
	local s0 = o.size or 0.55
	local d0 = o.delay or 0
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))
	local p = getPart()
	p.Shape = Enum.PartType.Ball
	p.Material = Enum.Material.Neon
	p.Color = o.color
	local dur = o.time or 0.09
	local function run()
		Anim(dur, "linear", function(e)
			local u = (e - 0.5) * 2 * sd
			local x = u * L * 0.5
			local y = curve * L * (1 - u * u) * 0.5
			local s = s0 * (1 - e * 0.55)
			p.Size = Vector3.new(s, s, s)
			p.CFrame = base * CFrame.new(x, y, 0)
			p.Transparency = (o.transparency or 0) + (1 - (o.transparency or 0)) * (e ^ 1.6)
		end, function() freePart(p) end)
	end
	if d0 > 0 then task.delay(d0, run) else run() end
end

-- 猫爪印：一个大肉球 + 四个小脚趾
local function PawPrint(o)
	local s = o.size or 0.55
	local life = o.life or 0.3
	local t0 = o.transparency or 0.04
	local d = o.delay or 0
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))
	Orb{
		pos = (base * CFrame.new(0, -s * 0.5, 0)).Position,
		size = s * 0.85, sizeEnd = s * 1.28,
		color = o.color, transparency = t0, life = life, delay = d,
		material = o.material,
	}
	local toes = o.toes or 4
	for i = 1, toes do
		local k = i - (toes + 1) / 2
		Orb{
			pos = (base * CFrame.new(k * s * 0.66, s * (0.42 + math.abs(k) * 0.06), 0)).Position,
			size = s * 0.33, sizeEnd = s * 0.5,
			color = o.color2 or o.color, transparency = t0,
			life = life * 0.85, delay = d + 0.025 * i,
			material = o.material,
		}
	end
end

-- 猫耳：用两条斜线勾出尖朝上的三角轮廓 + 内层小三角
local function CatEars(o)
	local s = o.size or 1
	local w = o.width or (s * 0.95)
	local h = o.height or (s * 1.15)
	local alpha = math.deg(math.atan((w / 2) / h))
	local L = math.sqrt((w / 2) ^ 2 + h ^ 2)
	local gapX = o.gap or (s * 1.0)
	local cy = o.offsetY or (s * 0.95)
	local th = o.thick or 0.09
	local life = o.life or 0.26
	local d = o.delay or 0
	for _, sgn in ipairs({ -1, 1 }) do
		-- 外轮廓：左斜边 / 右斜边
		Slash{pos=o.pos, angle=90 - alpha, offsetX=sgn*gapX - w*0.25, offsetY=cy, len=L, thick=th,
			color=o.color, transparency=o.transparency or 0, life=life, delay=d, ease="outExpo"}
		Slash{pos=o.pos, angle=90 + alpha, offsetX=sgn*gapX + w*0.25, offsetY=cy, len=L, thick=th,
			color=o.color, transparency=o.transparency or 0, life=life, delay=d + 0.012, ease="outExpo"}
		if o.inner ~= false then
			-- 内耳（粉色小三角）
			local iw, ih = w * 0.5, h * 0.55
			local ia = math.deg(math.atan((iw / 2) / ih))
			local iL = math.sqrt((iw / 2) ^ 2 + ih ^ 2)
			Slash{pos=o.pos, angle=90 - ia, offsetX=sgn*gapX - iw*0.25, offsetY=cy - h*0.12, len=iL, thick=th*0.75,
				color=o.color2 or o.color, transparency=0.05, life=life*0.85, delay=d + 0.03, ease="outExpo"}
			Slash{pos=o.pos, angle=90 + ia, offsetX=sgn*gapX + iw*0.25, offsetY=cy - h*0.12, len=iL, thick=th*0.75,
				color=o.color2 or o.color, transparency=0.05, life=life*0.85, delay=d + 0.042, ease="outExpo"}
		end
	end
end

-- 胡须：两侧各三条细线向外扇形发散
local function Whiskers(o)
	local n = o.count or 3
	local L = o.len or 1.75
	local fan = o.fan or 18
	local gapy = o.gap or 0.3
	local offx = o.offsetX or 0.75
	local col = o.color
	for _, sgn in ipairs({ -1, 1 }) do
		for i = 1, n do
			local k = i - (n + 1) / 2
			Slash{
				pos = o.pos,
				angle = -sgn * k * fan,
				offsetX = sgn * offx,
				offsetY = k * gapy + (o.offsetY or 0),
				len = L * (1 - math.abs(k) * 0.1),
				thick = o.thick or 0.045,
				color = col, transparency = o.transparency or 0.08,
				life = o.life or 0.2,
				delay = (o.delay or 0) + math.abs(k) * 0.012,
				ease = "outExpo",
			}
		end
	end
end

-- 铃铛：金球 + 下缘缝 + 光环 + 十字闪光
local function Bell(o)
	local s = o.size or 0.55
	local d = o.delay or 0
	Orb{pos=o.pos, size=s*0.45, sizeEnd=s*1.2, color=o.color or Color3.fromRGB(255,205,45),
		transparency=0, life=0.12, delay=d, ease="outBack"}
	Slash{pos=o.pos, angle=0, offsetY=-s*0.32, len=s*0.95, thick=0.055,
		color=o.color2 or Color3.fromRGB(255,242,175), life=0.14, delay=d + 0.02}
	Slash{pos=o.pos, angle=90, len=s*1.5, thick=0.045,
		color=o.color2 or Color3.fromRGB(255,242,175), life=0.16, delay=d + 0.03}
	Ring{pos=o.pos, radius=s*0.45, radiusEnd=s*1.7, thickness=0.14,
		color=o.color or Color3.fromRGB(255,205,45), transparency=0.2, life=0.22, delay=d}
end

-- 爱心：两个上圆 + 一个倒三角楔形
local function Heart(o)
	local s = o.size or 0.75
	local d = o.delay or 0
	local life = o.life or 0.45
	local t0 = o.transparency or 0
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))
	local col = o.color
	Orb{pos=(base * CFrame.new(-s*0.42, s*0.32, 0)).Position, size=s*0.66, sizeEnd=s*0.2,
		color=col, transparency=t0, life=life, delay=d}
	Orb{pos=(base * CFrame.new( s*0.42, s*0.32, 0)).Position, size=s*0.66, sizeEnd=s*0.2,
		color=col, transparency=t0, life=life, delay=d}
	local w = getPart()
	w.Shape = Enum.PartType.Wedge
	w.Material = o.material or Enum.Material.Neon
	w.Color = col
	local wSize = s * 1.32
	local wcf = base * CFrame.new(0, -s * 0.28, 0) * CFrame.Angles(0, 0, math.rad(45))
	local function runW()
		Anim(life, "outQuad", function(e)
			local sc = wSize * (1 - e * 0.25)
			w.Size = Vector3.new(sc, sc, sc)
			w.CFrame = wcf
			w.Transparency = t0 + (1 - t0) * (e ^ 1.3)
		end, function() freePart(w) end)
	end
	if d > 0 then task.delay(d, runW) else runW() end
end

-- ============================================================
--              爱心专用构件（原创 · 真几何，非换色）
-- ============================================================

-- 经典心形参数方程：x = 16sin³t，y = 13cos t − 5cos2t − 2cos3t − cos4t
-- 尖端朝下、上方双圆瓣，是真正意义上的心形（不是方块凑出来的）
-- 实测范围：y ∈ [-17, 11.92]（高 28.92），x ∈ [-16, 16]（宽 32）
local function heartPoint(t)
	local s = math.sin(t)
	return 16 * s * s * s,
		13 * math.cos(t) - 5 * math.cos(2 * t) - 2 * math.cos(3 * t) - math.cos(4 * t)
end

-- 心形归一化：size 为基准单位（实际高度 ≈ size×1.70，宽度 ≈ size×1.88）
local HEART_SCALE = 1 / 17
-- 右半边在 t ∈ [0.91, π] 上 y 单调递减（峰在 t≈0.91），填充二分必须用这个区间
local HEART_T_PEAK = 0.91

-- 心形描边：沿参数方程用 N 段短线拼出整颗心，逐段延迟 = 一笔一笔画出来的感觉
local function HeartOutline(o)
	local size = o.size or 2.2          -- 心形高度（studs）
	local segs = o.segs or 30           -- 分段数，越多越圆滑
	local th = (o.thick or 0.13) * CONFIG.LINE_THICK * styleThick
	local life = o.life or 0.32
	local t0 = o.transparency or 0
	local d0 = o.delay or 0
	local drawT = o.drawTime or 0.13    -- 描完整颗心所需时间
	local dir = o.dir or 1              -- 1 顺时针 / -1 逆时针
	local minW = CONFIG.MIN_LINE_WIDTH
	local scale = size * HEART_SCALE    -- 心形 y 范围 [-17, 11.92]，按 17 归一
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))

	for i = 1, segs do
		local u = (i - 1) / segs
		local t1 = u * 2 * math.pi * dir
		local t2 = ((i) / segs) * 2 * math.pi * dir
		local x1, y1 = heartPoint(t1)
		local x2, y2 = heartPoint(t2)
		local mx, my = (x1 + x2) * 0.5 * scale, (y1 + y2) * 0.5 * scale
		local dx, dy = (x2 - x1) * scale, (y2 - y1) * scale
		local len = math.sqrt(dx * dx + dy * dy)
		if len > 0.001 then
			local ux, uy = dx / len, dy / len
			-- part 长度沿 local Y，需把 Y 轴转到 (ux, uy)：θ = atan2(-ux, uy)
			local theta = math.atan2(-ux, uy)
			local w = math.max(th, minW)
			local seg = getPart()
			seg.Material = o.material or Enum.Material.Neon
			seg.Color = (o.color2 and (i % 2 == 0)) and o.color2 or o.color
			local cf = base * CFrame.new(mx, my, 0) * CFrame.Angles(0, 0, theta)
			local function run()
				Anim(life, o.ease or "outQuad", function(e)
					local ww = w * (1 - e) ^ 1.2
					seg.Size = Vector3.new(math.max(ww, 0.012), len, math.max(ww * 0.85, 0.012))
					seg.CFrame = cf
					seg.Transparency = t0 + (1 - t0) * (e ^ 1.25)
				end, function() freePart(seg) end)
			end
			local dly = d0 + u * drawT
			if dly > 0 then task.delay(dly, run) else run() end
		end
	end
end

-- 心形填充（柔光版）：按行求心形真实半宽，用横条把心"填"出来，带起搏缩放
local function HeartFill(o)
	local size = o.size or 2.2
	local rows = o.rows or 7
	local life = o.life or 0.3
	local t0 = o.transparency or 0.15
	local d0 = o.delay or 0
	local scale = size * HEART_SCALE
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))
	for r = 1, rows do
		local v = r / (rows + 1)                  -- 0..1 从顶到底
		local target = 11.92 - v * 28.92          -- 覆盖心形真实 y 范围
		-- 在单调下降区间 [T_PEAK, π] 上二分，求出该高度对应的 t
		local lo, hi = HEART_T_PEAK, math.pi
		for _ = 1, 14 do
			local mid = (lo + hi) * 0.5
			local _, hy = heartPoint(mid)
			if hy > target then lo = mid else hi = mid end
		end
		local hx = 16 * math.sin(lo) ^ 3
		local halfW = hx * scale
		if halfW > 0.05 then
			local p = getPart()
			p.Material = o.material or Enum.Material.Neon
			p.Color = o.color
			local w = halfW * 2
			local h = size * 0.2
			local cf = base * CFrame.new(0, target * scale, 0)
			local dly = d0 + v * 0.05
			local function run()
				Anim(life, "outQuad", function(e)
					local sc = 1 - e * 0.35
					p.Size = Vector3.new(w * sc, h * sc, math.max(h * 0.5 * sc, 0.012))
					p.CFrame = cf
					p.Transparency = t0 + (1 - t0) * (e ^ 1.2)
				end, function() freePart(p) end)
			end
			if dly > 0 then task.delay(dly, run) else run() end
		end
	end
end

-- 丘比特之箭：箭杆 + 箭头 V 形 + 尾羽 + 箭尖一颗心
local function CupidArrow(o)
	local L = o.len or (CONFIG.EFFECT_SIZE * 3.2)
	local ang = o.angle or 0
	local th = (o.thick or 0.09) * CONFIG.LINE_THICK * styleThick
	local d = o.delay or 0
	local life = o.life or 0.3
	local sd = o.dir or 1                 -- 1 从左到右飞
	local rodCol = o.color or Color3.fromRGB(255, 215, 120)
	local tipCol = o.color2 or Color3.fromRGB(255, 250, 252)
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(ang))

	-- 箭杆（用 Slash，长度随动画从 0 拉满 = 射进来的感觉）
	Slash{pos=o.pos, angle=ang, len=L, thick=th, color=rodCol, transparency=0,
		life=life, delay=d, ease="outExpo"}

	-- 箭头（两条斜线构成的 V，位于杆的前端）
	local tipX = sd * L * 0.5
	for _, sgn in ipairs({ -1, 1 }) do
		Slash{pos=o.pos, angle=ang + sgn * 62, offsetX=tipX * 0.55, offsetY=sgn * 0.001,
			len=L * 0.2, thick=th, color=tipCol, transparency=0, life=life * 0.85,
			delay=d + 0.02, ease="outExpo"}
	end
	-- 尾羽（后端两撇）
	local tailX = -sd * L * 0.42
	for _, sgn in ipairs({ -1, 1 }) do
		Slash{pos=o.pos, angle=ang + sgn * 38, offsetX=tailX, offsetY=sgn * 0.05,
			len=L * 0.16, thick=th * 0.8, color=rodCol, transparency=0.1,
			life=life * 0.8, delay=d + 0.035, ease="outExpo"}
	end
	-- 箭尖带一颗小爱心（射中靶心）
	Heart{
		pos = (base * CFrame.new(tipX * 0.98, 0, 0)).Position,
		size = (o.heartSize or 0.55), color = o.heartColor or Color3.fromRGB(255, 45, 95),
		life = life * 1.5, delay = d + 0.045,
	}
end

-- 爱心飘升：一堆小爱心从中心散开、旋转上浮、缩小消失
local function HeartFloat(o)
	local n = o.count or 8
	local R = o.spread or 2.4
	local life = o.life or 0.7
	local d0 = o.delay or 0
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
	for i = 1, n do
		local a = (i - 1) / n * 2 * math.pi + math.random() * 0.5
		local rad = R * (0.45 + math.random() * 0.75)
		local up = (o.rise or 2.6) * (0.6 + math.random() * 0.8)
		local spin = (math.random() - 0.5) * 2.6
		local sz = (o.size or 0.5) * (0.65 + math.random() * 0.7)
		local col = (o.color2 and (i % 2 == 0)) and o.color2 or o.color
		Heart{
			pos = (base * CFrame.new(math.cos(a) * rad * 0.4, 0, 0)).Position,
			size = sz, color = col, life = life * (0.75 + math.random() * 0.5),
			delay = d0 + (i - 1) * (o.stagger or 0.045),
		}
		-- 拖着走的小光点（表现飘）
		Orb{
			pos = (base * CFrame.new(math.cos(a) * rad, math.sin(a) * rad * 0.5, 0)).Position,
			size = sz * 0.7, sizeEnd = sz * 0.15,
			color = col, transparency = 0.1, life = life * 0.8,
			drift = Vector3.new(math.cos(a) * rad * 0.5, up, 0),
			delay = d0 + (i - 1) * (o.stagger or 0.045),
		}
	end
end

-- 心跳脉冲：咚—咚 两圈环，第二圈更小更快（模仿心跳双搏）
local function HeartPulse(o)
	local R = o.size or 2.4
	local d = o.delay or 0
	local col = o.color
	Ring{pos=o.pos, radius=0.5, radiusEnd=R, thickness=o.thickness or 0.3,
		color=col, transparency=o.transparency or 0.18, life=o.life or 0.28, delay=d}
	Ring{pos=o.pos, radius=0.5, radiusEnd=R * 0.62, thickness=(o.thickness or 0.3) * 0.8,
		color=o.color2 or col, transparency=0.22, life=(o.life or 0.28) * 0.7,
		delay=d + 0.13}
	Orb{pos=o.pos, size=R * 0.22, sizeEnd=R * 0.5, color=col, transparency=0,
		life=0.12, delay=d, ease="outBack"}
	Orb{pos=o.pos, size=R * 0.18, sizeEnd=R * 0.38, color=o.color2 or col, transparency=0,
		life=0.1, delay=d + 0.13, ease="outBack"}
end

-- ============================================================
--           虎杖专用构件（原创 · 黑闪 / 径庭拳）
-- ============================================================

-- 确定性伪随机：同一 seed 两次调用得到同一形状
-- （黑闪要"黑色外壳 + 红色内芯 + 白色高光"三层完全重合，随机形状必须可复现）
local function prand(i, seed)
	local v = math.sin(i * 12.9898 + (seed or 0) * 78.233) * 43758.5453
	return (v - math.floor(v)) - 0.5   -- [-0.5, 0.5]
end

-- 画一段线（所有虎杖构件的底层）：给定两端点，算出角度并把 part 的 Y 轴对齐过去
local function drawSeg(cf, ax, ay, bx, by, thick, col, mat, life, t0, delay, ease)
	local mx, my = (ax + bx) * 0.5, (ay + by) * 0.5
	local dx, dy = bx - ax, by - ay
	local len = math.sqrt(dx * dx + dy * dy)
	if len < 0.001 then return end
	local ux, uy = dx / len, dy / len
	-- part 长度沿 local Y；要把 (0,1) 转到 (ux,uy)：θ = atan2(-ux, uy)
	local theta = math.atan2(-ux, uy)
	local w = math.max(thick, CONFIG.MIN_LINE_WIDTH)
	local seg = getPart()
	seg.Material = mat or Enum.Material.Neon
	seg.Color = col
	local scf = cf * CFrame.new(mx, my, 0) * CFrame.Angles(0, 0, theta)
	local function run()
		Anim(life, ease or "outQuad", function(e)
			local ww = w * (1 - e) ^ 1.15
			seg.Size = Vector3.new(math.max(ww, 0.012), len, math.max(ww * 0.85, 0.012))
			seg.CFrame = scf
			seg.Transparency = t0 + (1 - t0) * (e ^ 1.2)
		end, function() freePart(seg) end)
	end
	if delay and delay > 0 then task.delay(delay, run) else run() end
end

-- 高品质冲击环（替代旧的 Ring）
-- 旧 Ring 用的是 Cylinder：正对镜头时是个"实心圆盘"，看起来就是一个廉价的圆圈。
-- ShockRing 改成用 N 段短片沿圆周拼出真正的"环"——有轮廓、有粗细衰减、有轻微不规则，
-- 扩散时每一段都在动，不再是整片圆盘平移。
local function ShockRing(o)
	local r0, r1 = o.radius or 0.5, o.radiusEnd or 4
	local th = (o.thickness or 0.3) * CONFIG.LINE_THICK * styleThick
	local life = o.life or 0.3
	local segs = o.segs or 26
	local t0, t1 = o.transparency or 0.08, 1
	local d0 = o.delay or 0
	local jitter = o.jitter or 0.05
	local seed = o.seed or 3
	local minW = CONFIG.MIN_LINE_WIDTH
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.roll or 0))
	local mat = o.material or Enum.Material.Neon
	local col = o.color

	local function run()
		local segs2 = {}
		for i = 1, segs do
			local p = getPart()
			p.Material = mat
			p.Color = col
			segs2[i] = {
				p = p,
				a1 = (i - 1) / segs * math.pi * 2,
				a2 = i / segs * math.pi * 2,
				j = 1 + prand(i, seed) * 2 * jitter,
			}
		end
		Anim(life, o.ease or "outCubic", function(e)
			local r = r0 + (r1 - r0) * e
			local w = math.max(th * (1 - e * 0.82), minW * 0.55)
			local tr = t0 + (t1 - t0) * (e ^ 1.15)
			for i, s in ipairs(segs2) do
				local rr = r * s.j
				local x1, y1 = math.cos(s.a1) * rr, math.sin(s.a1) * rr
				local x2, y2 = math.cos(s.a2) * rr, math.sin(s.a2) * rr
				local mx, my = (x1 + x2) * 0.5, (y1 + y2) * 0.5
				local dx, dy = x2 - x1, y2 - y1
				local len = math.sqrt(dx * dx + dy * dy)
				if len > 0.001 then
					local ux, uy = dx / len, dy / len
					local theta = math.atan2(-ux, uy)
					s.p.Size = Vector3.new(math.max(w, 0.012), len * 1.08, math.max(w * 0.88, 0.012))
					s.p.CFrame = base * CFrame.new(mx, my, 0) * CFrame.Angles(0, 0, theta)
					s.p.Transparency = tr
				end
			end
		end, function()
			for _, s in ipairs(segs2) do freePart(s.p) end
		end)
	end
	if d0 > 0 then task.delay(d0, run) else run() end
end

-- 锯齿闪电：折线电光，逐段延迟 = 电流窜出去
-- 从中心向两端窜（黑闪的炸法），两端渐细
local function ZigzagBolt(o)
	local L = o.len or (CONFIG.EFFECT_SIZE * 2.6)
	local n = o.segs or 9
	local jag = o.jag or 0.95          -- 锯齿幅度（越大越像闪电，越小越像直线）
	local th = (o.thick or 0.18) * CONFIG.LINE_THICK * styleThick
	local life = o.life or 0.28
	local t0 = o.transparency or 0
	local d0 = o.delay or 0
	local drawT = o.drawTime or 0.055
	local seed = o.seed or 1
	local taper = o.taper or 0.45
	local segLen = L / n
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))

	local pts = {}
	for i = 0, n do
		local x = (i - n / 2) * segLen
		local y = (i == 0 or i == n) and 0 or (prand(i, seed) * 2 * segLen * jag)
		pts[i] = { x = x, y = y }
	end
	for i = 1, n do
		local a, b = pts[i - 1], pts[i]
		local k = math.abs(i - (n + 1) / 2) / (n / 2 + 0.001)  -- 0=中心 1=端点
		local w = th * (1 - k * taper)
		drawSeg(base, a.x, a.y, b.x, b.y, w,
			(o.color2 and (i % 2 == 0)) and o.color2 or o.color,
			o.material, life, t0, d0 + (1 - k) * drawT, o.ease)
	end
end

-- 空间裂纹：从中心向外炸开的黑色粗裂纹，主干带分叉（像被打碎的玻璃）
-- 这是黑闪"空间被撕裂"的核心表现，比单纯的黑球像得多
local function SpaceCrack(o)
	local L = o.len or (CONFIG.EFFECT_SIZE * 2.8)
	local n = o.segs or 8
	local jag = o.jag or 0.95
	local th = (o.thick or 0.2) * CONFIG.LINE_THICK * styleThick
	local life = o.life or 0.34
	local t0 = o.transparency or 0
	local d0 = o.delay or 0
	local drawT = o.drawTime or 0.05
	local seed = o.seed or 1
	local segLen = L / n
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))

	local pts = {}
	for i = 0, n do
		local u = i / n
		pts[i] = { x = u * L, y = (i == 0) and 0 or (prand(i, seed) * 2 * segLen * jag) }
	end
	for i = 1, n do
		local a, b = pts[i - 1], pts[i]
		local u = i / n
		-- 主干：从中心向外，越远越细
		drawSeg(base, a.x, a.y, b.x, b.y, th * (1 - u * 0.5), o.color,
			o.material or Enum.Material.SmoothPlastic, life, t0, d0 + u * drawT)
		-- 分叉：每隔几段斜着长出短枝，制造"裂开"的感觉
		if o.branch ~= false and i >= 3 and i <= n - 2 and (i % 4 == 0) then
			local sgn = (prand(i + 50, seed) > 0) and 1 or -1
			local bl = L * 0.3 * (0.55 + math.abs(prand(i + 90, seed)))
			local ba = sgn * math.rad(38 + math.abs(prand(i + 7, seed)) * 78)
			drawSeg(base, a.x, a.y, a.x + math.cos(ba) * bl, a.y + math.sin(ba) * bl,
				th * 0.45, o.color, o.material or Enum.Material.SmoothPlastic,
				life * 0.8, t0 + 0.05, d0 + u * drawT + 0.022)
		end
	end
end

-- 黑闪闪电：黑色外壳 + 咒力内芯 + 白色高光，三层共用同一 seed 完全重合
local function BlackBolt(o)
	local seed = o.seed or math.random(1, 100000)
	local life = o.life or 0.3
	local d = o.delay or 0
	local th = o.thick or 0.18
	local segs = o.segs or 11
	local jag = o.jag or 0.95
	-- 1) 黑色外壳（SmoothPlastic 的黑才够"实"，Neon 的黑会发灰）
	ZigzagBolt{
		pos = o.pos, angle = o.angle, len = o.len, segs = segs, jag = jag, seed = seed,
		thick = th * (o.shellScale or 2.6), color = o.shellColor or C_KURO,
		material = Enum.Material.SmoothPlastic, transparency = 0.02,
		life = life, drawTime = o.drawTime, delay = d,
	}
	-- 2) 咒力内芯（红 / 紫白）
	ZigzagBolt{
		pos = o.pos, angle = o.angle, len = o.len, segs = segs, jag = jag, seed = seed,
		thick = th, color = o.color or C_AKA, color2 = o.color2,
		life = life * 0.85, drawTime = o.drawTime, delay = d + 0.012,
	}
	-- 3) 白色高光（最细、最先闪、最快消失）
	if o.highlight ~= false then
		ZigzagBolt{
			pos = o.pos, angle = o.angle, len = o.len, segs = segs, jag = jag, seed = seed,
			thick = th * 0.42, color = C_HAKU,
			life = life * 0.5, drawTime = o.drawTime, delay = d,
		}
	end
end

-- 放射状黑闪电（黑闪最标志性的画面），带网状连接环
local function BlackFlashWeb(o)
	local n = o.count or 12
	local base0 = o.startAngle or 0
	for i = 1, n do
		local ang = base0 + (i - 1) * (360 / n)
		if o.rand then ang = ang + (math.random() - 0.5) * 20 end
		BlackBolt{
			pos = o.pos, angle = ang,
			len = (o.len or CONFIG.EFFECT_SIZE * 3.0) * (o.rand and (0.7 + math.random() * 0.6) or 1),
			segs = o.segs or 11, jag = o.jag or 0.95, thick = o.thick or 0.18,
			color = o.color, color2 = o.color2, shellColor = o.shellColor,
			life = o.life or 0.3, delay = (o.delay or 0) + (o.stagger or 0.01) * i,
			drawTime = o.drawTime or 0.05, highlight = o.highlight,
		}
	end
	-- 网状连接环：把放射的闪电连起来，形成"网"而不是散开的线
	if o.web ~= false then
		local rr = (o.len or CONFIG.EFFECT_SIZE * 3.0)
		ShockRing{pos=o.pos, radius=rr*0.16, radiusEnd=rr*0.26, thickness=0.16,
			color=o.shellColor or C_KURO, transparency=0.08, life=0.28,
			delay=(o.delay or 0) + 0.03, segs=22, seed=131,
			material=Enum.Material.SmoothPlastic}
		ShockRing{pos=o.pos, radius=rr*0.3, radiusEnd=rr*0.42, thickness=0.12,
			color=o.color or C_AKA, transparency=0.24, life=0.34,
			delay=(o.delay or 0) + 0.05, segs=20, seed=137}
	end
end

-- 黑闪核心：咒力收束 → 白光爆闪 → 黑红双环扩散
-- 注意：不做大黑球（那样只会像"冒出个黑球"），用收缩环 + 极短的核心表达
local function BlackFlashCore(o)
	local R = o.size or 2.6
	local d = o.delay or 0
	-- 咒力螺旋收束
	Ring{pos=o.pos, radius=R*2.2, radiusEnd=R*0.22, thickness=0.32, color=C_AKA,
		transparency=0.3, life=0.11, ease="inQuad", delay=d}
	Ring{pos=o.pos, radius=R*1.5, radiusEnd=R*0.18, thickness=0.24, color=C_SHINKU,
		transparency=0.35, life=0.11, ease="inQuad", delay=d + 0.02}
	-- 极小极短的黑核（不是球，是"点"）
	Orb{pos=o.pos, size=R*0.2, sizeEnd=R*0.05, color=C_KURO, transparency=0,
		life=0.1, ease="inQuad", material=Enum.Material.SmoothPlastic, delay=d}
	-- 白光爆闪
	Orb{pos=o.pos, size=R*0.18, sizeEnd=R*1.9, color=C_HAKU, transparency=0,
		life=0.1, delay=d + 0.1}
	-- 黑环 + 红环扩散
	Ring{pos=o.pos, radius=0.4, radiusEnd=R*2.1, thickness=0.42, color=C_KURO,
		transparency=0.05, life=0.32, delay=d + 0.12,
		material=Enum.Material.SmoothPlastic}
	Ring{pos=o.pos, radius=0.4, radiusEnd=R*3.0, thickness=0.24, color=C_AKA,
		transparency=0.22, life=0.46, delay=d + 0.17}
end

-- 径庭拳：第一击 + 咒力滞后 0.16 秒才到的第二击（虎杖招牌，双段冲击）
local function DivergentWave(o)
	local R = o.size or 2.4
	local d = o.delay or 0
	local lag = o.lag or 0.16          -- 咒力滞后的时间差，精髓所在
	local c1 = o.color or C_HAKU
	local c2 = o.color2 or C_AKA

	-- ===== 第一击：规矩、克制、白 =====
	Orb{pos=o.pos, size=R*0.28, sizeEnd=R*0.86, color=c1, transparency=0,
		life=0.12, delay=d, ease="outBack"}
	Ring{pos=o.pos, radius=0.35, radiusEnd=R*1.3, thickness=0.28, color=c1,
		transparency=0.14, life=0.24, delay=d}
	Impact{pos=o.pos, rays=4, rayAngle=45, rayLen=R*0.9, rayThick=0.14,
		color=c1, color2=c2, coreSize=0.32, life=0.18, delay=d}

	-- ===== 第二击：滞后到达，红黑、更大、更狠 =====
	task.delay(lag, function()
		Orb{pos=o.pos, size=R*0.36, sizeEnd=R*1.4, color=c2, transparency=0,
			life=0.18, ease="outBack"}
		Ring{pos=o.pos, radius=0.35, radiusEnd=R*2.1, thickness=0.4, color=c2,
			transparency=0.1, life=0.32}
		Ring{pos=o.pos, radius=0.35, radiusEnd=R*1.5, thickness=0.3, color=C_KURO,
			transparency=0.18, life=0.26, delay=0.03, material=Enum.Material.SmoothPlastic}
		Impact{pos=o.pos, rays=6, rayLen=R*1.3, rayThick=0.16,
			color=c2, color2=C_SHINKU, coreSize=0.42, life=0.22}
		Rays{pos=o.pos, count=6, len=R*1.7, thick=0.14, color=C_AKA, color2=C_SHINKU,
			life=0.24, rand=true}
	end)
end

-- 拳冲击：紧凑十字 + 核心闪光 + 冲击波
local function FistImpact(o)
	local R = o.size or 1.8
	local d = o.delay or 0
	local c1 = o.color or C_HAKU
	local c2 = o.color2 or C_AKA
	local ang = o.angle or 0
	-- 十字（粗）
	Slash{pos=o.pos, angle=ang, len=R*1.35, thick=0.2, color=c2, transparency=0,
		life=o.life or 0.22, delay=d, ease="outExpo"}
	Slash{pos=o.pos, angle=ang+90, len=R*1.25, thick=0.18, color=c1, transparency=0,
		life=(o.life or 0.22)*0.9, delay=d + 0.015, ease="outExpo"}
	-- 核心
	Orb{pos=o.pos, size=R*0.32, sizeEnd=R*0.78, color=c1, transparency=0,
		life=0.13, delay=d, ease="outBack"}
	-- 冲击波
	Ring{pos=o.pos, radius=0.3, radiusEnd=R*1.15, thickness=0.26, color=c2,
		transparency=0.18, life=0.22, delay=d}
end

-- 弧形扫击轨迹：卍踢用。沿抛物线铺开，宽度从头到尾渐变（拖尾感）
local function SwingArc(o)
	local L = o.len or (CONFIG.EFFECT_SIZE * 3.2)
	local curve = o.curve or 0.52
	local segs = o.segs or 14
	local th = (o.thick or 0.22) * CONFIG.LINE_THICK * styleThick
	local life = o.life or 0.28
	local t0 = o.transparency or 0
	local d0 = o.delay or 0
	local sweepT = o.sweepTime or 0.09
	local sd = o.sweepDir or 1
	local taper = o.taper or 0.6
	local base = faceCamCF(o.pos)
		* CFrame.new(o.offsetX or 0, o.offsetY or 0, o.offsetZ or 0)
		* CFrame.Angles(0, 0, math.rad(o.angle or 0))

	for i = 1, segs do
		local t1, t2 = (i - 1) / segs, i / segs
		local u1, u2 = (t1 - 0.5) * 2 * sd, (t2 - 0.5) * 2 * sd
		local x1, y1 = u1 * L * 0.5, curve * L * (1 - u1 * u1) * 0.5
		local x2, y2 = u2 * L * 0.5, curve * L * (1 - u2 * u2) * 0.5
		local w = math.max(th, CONFIG.MIN_LINE_WIDTH) * (1 - t1 * taper)
		drawSeg(base, x1, y1, x2, y2, w, o.color, o.material, life, t0,
			d0 + t1 * sweepT, o.ease)
	end
end

-- 咒力火花：黑红双色粒子 + 黑烟
local function CursedSparks(o)
	local n = o.count or 14
	Burst{pos=o.pos, color=o.color or C_AKA, color2=o.color2 or C_SHINKU,
		count=n, speed=o.speed or 30, size=o.size or 0.3, life=o.life or 0.24,
		delay=o.delay or 0}
	if o.smoke ~= false then
		Burst{pos=o.pos, color=C_KURO, color2=C_KOKUSEN, count=math.floor(n * 0.5),
			speed=(o.speed or 30) * 0.45, size=(o.size or 0.3) * 2, life=(o.life or 0.24) * 2,
			delay=o.delay or 0, texture="rbxasset://textures/particles/smoke_main.dds", emission=0}
	end
end

-- 猫眼闪光：两点亮光一闪（终结技起手）
local function CatEyes(o)
	local s = o.size or 0.32
	local g = o.gap or 0.62
	local base = faceCamCF(o.pos) * CFrame.new(o.offsetX or 0, o.offsetY or 0, 0)
	for _, sgn in ipairs({ -1, 1 }) do
		Orb{
			pos = (base * CFrame.new(sgn * g, 0, 0)).Position,
			size = s * 1.7, sizeEnd = s * 0.15,
			color = o.color, transparency = 0, life = o.life or 0.2,
			delay = o.delay or 0, ease = "inQuad",
		}
	end
end

local shake = { mag = 0, t = 0, d = 0.001 }
local function Shake(mag, dur)
	if not CONFIG.SHAKE_ENABLED then return end
	local m = mag * CONFIG.SHAKE_SCALE
	if m <= shake.mag and shake.t > 0 then return end
	shake.mag, shake.t, shake.d = m, dur, dur
end

local fovKick, baseFov = 0, Camera.FieldOfView
RS:BindToRenderStep("TSB_HitFeedback", Enum.RenderPriority.Camera.Value + 1, function(dt)
	if shake.t > 0 then
		shake.t = shake.t - dt
		local k = math.clamp(shake.t / shake.d, 0, 1)
		local m = shake.mag * k * k
		Camera.CFrame = Camera.CFrame * CFrame.Angles(
			math.rad((math.random() - 0.5) * m),
			math.rad((math.random() - 0.5) * m),
			math.rad((math.random() - 0.5) * m * 0.5))
	end
	if CONFIG.FOV_KICK > 0 then
		if fovKick > 0.01 then
			fovKick = fovKick * math.max(0, 1 - dt * 11)
			Camera.FieldOfView = baseFov + fovKick
		else
			baseFov = Camera.FieldOfView
		end
	end
end)
local function kickFov(mult)
	if CONFIG.FOV_KICK <= 0 then return end
	baseFov = Camera.FieldOfView
	fovKick = CONFIG.FOV_KICK * (mult or 1)
end

-- 全屏闪色（黑闪的"闪"就靠它）：color 闪一下，peak 是最大不透明度，dur 是时长
-- 前向声明，真正的实现在文件末尾 GUI 建好之后赋值
local screenFlash

local C_WHITE = Color3.fromRGB(255, 255, 255)
local C_RED = Color3.fromRGB(230, 30, 35)
local C_DRED = Color3.fromRGB(120, 0, 0)
local C_BLACK = Color3.fromRGB(8, 8, 12)
local C_GOLD = Color3.fromRGB(255, 205, 45)
local C_PALE = Color3.fromRGB(255, 242, 175)
local C_GRAY = Color3.fromRGB(60, 60, 70)

-- 猫娘配色
local C_PINK = Color3.fromRGB(255, 85, 165)
local C_SOFTPINK = Color3.fromRGB(255, 170, 212)
local C_CREAM = Color3.fromRGB(255, 246, 252)
local C_LILAC = Color3.fromRGB(196, 130, 255)
local C_PLUM = Color3.fromRGB(126, 22, 72)
local C_BELL = Color3.fromRGB(255, 210, 60)
local C_EYE = Color3.fromRGB(255, 215, 80)

-- 爱心配色
local C_ROSE = Color3.fromRGB(255, 40, 95)      -- 玫瑰红（主）
local C_HOTPINK_H = Color3.fromRGB(255, 75, 145)
local C_BLOSSOM = Color3.fromRGB(255, 165, 200) -- 柔花瓣粉
local C_PEARL = Color3.fromRGB(255, 248, 251)   -- 珠光白
local C_ARROW = Color3.fromRGB(255, 218, 125)   -- 箭的金
local C_CRIMSON = Color3.fromRGB(150, 8, 45)    -- 深红（坍缩用）

-- 虎杖 / 黑闪配色
local C_KURO = Color3.fromRGB(12, 10, 16)       -- 黒（闪电外壳）
local C_AKA = Color3.fromRGB(255, 45, 55)       -- 赤（呪力）
local C_SHINKU = Color3.fromRGB(158, 0, 18)     -- 深紅
local C_KOKUSEN = Color3.fromRGB(38, 14, 58)    -- 黒閃の紫黒（空间扭曲）
local C_DENKOU = Color3.fromRGB(196, 186, 255)  -- 電光（紫白）
local C_HAKU = Color3.fromRGB(255, 250, 245)    -- 白（冲击闪光）
local C_HOOD = Color3.fromRGB(216, 52, 62)      -- 虎杖连帽衫的红

-- ===================== 宿傩斩 =====================
local function sukuna_horizontal(pos, p)
	local R = CONFIG.EFFECT_SIZE
	Slash{pos=pos, angle=0, len=R*3.4*p, thick=0.42, color=C_RED, transparency=0.45, life=0.3, sweep=R*0.7, ease="outQuad"}
	Slash{pos=pos, angle=0, len=R*4.4*p, thick=0.17, color=C_WHITE, transparency=0, life=0.22, sweep=R*1.5, sweepDir=1, ease="outExpo"}
	Slash{pos=pos, angle=0, len=R*3.8*p, thick=0.09, color=C_WHITE, transparency=0, life=0.15, delay=0.015, sweep=R*1.7, ease="outExpo"}
	Slash{pos=pos, angle=-7, offsetY=0.25, len=R*2.6*p, thick=0.10, color=C_WHITE, transparency=0.1, life=0.18, delay=0.02, sweep=R*0.8}
	Slash{pos=pos, angle=11, offsetY=-0.3, len=R*2.2*p, thick=0.09, color=C_RED, transparency=0.15, life=0.2, delay=0.03, sweep=R*0.7}
	Orb{pos=pos, size=0.5*p, sizeEnd=2.8*p, color=C_WHITE, transparency=0, life=0.15}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.6, thickness=0.26, color=C_WHITE, transparency=0.3, life=0.26}
	for i = 1, 3 do
		Slash{pos=pos, angle=(i-2)*7, len=R*1.8*p, thick=0.10, color=(i%2==0) and C_RED or C_DRED, transparency=0.05, life=0.28, delay=0.03*i, sweep=R*0.4}
	end
	Impact{pos=pos, rays=4, rayAngle=10, rayLen=R*0.9, rayThick=0.10, color=C_WHITE, color2=C_RED, coreSize=0.3, life=0.16}
	Burst{pos=pos, color=C_WHITE, color2=C_RED, count=16, speed=28, size=0.28, life=0.22}
	Shake(0.45 * p, 0.09); kickFov(1 * p)
end

local function sukuna_vertical(pos, p)
	local R = CONFIG.EFFECT_SIZE
	Slash{pos=pos, angle=90, len=R*3.6*p, thick=0.42, color=C_RED, transparency=0.45, life=0.3, sweep=R*0.8, sweepDir=-1, ease="outQuad"}
	Slash{pos=pos, angle=90, len=R*4.6*p, thick=0.17, color=C_WHITE, transparency=0, life=0.22, sweep=R*1.6, sweepDir=-1, ease="outExpo"}
	Slash{pos=pos, angle=90, len=R*4.0*p, thick=0.09, color=C_WHITE, transparency=0, life=0.15, delay=0.015, sweep=R*1.8, sweepDir=-1, ease="outExpo"}
	Slash{pos=pos, angle=83, offsetX=0.3, len=R*2.4*p, thick=0.10, color=C_WHITE, transparency=0.1, life=0.18, delay=0.02, sweep=R*0.8, sweepDir=-1}
	Slash{pos=pos, angle=101, offsetX=-0.28, len=R*2.0*p, thick=0.09, color=C_RED, transparency=0.15, life=0.2, delay=0.03, sweep=R*0.7, sweepDir=-1}
	Orb{pos=pos + Vector3.new(0, -0.7, 0), size=0.6*p, sizeEnd=3.2*p, color=C_WHITE, transparency=0, life=0.17}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.7, thickness=0.26, color=C_WHITE, transparency=0.3, life=0.28}
	Rays{pos=pos, count=4, startAngle=90, len=R*1.9*p, thick=0.12, color=C_RED, life=0.22, delay=0.02}
	for i = 1, 3 do
		Slash{pos=pos, angle=90 + (i-2)*6, len=R*2.0*p, thick=0.10, color=(i%2==0) and C_RED or C_DRED, transparency=0.05, life=0.3, delay=0.03*i, sweep=R*0.4, sweepDir=-1}
	end
	Burst{pos=pos, color=C_WHITE, color2=C_RED, count=18, speed=32, size=0.28, life=0.24}
	Shake(0.55 * p, 0.1); kickFov(1.1 * p)
end

local function sukuna_diagonal(pos, p)
	local R = CONFIG.EFFECT_SIZE
	Slash{pos=pos, angle=135, len=R*4.0*p, thick=0.16, color=C_WHITE, transparency=0, life=0.22, sweep=R*1.4, ease="outExpo"}
	Slash{pos=pos, angle=135, len=R*3.4*p, thick=0.4, color=C_RED, transparency=0.45, life=0.3, sweep=R*0.7, ease="outQuad"}
	Slash{pos=pos, angle=135, len=R*3.4*p, thick=0.09, color=C_WHITE, transparency=0, life=0.15, delay=0.015, sweep=R*1.6, ease="outExpo"}
	Slash{pos=pos, angle=45, len=R*3.8*p, thick=0.16, color=C_WHITE, transparency=0, life=0.22, delay=0.055, sweep=R*1.4, ease="outExpo"}
	Slash{pos=pos, angle=45, len=R*3.2*p, thick=0.09, color=C_RED, transparency=0.05, life=0.16, delay=0.07, sweep=R*1.5, ease="outExpo"}
	Orb{pos=pos, size=0.55*p, sizeEnd=3.2*p, color=C_WHITE, transparency=0, life=0.17}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.9, thickness=0.28, color=C_WHITE, transparency=0.3, life=0.3}
	for i = 1, 4 do
		Slash{pos=pos, angle=(i-1)*90 + 45 + math.random(-10,10), len=R*1.9*p, thick=0.10, color=(i%2==0) and C_RED or C_DRED, transparency=0.05, life=0.3, delay=0.035*i, sweep=R*0.35}
	end
	Impact{pos=pos, rays=6, rayLen=R*1.0, rayThick=0.10, color=C_WHITE, color2=C_RED, coreSize=0.34, life=0.18, delay=0.02}
	Burst{pos=pos, color=C_WHITE, color2=C_RED, count=22, speed=32, size=0.3, life=0.26}
	Shake(0.6 * p, 0.11); kickFov(1.15 * p)
end

local function sukuna_domain(pos, p, isFinisher)
	local R = CONFIG.EFFECT_SIZE
	local bars = isFinisher and 7 or 5
	Orb{pos=pos, size=2.4*p, sizeEnd=0.5*p, color=C_BLACK, transparency=0.05, transparencyEnd=0.02, life=0.14, material=Enum.Material.SmoothPlastic}
	task.delay(0.13, function()
		for i = 1, bars do
			local off = (i - (bars + 1) / 2) * (R * 0.4)
			Slash{pos=pos, angle=22, offsetY=off, len=R*4.6*p, thick=0.95, color=C_WHITE, transparency=0.02, life=0.3, ease="outExpo", sweep=R*0.6, delay=0.015*i}
			Slash{pos=pos, angle=22, offsetY=off, offsetZ=-0.06, len=R*4.3*p, thick=0.6, color=C_BLACK, transparency=0.02, life=0.28, ease="outExpo", sweep=R*0.6, delay=0.015*i, material=Enum.Material.SmoothPlastic}
		end
	end)
	task.delay(0.21, function()
		Orb{pos=pos, size=0.6*p, sizeEnd=5.2*p, color=C_WHITE, transparency=0, life=0.22}
		Ring{pos=pos, radius=0.6, radiusEnd=R*2.6*p, thickness=0.42, color=C_WHITE, transparency=0.12, life=0.34}
		Ring{pos=pos, radius=0.6, radiusEnd=R*3.4*p, thickness=0.22, color=C_RED, transparency=0.28, life=0.5, delay=0.05}
		Rays{pos=pos, count=isFinisher and 12 or 8, len=R*2.4*p, thick=0.12, color=C_RED, color2=C_DRED, life=0.34, rand=true}
		Rays{pos=pos, count=isFinisher and 8 or 6, len=R*1.5*p, thick=0.09, color=C_WHITE, life=0.26, delay=0.04, rand=true}
		Burst{pos=pos, color=C_WHITE, color2=C_RED, count=isFinisher and 44 or 32, speed=40, size=0.36, life=0.34}
		Burst{pos=pos, color=C_BLACK, color2=C_GRAY, count=isFinisher and 24 or 16, speed=14, size=0.7, life=0.5, texture="rbxasset://textures/particles/smoke_main.dds", emission=0}
		Shards{pos=pos, count=isFinisher and 14 or 9, dist=R*1.5*p, size=0.4*p, color=C_BLACK, color2=C_GRAY, life=0.42}
		Shake((isFinisher and 2.0 or 1.4) * p, isFinisher and 0.3 or 0.22)
		kickFov((isFinisher and 2.0 or 1.5) * p)
	end)
	if isFinisher then
		task.delay(0.44, function()
			Orb{pos=pos, size=3.2*p, sizeEnd=0.2*p, color=C_BLACK, transparency=0.08, life=0.28, material=Enum.Material.SmoothPlastic}
			Ring{pos=pos, radius=R*3.0, radiusEnd=R*0.5, thickness=0.5, color=C_WHITE, transparency=0.18, life=0.3, ease="inQuad"}
			Rays{pos=pos, count=10, len=R*2.8*p, thick=0.13, color=C_RED, life=0.36, rand=true}
			Burst{pos=pos, color=C_RED, color2=C_DRED, count=38, speed=34, size=0.4, life=0.4}
			Shake(1.5, 0.3); kickFov(1.6)
		end)
	end
end

local function effect_sukuna_slash(pos, direction, isFinisher)
	local p = isFinisher and 1.3 or 1
	if direction == 1 then sukuna_horizontal(pos, p)
	elseif direction == 2 then sukuna_vertical(pos, p)
	elseif direction == 3 then sukuna_diagonal(pos, p)
	else sukuna_domain(pos, p, isFinisher) end
end

-- ===================== TF2 暴击 =====================
local function tf2_horizontal(pos, p)
	local R = CONFIG.EFFECT_SIZE
	Slash{pos=pos, angle=0, len=R*3.6*p, thick=0.15, color=C_GOLD, transparency=0, life=0.24, sweep=R*0.5, ease="outExpo"}
	Slash{pos=pos, angle=0, len=R*3.0*p, thick=0.09, color=C_PALE, transparency=0, life=0.16, delay=0.02, sweep=R*0.6, ease="outExpo"}
	Slash{pos=pos, angle=90, len=R*1.9*p, thick=0.13, color=C_PALE, transparency=0, life=0.2, delay=0.03, sweep=R*0.3}
	Slash{pos=pos, angle=90, len=R*1.5*p, thick=0.09, color=C_WHITE, transparency=0, life=0.14, delay=0.05}
	Impact{pos=pos, rays=6, rayAngle=8, rayLen=R*1.0*p, rayThick=0.11, color=C_GOLD, color2=C_PALE, coreSize=0.42, coreColor=C_WHITE, life=0.19}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.4*p, thickness=0.2, color=C_GOLD, transparency=0.22, life=0.24}
	Burst{pos=pos, color=C_GOLD, color2=C_PALE, count=18, speed=30, size=0.3, life=0.24}
	Shake(0.4 * p, 0.08); kickFov(0.9 * p)
end

local function tf2_vertical(pos, p)
	local R = CONFIG.EFFECT_SIZE
	Slash{pos=pos, angle=90, len=R*3.8*p, thick=0.15, color=C_GOLD, transparency=0, life=0.24, sweep=R*0.5, sweepDir=-1, ease="outExpo"}
	Slash{pos=pos, angle=90, len=R*3.2*p, thick=0.09, color=C_PALE, transparency=0, life=0.16, delay=0.02, sweep=R*0.6, sweepDir=-1, ease="outExpo"}
	Slash{pos=pos, angle=0, len=R*1.9*p, thick=0.13, color=C_PALE, transparency=0, life=0.2, delay=0.03, sweep=R*0.3}
	Slash{pos=pos, angle=0, len=R*1.5*p, thick=0.09, color=C_WHITE, transparency=0, life=0.14, delay=0.05}
	Impact{pos=pos, rays=6, rayAngle=82, rayLen=R*1.0*p, rayThick=0.11, color=C_GOLD, color2=C_PALE, coreSize=0.42, coreColor=C_WHITE, life=0.19}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.5*p, thickness=0.2, color=C_GOLD, transparency=0.22, life=0.24}
	Burst{pos=pos, color=C_GOLD, color2=C_PALE, count=20, speed=32, size=0.3, life=0.24}
	Shake(0.5 * p, 0.09); kickFov(0.95 * p)
end

local function tf2_diagonal(pos, p)
	local R = CONFIG.EFFECT_SIZE
	Slash{pos=pos, angle=135, len=R*3.4*p, thick=0.15, color=C_GOLD, transparency=0, life=0.24, sweep=R*0.5, ease="outExpo"}
	Slash{pos=pos, angle=135, len=R*2.8*p, thick=0.09, color=C_PALE, transparency=0, life=0.16, delay=0.02, sweep=R*0.6, ease="outExpo"}
	Slash{pos=pos, angle=45, len=R*3.2*p, thick=0.15, color=C_GOLD, transparency=0, life=0.24, delay=0.05, sweep=R*0.5, ease="outExpo"}
	Slash{pos=pos, angle=45, len=R*2.6*p, thick=0.09, color=C_PALE, transparency=0, life=0.16, delay=0.07, sweep=R*0.6, ease="outExpo"}
	Impact{pos=pos, rays=8, rayLen=R*1.1*p, rayThick=0.11, color=C_GOLD, color2=C_PALE, coreSize=0.45, coreColor=C_WHITE, life=0.2}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.7*p, thickness=0.22, color=C_GOLD, transparency=0.22, life=0.26}
	Burst{pos=pos, color=C_GOLD, color2=C_PALE, count=24, speed=34, size=0.32, life=0.26}
	Shake(0.55 * p, 0.1); kickFov(1.05 * p)
end

local function tf2_crit_burst(pos, p, isFinisher)
	local R = CONFIG.EFFECT_SIZE
	local rays = isFinisher and 14 or 10
	Orb{pos=pos, size=0.35*p, sizeEnd=1.1*p, color=C_PALE, transparency=0, life=0.09, ease="outBack"}
	task.delay(0.09, function()
		Rays{pos=pos, count=rays, len=R*3.4*p, thick=0.15, color=C_GOLD, color2=C_PALE, life=0.3, rand=true, ease="outExpo"}
		Rays{pos=pos, count=rays, len=R*2.2*p, thick=0.09, color=C_WHITE, life=0.22, delay=0.03, rand=true, ease="outExpo"}
		Orb{pos=pos, size=0.6*p, sizeEnd=5.0*p, color=C_PALE, transparency=0, life=0.22}
		Ring{pos=pos, radius=0.6, radiusEnd=R*2.4*p, thickness=0.44, color=C_PALE, transparency=0.1, life=0.32}
		Ring{pos=pos, radius=0.6, radiusEnd=R*3.2*p, thickness=0.28, color=C_GOLD, transparency=0.2, life=0.44, delay=0.05}
		Ring{pos=pos, radius=0.6, radiusEnd=R*4.0*p, thickness=0.16, color=C_GOLD, transparency=0.32, life=0.58, delay=0.11}
		Burst{pos=pos, color=C_GOLD, color2=C_PALE, count=isFinisher and 52 or 36, speed=36, size=0.42, life=0.4}
		Burst{pos=pos, color=C_PALE, count=isFinisher and 28 or 18, speed=54, size=0.26, life=0.28}
		Shake((isFinisher and 1.9 or 1.3) * p, isFinisher and 0.28 or 0.2)
		kickFov((isFinisher and 1.9 or 1.4) * p)
	end)
	if isFinisher then
		task.delay(0.36, function()
			Rays{pos=pos, count=8, len=R*4.4, thick=0.18, color=C_GOLD, color2=C_PALE, life=0.38, rand=true, ease="outExpo"}
			Ring{pos=pos, radius=R*3.4, radiusEnd=R*5.0, thickness=0.4, color=C_PALE, transparency=0.14, life=0.38}
			Burst{pos=pos, color=C_GOLD, color2=C_PALE, count=42, speed=42, size=0.5, life=0.46}
			Shake(1.4, 0.28); kickFov(1.5)
		end)
	end
end

local function effect_tf2_crit(pos, direction, isFinisher)
	local p = isFinisher and 1.3 or 1
	if direction == 1 then tf2_horizontal(pos, p)
	elseif direction == 2 then tf2_vertical(pos, p)
	elseif direction == 3 then tf2_diagonal(pos, p)
	else tf2_crit_burst(pos, p, isFinisher) end
end

-- ===================== 边狱巴士 =====================
local function limbus_dir(pos, p, axis)
	local R = CONFIG.EFFECT_SIZE
	Orb{pos=pos, size=1.2*p, sizeEnd=0.3*p, color=C_BLACK, transparency=0.1, life=0.13, material=Enum.Material.Slate}
	task.delay(0.13, function()
		for i = 1, 4 do
			Slash{pos=pos, angle=axis + (i-2.5)*13, len=R*2.2*p, thick=0.13, color=(i%2==0) and C_RED or C_DRED, transparency=0.05, life=0.24, delay=0.02*i, sweep=R*0.5}
		end
		Slash{pos=pos, angle=axis, len=R*3.0*p, thick=0.22, color=C_BLACK, transparency=0.15, life=0.26, material=Enum.Material.SmoothPlastic}
		Shards{pos=pos, count=10, dist=R*1.3*p, size=0.42*p, color=C_BLACK, color2=C_GRAY, life=0.4}
		Ring{pos=pos, radius=0.5, radiusEnd=R*1.5*p, thickness=0.24, color=C_RED, transparency=0.26, life=0.28}
		Burst{pos=pos, color=C_BLACK, color2=C_GRAY, count=14, speed=16, size=0.55, life=0.4, texture="rbxasset://textures/particles/smoke_main.dds", emission=0}
		Burst{pos=pos, color=C_RED, color2=C_DRED, count=14, speed=28, size=0.28, life=0.26}
		Shake(0.5 * p, 0.11); kickFov(0.9 * p)
	end)
end

local function limbus_collapse(pos, p, isFinisher)
	local R = CONFIG.EFFECT_SIZE
	Orb{pos=pos, size=2.6*p, sizeEnd=0.25*p, color=C_BLACK, transparency=0.08, life=0.19, ease="inQuad", material=Enum.Material.Slate}
	Orb{pos=pos, size=1.2*p, sizeEnd=3.2*p, color=C_RED, transparency=0.7, life=0.19}
	task.delay(0.19, function()
		Shards{pos=pos, count=isFinisher and 22 or 16, dist=R*1.8*p, size=0.5*p, color=C_BLACK, color2=C_GRAY, life=0.46}
		Ring{pos=pos, radius=0.5, radiusEnd=R*3.0*p, thickness=0.38, color=C_RED, transparency=0.14, life=0.36}
		Ring{pos=pos, radius=0.5, radiusEnd=R*2.0*p, thickness=0.26, color=C_BLACK, transparency=0.32, life=0.28}
		Rays{pos=pos, count=isFinisher and 12 or 8, len=R*2.2*p, thick=0.12, color=C_RED, color2=C_DRED, life=0.34, rand=true}
		Burst{pos=pos, color=C_BLACK, color2=C_GRAY, count=isFinisher and 32 or 22, speed=18, size=0.7, life=0.52, texture="rbxasset://textures/particles/smoke_main.dds", emission=0}
		Burst{pos=pos, color=C_RED, color2=C_DRED, count=isFinisher and 38 or 26, speed=36, size=0.32, life=0.32}
		Shake((isFinisher and 1.8 or 1.3) * p, isFinisher and 0.3 or 0.22)
		kickFov((isFinisher and 1.7 or 1.2) * p)
	end)
end

local function effect_limbus_shatter(pos, direction, isFinisher)
	local p = isFinisher and 1.3 or 1
	if direction == 1 then limbus_dir(pos, p, 0)
	elseif direction == 2 then limbus_dir(pos, p, 90)
	elseif direction == 3 then limbus_dir(pos, p, 135)
	else limbus_collapse(pos, p, isFinisher) end
end

-- ===================== 猫娘打击（重做版）=====================
-- M1：横向挥爪 + 胡须 + 猫爪印
local function cat_horizontal(pos, p)
	local R = CONFIG.EFFECT_SIZE
	local L = R * 2.7 * p
	local curve = 0.4

	-- 粉色底光（拱形，粗，柔和）
	ClawArc{pos=pos, angle=0, claws=3, len=L*1.15, curve=curve, thick=0.11, color=C_SOFTPINK,
		gap=0.56, life=0.3, transparency=0.35, sweepTime=0.075, sweepDir=1, segs=8}
	-- 主爪痕（三道拱形，从右往左划过）
	ClawArc{pos=pos, angle=0, claws=3, len=L, curve=curve, thick=0.15, color=C_PINK,
		gap=0.56, life=0.3, sweepTime=0.08, sweepDir=1, segs=10}
	-- 爪尖亮点沿弧扫过
	ClawTip{pos=pos, angle=0, len=L, curve=curve, size=0.6*p, time=0.085, sweepDir=1, color=C_CREAM}
	-- 奶白高光爪（细，延迟一点，像爪痕的反光）
	ClawArc{pos=pos, angle=0, claws=3, len=L*1.08, curve=curve, thick=0.045, color=C_CREAM,
		gap=0.56, life=0.22, delay=0.03, sweepTime=0.075, sweepDir=1, segs=8}

	-- 猫爪印
	PawPrint{pos=pos, size=0.62*p, angle=-14, color=C_PINK, color2=C_SOFTPINK, life=0.34}
	-- 胡须
	Whiskers{pos=pos, count=3, len=1.9*p, color=C_CREAM, life=0.22, delay=0.05}
	-- 冲击与粒子
	Orb{pos=pos, size=0.45*p, sizeEnd=2.3*p, color=C_SOFTPINK, transparency=0, life=0.16}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.45*p, thickness=0.24, color=C_PINK, transparency=0.25, life=0.26}
	Burst{pos=pos, color=C_PINK, color2=C_CREAM, count=18, speed=30, size=0.3, life=0.24}
	Burst{pos=pos, color=C_SOFTPINK, count=10, speed=12, size=0.55, life=0.5,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0.4}

	Shake(0.42 * p, 0.09); kickFov(0.9 * p)
end

-- M2：自上而下劈爪 + 猫耳闪现 + 猫爪印
local function cat_vertical(pos, p)
	local R = CONFIG.EFFECT_SIZE
	local L = R * 2.8 * p
	local curve = 0.38

	ClawArc{pos=pos, angle=90, claws=3, len=L*1.15, curve=curve, thick=0.11, color=C_SOFTPINK,
		gap=0.56, life=0.3, transparency=0.35, sweepTime=0.075, sweepDir=-1, segs=8}
	ClawArc{pos=pos, angle=90, claws=3, len=L, curve=curve, thick=0.15, color=C_PINK,
		gap=0.56, life=0.3, sweepTime=0.08, sweepDir=-1, segs=10}
	ClawTip{pos=pos, angle=90, len=L, curve=curve, size=0.6*p, time=0.085, sweepDir=-1, color=C_CREAM}
	ClawArc{pos=pos, angle=90, claws=3, len=L*1.08, curve=curve, thick=0.045, color=C_CREAM,
		gap=0.56, life=0.22, delay=0.03, sweepTime=0.075, sweepDir=-1, segs=8}

	-- 猫耳（在上方两侧闪现）
	CatEars{pos=pos, size=1.05*p, color=C_PINK, color2=C_SOFTPINK, life=0.3, offsetY=R*0.95*p}
	PawPrint{pos=pos, size=0.62*p, angle=-14, color=C_PINK, color2=C_SOFTPINK, life=0.34}
	Whiskers{pos=pos, count=3, len=1.9*p, color=C_CREAM, life=0.22, delay=0.05, offsetY=-0.15}

	Orb{pos=pos, size=0.5*p, sizeEnd=2.5*p, color=C_SOFTPINK, transparency=0, life=0.17}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.55*p, thickness=0.24, color=C_PINK, transparency=0.25, life=0.28}
	Burst{pos=pos, color=C_PINK, color2=C_CREAM, count=20, speed=32, size=0.3, life=0.26}
	Burst{pos=pos, color=C_SOFTPINK, count=12, speed=12, size=0.55, life=0.5,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0.4}

	Shake(0.52 * p, 0.1); kickFov(0.98 * p)
end

-- M3：斜向 X 双爪 + 铃铛 + 双爪印
local function cat_diagonal(pos, p)
	local R = CONFIG.EFFECT_SIZE
	local L = R * 2.7 * p
	local curve = 0.4

	-- 第一道（135°）
	ClawArc{pos=pos, angle=135, claws=3, len=L*1.12, curve=curve, thick=0.1, color=C_SOFTPINK,
		gap=0.54, life=0.3, transparency=0.35, sweepTime=0.075, sweepDir=1, segs=8}
	ClawArc{pos=pos, angle=135, claws=3, len=L, curve=curve, thick=0.14, color=C_PINK,
		gap=0.54, life=0.3, sweepTime=0.08, sweepDir=1, segs=9}
	ClawTip{pos=pos, angle=135, len=L, curve=curve, size=0.55*p, time=0.08, sweepDir=1, color=C_CREAM}

	-- 第二道（45°，延迟形成交叉）
	ClawArc{pos=pos, angle=45, claws=3, len=L*1.12, curve=curve, thick=0.1, color=C_LILAC,
		gap=0.54, life=0.3, transparency=0.35, delay=0.06, sweepTime=0.075, sweepDir=-1, segs=8}
	ClawArc{pos=pos, angle=45, claws=3, len=L, curve=curve, thick=0.14, color=C_SOFTPINK,
		gap=0.54, life=0.3, delay=0.06, sweepTime=0.08, sweepDir=-1, segs=9}
	ClawTip{pos=pos, angle=45, len=L, curve=curve, size=0.55*p, time=0.08, sweepDir=-1,
		color=C_CREAM, delay=0.06}

	-- 高光
	ClawArc{pos=pos, angle=135, claws=3, len=L*1.06, curve=curve, thick=0.045, color=C_CREAM,
		gap=0.54, life=0.2, delay=0.03, sweepTime=0.075, sweepDir=1, segs=7}
	ClawArc{pos=pos, angle=45, claws=3, len=L*1.06, curve=curve, thick=0.045, color=C_CREAM,
		gap=0.54, life=0.2, delay=0.09, sweepTime=0.075, sweepDir=-1, segs=7}

	-- 双爪印 + 铃铛
	PawPrint{pos=pos, size=0.66*p, angle=20, color=C_PINK, color2=C_SOFTPINK, life=0.36}
	PawPrint{pos=pos, size=0.48*p, angle=-35, offsetY=-0.75*p, offsetX=0.6*p,
		color=C_LILAC, color2=C_CREAM, life=0.32, delay=0.08}
	Bell{pos=pos, size=0.6*p, color=C_BELL, color2=C_CREAM, offsetY=0.9*p, delay=0.1}
	Whiskers{pos=pos, count=3, len=1.8*p, color=C_CREAM, life=0.2, delay=0.12}

	Orb{pos=pos, size=0.55*p, sizeEnd=2.7*p, color=C_SOFTPINK, transparency=0, life=0.18}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.75*p, thickness=0.26, color=C_PINK, transparency=0.25, life=0.3}
	Burst{pos=pos, color=C_PINK, color2=C_CREAM, count=24, speed=34, size=0.32, life=0.28}
	Burst{pos=pos, color=C_LILAC, count=14, speed=14, size=0.5, life=0.5,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0.4}

	Shake(0.58 * p, 0.11); kickFov(1.08 * p)
end

-- M4：猫娘狂乱连击（多道爪痕连打 + 猫眼 + 大爪印 + 猫耳 + 爱心）
local function cat_finale(pos, p, isFinisher)
	local R = CONFIG.EFFECT_SIZE
	local L = R * 2.9 * p
	local curve = 0.38

	-- 起手：深紫收缩 + 猫眼闪光
	Orb{pos=pos, size=2.1*p, sizeEnd=0.26*p, color=C_PLUM, transparency=0.06, life=0.16, ease="inQuad"}
	CatEyes{pos=pos, size=0.34*p, gap=0.66*p, color=C_EYE, life=0.22, delay=0.02}

	task.delay(0.16, function()
		-- ===== 狂乱连击：连续多道爪痕，角度与方向交替 =====
		local seq = {
			{ angle = -18, dir =  1, d = 0.00 },
			{ angle =  26, dir = -1, d = 0.055 },
			{ angle =  68, dir =  1, d = 0.11 },
			{ angle = -64, dir = -1, d = 0.165 },
			{ angle = 112, dir =  1, d = 0.22 },
			{ angle = -108, dir = -1, d = 0.275 },
		}
		if isFinisher then
			table.insert(seq, { angle = 158, dir =  1, d = 0.33 })
			table.insert(seq, { angle =  8,  dir = -1, d = 0.375 })
		end

		for i, s in ipairs(seq) do
			local col = (i % 2 == 0) and C_LILAC or C_PINK
			ClawArc{pos=pos, angle=s.angle, claws=3, len=L, curve=curve, thick=0.15, color=col,
				gap=0.56, life=0.32, sweepTime=0.07, sweepDir=s.dir, segs=9, delay=s.d}
			ClawTip{pos=pos, angle=s.angle, len=L, curve=curve, size=0.55*p, time=0.075,
				sweepDir=s.dir, color=C_CREAM, delay=s.d}
			-- 高光层只加在一半的组上，控制零件数量
			if i % 2 == 1 then
				ClawArc{pos=pos, angle=s.angle, claws=3, len=L*1.08, curve=curve, thick=0.045,
					color=C_CREAM, gap=0.56, life=0.22, sweepTime=0.07, sweepDir=s.dir,
					segs=7, delay=s.d + 0.02}
			end
		end

		-- ===== 超大猫爪印拍下 =====
		PawPrint{pos=pos, size=1.55*p, angle=-12, color=C_PINK, color2=C_SOFTPINK, life=0.46, delay=0.1}
		PawPrint{pos=pos, size=0.95*p, angle=28, offsetY=1.3*p, offsetX=-0.8*p,
			color=C_LILAC, color2=C_CREAM, life=0.4, delay=0.2}

		-- ===== 猫耳 + 爱心 =====
		CatEars{pos=pos, size=1.35*p, color=C_PINK, color2=C_SOFTPINK, life=0.34,
			offsetY=R*1.05*p, delay=0.12}
		Heart{pos=pos, size=0.85*p, color=C_PINK, life=0.5, offsetY=-1.5*p, offsetX=1.2*p,
			delay=0.18, angle=-18}
		Heart{pos=pos, size=0.7*p, color=C_SOFTPINK, life=0.5, offsetY=-1.6*p, offsetX=-1.3*p,
			delay=0.26, angle=16}
		if isFinisher then
			Heart{pos=pos, size=1.15*p, color=C_PINK, life=0.55, offsetY=-1.9*p,
				delay=0.34, angle=0}
		end

		-- ===== 铃铛金光 + 光环 =====
		Bell{pos=pos, size=0.75*p, color=C_BELL, color2=C_CREAM, offsetY=1.7*p, delay=0.14}
		Orb{pos=pos, size=0.7*p, sizeEnd=4.6*p, color=C_CREAM, transparency=0, life=0.22, delay=0.1}
		Ring{pos=pos, radius=0.6, radiusEnd=R*2.4*p, thickness=0.4, color=C_SOFTPINK,
			transparency=0.12, life=0.34, delay=0.12}
		Ring{pos=pos, radius=0.6, radiusEnd=R*3.1*p, thickness=0.26, color=C_PINK,
			transparency=0.22, life=0.46, delay=0.17}

		-- ===== 毛絮与粒子 =====
		Rays{pos=pos, count=isFinisher and 12 or 9, len=R*2.4*p, thick=0.12, color=C_PINK,
			color2=C_LILAC, life=0.32, rand=true, delay=0.12}
		Burst{pos=pos, color=C_PINK, color2=C_CREAM, count=isFinisher and 46 or 34,
			speed=38, size=0.38, life=0.36, delay=0.12}
		Burst{pos=pos, color=C_LILAC, color2=C_CREAM, count=isFinisher and 26 or 16,
			speed=56, size=0.24, life=0.26, delay=0.14}
		Burst{pos=pos, color=C_SOFTPINK, count=isFinisher and 26 or 18, speed=13, size=0.7,
			life=0.6, delay=0.12, texture="rbxasset://textures/particles/smoke_main.dds", emission=0.35}
		Shards{pos=pos, count=isFinisher and 14 or 10, dist=R*1.4*p, size=0.38*p, color=C_PINK,
			color2=C_SOFTPINK, material=Enum.Material.Neon, life=0.42, delay=0.15}

		Shake((isFinisher and 1.65 or 1.2) * p, isFinisher and 0.28 or 0.2)
		kickFov((isFinisher and 1.6 or 1.2) * p)
	end)

	if isFinisher then
		task.delay(0.4, function()
			-- 收尾：粉色坍缩 + 最后一记大爪
			Orb{pos=pos, size=3.0*p, sizeEnd=0.2*p, color=C_PLUM, transparency=0.06, life=0.28, ease="inQuad"}
			Ring{pos=pos, radius=R*3.0*p, radiusEnd=R*0.5, thickness=0.44, color=C_CREAM,
				transparency=0.16, life=0.3, ease="inQuad"}
			ClawArc{pos=pos, angle=20, claws=4, len=R*3.6*p, curve=0.34, thick=0.17, color=C_PINK,
				gap=0.6, life=0.36, sweepTime=0.08, sweepDir=1, segs=10}
			CatEyes{pos=pos, size=0.42*p, gap=0.8*p, color=C_EYE, life=0.26}
			PawPrint{pos=pos, size=1.85*p, angle=-12, color=C_PINK, color2=C_CREAM, life=0.5}
			Burst{pos=pos, color=C_PINK, color2=C_CREAM, count=40, speed=42, size=0.44, life=0.42}
			Shake(1.35 * p, 0.3); kickFov(1.5 * p)
		end)
	end
end

local function effect_cat_girl(pos, direction, isFinisher)
	local p = isFinisher and 1.3 or 1
	if direction == 1 then cat_horizontal(pos, p)
	elseif direction == 2 then cat_vertical(pos, p)
	elseif direction == 3 then cat_diagonal(pos, p)
	else cat_finale(pos, p, isFinisher) end
end

-- ===================== 爱心打击（新增 · 音效 7851649484）=====================
-- M1：横向爱心斩 + 心形描边展开
local function love_horizontal(pos, p)
	local R = CONFIG.EFFECT_SIZE

	-- 柔粉外光横斩
	Slash{pos=pos, angle=0, len=R*3.5*p, thick=0.3, color=C_BLOSSOM, transparency=0.35,
		life=0.28, sweep=R*0.6, ease="outQuad"}
	-- 主斩（玫瑰红）
	Slash{pos=pos, angle=0, len=R*4.0*p, thick=0.15, color=C_ROSE, transparency=0,
		life=0.24, sweep=R*1.2, ease="outExpo"}
	-- 珠光高光
	Slash{pos=pos, angle=0, len=R*3.4*p, thick=0.05, color=C_PEARL, transparency=0,
		life=0.16, delay=0.02, sweep=R*1.4, ease="outExpo"}

	-- 真正的心形描边（一笔一笔展开）
	HeartOutline{pos=pos, size=2.6*p, thick=0.14, color=C_ROSE, color2=C_HOTPINK_H,
		life=0.34, drawTime=0.13, segs=30, delay=0.03}
	-- 心形柔光填充
	HeartFill{pos=pos, size=2.6*p, rows=7, color=C_BLOSSOM, life=0.3, delay=0.06}
	-- 中心实心爱心
	Heart{pos=pos, size=0.85*p, color=C_ROSE, life=0.42, delay=0.05}

	-- 心跳脉冲 + 小爱心飘散
	HeartPulse{pos=pos, size=R*1.5*p, color=C_HOTPINK_H, color2=C_BLOSSOM, life=0.26}
	HeartFloat{pos=pos, count=6, spread=2.0*p, size=0.5*p, rise=2.2*p,
		color=C_ROSE, color2=C_BLOSSOM, life=0.6, delay=0.08}

	Orb{pos=pos, size=0.5*p, sizeEnd=2.5*p, color=C_PEARL, transparency=0, life=0.16}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.5*p, thickness=0.24, color=C_ROSE,
		transparency=0.25, life=0.26}
	Burst{pos=pos, color=C_ROSE, color2=C_PEARL, count=20, speed=30, size=0.3, life=0.24}
	Burst{pos=pos, color=C_BLOSSOM, count=10, speed=12, size=0.55, life=0.5,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0.4}

	Shake(0.44 * p, 0.09); kickFov(0.92 * p)
end

-- M2：纵向爱心劈 + 心形描边（略倾斜）+ 心跳双搏
local function love_vertical(pos, p)
	local R = CONFIG.EFFECT_SIZE

	Slash{pos=pos, angle=90, len=R*3.7*p, thick=0.3, color=C_BLOSSOM, transparency=0.35,
		life=0.28, sweep=R*0.6, sweepDir=-1, ease="outQuad"}
	Slash{pos=pos, angle=90, len=R*4.2*p, thick=0.15, color=C_ROSE, transparency=0,
		life=0.24, sweep=R*1.2, sweepDir=-1, ease="outExpo"}
	Slash{pos=pos, angle=90, len=R*3.6*p, thick=0.05, color=C_PEARL, transparency=0,
		life=0.16, delay=0.02, sweep=R*1.4, sweepDir=-1, ease="outExpo"}

	HeartOutline{pos=pos, size=2.8*p, angle=-8, thick=0.14, color=C_ROSE,
		color2=C_HOTPINK_H, life=0.34, drawTime=0.13, segs=30, delay=0.03}
	HeartFill{pos=pos, size=2.8*p, angle=-8, rows=7, color=C_BLOSSOM, life=0.3, delay=0.06}
	Heart{pos=pos, size=0.9*p, color=C_ROSE, life=0.44, delay=0.05}

	-- 上下两颗小心
	Heart{pos=pos, size=0.5*p, offsetY=1.5*p, offsetX=-0.9*p, color=C_HOTPINK_H,
		life=0.4, delay=0.1, angle=18}
	Heart{pos=pos, size=0.5*p, offsetY=1.5*p, offsetX=0.9*p, color=C_HOTPINK_H,
		life=0.4, delay=0.16, angle=-18}

	HeartPulse{pos=pos, size=R*1.6*p, color=C_HOTPINK_H, color2=C_BLOSSOM, life=0.28}
	HeartFloat{pos=pos, count=7, spread=2.1*p, size=0.5*p, rise=2.4*p,
		color=C_ROSE, color2=C_BLOSSOM, life=0.62, delay=0.08}

	Orb{pos=pos, size=0.55*p, sizeEnd=2.7*p, color=C_PEARL, transparency=0, life=0.17}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.6*p, thickness=0.24, color=C_ROSE,
		transparency=0.25, life=0.28}
	Burst{pos=pos, color=C_ROSE, color2=C_PEARL, count=22, speed=32, size=0.3, life=0.26}
	Burst{pos=pos, color=C_BLOSSOM, count=12, speed=12, size=0.55, life=0.5,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0.4}

	Shake(0.54 * p, 0.1); kickFov(1.0 * p)
end

-- M3：斜向 X 双斩 + 双心形描边 + 丘比特之箭
local function love_diagonal(pos, p)
	local R = CONFIG.EFFECT_SIZE

	-- 第一道（135°）
	Slash{pos=pos, angle=135, len=R*3.6*p, thick=0.16, color=C_ROSE, transparency=0,
		life=0.24, sweep=R*1.0, ease="outExpo"}
	Slash{pos=pos, angle=135, len=R*3.0*p, thick=0.05, color=C_PEARL, transparency=0,
		life=0.16, delay=0.02, sweep=R*1.1, ease="outExpo"}
	-- 第二道（45°，延迟形成 X）
	Slash{pos=pos, angle=45, len=R*3.4*p, thick=0.16, color=C_HOTPINK_H, transparency=0,
		life=0.24, delay=0.055, sweep=R*1.0, ease="outExpo"}
	Slash{pos=pos, angle=45, len=R*2.8*p, thick=0.05, color=C_PEARL, transparency=0,
		life=0.16, delay=0.07, sweep=R*1.1, ease="outExpo"}

	-- 双心形描边（一正一斜，错开时间）
	HeartOutline{pos=pos, size=2.5*p, angle=-18, thick=0.13, color=C_ROSE,
		color2=C_HOTPINK_H, life=0.34, drawTime=0.12, segs=28, delay=0.05}
	HeartOutline{pos=pos, size=2.2*p, angle=20, offsetY=-0.2, thick=0.12, color=C_HOTPINK_H,
		color2=C_BLOSSOM, life=0.32, drawTime=0.12, segs=26, delay=0.13}
	HeartFill{pos=pos, size=2.5*p, angle=-18, rows=6, color=C_BLOSSOM, life=0.28, delay=0.08}

	-- 中心大爱心 + 两颗小爱心
	Heart{pos=pos, size=0.95*p, color=C_ROSE, life=0.44, delay=0.06}
	Heart{pos=pos, size=0.48*p, offsetY=1.3*p, offsetX=-1.0*p, color=C_HOTPINK_H,
		life=0.4, delay=0.12, angle=20}
	Heart{pos=pos, size=0.48*p, offsetY=1.3*p, offsetX=1.0*p, color=C_BLOSSOM,
		life=0.4, delay=0.18, angle=-20}

	-- 丘比特之箭斜射穿过
	CupidArrow{pos=pos, angle=135, len=R*3.4*p, thick=0.08, color=C_ARROW,
		color2=C_PEARL, heartColor=C_ROSE, heartSize=0.5*p, life=0.3, delay=0.02}

	HeartPulse{pos=pos, size=R*1.7*p, color=C_HOTPINK_H, color2=C_BLOSSOM, life=0.3}
	HeartFloat{pos=pos, count=8, spread=2.3*p, size=0.52*p, rise=2.5*p,
		color=C_ROSE, color2=C_BLOSSOM, life=0.65, delay=0.1}

	Orb{pos=pos, size=0.6*p, sizeEnd=2.9*p, color=C_PEARL, transparency=0, life=0.18}
	Ring{pos=pos, radius=0.5, radiusEnd=R*1.8*p, thickness=0.26, color=C_ROSE,
		transparency=0.25, life=0.3}
	Burst{pos=pos, color=C_ROSE, color2=C_PEARL, count=26, speed=34, size=0.32, life=0.28}
	Burst{pos=pos, color=C_BLOSSOM, count=14, speed=14, size=0.5, life=0.5,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0.4}

	Shake(0.6 * p, 0.11); kickFov(1.1 * p)
end

-- M4：丘比特之心（起手心跳 → 金箭射穿 → 巨型心形描边 → 爱心风暴）
local function love_finale(pos, p, isFinisher)
	local R = CONFIG.EFFECT_SIZE

	-- 起手：深红收缩 + 心跳双搏（咚—咚）
	Orb{pos=pos, size=2.2*p, sizeEnd=0.28*p, color=C_CRIMSON, transparency=0.06,
		life=0.16, ease="inQuad"}
	HeartPulse{pos=pos, size=R*1.6*p, color=C_ROSE, color2=C_HOTPINK_H, life=0.26, delay=0.02}

	task.delay(0.16, function()
		-- ===== 丘比特之箭从三个角度射入、穿透靶心 =====
		CupidArrow{pos=pos, angle=20, len=R*3.8*p, thick=0.09, color=C_ARROW,
			color2=C_PEARL, heartColor=C_ROSE, heartSize=0.6*p, life=0.32, dir=1}
		CupidArrow{pos=pos, angle=160, len=R*3.6*p, thick=0.09, color=C_ARROW,
			color2=C_PEARL, heartColor=C_HOTPINK_H, heartSize=0.55*p, life=0.32,
			delay=0.05, dir=-1}
		if isFinisher then
			CupidArrow{pos=pos, angle=-70, len=R*4.0*p, thick=0.1, color=C_ARROW,
				color2=C_PEARL, heartColor=C_ROSE, heartSize=0.65*p, life=0.34,
				delay=0.1, dir=1}
		end

		-- ===== 巨型心形描边（主角）=====
		HeartOutline{pos=pos, size=4.4*p, thick=0.19, color=C_ROSE, color2=C_HOTPINK_H,
			life=0.46, drawTime=0.18, segs=38, delay=0.06}
		HeartFill{pos=pos, size=4.4*p, rows=9, color=C_BLOSSOM, life=0.4, delay=0.16}
		-- 内层小心描边（套娃感）
		HeartOutline{pos=pos, size=2.6*p, thick=0.13, color=C_PEARL, life=0.38,
			drawTime=0.15, segs=28, delay=0.2}

		-- ===== 中心大爱心实体 + 四周小爱心 =====
		Heart{pos=pos, size=1.5*p, color=C_ROSE, life=0.55, delay=0.12}
		for i = 1, (isFinisher and 8 or 6) do
			local a = (i - 1) / (isFinisher and 8 or 6) * 2 * math.pi
			Heart{
				pos = pos, size = 0.55 * p,
				offsetX = math.cos(a) * 2.4 * p, offsetY = math.sin(a) * 2.0 * p,
				color = (i % 2 == 0) and C_HOTPINK_H or C_BLOSSOM,
				life = 0.5, delay = 0.16 + i * 0.03,
				angle = math.deg(a) * 0.25,
			}
		end

		-- ===== 爱心风暴（飘升）=====
		HeartFloat{pos=pos, count=isFinisher and 16 or 11, spread=3.0*p, size=0.6*p,
			rise=3.0*p, color=C_ROSE, color2=C_BLOSSOM, life=0.8, delay=0.18, stagger=0.04}

		-- ===== 光环 + 射线 + 粒子 =====
		Orb{pos=pos, size=0.75*p, sizeEnd=4.8*p, color=C_PEARL, transparency=0,
			life=0.22, delay=0.1}
		Ring{pos=pos, radius=0.6, radiusEnd=R*2.5*p, thickness=0.4, color=C_HOTPINK_H,
			transparency=0.12, life=0.34, delay=0.12}
		Ring{pos=pos, radius=0.6, radiusEnd=R*3.2*p, thickness=0.26, color=C_ROSE,
			transparency=0.22, life=0.46, delay=0.17}
		Rays{pos=pos, count=isFinisher and 14 or 10, len=R*2.5*p, thick=0.12,
			color=C_ROSE, color2=C_HOTPINK_H, life=0.32, rand=true, delay=0.12}
		Burst{pos=pos, color=C_ROSE, color2=C_PEARL, count=isFinisher and 50 or 36,
			speed=38, size=0.38, life=0.36, delay=0.12}
		Burst{pos=pos, color=C_BLOSSOM, count=isFinisher and 28 or 18, speed=56,
			size=0.24, life=0.26, delay=0.14}
		Burst{pos=pos, color=C_HOTPINK_H, count=isFinisher and 26 or 18, speed=13,
			size=0.7, life=0.6, delay=0.12,
			texture="rbxasset://textures/particles/smoke_main.dds", emission=0.35}

		Shake((isFinisher and 1.7 or 1.25) * p, isFinisher and 0.28 or 0.2)
		kickFov((isFinisher and 1.6 or 1.22) * p)
	end)

	if isFinisher then
		task.delay(0.44, function()
			-- 收尾：心形坍缩 + 最后一记巨心 + 爱心雨
			Orb{pos=pos, size=3.2*p, sizeEnd=0.2*p, color=C_CRIMSON, transparency=0.06,
				life=0.28, ease="inQuad"}
			Ring{pos=pos, radius=R*3.0*p, radiusEnd=R*0.5, thickness=0.44, color=C_PEARL,
				transparency=0.16, life=0.3, ease="inQuad"}
			HeartOutline{pos=pos, size=5.2*p, thick=0.2, color=C_ROSE, life=0.5,
				drawTime=0.2, segs=40}
			Heart{pos=pos, size=2.2*p, color=C_ROSE, life=0.6}
			HeartFloat{pos=pos, count=18, spread=3.6*p, size=0.68*p, rise=3.4*p,
				color=C_ROSE, color2=C_BLOSSOM, life=0.9, stagger=0.03}
			Burst{pos=pos, color=C_ROSE, color2=C_PEARL, count=42, speed=42, size=0.44, life=0.42}
			Shake(1.4 * p, 0.3); kickFov(1.5 * p)
		end)
	end
end

local function effect_love_heart(pos, direction, isFinisher)
	local p = isFinisher and 1.3 or 1
	if direction == 1 then love_horizontal(pos, p)
	elseif direction == 2 then love_vertical(pos, p)
	elseif direction == 3 then love_diagonal(pos, p)
	else love_finale(pos, p, isFinisher) end
end

-- ===================== 虎杖打击（重制 · 加粗 + 真黑闪）=====================
-- M1：直拳 Jab —— 短促、紧凑、白色冲击 + 红咒力（整体加粗）
local function yuji_jab(pos, p)
	local R = CONFIG.EFFECT_SIZE

	FistImpact{pos=pos, size=1.85*p, color=C_HAKU, color2=C_AKA, life=0.24}
	-- 出拳主冲击（粗）
	Slash{pos=pos, angle=0, len=R*2.7*p, thick=0.22, color=C_HOOD, transparency=0.1,
		life=0.22, sweep=R*0.5, ease="outExpo"}
	Slash{pos=pos, angle=0, len=R*2.2*p, thick=0.09, color=C_HAKU, transparency=0,
		life=0.15, delay=0.02, sweep=R*0.6, ease="outExpo"}
	-- 侧向气流
	Slash{pos=pos, angle=-20, offsetY=0.6, len=R*1.5*p, thick=0.12, color=C_AKA,
		transparency=0.2, life=0.18, delay=0.03, sweep=R*0.45}
	Slash{pos=pos, angle=20, offsetY=-0.6, len=R*1.5*p, thick=0.12, color=C_AKA,
		transparency=0.2, life=0.18, delay=0.045, sweep=R*0.45}
	-- 小咒力闪电（虎杖的咒力会漏电）
	BlackBolt{pos=pos, angle=8, len=R*1.9*p, thick=0.1, color=C_AKA,
		life=0.18, delay=0.02, segs=7, jag=0.8, highlight=true}

	Ring{pos=pos, radius=0.35, radiusEnd=R*1.25*p, thickness=0.26, color=C_HAKU,
		transparency=0.2, life=0.24}
	CursedSparks{pos=pos, count=14, speed=28, size=0.3, life=0.24, smoke=false}

	Shake(0.42 * p, 0.08); kickFov(0.9 * p)
end

-- M2：上勾拳 Uppercut —— 由下往上顶，带上升气流与咒力柱
local function yuji_uppercut(pos, p)
	local R = CONFIG.EFFECT_SIZE

	FistImpact{pos=pos, size=2.0*p, angle=90, color=C_HAKU, color2=C_AKA, life=0.24}
	-- 上顶主冲击
	Slash{pos=pos, angle=90, len=R*3.0*p, thick=0.24, color=C_HOOD, transparency=0.1,
		life=0.24, sweep=R*0.65, sweepDir=-1, ease="outExpo"}
	Slash{pos=pos, angle=90, len=R*2.4*p, thick=0.09, color=C_HAKU, transparency=0,
		life=0.16, delay=0.02, sweep=R*0.75, sweepDir=-1, ease="outExpo"}
	-- 上升气流：四道向上细线，错开
	for i = 1, 4 do
		Slash{pos=pos, angle=90 + (i-2.5)*8, offsetX=(i-2.5)*0.52, len=R*2.1*p,
			thick=0.1, color=C_AKA, transparency=0.15, life=0.22,
			delay=0.03 + i*0.015, sweep=R*1.0, sweepDir=-1}
	end
	-- 咒力柱（上升的黑红闪电）
	BlackBolt{pos=pos, angle=90, len=R*2.5*p, thick=0.11, color=C_AKA,
		life=0.22, delay=0.04, segs=8, jag=0.85}
	-- 上冲核心
	Orb{pos=pos, size=0.44*p, sizeEnd=1.6*p, color=C_AKA, transparency=0.25,
		life=0.2, drift=Vector3.new(0, 2.4*p, 0)}

	Ring{pos=pos, radius=0.35, radiusEnd=R*1.4*p, thickness=0.28, color=C_AKA,
		transparency=0.2, life=0.26}
	CursedSparks{pos=pos, count=18, speed=32, size=0.32, life=0.26, smoke=false}

	Shake(0.54 * p, 0.09); kickFov(1.0 * p)
end

-- M3：卍踢 Manji Kick —— 大幅弧线扫击 + 交叉第二道 + 咒力裂纹
local function yuji_manji_kick(pos, p)
	local R = CONFIG.EFFECT_SIZE

	-- 主扫击弧线（粗）
	SwingArc{pos=pos, angle=20, len=R*4.0*p, curve=0.56, thick=0.26, color=C_HOOD,
		life=0.28, sweepTime=0.1, sweepDir=1}
	-- 白色拖影
	SwingArc{pos=pos, angle=20, len=R*4.2*p, curve=0.56, thick=0.09, color=C_HAKU,
		life=0.2, delay=0.025, sweepTime=0.1, sweepDir=1}
	-- 交叉的第二道（卍字的交叉感），反向
	SwingArc{pos=pos, angle=-30, len=R*3.6*p, curve=0.5, thick=0.22, color=C_SHINKU,
		life=0.28, delay=0.075, sweepTime=0.1, sweepDir=-1}
	SwingArc{pos=pos, angle=-30, len=R*3.8*p, curve=0.5, thick=0.09, color=C_AKA,
		life=0.2, delay=0.1, sweepTime=0.1, sweepDir=-1}

	-- 咒力裂纹（踢击撕裂空气）
	for i = 1, 4 do
		SpaceCrack{pos=pos, angle=(i-1)*90 + 20, len=R*1.9*p, segs=7, jag=0.9,
			thick=0.13, color=C_KURO, life=0.26, delay=0.05 + i*0.012, seed=i*53}
	end

	FistImpact{pos=pos, size=2.1*p, angle=20, color=C_HAKU, color2=C_AKA, life=0.26, delay=0.05}
	Ring{pos=pos, radius=0.4, radiusEnd=R*1.7*p, thickness=0.3, color=C_AKA,
		transparency=0.2, life=0.3, delay=0.05}
	CursedSparks{pos=pos, count=22, speed=36, size=0.34, life=0.28}

	Shake(0.62 * p, 0.1); kickFov(1.1 * p)
end

-- M4：径庭拳 Divergent Fist —— 第一击 + 咒力滞后 0.16s 的第二击
local function yuji_divergent(pos, p)
	local R = CONFIG.EFFECT_SIZE

	-- 起手：咒力聚到拳上（收束环，不是球）
	Ring{pos=pos, radius=R*1.5*p, radiusEnd=R*0.18, thickness=0.3, color=C_AKA,
		transparency=0.35, life=0.12, ease="inQuad"}
	Orb{pos=pos, size=R*0.5, sizeEnd=R*0.14, color=C_AKA, transparency=0.45,
		life=0.12, ease="inQuad"}
	-- 出拳的两道交叉冲击（粗）
	Slash{pos=pos, angle=15, len=R*3.3*p, thick=0.24, color=C_HOOD, transparency=0,
		life=0.24, sweep=R*0.85, ease="outExpo"}
	Slash{pos=pos, angle=-15, len=R*3.0*p, thick=0.2, color=C_AKA, transparency=0,
		life=0.24, delay=0.05, sweep=R*0.85, ease="outExpo"}
	Slash{pos=pos, angle=15, len=R*2.7*p, thick=0.08, color=C_HAKU, transparency=0,
		life=0.16, delay=0.02, sweep=R*0.95, ease="outExpo"}

	-- 双段冲击（核心：第二击延迟到达）
	DivergentWave{pos=pos, size=2.4*p, lag=0.16, color=C_HAKU, color2=C_AKA}

	-- 咒力火花与黑色碎片
	CursedSparks{pos=pos, count=20, speed=34, size=0.34, life=0.28}
	task.delay(0.16, function()
		CursedSparks{pos=pos, count=30, speed=42, size=0.38, life=0.32}
		Shards{pos=pos, count=10, dist=R*1.3*p, size=0.42*p, color=C_KURO,
			color2=C_SHINKU, material=Enum.Material.SmoothPlastic, life=0.42}
		-- 第二击时撕裂空间
		for i = 1, 5 do
			SpaceCrack{pos=pos, angle=(i-1)*72, len=R*2.0*p, segs=8, jag=0.9,
				thick=0.14, color=C_KURO, life=0.3, seed=i*29}
		end
	end)

	-- 两段震动：第一击轻、第二击重（体现"滞后"的重量感）
	Shake(0.55 * p, 0.09); kickFov(1.05 * p)
	task.delay(0.16, function()
		Shake(1.0 * p, 0.17); kickFov(1.4 * p)
	end)
end

-- 击杀 → 黑闪 Black Flash（重制版）
-- 原作里黑闪 = 咒力在 0.000001 秒内与物理冲击重合 → 空间产生扭曲
-- 表现顺序：咒力收束 → 【闪】全屏白光爆闪 → 空间被撕出黑裂纹 → 黑红闪电网炸开 → 冲击波 → 余波
local function yuji_blackflash(pos, p)
	local R = CONFIG.EFFECT_SIZE

	-- ===== P0 (0 ~ 0.10s) 咒力收束：空间被吸向一点 =====
	BlackFlashCore{pos=pos, size=2.8*p, delay=0}

	-- ===== P1 (0.10s) 「闪」：极短白光爆闪 + 全屏闪白 =====
	task.delay(0.10, function()
		Rays{pos=pos, count=14, len=R*2.8*p, thick=0.18, color=C_HAKU,
			transparency=0, life=0.09, ease="outExpo"}
		if screenFlash then screenFlash(Color3.new(1, 1, 1), 0.6, 0.11) end
		Shake(0.9 * p, 0.08)
	end)

	-- ===== P2 (0.14s) 空间撕裂：黑色裂纹向外炸开（像打碎的玻璃）=====
	task.delay(0.14, function()
		for i = 1, 8 do
			local a = (i - 1) / 8 * 360 + math.random(-14, 14)
			SpaceCrack{pos=pos, angle=a, len=R*3.0*p, segs=11, jag=0.98, thick=0.22,
				color=C_KURO, life=0.36, delay=0.012 * i, drawTime=0.05, seed=i * 37}
		end
		if screenFlash then screenFlash(Color3.new(0, 0, 0), 0.42, 0.16) end
	end)

	-- ===== P3 (0.17s) 黑红闪电网炸开（黑闪最标志性的画面）=====
	task.delay(0.17, function()
		BlackFlashWeb{pos=pos, count=13, len=R*3.4*p, thick=0.2, color=C_AKA,
			color2=C_SHINKU, shellColor=C_KURO, life=0.34, rand=true,
			segs=12, jag=0.98, stagger=0.009, web=true}
		BlackFlashWeb{pos=pos, count=7, startAngle=22, len=R*2.2*p, thick=0.13,
			color=C_DENKOU, shellColor=C_KOKUSEN, life=0.28, delay=0.05,
			rand=true, segs=10, jag=0.92, web=false}
	end)

	-- ===== P4 (0.19s) 冲击波 + 碎片 =====
	task.delay(0.19, function()
		ShockRing{pos=pos, radius=0.4, radiusEnd=R*2.6*p, thickness=0.34, color=C_KURO,
			transparency=0.02, life=0.34, segs=28, seed=11,
			material=Enum.Material.SmoothPlastic}
		ShockRing{pos=pos, radius=0.4, radiusEnd=R*3.6*p, thickness=0.2, color=C_AKA,
			transparency=0.16, life=0.5, delay=0.05, segs=26, seed=23}
		ShockRing{pos=pos, radius=R*3.0*p, radiusEnd=R*0.6, thickness=0.24, color=C_KOKUSEN,
			transparency=0.2, life=0.4, ease="inQuad", delay=0.06, segs=24, seed=31}
		Impact{pos=pos, rays=9, rayLen=R*1.5*p, rayThick=0.16, color=C_HAKU,
			color2=C_AKA, coreSize=0.5, life=0.26}
		-- 黑色空间碎片（SmoothPlastic 的黑才够"实"）
		Shards{pos=pos, count=16, dist=R*1.7*p, size=0.46*p, color=C_KURO,
			color2=C_SHINKU, material=Enum.Material.SmoothPlastic, life=0.46}
		CursedSparks{pos=pos, count=52, speed=46, size=0.42, life=0.36}
		Burst{pos=pos, color=C_DENKOU, color2=C_HAKU, count=34, speed=62,
			size=0.28, life=0.28}
		Burst{pos=pos, color=C_KURO, color2=C_KOKUSEN, count=26, speed=15, size=0.85,
			life=0.62, texture="rbxasset://textures/particles/smoke_main.dds", emission=0}

		Shake(2.0 * p, 0.32); kickFov(1.85 * p)
	end)

	-- ===== P5 (0.48s) 余波：第二记更小的黑闪（黑闪常连着来）=====
	task.delay(0.48, function()
		if screenFlash then screenFlash(Color3.new(0, 0, 0), 0.34, 0.14) end
		Orb{pos=pos, size=R*1.9*p, sizeEnd=R*0.08*p, color=C_KOKUSEN, transparency=0.05,
			life=0.26, ease="inQuad", material=Enum.Material.SmoothPlastic}
		BlackFlashWeb{pos=pos, count=9, startAngle=16, len=R*2.8*p, thick=0.16,
			color=C_AKA, color2=C_SHINKU, shellColor=C_KURO, life=0.3,
			rand=true, segs=8, jag=0.95, web=false, highlight=false}
		ShockRing{pos=pos, radius=R*3.2*p, radiusEnd=R*0.5, thickness=0.3, color=C_HAKU,
			transparency=0.12, life=0.32, ease="inQuad", segs=26, seed=41}
		ShockRing{pos=pos, radius=0.4, radiusEnd=R*2.2*p, thickness=0.22, color=C_AKA,
			transparency=0.18, life=0.36, segs=24, seed=53}
		CursedSparks{pos=pos, count=38, speed=40, size=0.38, life=0.34}
		Shards{pos=pos, count=12, dist=R*1.4*p, size=0.4*p, color=C_KURO,
			color2=C_SHINKU, material=Enum.Material.SmoothPlastic, life=0.4}
		Shake(1.4 * p, 0.28); kickFov(1.45 * p)
	end)
end

-- 一记黑闪的通用演出（普通击杀与连击共用，scale 控制强度）
local function blackflashHit(pos, p, scale, seedBase)
	local R = CONFIG.EFFECT_SIZE
	local s = scale or 1

	-- 白光爆闪
	Orb{pos=pos, size=R*0.24*s, sizeEnd=R*2.0*s, color=C_HAKU, transparency=0, life=0.1}
	-- 空间撕裂裂纹
	local cracks = math.floor(8 * s)
	for i = 1, cracks do
		local a = (i - 1) / cracks * 360 + math.random(-14, 14)
		SpaceCrack{pos=pos, angle=a, len=R*3.0*p*s, segs=9, jag=0.98, thick=0.22,
			color=C_KURO, life=0.36, delay=0.012 * i, drawTime=0.05,
			seed=(seedBase or 1) * 37 + i * 17}
	end
	-- 黑红闪电网
	BlackFlashWeb{pos=pos, count=math.floor(11 * s), len=R*3.4*p*s, thick=0.2,
		color=C_AKA, color2=C_SHINKU, shellColor=C_KURO, life=0.34, rand=true,
		segs=8, jag=0.98, stagger=0.009, web=true, delay=0.02}
	BlackFlashWeb{pos=pos, count=math.floor(6 * s), startAngle=22, len=R*2.2*p*s,
		thick=0.13, color=C_DENKOU, shellColor=C_KOKUSEN, life=0.28, delay=0.07,
		rand=true, segs=8, jag=0.92, web=false, highlight=false}
	-- 冲击波
	ShockRing{pos=pos, radius=0.4, radiusEnd=R*2.6*p*s, thickness=0.34, color=C_KURO,
		transparency=0.02, life=0.34, segs=28, seed=61,
		material=Enum.Material.SmoothPlastic, delay=0.04}
	ShockRing{pos=pos, radius=0.4, radiusEnd=R*3.6*p*s, thickness=0.2, color=C_AKA,
		transparency=0.16, life=0.5, delay=0.09, segs=26, seed=71}
	Impact{pos=pos, rays=9, rayLen=R*1.5*p*s, rayThick=0.16, color=C_HAKU,
		color2=C_AKA, coreSize=0.5, life=0.26, delay=0.04}
	CursedSparks{pos=pos, count=math.floor(52 * s), speed=46, size=0.42, life=0.36, delay=0.04}
	Burst{pos=pos, color=C_KURO, color2=C_KOKUSEN, count=math.floor(26 * s), speed=15,
		size=0.85, life=0.62, delay=0.04,
		texture="rbxasset://textures/particles/smoke_main.dds", emission=0}
end

-- 一技能击杀 → 大量黑闪（连击四记，越来越猛）
-- 原作里虎杖进入"flow state"会连续打出黑闪，这里就照那个感觉做
local function yuji_blackflash_barrage(pos, p)
	local R = CONFIG.EFFECT_SIZE

	-- 起手：咒力疯狂收束（比单发黑闪更强的吸力）
	ShockRing{pos=pos, radius=R*3.2*p, radiusEnd=R*0.2, thickness=0.3, color=C_AKA,
		transparency=0.26, life=0.14, ease="inQuad", segs=26, seed=83}
	ShockRing{pos=pos, radius=R*2.2*p, radiusEnd=R*0.16, thickness=0.24, color=C_SHINKU,
		transparency=0.3, life=0.14, ease="inQuad", delay=0.03, segs=24, seed=97}
	Orb{pos=pos, size=R*0.8*p, sizeEnd=R*0.06, color=C_KURO, transparency=0,
		life=0.14, ease="inQuad", material=Enum.Material.SmoothPlastic}

	-- 第一记（0.14s）：标准强度
	task.delay(0.14, function()
		if screenFlash then screenFlash(Color3.new(1, 1, 1), 0.65, 0.12) end
		blackflashHit(pos, p, 1.0, 1)
		Shake(2.0 * p, 0.32); kickFov(1.85 * p)
	end)

	-- 第二记（0.40s）：更强
	task.delay(0.40, function()
		if screenFlash then screenFlash(Color3.new(0, 0, 0), 0.45, 0.15) end
		blackflashHit(pos, p, 1.25, 2)
		Shards{pos=pos, count=18, dist=R*1.8*p, size=0.5*p, color=C_KURO,
			color2=C_SHINKU, material=Enum.Material.SmoothPlastic, life=0.46}
		Shake(2.3 * p, 0.34); kickFov(2.0 * p)
	end)

	-- 第三记（0.68s）：最强，同时向外抛出多道黑闪
	task.delay(0.68, function()
		if screenFlash then screenFlash(Color3.new(1, 1, 1), 0.75, 0.14) end
		blackflashHit(pos, p, 1.5, 3)
		-- 周围同时炸开三记小黑闪（"大量"就靠这个）
		for k = 1, 3 do
			local a = (k - 1) / 3 * 360 + 30
			local rad = R * 1.6 * p
			local off = pos + Vector3.new(math.cos(math.rad(a)) * rad,
				math.sin(math.rad(a)) * rad * 0.6, 0)
			task.delay(0.05 * k, function()
				blackflashHit(off, p, 0.75, 10 + k)
				Shake(1.2 * p, 0.2)
			end)
		end
		Shards{pos=pos, count=22, dist=R*2.0*p, size=0.55*p, color=C_KURO,
			color2=C_SHINKU, material=Enum.Material.SmoothPlastic, life=0.5}
		Shake(2.7 * p, 0.38); kickFov(2.3 * p)
	end)

	-- 终末（1.05s）：全场空间坍缩 + 最大规模闪电网
	task.delay(1.05, function()
		if screenFlash then screenFlash(Color3.new(0, 0, 0), 0.6, 0.24) end
		Orb{pos=pos, size=R*3.4*p, sizeEnd=R*0.05*p, color=C_KOKUSEN, transparency=0.04,
			life=0.34, ease="inQuad", material=Enum.Material.SmoothPlastic}
		-- 最大规模：20 道闪电网
		BlackFlashWeb{pos=pos, count=16, len=R*4.2*p, thick=0.24, color=C_AKA,
			color2=C_SHINKU, shellColor=C_KURO, life=0.42, rand=true,
			segs=8, jag=1.05, stagger=0.007, web=true}
		BlackFlashWeb{pos=pos, count=8, startAngle=14, len=R*2.8*p, thick=0.15,
			color=C_DENKOU, shellColor=C_KOKUSEN, life=0.34, delay=0.06,
			rand=true, segs=8, jag=0.95, web=false, highlight=false}
		for i = 1, 10 do
			SpaceCrack{pos=pos, angle=(i-1)*36, len=R*3.6*p, segs=9, jag=1.0,
				thick=0.24, color=C_KURO, life=0.44, delay=0.01*i, seed=i*61}
		end
		ShockRing{pos=pos, radius=R*4.0*p, radiusEnd=R*0.4, thickness=0.32, color=C_HAKU,
			transparency=0.1, life=0.36, ease="inQuad", segs=30, seed=103}
		ShockRing{pos=pos, radius=0.4, radiusEnd=R*4.4*p, thickness=0.22, color=C_AKA,
			transparency=0.16, life=0.5, segs=28, seed=113}
		CursedSparks{pos=pos, count=70, speed=52, size=0.46, life=0.4}
		Shards{pos=pos, count=26, dist=R*2.2*p, size=0.6*p, color=C_KURO,
			color2=C_SHINKU, material=Enum.Material.SmoothPlastic, life=0.54}
		Shake(3.0 * p, 0.44); kickFov(2.6 * p)
	end)
end

local function effect_yuji(pos, direction, isFinisher, killKind)
	local p = isFinisher and 1.3 or 1
	if direction == 1 then yuji_jab(pos, p)
	elseif direction == 2 then yuji_uppercut(pos, p)
	elseif direction == 3 then yuji_manji_kick(pos, p)
	else
		if not isFinisher then
			yuji_divergent(pos, p)                      -- 普攻第四段 = 径庭拳
		elseif killKind == 1 then
			yuji_blackflash_barrage(pos, p)             -- 一技能击杀 = 大量黑闪
		end
		-- 普攻击杀（killKind == 0）：按需求已取消，不放黑闪、不放音效
	end
end

local STYLE_FUNCS = {
	sukuna_slash = effect_sukuna_slash,
	tf2_crit = effect_tf2_crit,
	limbus_shatter = effect_limbus_shatter,
	cat_girl = effect_cat_girl,
	love_heart = effect_love_heart,
	yuji = effect_yuji,
}

-- ===================== 命中判定 =====================
local lastSwingTime = 0
local lastInputTime = 0
local mouseHeld = false
local lastHitPerTarget = {}
local lastDamageTime = {}   -- 我最近一次对该目标造成伤害的时间（击杀归因兜底）
local lastSkill1Time = 0    -- 最近一次按一技能键的时间

local function markSwing()
	local now = tick()
	lastSwingTime = now
	lastInputTime = now
end
local function isSwinging()
	if mouseHeld then return true end
	return (tick() - lastSwingTime) <= CONFIG.ATTACK_LOCK_TIME
end

UIS.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		mouseHeld = true
		markSwing()
	end
end)
UIS.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		mouseHeld = false
	end
end)
RS.Heartbeat:Connect(function()
	if mouseHeld then markSwing() end
end)

local animConn = nil
local function hookSwingAnim(char)
	if animConn then animConn:Disconnect(); animConn = nil end
	local hum = char:WaitForChild("Humanoid", 5)
	if not hum then return end
	animConn = hum.AnimationPlayed:Connect(function(track)
		local pr = track.Priority
		if pr == Enum.AnimationPriority.Action or pr == Enum.AnimationPriority.Action1 or pr == Enum.AnimationPriority.Action2 or pr == Enum.AnimationPriority.Action3 or pr == Enum.AnimationPriority.Action4 then
			markSwing()
		end
	end)
end

local currentEffectIndex = 1
local selectedEffect = EFFECTS[1]
local activeConnections = {}
local comboCount = 0
local lastHitTime = 0

-- killKind：0 = 普攻击杀，1 = 一技能击杀
local function fireEffect(pos, direction, isFinisher, killKind)
	ensureFolders()
	if not selectedEffect then return end
	-- 应用该风格的线条粗细倍率（虎杖等风格会更粗）
	styleThick = selectedEffect.thick or 1
	local func = STYLE_FUNCS[selectedEffect.key]
	if func then
		local ok, err = pcall(func, pos, direction, isFinisher, killKind)
		if not ok then
			-- 以前错误被 pcall 静默吞掉，特效出问题完全查不到原因；现在报出来
			warn("[TSB M1] 特效执行出错（" .. tostring(selectedEffect.key) .. "）：" .. tostring(err))
		end
	end

	-- 音效：普攻优先用 perDirection[direction]（可按段位配不同音），否则用 sound
	-- 终结用 soundFinisher；一技能击杀用 soundSkill1（设为 nil 就不放）
	local id, cut
	if isFinisher and killKind == 1 then
		id = selectedEffect.soundSkill1 and pickSound(selectedEffect.soundSkill1) or nil
		cut = selectedEffect.finisherCut
	elseif isFinisher then
		id = selectedEffect.soundFinisher and pickSound(selectedEffect.soundFinisher) or nil
		cut = selectedEffect.finisherCut
	else
		local pd = selectedEffect.perDirection
		if pd and pd[direction] then
			id = pickSound(pd[direction])
			cut = selectedEffect.soundCut
		else
			id = pickSound(selectedEffect.sound)
			cut = selectedEffect.soundCut
		end
	end
	playSound(id, selectedEffect.volume or CONFIG.SOUND_VOLUME, cut)
end

local function facingTarget(targetCharacter)
	local myChar = player.Character
	if not myChar then return false end
	local myRoot = myChar:FindFirstChild("HumanoidRootPart")
	local tgRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")
	if not myRoot or not tgRoot then return false end
	local delta = tgRoot.Position - myRoot.Position
	if delta.Magnitude > CONFIG.LOCK_RANGE then return false end
	if delta.Magnitude < 0.001 then return true end
	return myRoot.CFrame.LookVector:Dot(delta.Unit) >= CONFIG.LOCK_ANGLE_DOT
end

local function triggerHitEffect(targetCharacter)
	local now = tick()
	if now - lastHitTime > CONFIG.COMBO_RESET_TIME then comboCount = 0 end
	comboCount = (comboCount % 4) + 1
	lastHitTime = now
	local root = targetCharacter:FindFirstChild("HumanoidRootPart")
	if not root then return end
	fireEffect(root.Position, comboCount, false)
end

local function triggerFinisherEffect(targetCharacter, killKind)
	local root = targetCharacter:FindFirstChild("HumanoidRootPart")
	if not root then return end
	fireEffect(root.Position, 4, true, killKind or 0)
end

-- 宽松版朝向判定：给技能击杀用（技能常带位移，角度不限、距离放宽）
-- 取不到角色信息时放行而不是否掉——宁可多触发一次，也不要像 v20 那样整个不触发
local function facingTargetLoose(targetCharacter)
	local myChar = player.Character
	if not myChar then return true end
	local myRoot = myChar:FindFirstChild("HumanoidRootPart")
	local tgRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")
	if not myRoot or not tgRoot then return true end
	return (tgRoot.Position - myRoot.Position).Magnitude <= CONFIG.SKILL_LOCK_RANGE
end

-- 判断这次击杀算「一技能击杀」还是「普攻击杀」，返回 nil 表示不算我杀的
local function classifyKill(character)
	-- 1) 一技能：按过技能键且在窗口内
	if tick() - lastSkill1Time <= CONFIG.SKILL1_WINDOW then
		if facingTargetLoose(character) then return 1 end
	end
	-- 2) 普攻：正在挥击 + 朝向目标
	if isSwinging() and facingTarget(character) then return 0 end
	-- 3) 兜底：我最近打过他（技能飞行有延迟，挥击状态可能已经过去）
	local t = lastDamageTime[character]
	if t and (tick() - t) <= CONFIG.KILL_CREDIT_TIME then
		if facingTargetLoose(character) then return 0 end
	end
	return nil
end

local function disconnectHumanoid(humanoid)
	local data = activeConnections[humanoid]
	if data then
		if data.conn then data.conn:Disconnect() end
		if data.diedConn then data.diedConn:Disconnect() end
		activeConnections[humanoid] = nil
	end
end

-- ===================== 可打假人（v23 新增）=====================
-- TSB 的假人不在 Players 里，是 workspace 下的普通 Model，所以从 workspace 扫描。
-- 命中/击杀判定完全复用现有的 setupTarget：它只依赖 Humanoid + HumanoidRootPart，
-- 对假人一样适用，不用另写一套。
-- 前向声明：scanDummies 会调用 setupTarget，而 setupTarget 定义在它后面。
-- 不声明的话 Lua 会把 setupTarget 当全局变量找，结果是 nil，一扫到假人就崩。
local setupTarget

local DUMMY_KEYS = {
	"weakest dummy", "blocking dummy", "attacking dummy",
	"training dummy", "testing dummy", "target dummy",
	"dummy", "假人", "人偶",
}

local function isPlayerCharacter(m)
	local ok, pl = pcall(function() return Players:GetPlayerFromCharacter(m) end)
	return ok and pl ~= nil
end

local function dummyRoot(m)
	return m:FindFirstChild("HumanoidRootPart") or m:FindFirstChild("Torso")
		or m:FindFirstChild("UpperTorso") or m:FindFirstChild("Head")
end

local function isDummy(m)
	if not m:IsA("Model") then return false end
	if m == player.Character then return false end
	if isPlayerCharacter(m) then return false end
	if not m:FindFirstChildOfClass("Humanoid") then return false end
	if not dummyRoot(m) then return false end
	local n = m.Name:lower()
	for _, k in ipairs(DUMMY_KEYS) do
		if n:find(k, 1, true) then return true end
	end
	return false
end

local dummyCount = 0
local function scanDummies()
	if not CONFIG.HIT_DUMMY then return end
	local ok, list = pcall(function() return workspace:GetDescendants() end)
	if not ok or not list then return end
	local found = 0
	for _, obj in ipairs(list) do
		if isDummy(obj) then
			found = found + 1
			local hum = obj:FindFirstChildOfClass("Humanoid")
			if hum and not activeConnections[hum] then
				setupTarget(obj)   -- 与玩家完全同一套命中/击杀逻辑
			end
		end
	end
	dummyCount = found
end

-- 独立心跳定期重扫：假人被打死会重生，重生后是新 Model 需要重新挂监听
local dummyScanTimer = 0
RS.Heartbeat:Connect(function(dt)
	dummyScanTimer = dummyScanTimer + dt
	if dummyScanTimer >= CONFIG.DUMMY_SCAN_EVERY then
		dummyScanTimer = 0
		pcall(scanDummies)
	end
end)

setupTarget = function(character)
	local humanoid = character:WaitForChild("Humanoid", 5)
	if not humanoid then return end
	if activeConnections[humanoid] then return end
	local lastHealth = humanoid.Health
	local conn = humanoid.HealthChanged:Connect(function(newHealth)
		if newHealth <= 0 and lastHealth > 0 then
			lastHealth = newHealth
			-- v21：不再只认 isSwinging()，技能击杀也能触发
			local kind = classifyKill(character)
			if kind ~= nil then
				triggerFinisherEffect(character, kind)
			end
			return
		end
		if newHealth >= lastHealth then lastHealth = newHealth; return end
		lastHealth = newHealth
		-- 记录"我打过他"的时间，作为击杀归因的兜底依据
		-- （不要求 isSwinging，技能伤害同样记上，这样技能延迟命中也能归因）
		if facingTargetLoose(character) then
			lastDamageTime[character] = tick()
		end
		if not isSwinging() then return end
		if not facingTarget(character) then return end
		local t = tick()
		if lastHitPerTarget[character] and (t - lastHitPerTarget[character]) < CONFIG.HIT_THROTTLE then return end
		lastHitPerTarget[character] = t
		triggerHitEffect(character)
	end)
	local diedConn = humanoid.Died:Connect(function() end)
	activeConnections[humanoid] = { conn = conn, diedConn = diedConn, character = character }
end

local function onCharacterAdded(character)
	if character == player.Character then return end
	setupTarget(character)
end
local function initPlayer(p)
	if p ~= player and p.Character then setupTarget(p.Character) end
	p.CharacterAdded:Connect(onCharacterAdded)
end
for _, p in pairs(Players:GetPlayers()) do initPlayer(p) end
Players.PlayerAdded:Connect(initPlayer)

player.CharacterAdded:Connect(function(char)
	task.wait(0.3)
	hookSwingAnim(char)
	for humanoid, data in pairs(activeConnections) do
		if not data.character:IsDescendantOf(game) then disconnectHumanoid(humanoid) end
	end
	for _, p in pairs(Players:GetPlayers()) do
		if p ~= player and p.Character then setupTarget(p.Character) end
	end
end)
if player.Character then hookSwingAnim(player.Character) end

-- 假人若是后来生成 / 重生的，靠这个立刻挂上，不用等下一轮扫描
workspace.DescendantAdded:Connect(function(obj)
	if not CONFIG.HIT_DUMMY then return end
	if isDummy(obj) then
		task.wait(0.25)
		local hum = obj:FindFirstChildOfClass("Humanoid")
		if hum and not activeConnections[hum] then setupTarget(obj) end
	end
end)

scanDummies()   -- 启动时先扫一遍
task.delay(1.6, function()
	if CONFIG.HIT_DUMMY and dummyCount > 0 then
		danmaku(string.format(T("dummyFound"), dummyCount))
	end
end)

local cleanupTimer = 0
RS.Heartbeat:Connect(function(dt)
	cleanupTimer = cleanupTimer + dt
	if cleanupTimer >= 1 then
		cleanupTimer = 0
		ensureFolders()
		for k, t in pairs(lastHitPerTarget) do
			if tick() - t > 2 then lastHitPerTarget[k] = nil end
		end
		for _, p in pairs(Players:GetPlayers()) do
			if p ~= player and p.Character and p.Character:FindFirstChild("Humanoid") then
				local h = p.Character.Humanoid
				if not activeConnections[h] then setupTarget(p.Character) end
			end
		end
		for humanoid, data in pairs(activeConnections) do
			if not data.character:IsDescendantOf(game) then disconnectHumanoid(humanoid) end
		end
	end
end)

-- ===================== 提示 UI =====================
local noticeGui = Instance.new("ScreenGui")
noticeGui.Name = "TSBNotice"
noticeGui.ResetOnSpawn = false
noticeGui.IgnoreGuiInset = true
noticeGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
noticeGui.Parent = player:WaitForChild("PlayerGui")

local noticeLabel = Instance.new("TextLabel")
noticeLabel.Size = UDim2.new(1, 0, 0, 50)
noticeLabel.Position = UDim2.new(0, 0, 0.35, 0)
noticeLabel.BackgroundTransparency = 1
noticeLabel.Text = ""
noticeLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
noticeLabel.TextStrokeTransparency = 0.3
noticeLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
noticeLabel.Font = Enum.Font.SourceSansBold
noticeLabel.TextSize = 32
noticeLabel.TextTransparency = 1
noticeLabel.Parent = noticeGui

-- 全屏闪光层：黑闪的"闪"就靠它（白光爆闪 / 瞬间变暗）
local flashFrame = Instance.new("Frame")
flashFrame.Name = "BlackFlashOverlay"
flashFrame.Size = UDim2.new(1.2, 0, 1.2, 0)
flashFrame.Position = UDim2.new(-0.1, 0, -0.1, 0)
flashFrame.BackgroundColor3 = Color3.new(1, 1, 1)
flashFrame.BackgroundTransparency = 1
flashFrame.BorderSizePixel = 0
flashFrame.ZIndex = 0
flashFrame.Visible = true
flashFrame.Parent = noticeGui

local flashTween = nil
screenFlash = function(color, peak, dur)
	flashFrame.BackgroundColor3 = color or Color3.new(1, 1, 1)
	if flashTween then flashTween:Cancel() end
	flashFrame.BackgroundTransparency = 1 - math.clamp(peak or 0.5, 0, 1)
	flashTween = TS:Create(flashFrame,
		TweenInfo.new(math.max(dur or 0.12, 0.02), Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
		{ BackgroundTransparency = 1 })
	flashTween:Play()
end

-- ===================== 多语言 / 弹幕 / 语言菜单（v23 新增）=====================
-- 语言选项：按你的要求，菜单项是英文标注（Chinese / English / Vietnamese）
local LANGS = {
	{ key = "zh", en = "Chinese",    native = "中文" },
	{ key = "en", en = "English",    native = "English" },
	{ key = "vi", en = "Vietnamese", native = "Tiếng Việt" },
}
local curLang = "zh"

local STR = {
	zh = {
		styleReady  = "M4 变体已就绪",
		testNormal  = "普攻音效",
		testFin     = "终结音效",
		testSkill1  = "一技能击杀音效",
		dummyFound  = "已扫描到 %d 个假人",
		langSet     = "语言已设置",
		chooseLang  = "选择语言",
		keyHint     = "默认按键 %s 切换特效 ｜ 绑定：Ctrl + 任意键",
		boundTo     = "已绑定 %s 键",
	},
	en = {
		styleReady  = "M4 variant ready",
		testNormal  = "Normal hit sound",
		testFin     = "Finisher sound",
		testSkill1  = "Skill-1 kill sound",
		dummyFound  = "Found %d dummy(es)",
		langSet     = "Language set",
		chooseLang  = "Choose Language",
		keyHint     = "Default key %s to switch effects | Rebind: Ctrl + any key",
		boundTo     = "Bound to [%s]",
	},
	vi = {
		styleReady  = "Biến thể M4 đã sẵn sàng",
		testNormal  = "Âm thanh đánh thường",
		testFin     = "Âm thanh kết liễu",
		testSkill1  = "Âm thanh kết liễu bằng kỹ năng 1",
		dummyFound  = "Đã tìm thấy %d Dummy",
		langSet     = "Đã chọn ngôn ngữ",
		chooseLang  = "Chọn ngôn ngữ",
		keyHint     = "Phím mặc định %s để đổi hiệu ứng | Gán lại: Ctrl + phím bất kỳ",
		boundTo     = "Đã gán [%s]",
	},
}
local function T(key)
	local t = STR[curLang] or STR.zh
	return t[key] or (STR.zh[key] or key)
end

-- ---------- 弹幕：从右向左滚动 ----------
-- 数字键/符号键显示成好看的名字（1 而不是 One）
local KEY_ALIAS = {
	One="1", Two="2", Three="3", Four="4", Five="5",
	Six="6", Seven="7", Eight="8", Nine="9", Zero="0",
	LeftControl="L-Ctrl", RightControl="R-Ctrl",
	LeftShift="L-Shift", RightShift="R-Shift",
	LeftAlt="L-Alt", RightAlt="R-Alt",
	Return="Enter", BackSlash="\\", Slash="/",
	Minus="-", Equals="=", Period=".", Comma=",",
	Semicolon=";", Quote="'", LeftBracket="[", RightBracket="]",
}
local function keyName(kc)
	local nm = tostring(kc):gsub("Enum%.KeyCode%.", "")
	return KEY_ALIAS[nm] or nm
end

-- 切换风格的可绑定键，默认 G；Ctrl + 任意键 可重新绑定
local boundStyleKey = Enum.KeyCode.G

local danmakuSlot = 0
local function danmaku(text)
	danmakuSlot = (danmakuSlot + 1) % 4
	local lbl = Instance.new("TextLabel")
	lbl.Size = IS_MOBILE and UDim2.new(0.94, 0, 0, 40) or UDim2.new(0, 520, 0, 44)
	lbl.Position = UDim2.new(IS_MOBILE and 0.03 or 0, 0, 0.08 + danmakuSlot * 0.09, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = text
	lbl.TextColor3 = Color3.fromRGB(255, 224, 130)
	lbl.TextStrokeTransparency = 0.25
	lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	lbl.Font = Enum.Font.SourceSansBold
	lbl.TextSize = uiText(22)
	lbl.TextXAlignment = Enum.TextXAlignment.Center
	lbl.Position = UDim2.new(1.05, 0, 0.08 + danmakuSlot * 0.09, 0)
	lbl.Parent = noticeGui
	local tw = TS:Create(lbl, TweenInfo.new(7, Enum.EasingStyle.Linear),
		{ Position = UDim2.new(-1.05, 0, lbl.Position.Y.Scale, 0) })
	tw:Play()
	tw.Completed:Connect(function() lbl:Destroy() end)
end

-- 夜神月大笑播放器
-- ===== 夜神月大笑 ID 播放器 =====
-- 前向声明：playKiraById 会调用 showNotice，而 showNotice 定义在后面。
-- 不声明的话 Lua 会当全局变量找 → nil → 一按 H 就崩。
local showNotice

-- 实测只有第一个能响，其余三个全部失效，故只保留这一个
local KIRA_ID = "7473415590"   -- Light Yagami Laugh / Kira's Laugh

local function playKiraById(id, label)
	lastSoundTime = 0
	playSound(id, CONFIG.KIRA_VOLUME, nil)
	showNotice("🎬 夜神月 " .. (label or "") .. " [" .. id .. "]")
	danmaku("夜神月 " .. (label or "") .. " ｜ ID " .. id)
end

-- H：试听（现在只有一个有效 ID，就是它）
local function playKiraNext()
	playKiraById(KIRA_ID, "Kira's Laugh")
end

-- 选完语言后播放的彩蛋：优先用锁定的那个
local function playSceneDialogue()
	lastSoundTime = 0
	playSound(KIRA_ID, CONFIG.KIRA_VOLUME, nil)
end

-- ---------- 语言选择菜单：从屏幕下方平滑升到正中 ----------
local langFrame = Instance.new("Frame")
langFrame.Name = "TSBLangMenu"
langFrame.AnchorPoint = Vector2.new(0.5, 0.5)
langFrame.Size = IS_MOBILE and UDim2.new(0.86, 0, 0, 236) or UDim2.new(0, 300, 0, 236)
langFrame.Position = UDim2.new(0.5, 0, 1.6, 0)   -- 起点：屏幕下方外面
langFrame.BackgroundColor3 = Color3.fromRGB(14, 14, 20)
langFrame.BackgroundTransparency = 0.12
langFrame.BorderSizePixel = 0
langFrame.Parent = noticeGui

local langCorner = Instance.new("UICorner")
langCorner.CornerRadius = UDim.new(0, 10)
langCorner.Parent = langFrame

local langTitle = Instance.new("TextLabel")
langTitle.Size = UDim2.new(1, 0, 0, 52)
langTitle.Position = UDim2.new(0, 0, 0, 8)
langTitle.BackgroundTransparency = 1
langTitle.Text = "Choose Language"
langTitle.TextColor3 = Color3.fromRGB(255, 224, 130)
langTitle.Font = Enum.Font.SourceSansBold
langTitle.TextSize = uiText(24)
langTitle.Parent = langFrame

local langSub = Instance.new("TextLabel")
langSub.Size = UDim2.new(1, 0, 0, 22)
langSub.Position = UDim2.new(0, 0, 0, 44)
langSub.BackgroundTransparency = 1
langSub.Text = STR.zh.chooseLang
langSub.TextColor3 = Color3.fromRGB(150, 142, 128)
langSub.Font = Enum.Font.SourceSans
langSub.TextSize = 14
langSub.Parent = langFrame

local langButtons = {}
local function closeLangMenu()
	local tw = TS:Create(langFrame, TweenInfo.new(0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
		{ Position = UDim2.new(0.5, 0, 1.6, 0) })
	tw:Play()
	tw.Completed:Connect(function() langFrame.Visible = false end)
end

for i, lg in ipairs(LANGS) do
	local b = Instance.new("TextButton")
	b.Size = IS_MOBILE and UDim2.new(0.86, 0, 0, 46) or UDim2.new(0, 260, 0, 46)
	b.Position = IS_MOBILE and UDim2.new(0.07, 0, 0, 74 + (i - 1) * 52)
	             or UDim2.new(0.5, -130, 0, 74 + (i - 1) * 52)
	b.BackgroundColor3 = Color3.fromRGB(32, 30, 40)
	b.BorderSizePixel = 0
	b.TextColor3 = Color3.fromRGB(235, 230, 220)
	b.Font = Enum.Font.SourceSansBold
	b.TextSize = 20
	-- 主文字用英文标注（你要的），下面一行小字是母语
	b.Text = lg.en .. "  (" .. lg.native .. ")"
	b.Parent = langFrame
	local bc = Instance.new("UICorner")
	bc.CornerRadius = UDim.new(0, 8)
	bc.Parent = b
	langButtons[i] = b
	b.MouseButton1Click:Connect(function()
		curLang = lg.key
		langSub.Text = T("chooseLang")
		b.BackgroundColor3 = Color3.fromRGB(201, 162, 39)
		b.TextColor3 = Color3.fromRGB(20, 18, 12)
		danmaku(T("langSet") .. " · " .. lg.en)
		-- 选完语言后再来一条：用所选语言说明默认按键 + 怎么绑定
		task.delay(0.55, function()
			danmaku(string.format(T("keyHint"), keyName(boundStyleKey)))
		end)
		-- 选完语言 1 秒后放这段场景对白
		task.delay(1.0, playSceneDialogue)
		task.delay(0.5, closeLangMenu)
	end)
end

-- 升起动画：从下方平滑滑到屏幕中间
task.delay(0.9, function()
	local tw = TS:Create(langFrame, TweenInfo.new(0.9, Enum.EasingStyle.Quart, Enum.EasingDirection.Out),
		{ Position = UDim2.new(0.5, 0, 0.5, 0) })
	tw:Play()
end)

-- 开场弹幕
danmaku("script by golden")

local noticeTween = nil
showNotice = function(text)
	if noticeTween then noticeTween:Cancel() end
	noticeLabel.Text = text
	noticeLabel.TextTransparency = 0
	noticeLabel.TextStrokeTransparency = 0.3
	noticeTween = TS:Create(noticeLabel, TweenInfo.new(CONFIG.NOTICE_DURATION, Enum.EasingStyle.Quad, Enum.EasingDirection.In), { TextTransparency = 1, TextStrokeTransparency = 1 })
	noticeTween:Play()
end

-- 一技能键：按下就记时间，用于判定「一技能击杀」
UIS.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if input.KeyCode == CONFIG.SKILL1_KEY then
		lastSkill1Time = tick()
	end
end)

-- 切换风格（PC 键盘和手机按钮共用同一段逻辑）
local function switchStyle()
	currentEffectIndex = currentEffectIndex + 1
	if currentEffectIndex > #EFFECTS then currentEffectIndex = 1 end
	selectedEffect = EFFECTS[currentEffectIndex]
	comboCount = 0
	showNotice(selectedEffect.name .. " · " .. T("styleReady"))
end

UIS.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
	local kc = input.KeyCode
	-- Ctrl + 任意键 = 把「切换风格」重新绑定到那个键
	local ctrlDown = UIS:IsKeyDown(Enum.KeyCode.LeftControl)
		or UIS:IsKeyDown(Enum.KeyCode.RightControl)
	if ctrlDown then
		if kc ~= Enum.KeyCode.LeftControl and kc ~= Enum.KeyCode.RightControl
			and kc ~= Enum.KeyCode.Escape then
			boundStyleKey = kc
			danmaku(string.format(T("boundTo"), keyName(kc)))
		end
		return
	end

	-- 切换风格（默认 G，可用 Ctrl+任意键 改）
	if kc == boundStyleKey then
		switchStyle()
	end
	-- R：试听 M1~M3 的音效
	if input.KeyCode == Enum.KeyCode.R then
		lastSoundTime = 0
		playSound(pickSound(selectedEffect.sound), selectedEffect.volume or CONFIG.SOUND_VOLUME, selectedEffect.soundCut)
		showNotice("🔊 " .. T("testNormal") .. "：" .. selectedEffect.name)
	end
	-- T：试听 M4 / 终结音效
	if input.KeyCode == Enum.KeyCode.T then
		lastSoundTime = 0
		playSound(pickSound(selectedEffect.soundFinisher or selectedEffect.sound), selectedEffect.volume or CONFIG.SOUND_VOLUME, selectedEffect.finisherCut)
		showNotice("🔊 " .. T("testFin") .. "：" .. selectedEffect.name)
	end
	-- Y：试听一技能击杀音效
	if input.KeyCode == Enum.KeyCode.Y then
		lastSoundTime = 0
		playSound(pickSound(selectedEffect.soundSkill1 or selectedEffect.soundFinisher or selectedEffect.sound), selectedEffect.volume or CONFIG.SOUND_VOLUME, selectedEffect.finisherCut)
		showNotice("🔊 " .. T("testSkill1") .. "：" .. selectedEffect.name)
	end
	-- H：试听夜神月大笑
	if input.KeyCode == Enum.KeyCode.H then
		playKiraNext()
	end
end)

-- ===================== 移动端：虚拟按钮面板 =====================
-- 手机没键盘，G / H / R / T / Y 全按不了，所以给一套屏幕按钮。
-- PC 上 IS_MOBILE = false，这段完全不执行，PC 体验一点不变。
if IS_MOBILE then
	local panel = Instance.new("ScreenGui")
	panel.Name = "TSBMobilePanel"
	panel.ResetOnSpawn = false
	panel.IgnoreGuiInset = true
	panel.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	panel.Parent = player:WaitForChild("PlayerGui")

	-- 可拖动的容器（避免挡住战斗视野）
	local holder = Instance.new("Frame")
	holder.Name = "Holder"
	holder.Size = UDim2.new(0.30, 0, 0.42, 0)
	holder.Position = UDim2.new(0.01, 0, 0.30, 0)
	holder.BackgroundTransparency = 1
	holder.Active = true
	holder.Draggable = true
	holder.Parent = panel

	local BTN = {
		{ txt = "🎨 风格",      fn = function() switchStyle() end },
		{ txt = "🎬 夜神月",    fn = function() playKiraNext() end },
		{ txt = "🔊 普攻音",    fn = function()
			lastSoundTime = 0
			playSound(pickSound(selectedEffect.sound), selectedEffect.volume or CONFIG.SOUND_VOLUME, selectedEffect.soundCut)
			showNotice("🔊 " .. T("testNormal") .. "：" .. selectedEffect.name)
		end },
		{ txt = "🔊 终结音",    fn = function()
			lastSoundTime = 0
			playSound(pickSound(selectedEffect.soundFinisher or selectedEffect.sound), selectedEffect.volume or CONFIG.SOUND_VOLUME, selectedEffect.finisherCut)
			showNotice("🔊 " .. T("testFin") .. "：" .. selectedEffect.name)
		end },
		{ txt = "🔊 技能音",    fn = function()
			lastSoundTime = 0
			playSound(pickSound(selectedEffect.soundSkill1 or selectedEffect.soundFinisher or selectedEffect.sound), selectedEffect.volume or CONFIG.SOUND_VOLUME, selectedEffect.finisherCut)
			showNotice("🔊 " .. T("testSkill1") .. "：" .. selectedEffect.name)
		end },
		{ txt = "🎯 假人",      fn = function()
			CONFIG.HIT_DUMMY = not CONFIG.HIT_DUMMY
			showNotice("🎯 打假人：" .. (CONFIG.HIT_DUMMY and "开" or "关"))
		end },
	}

	for i, d in ipairs(BTN) do
		local b = Instance.new("TextButton")
		b.Size = UDim2.new(1, 0, 0, 52)
		b.Position = UDim2.new(0, 0, 0, (i - 1) * 58)
		b.BackgroundColor3 = Color3.fromRGB(20, 18, 26)
		b.BackgroundTransparency = 0.25
		b.BorderSizePixel = 0
		b.TextColor3 = Color3.fromRGB(235, 230, 220)
		b.Font = Enum.Font.SourceSansBold
		b.TextSize = 17
		b.Text = d.txt
		b.Parent = holder
		local bc = Instance.new("UICorner")
		bc.CornerRadius = UDim.new(0, 8)
		bc.Parent = b
		b.MouseButton1Click:Connect(d.fn)
	end

	task.delay(2.2, function()
		danmaku("移动端面板已启用 ｜ 可拖动 ｜ Mobile panel enabled")
	end)
end

print("✅ TSB M1 特效脚本 v24 已加载（可打假人 + 语言菜单 + 移动端适配）")
print("⚔️ 宿傩斩：M1横斩 / M2纵劈 / M3斜X斩 / M4「解」空间斩")
print("💥 TF2暴击：M1横十字 / M2竖十字 / M3斜十字 / M4全向金色爆发")
print("🕳️ 边狱巴士：横向崩 / 纵向崩 / 斜向崩 / M4完全坍缩")
print("🐱 猫娘打击：M1横爪+胡须 / M2竖爪+猫耳 / M3斜X双爪+铃铛 / M4狂乱连击")
print("💗 爱心打击：M1横斩+心形描边 / M2竖劈+心形描边 / M3斜X+双心+丘比特箭 / M4丘比特之心")
print("👊 虎杖打击：M1直拳 / M2上勾拳 / M3卍踢 / M4径庭拳(双段) / 击杀=黑闪(6段重制)")
print("🖤 黑闪：咒力收束 → 全屏闪白 → 空间撕裂 → 黑红闪电网 → 冲击波 → 余波")
print("📏 虎杖线条 thick=1.9（约原版 5.3 倍粗），其他风格不受影响")
print("🔊 虎杖音效：M1~M4 各段不同 | 击杀放虎杖日语「黒閃」18466149472")
print("🔊 黒閃备胎：81528757439622（日语）| 12764933067 / 9114314398（纯打击音）")
print("🚫 v22：已取消「普攻打死 → 黑闪 + 音效」，虎杖现在只有一技能击杀才出黑闪")
print("🎯 v23：可打假人已开启（CONFIG.HIT_DUMMY），假人与玩家共用同一套命中判定")
print("🌐 v23：开场弹幕 script by golden + 语言菜单（Chinese / English / Vietnamese）")
print("🎬 夜神月大笑：7473415590（选完语言 1 秒后播放｜按 H 试听）")
print("📱 移动端：检测到手机会自动启用虚拟按钮面板（可拖动）；PC 不受影响")
print("💥 一技能击杀：连打四记大量黑闪 + 独立音效（Y 试听）")
print("⌨️ G 切风格 | R 听普攻音 | T 听终结音 | Y 听一技能击杀音")

