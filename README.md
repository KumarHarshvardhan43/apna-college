# apna-college
local screenshotCount = 0  -- Track screenshot count
local downPressesPerImage = 18  -- Adjusted based on image height

-- Get built-in display center
local function getBuiltInScreenCenter()
  local screens = hs.screen.allScreens()
  for _, screen in ipairs(screens) do
    if screen:name():lower():find("built%-in") then
      local frame = screen:frame()
      return {
        x = frame.x + frame.w / 2,
        y = frame.y + frame.h / 2
      }
    end
  end
  -- Fallback to primary screen
  local primary = hs.screen.primaryScreen():frame()
  return {
    x = primary.x + primary.w / 2,
    y = primary.y + primary.h / 2
  }
end

-- Focus OneNote and paste at current caret position
local function activateOneNoteAndPaste()
  hs.application.launchOrFocus("Microsoft OneNote")
  hs.timer.doAfter(0.5, function()
    hs.eventtap.keyStroke({"cmd"}, "v") -- Just paste at the caret
    hs.timer.doAfter(0.3, function()
      hs.alert("✅ Screenshot pasted at position " .. (screenshotCount + 1))
      screenshotCount = screenshotCount + 1
      returnToBuiltInDisplay()
    end)
  end)
end

-- Move cursor back to built-in display
local function returnToBuiltInDisplay()
  local point = getBuiltInScreenCenter()
  hs.mouse.setAbsolutePosition(point)
end

-- Capture screenshot using AppleScript
local function captureScreenshot()
  return hs.osascript.applescript([[
    do shell script "screencapture -c"
    return true
  ]])
end

-- Watch OneNote activation to reset position
local function watchForOneNoteActivation()
  hs.application.watcher.new(function(appName, eventType)
    if appName == "Microsoft OneNote" and eventType == hs.application.watcher.activated then
      screenshotCount = 0
      hs.alert("🔄 Reset screenshot sequence")
    end
  end):start()
end

watchForOneNoteActivation()

-- Main screenshot bind
hs.hotkey.bind({}, "`", function()
  hs.alert("📸 Taking screenshot...")
  hs.pasteboard.clearContents()

  if not captureScreenshot() then
    hs.alert("❌ Screenshot failed!")
    return
  end

  local function waitForImage(attempts)
    attempts = attempts or 0
    if hs.pasteboard.readImage() then
      activateOneNoteAndPaste()
    elseif attempts < 20 then
      hs.timer.doAfter(0.15, function() waitForImage(attempts + 1) end)
    else
      hs.alert("❌ Timed out waiting for screenshot!")
    end
  end

  waitForImage()
end)

-- Manual reset with Shift + `
hs.hotkey.bind({"shift"}, "`", function()
  screenshotCount = 0
  hs.alert("🔄 Manually reset screenshot sequence")
end)

-- Show startup status
hs.alert.show("🟢 Screenshot automation ready\nCurrent sequence: " .. screenshotCount)

-- Switch to text mode in OneNote when pressing Ctrl + Cmd
hs.hotkey.bind({"ctrl", "cmd"}, "t", function()
  local point = {x = 28, y = 93}
  hs.mouse.setAbsolutePosition(point)
  hs.eventtap.leftClick()
  hs.alert("✏️ Switched to Text Mode")
end)
-- Focus back to lecture screen (built-in display) and left-click
hs.hotkey.bind({"ctrl", "alt", "cmd"}, "v", function()
  local point = getBuiltInScreenCenter()
  hs.mouse.setAbsolutePosition(point)
  hs.eventtap.leftClick(point)
  hs.alert("🎯")
end)
