---
title: 我完全不会 Lua，但我用嘴做了一个 OCR
slug: build-a-macos-vision-ocr-with-ai-agent-by-voice
tags: [Life, AI, Mac]
date: 2026-04-09T22:35:00+08:00
---

我前几天折腾了一个自己挺喜欢的小东西：在 macOS 上按一个快捷键，框选一块区域，直接把图里的字识别出来，然后塞进剪贴板。<!--more-->

功能不复杂，真正有意思的是另一件事：我几乎不会 Lua，这个东西还是做出来了。而且我全程基本没怎么敲代码，主要靠嘴说。准确点说，是我把需求、偏好、报错现象讲给 Codex，它去查、去试、去改，我负责盯方向和验收。

这篇不想再空讲“AI 改变生产方式”这种大话了，没劲。就讲三件事：

1. 这个 OCR 最后是怎么实现的
2. 中间我为什么放弃了几条看起来更直觉的路
3. 最后能直接用的 Lua 脚本长什么样

## 我想要的，其实不是 OCR

我想要的是一个顺手的动作。

按下 `Ctrl + A`，框一下，文字就进剪贴板。没有额外窗口，没有一堆配置，没有“请把图片拖到这里”，也没有成功以后再弹个提示来打断我。

这点很重要。因为很多工具的问题不在“不能用”，而在“每次用都要被它打扰一下”。

所以我一开始就把要求卡得比较死：

- 快捷键触发
- 支持框选截图
- 尽量只用 macOS 自带能力
- 不装额外 OCR 软件
- 最好只有一个 Lua 主入口文件
- 成功时别弹窗，失败再说

你看，真正难的不是“OCR 能不能做”，而是把链路收得够短。

## 一开始最容易想到的路，反而不是我想要的

最直觉的方案当然是装个 `tesseract` 之类的东西，截图之后把图片丢过去识别。

这条路能不能走？当然能。

但我很快就不想走了。原因也很简单：我只是想做个平时自己顺手用的小 OCR，不想为了这点事再引一套额外依赖。你现在觉得装一次也就装一次，等你哪天把它分享给别人，或者自己换台机器，就知道这种“也不复杂”的依赖到底有多烦。

所以目标很快就变成了：

`Hammerspoon 负责交互 + macOS Vision 负责识别`

这条路的好处是干净。系统本来就有的能力，能用就别再往外搬。

## 真正麻烦的地方，不是识别，而是桥接

如果只是讲原理，这个东西其实只有三段：

1. Hammerspoon 绑定快捷键，调用系统截图
2. 截图落到临时文件
3. 调用 macOS 自带的 Vision 做文字识别，再把结果写进剪贴板

问题出在第三步。

我一开始以为，既然 Hammerspoon 里跑的是 Lua，那能不能直接在 Lua 里把 Vision 框架调起来？后来确认，不行，至少在现成这套环境里不行。Hammerspoon 的 Lua 很适合做调度，但不是那种你想调什么 Objective-C 框架就能直接调什么的运行时。

这时候如果硬拧，就会开始出现一种很烦的局面：Lua 一点，Shell 一点，AppleScript 一点，再夹点别的东西，最后能跑是能跑，但结构越来越丑。

我中间最不满意的也就是这个。

后来把思路拧回来以后，事情反而顺了：

- Lua 继续当主入口
- Lua 负责截图、权限判断、临时文件、剪贴板
- Lua 动态生成一段临时 AppleScript
- `osascript` 执行这段脚本
- AppleScriptObjC 去调 Vision

也就是说，表面上我还是只维护一个 `ocr.lua`，真正的 OCR 识别交给系统原生框架。这个结构我能接受，因为它至少是收敛的。

## 这几个细节，不处理的话用起来会很烦

代码真正写出来以后，后面花时间的地方反而是一些小事。

### 1. 屏幕录制权限得先拦

系统截图这一步如果没有权限，用户看到的体验会很差。与其等它报个莫名其妙的错，不如先检查一次 `screenRecordingState`，没权限就直接提示去系统设置里开。

