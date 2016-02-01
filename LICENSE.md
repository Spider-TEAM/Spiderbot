
local function isBotAllowed (userId, chatId)
  local hash = 'anti-bot:allowed:'..chatId..':'..userId
  local banned = redis:get(hash)
  return banned
end

local function allowBot (userId, chatId)
  local hash = 'anti-bot:allowed:'..chatId..':'..userId
  redis:set(hash, true)
end

local function disallowBot (userId, chatId)
  local hash = 'anti-bot:allowed:'..chatId..':'..userId
  redis:del(hash)
end

-- Is anti-bot enabled on chat
local function isAntiBotEnabled (chatId)
  local hash = 'anti-bot:enabled:'..chatId
  local enabled = redis:get(hash)
  return enabled
end

local function enableAntiBot (chatId)
  local hash = 'anti-bot:enabled:'..chatId
  redis:set(hash, true)
end

local function disableAntiBot (chatId)
  local hash = 'anti-bot:enabled:'..chatId
  redis:del(hash)
end

local function isABot (user)
  -- Flag its a bot 0001000000000000
  local binFlagIsBot = 4096
  local result = bit32.band(user.flags, binFlagIsBot)
  return result == binFlagIsBot
end

local function kickUser(userId, chatId)
  local chat = 'chat#id'..chatId
  local user = 'user#id'..userId
  chat_del_user(chat, user, function (data, success, result)
    if success ~= 1 then
      print('I can\'t kick '..data.user..' but should be kicked')
    end
  end, {chat=chat, user=user})
end

local function run (msg, matches)
  -- We wont return text if is a service msg
  if matches[1] ~= 'chat_add_user' and matches[1] ~= 'chat_add_user_link' then
    if msg.to.type ~= 'chat' then
      return 'Anti-flood works only on channels'
    end
  end

  local chatId = msg.to.id
  if matches[1] == 'enable' then
    enableAntiBot(chatId)
    return 'Anti-bot enabled on this chat'
  end
  if matches[1] == 'disable' then
    disableAntiBot(chatId)
    return 'Anti-bot disabled on this chat'
  end
  if matches[1] == 'allow' then
    local userId = matches[2]
    allowBot(userId, chatId)
    return 'Bot '..userId..' allowed'
  end
  if matches[1] == 'disallow' then
    local userId = matches[2]
    disallowBot(userId, chatId)
    return 'Bot '..userId..' disallowed'
  end
  if matches[1] == 'chat_add_user' or matches[1] == 'chat_add_user_link' then
    local user = msg.action.user or msg.from
    if isABot(user) then
      print('It\'s a bot!')
      if isAntiBotEnabled(chatId) then
        print('Anti bot is enabled')
        local userId = user.id
        if not isBotAllowed(userId, chatId) then
          kickUser(userId, chatId)
        else
          print('This bot is allowed')
        end
      end
    end
  end
end

return {
  description = 'When bot enters group kick it.',
  usage = {
    '!antibot enable: Enable Anti-bot on current chat',
    '!antibot disable: Disable Anti-bot on current chat',
    '!antibot allow <botId>: Allow <botId> on this chat',
    '!antibot disallow <botId>: Disallow <botId> on this chat'
  },
  patterns = {
    '^!antibot (allow) (%d+)$',
    '^!antibot (disallow) (%d+)$',
    '^!antibot (enable)$',
    '^!antibot (disable)$',
    '^!!tgservice (chat_add_user)$',
    '^!!tgservice (chat_add_user_link)$'
  },
  run = run
}
local NUM_MSG_MAX = 4 -- Max number of messages per TIME_CHECK seconds
local TIME_CHECK = 2

local function kick_user(user_id, chat_id)
  local chat = 'chat#id'..chat_id
  local user = 'user#id'..user_id
  chat_del_user(chat, user, function (data, success, result)
    if success ~= 1 then
      local text = 'I can\'t kick '..data.user..' but should be kicked'
      send_msg(data.chat, '', ok_cb, nil)
    end
  end, {chat=chat, user=user})
end