### 2. 成功别弹窗

我一开始的版本，识别成功之后会弹一个提示。这个设计我后来很快就否了。

原因很朴素：成功本来就是正常路径。正常路径还出来刷存在感，这种交互我自己都嫌烦。最后只保留了失败和取消时的提示，成功就安静地把文字写进剪贴板。

### 3. 识别结果直接进剪贴板

如果 OCR 完了还得再点一次复制，那这个工具就已经输了。快捷动作之所以有价值，就是因为它能少一次手。

### 4. 识别语言别乱开

我最后只给了这几个识别语言：

```text
zh-Hans
zh-Hant
en-US
```

我平时用它，中文和英文够了。语言开太多，有时候反而容易给自己添乱。

## 最终脚本

脚本现在就在我本机的 `~/.hammerspoon/ocr.lua` 里，能直接用。核心逻辑就是下面这份。

把它放到 `~/.hammerspoon/ocr.lua`：

```lua
local hs = hs
local hotkey = hs.hotkey
local task = hs.task
local alert = hs.alert
local pasteboard = hs.pasteboard
local timer = hs.timer
local screenRecordingState = hs.screenRecordingState

local ocr = {}

local screenshotCommand = "/usr/sbin/screencapture"
local osascriptCommand = "/usr/bin/osascript"

local function trim(text)
  return (text or ""):gsub("^%s+", ""):gsub("%s+$", "")
end

local function stripProcessTrailingNewline(text)
  return (text or ""):gsub("\r?\n$", "")
end

local function removeFile(path)
  if path and #path > 0 then
    os.remove(path)
  end
end

local function tempPath(suffix)
  return os.tmpname() .. suffix
end

local function copyOCRResult(text)
  pasteboard.setContents(text)
end

local function appleScriptQuote(value)
  return '"' .. tostring(value):gsub("\\", "\\\\"):gsub('"', '\\"') .. '"'
end

local function writeFile(path, content)
  local file, err = io.open(path, "w")
  if not file then
    return nil, err
  end

  file:write(content)
  file:close()
  return true
end

local function buildOCRAppleScript(imagePath)
  local quotedPath = appleScriptQuote(imagePath)
  return table.concat({
    'use framework "Foundation"',
    'use framework "Vision"',
    'use scripting additions',
    "",
    "set theFile to current application's |NSURL|'s fileURLWithPath:" .. quotedPath,
    "set requestHandler to current application's VNImageRequestHandler's alloc()'s initWithURL:theFile options:(missing value)",
    "set theRequest to current application's VNRecognizeTextRequest's alloc()'s init()",
    "theRequest's setRecognitionLevel:(current application's VNRequestTextRecognitionLevelAccurate)",
    'theRequest\'s setRecognitionLanguages:{"zh-Hans", "zh-Hant", "en-US"}',
    "theRequest's setUsesLanguageCorrection:false",
    "requestHandler's performRequests:(current application's NSArray's arrayWithObject:(theRequest)) |error|:(missing value)",
    "set theResults to theRequest's results()",
    'if theResults is missing value then return ""',
    "set theArray to current application's NSMutableArray's new()",
    "repeat with aResult in theResults",
    "    set theCandidates to aResult's topCandidates:1",
    "    if (theCandidates's |count|()) > 0 then",
    "        (theArray's addObject:(((theCandidates's objectAtIndex:0)'s |string|())))",
    "    end if",
    "end repeat",
    "return (theArray's componentsJoinedByString:linefeed) as text",
  }, "\n")
end

local function runAppleScriptOCR(imagePath, callback)
  local scriptPath = tempPath(".applescript")
  local ok, err = writeFile(scriptPath, buildOCRAppleScript(imagePath))
  if not ok then
    callback(nil, "无法创建临时 AppleScript: " .. tostring(err))
    return
  end

  local ocrTask = task.new(osascriptCommand, function(exitCode, stdout, stderr)
    removeFile(scriptPath)

    if exitCode ~= 0 then
      local message = trim(stderr)
      if message == "" then
        message = "系统 OCR 执行失败"
      end
      callback(nil, message)
      return
    end

    callback(stripProcessTrailingNewline(stdout), nil)
  end, {scriptPath})

  if not ocrTask then
    removeFile(scriptPath)
    callback(nil, "无法启动 osascript")
    return
  end

  ocrTask:start()
end

function ocr.runOnImage(imagePath)
  runAppleScriptOCR(imagePath, function(text, err)
    removeFile(imagePath)

    if not text then
      alert.show(err, 3)
      return
    end

    if text == "" then
      alert.show("没有识别到文本", 2)
      return
    end

    copyOCRResult(text)
  end)
end

function ocr.captureSelectionAndRecognize()
  if not screenRecordingState(false) then
    screenRecordingState(true)
    alert.show("请先给 Hammerspoon 开启屏幕录制权限，然后重新尝试 OCR", 3)
    return
  end

  local imagePath = tempPath(".png")
  task.new(screenshotCommand, function(exitCode, _, stderr)
    if exitCode ~= 0 then
      removeFile(imagePath)

      local message = trim(stderr)
      if not screenRecordingState(false) then
        message = "缺少屏幕录制权限，请在系统设置里允许 Hammerspoon"
      elseif message == "" then
        message = "截图已取消"
      end

      alert.show(message, 1.8)
      return
    end

    timer.doAfter(0.05, function()
      ocr.runOnImage(imagePath)
    end)
  end, {"-i", "-x", imagePath}):start()
end

local modifiers = {"ctrl"}
local triggerKey = "a"

if hotkey.assignable(modifiers, triggerKey) then
  hotkey.bind(modifiers, triggerKey, nil, function()
    ocr.captureSelectionAndRecognize()
  end)
else
  alert.show("Ctrl+" .. triggerKey .. " 已被系统占用，OCR 热键未绑定")
end

return ocr
```

然后在 `~/.hammerspoon/init.lua` 里加一句：

```lua
require("ocr")
```

重载 Hammerspoon 之后，按 `Ctrl + A` 就能用。

## 这份脚本到底干了什么

如果把上面一大段压成几句人话，其实就是：

1. 按 `Ctrl + A`
2. 调 `screencapture -i -x` 让你框选
3. 把截图存成临时 PNG
4. 现场生成一段 AppleScript
5. AppleScript 通过 Vision 识别文字
6. 把返回结果塞进剪贴板
7. 删掉临时图片和临时脚本

所以这东西虽然叫 `ocr.lua`，本质上其实是个调度器。真正干识别的是 Vision，真正把 Vision 调起来的是 AppleScriptObjC。

## 这次最让我有感觉的，不是 Lua

说到底，我这次不是学会了 Lua。

我只是第一次特别明确地感受到，很多过去“得先学会 X，才能开始做 Y”的事情，现在已经没那么绝对了。

当然，前提不是你可以什么都不懂。恰恰相反，你得更清楚自己到底要什么。

比如这次真正由我拍板的，不是某个 API 怎么写，而是这些事：

- 我不要第三方 OCR 依赖
- 我不要成功弹窗
- 我希望它最后还是一个单文件入口
- 哪条路虽然能跑，但结构太脏，我不要

这些判断以前也重要，只是以前你还得自己把所有实现细节一并扛下来。现在实现这部分，可以明显往 AI 那边分了。

这就是我为什么会说，这玩意儿是“用嘴做出来的”。

不是说我一点没参与。恰恰相反，我参与得很深，只是参与的位置变了。

## 最后

这篇原本只写了思路，没把脚本贴出来，确实差一口气。技术文讲到“这里其实能做”，结果不放代码，读者看完多半只会想一句：行吧，那你倒是把东西给我。

现在补上了，文章才算完整。

而且说实话，这个 OCR 本身没多大。真正让我上头的是另一件事：我不会 Lua，但我还是把它做出来了。这个感觉挺新鲜，也挺实在。