local function run (msg, matches)
  if msg.to.type ~= 'chat' then
    return 'Anti-flood works only on channels'
  else
    local chat = msg.to.id
    local hash = 'anti-flood:enabled:'..chat
    if matches[1] == 'enable' then
      redis:set(hash, true)
      return 'Anti-flood enabled on chat'
    end
    if matches[1] == 'disable' then
      redis:del(hash)
      return 'Anti-flood disabled on chat'
    end
  end
end

local function pre_process (msg)
  -- Ignore service msg
  if msg.service then
    print('Service message')
    return msg
  end

  local hash_enable = 'anti-flood:enabled:'..msg.to.id
  local enabled = redis:get(hash_enable)

  if enabled then
    print('anti-flood enabled')
    -- Check flood
    if msg.from.type == 'user' then
      -- Increase the number of messages from the user on the chat
      local hash = 'anti-flood:'..msg.from.id..':'..msg.to.id..':msg-num'
      local msgs = tonumber(redis:get(hash) or 0)
      if msgs > NUM_MSG_MAX then
        local receiver = get_receiver(msg)
        local user = msg.from.id
        local text = 'User '..user..' is flooding'
        local chat = msg.to.id

        send_msg(receiver, text, ok_cb, nil)
        if msg.to.type ~= 'chat' then
          print("Flood in not a chat group!")
        elseif user == tostring(our_id) then
          print('I won\'t kick myself')
        elseif is_sudo(msg) then
          print('I won\'t kick an admin!')
        else
          -- Ban user
          -- TODO: Check on this plugin bans
          local bhash = 'banned:'..msg.to.id..':'..msg.from.id
          redis:set(bhash, true)
          kick_user(user, chat)
        end
        msg = nil
      end
      redis:setex(hash, TIME_CHECK, msgs+1)
    end
  end
  return msg
end

return {
  description = 'Plugin to kick flooders from group.',
  usage = {},
  patterns = {
    '^!antiflood (enable)$',
    '^!antiflood (disable)$'
  },
  run = run,
  privileged = true,
  pre_process = pre_process
}

--An empty table for solving multiple kicking problem(thanks to @topkecleon )
kicktable = {}

do

local TIME_CHECK = 2 -- seconds
local data = load_data(_config.moderation.data)
-- Save stats, ban user
local function pre_process(msg)
  -- Ignore service msg
  if msg.service then
    return msg
  end
  if msg.from.id == our_id then
    return msg
  end
  --Load moderation data
  local data = load_data(_config.moderation.data)
  if data[tostring(msg.to.id)] then
    --Check if flood is one or off
    if data[tostring(msg.to.id)]['settings']['flood'] == 'no' then
      return msg
    end
  end
  -- Save user on Redis
  if msg.from.type == 'user' then
    local hash = 'user:'..msg.from.id
    print('Saving user', hash)
    if msg.from.print_name then
      redis:hset(hash, 'print_name', msg.from.print_name)
    end
    if msg.from.first_name then
      redis:hset(hash, 'first_name', msg.from.first_name)
    end
    if msg.from.last_name then
      redis:hset(hash, 'last_name', msg.from.last_name)
    end
  end

  -- Save stats on Redis
  if msg.to.type == 'chat' then
    -- User is on chat
    local hash = 'chat:'..msg.to.id..':users'
    redis:sadd(hash, msg.from.id)
  end



  -- Total user msgs
  local hash = 'msgs:'..msg.from.id..':'..msg.to.id
  redis:incr(hash)

  -- Check flood
  if msg.from.type == 'user' then
    local hash = 'user:'..msg.from.id..':msgs'
    local msgs = tonumber(redis:get(hash) or 0)
    local data = load_data(_config.moderation.data)
    local NUM_MSG_MAX = 5
    if data[tostring(msg.to.id)] then
      if data[tostring(msg.to.id)]['settings']['flood_msg_max'] then
        NUM_MSG_MAX = tonumber(data[tostring(msg.to.id)]['settings']['flood_msg_max'])--Obtain group flood sensitivity
      end
    end
    local max_msg = NUM_MSG_MAX * 1
    if msgs > max_msg then
      local user = msg.from.id
      -- Ignore mods,owner and admins
      if is_momod(msg) then 
        return msg
      end
      local chat = msg.to.id
      local user = msg.from.id
      -- Return end if user was kicked before
      if kicktable[user] == true then
        return
      end
      kick_user(user, chat)
      local name = user_print_name(msg.from)
      --save it to log file
      savelog(msg.to.id, name.." ["..msg.from.id.."] spammed and kicked ! ")
      -- incr it on redis
      local gbanspam = 'gban:spam'..msg.from.id
      redis:incr(gbanspam)
      local gbanspam = 'gban:spam'..msg.from.id
      local gbanspamonredis = redis:get(gbanspam)
      --Check if user has spammed is group more than 4 times  
      if gbanspamonredis then
        if tonumber(gbanspamonredis) ==  4 and not is_owner(msg) then
          --Global ban that user
          banall_user(msg.from.id)
          local gbanspam = 'gban:spam'..msg.from.id
          --reset the counter
          redis:set(gbanspam, 0)
          local username = " "
          if msg.from.username ~= nil then
            username = msg.from.username
          end
          local name = user_print_name(msg.from)
          --Send this to that chat
          send_large_msg("chat#id"..msg.to.id, "User [ "..name.." ]"..msg.from.id.." Globally banned (spamming)")
          local log_group = 1 --set log group caht id
          --send it to log group
          send_large_msg("chat#id"..log_group, "User [ "..name.." ] ( @"..username.." )"..msg.from.id.." Globally banned from ( "..msg.to.print_name.." ) [ "..msg.to.id.." ] (spamming)")
        end
      end
      kicktable[user] = true
      msg = nil
    end
    redis:setex(hash, TIME_CHECK, msgs+1)
  end
  return msg
end

local function cron()
  --clear that table on the top of the plugins
	kicktable = {}
end

return {
  patterns = {},
  cron = cron,
  pre_process = pre_process
}

end
-- Function reference: http://mathjs.org/docs/reference/functions/categorical.html

local function mathjs(exp)
  local url = 'http://api.mathjs.org/v1/'
  url = url..'?expr='..URL.escape(exp)
  local b,c = http.request(url)
  local text = nil
  if c == 200 then
    text = 'Result: '..b
  
  elseif c == 400 then
    text = b
  else
    text = 'Unexpected error\n'
      ..'Is api.mathjs.org up?'
  end
  return text
end

local function run(msg, matches)
  return mathjs(matches[1])
end

return {
  description = "Calculate math expressions with mathjs API",
  usage = "!calc [expression]: evaluates the expression and sends the result.",
  patterns = {
    "^!calc (.*)$"
  },
  run = run
}
local function callback(extra, success, result)
  if success then
    print('File downloaded to:', result)
  else
    print('Error downloading: '..extra)
  end
end

local function run(msg, matches)
  if msg.media then
    if msg.media.type == 'document' then
      load_document(msg.id, callback, msg.id)
    end
    if msg.media.type == 'photo' then
      load_photo(msg.id, callback, msg.id)
    end
    if msg.media.type == 'video' then
      load_video(msg.id, callback, msg.id)
    end
    if msg.media.type == 'audio' then
      load_audio(msg.id, callback, msg.id)
    end
  end
end

local function pre_process(msg)
  if not msg.text and msg.media then
    msg.text = '['..msg.media.type..']'
  end
  return msg
end

return {
  description = "When bot receives a media msg, download the media.",
  usage = "",
  run = run,
  patterns = {
    '%[(document)%]',
    '%[(photo)%]',
    '%[(video)%]',
    '%[(audio)%]'
  },
  pre_process = pre_process
}do
local mime = require("mime")

local google_config = load_from_file('data/google.lua')
local cache = {}

--[[
local function send_request(url)
  local t = {}
  local options = {
    url = url,
    sink = ltn12.sink.table(t),
    method = "GET"
  }
  local a, code, headers, status = http.request(options)
  return table.concat(t), code, headers, status
end]]--

local function get_google_data(text)
  local url = "http://ajax.googleapis.com/ajax/services/search/images?"
  url = url.."v=1.0&rsz=5"
  url = url.."&q="..URL.escape(text)
  url = url.."&imgsz=small|medium|large"
  if google_config.api_keys then
    local i = math.random(#google_config.api_keys)
    local api_key = google_config.api_keys[i]
    if api_key then
      url = url.."&key="..api_key
    end
  end

  local res, code = http.request(url)
  
  if code ~= 200 then 
    print("HTTP Error code:", code)
    return nil 
  end
  
  local google = json:decode(res)
  return google
end

-- Returns only the useful google data to save on cache
local function simple_google_table(google)
  local new_table = {}
  new_table.responseData = {}
  new_table.responseDetails = google.responseDetails
  new_table.responseStatus = google.responseStatus
  new_table.responseData.results = {}
  local results = google.responseData.results
  for k,result in pairs(results) do
    new_table.responseData.results[k] = {}
    new_table.responseData.results[k].unescapedUrl = result.unescapedUrl
    new_table.responseData.results[k].url = result.url
  end
  return new_table
end

local function save_to_cache(query, data)
  -- Saves result on cache
  if string.len(query) <= 7 then
    local text_b64 = mime.b64(query)
    if not cache[text_b64] then
      local simple_google = simple_google_table(data)
      cache[text_b64] = simple_google
    end
  end
end

local function process_google_data(google, receiver, query)
  if google.responseStatus == 403 then
    local text = 'ERROR: Reached maximum searches per day'
    send_msg(receiver, text, ok_cb, false)

  elseif google.responseStatus == 200 then
    local data = google.responseData

    if not data or not data.results or #data.results == 0 then
      local text = 'Image not found.'
      send_msg(receiver, text, ok_cb, false)
      return false
    end

    -- Random image from table
    local i = math.random(#data.results)
    local url = data.results[i].unescapedUrl or data.results[i].url
    local old_timeout = http.TIMEOUT or 10
    http.TIMEOUT = 5
    send_photo_from_url(receiver, url)
    http.TIMEOUT = old_timeout

    save_to_cache(query, google)
  
  else
    local text = 'ERROR!'
    send_msg(receiver, text, ok_cb, false)
  end
end

function run(msg, matches)
  local receiver = get_receiver(msg)
  local text = matches[1]
  local text_b64 = mime.b64(text)
  local cached = cache[text_b64]
  if cached then
    process_google_data(cached, receiver, text)
  else
    local data = get_google_data(text)    
    process_google_data(data, receiver, text)
  end
end

return {
  description = "Search image with Google API and sends it.", 
  usage = "!img [term]: Random search an image with Google API.",
  patterns = {
    "^!img (.*)$"
  }, 
  run = run
}

end
local function kick_user(user_id, chat_id)
  local chat = 'chat#id'..chat_id
  local user = 'user#id'..user_id
  chat_del_user(chat, user, function (data, success, result)
    if success ~= 1 then
      send_msg(data.chat, 'Error while kicking user', ok_cb, nil)
    end
  end, {chat=chat, user=user})
end

local function run (msg, matches)
  local user = msg.from.id
  local chat = msg.to.id

  if msg.to.type ~= 'chat' then
    return "Not a chat group!"
  elseif user == tostring(our_id) then
    --[[ A robot must protect its own existence as long as such protection does
    not conflict with the First or Second Laws. ]]--
    return "I won't kick myself!"
  elseif is_sudo(msg) then
    return "I won't kick an admin!"
  else
    kick_user(user, chat)
  end
end

return {
  description = "Bot kicks user",
  usage = {
    "!kickme"
  },
  patterns = {
    "^!kickme$"
  },
  run = run
}
do

local function parsed_url(link)
  local parsed_link = URL.parse(link)
  local parsed_path = URL.parse_path(parsed_link.path)
  return parsed_path[2]
end

function run(msg, matches)
  local hash = parsed_url(matches[1])   
  join = import_chat_link(hash,ok_cb,false)
end


return {
  description = "Invite me into a group chat", 
  usage = "!inviteme [invite link]",
  patterns = {
    "^!inviteme (.*)$"
  }, 
  run = run
}

end
do

-- Returns the key (index) in the config.enabled_plugins table
local function plugin_enabled( name )
  for k,v in pairs(_config.enabled_plugins) do
    if name == v then
      return k
    end
  end
  -- If not found
  return false
end

-- Returns true if file exists in plugins folder
local function plugin_exists( name )
  for k,v in pairs(plugins_names()) do
    if name..'.lua' == v then
      return true
    end
  end
  return false
end

local function list_plugins(only_enabled)
  local text = ''
  for k, v in pairs( plugins_names( )) do
    --  ✔ enabled, ❌ disabled
    local status = '❌'
    -- Check if is enabled
    for k2, v2 in pairs(_config.enabled_plugins) do
      if v == v2..'.lua' then 
        status = '✔' 
      end
    end
    if not only_enabled or status == '✔' then
      -- get the name
      v = string.match (v, "(.*)%.lua")
      text = text..v..'  '..status..'\n'
    end
  end
  return text
end

local function reload_plugins( )
  plugins = {}
  load_plugins()
  return list_plugins(true)
end


local function enable_plugin( plugin_name )
  print('checking if '..plugin_name..' exists')
  -- Check if plugin is enabled
  if plugin_enabled(plugin_name) then
    return 'Plugin '..plugin_name..' is enabled'
  end
  -- Checks if plugin exists
  if plugin_exists(plugin_name) then
    -- Add to the config table
    table.insert(_config.enabled_plugins, plugin_name)
    print(plugin_name..' added to _config table')
    save_config()
    -- Reload the plugins
    return reload_plugins( )
  else
    return 'Plugin '..plugin_name..' does not exists'
  end
end

local function disable_plugin( name, chat )
  -- Check if plugins exists
  if not plugin_exists(name) then
    return 'Plugin '..name..' does not exists'
  end
  local k = plugin_enabled(name)
  -- Check if plugin is enabled
  if not k then
    return 'Plugin '..name..' not enabled'
  end
  -- Disable and reload
  table.remove(_config.enabled_plugins, k)
  save_config( )
  return reload_plugins(true)    
end

local function disable_plugin_on_chat(receiver, plugin)
  if not plugin_exists(plugin) then
    return "Plugin doesn't exists"
  end

  if not _config.disabled_plugin_on_chat then
    _config.disabled_plugin_on_chat = {}
  end

  if not _config.disabled_plugin_on_chat[receiver] then
    _config.disabled_plugin_on_chat[receiver] = {}
  end

  _config.disabled_plugin_on_chat[receiver][plugin] = true

  save_config()
  return 'Plugin '..plugin..' disabled on this chat'
end

local function reenable_plugin_on_chat(receiver, plugin)
  if not _config.disabled_plugin_on_chat then
    return 'There aren\'t any disabled plugins'
  end

  if not _config.disabled_plugin_on_chat[receiver] then
    return 'There aren\'t any disabled plugins for this chat'
  end

  if not _config.disabled_plugin_on_chat[receiver][plugin] then
    return 'This plugin is not disabled'
  end

  _config.disabled_plugin_on_chat[receiver][plugin] = false
  save_config()
  return 'Plugin '..plugin..' is enabled again'
end

local function run(msg, matches)
  -- Show the available plugins 
  if matches[1] == '!plugins' then
    return list_plugins()
  end

  -- Re-enable a plugin for this chat
  if matches[1] == 'enable' and matches[3] == 'chat' then
    local receiver = get_receiver(msg)
    local plugin = matches[2]
    print("enable "..plugin..' on this chat')
    return reenable_plugin_on_chat(receiver, plugin)
  end

  -- Enable a plugin
  if matches[1] == 'enable' then
    local plugin_name = matches[2]
    print("enable: "..matches[2])
    return enable_plugin(plugin_name)
  end

  -- Disable a plugin on a chat
  if matches[1] == 'disable' and matches[3] == 'chat' then
    local plugin = matches[2]
    local receiver = get_receiver(msg)
    print("disable "..plugin..' on this chat')
    return disable_plugin_on_chat(receiver, plugin)
  end

  -- Disable a plugin
  if matches[1] == 'disable' then
    print("disable: "..matches[2])
    return disable_plugin(matches[2])
  end

  -- Reload all the plugins!
  if matches[1] == 'reload' then
    return reload_plugins(true)
  end
end

return {
  description = "Plugin to manage other plugins. Enable, disable or reload.", 
  usage = {
    "!plugins: list all plugins.", 
    "!plugins enable [plugin]: enable plugin.",
    "!plugins disable [plugin]: disable plugin.",
    "!plugins disable [plugin] chat: disable plugin only this chat.",
    "!plugins reload: reloads all plugins." },
  patterns = {
    "^!plugins$",
    "^!plugins? (enable) ([%w_%.%-]+)$",
    "^!plugins? (disable) ([%w_%.%-]+)$",
    "^!plugins? (enable) ([%w_%.%-]+) (chat)",
    "^!plugins? (disable) ([%w_%.%-]+) (chat)",
    "^!plugins? (reload)$" },
  run = run,
  privileged = true
}

endlocal add_user_cfg = load_from_file('data/add_user_cfg.lua')

local function template_add_user(base, to_username, from_username, chat_name, chat_id)
   base = base or ''
   to_username = '@' .. (to_username or '')
   from_username = '@' .. (from_username or '')
   chat_name = chat_name or ''
   chat_id = "chat#id" .. (chat_id or '')
   if to_username == "@" then
      to_username = ''
   end
   if from_username == "@" then
      from_username = ''
   end
   base = string.gsub(base, "{to_username}", to_username)
   base = string.gsub(base, "{from_username}", from_username)
   base = string.gsub(base, "{chat_name}", chat_name)
   base = string.gsub(base, "{chat_id}", chat_id)
   return base
end

function chat_new_user_link(msg)
   local pattern = add_user_cfg.initial_chat_msg
   local to_username = msg.from.username
   local from_username = '[link](@' .. (msg.action.link_issuer.username or '') .. ')'
   local chat_name = msg.to.print_name
   local chat_id = msg.to.id
   pattern = template_add_user(pattern, to_username, from_username, chat_name, chat_id)
   if pattern ~= '' then
      local receiver = get_receiver(msg)
      send_msg(receiver, pattern, ok_cb, false)
   end
end

function chat_new_user(msg)
   local pattern = add_user_cfg.initial_chat_msg
   local to_username = msg.action.user.username
   local from_username = msg.from.username
   local chat_name = msg.to.print_name
   local chat_id = msg.to.id
   pattern = template_add_user(pattern, to_username, from_username, chat_name, chat_id)
   if pattern ~= '' then
      local receiver = get_receiver(msg)
      send_msg(receiver, pattern, ok_cb, false)
   end
end


local function run(msg, matches)
   if not msg.service then
      return "Are you trying to troll me?"
   end
   if matches[1] == "chat_add_user" then
      chat_new_user(msg)
   elseif matches[1] == "chat_add_user_link" then
      chat_new_user_link(msg)
   end
end

return {
   description = "Service plugin that sends a custom message when an user enters a chat.",
   usage = "",
   patterns = {
      "^!!tgservice (chat_add_user)$",
      "^!!tgservice (chat_add_user_link)$"
   },
   run = run
}
local function run(msg, matches)
   -- avoid this plugins to process user messages
   if not msg.service then
      -- return "Are you trying to troll me?"
      return nil
   end
   print("Service message received: " .. matches[1])
end


return {
   description = "Template for service plugins",
   usage = "",
   patterns = {
      "^!!tgservice (.*)$" -- Do not use the (.*) match in your service plugin
   },
   run = run
}
local helpers = require "OAuth.helpers"

local base = 'https://screenshotmachine.com/'
local url = base .. 'processor.php'

local function get_webshot_url(param)
   local response_body = {}
   local request_constructor = {
      url = url,
      method = "GET",
      sink = ltn12.sink.table(response_body),
      headers = {
         referer = base,
         dnt = "1",
         origin = base,
         ["User-Agent"] = "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.101 Safari/537.36"
      },
      redirect = false
   }

   local arguments = {
      urlparam = param,
      size = "FULL"
   }

   request_constructor.url = url .. "?" .. helpers.url_encode_arguments(arguments)

   local ok, response_code, response_headers, response_status_line = https.request(request_constructor)
   if not ok or response_code ~= 200 then
      return nil
   end

   local response = table.concat(response_body)
   return string.match(response, "href='(.-)'")
end

local function run(msg, matches)
   local find = get_webshot_url(matches[1])
   if find then
      local imgurl = base .. find
      local receiver = get_receiver(msg)
      send_photo_from_url(receiver, imgurl)
   end
end


return {
   description = "Send an screenshot of a website.",
   usage = {
      "!webshot [url]: Take an screenshot of the web and send it back to you."
   },
   patterns = {
      "^!webshot (https?://[%w-_%.%?%.:/%+=&]+)$",
   },
   run = run
}
