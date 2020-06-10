local fs = require("filesystem")
local component = require("component")
local robot = component.robot
local nav = component.navigation
local geo = component.geolyzer
local modem = component.modem

local settings = {
  ["farm_type"] = "wheat",
  ["base"] = "Farm1",
  ["waypoint_radius"] = 16,
  ["sleep_time"] = 60
}

local cx, cy, cz = 0, 0, 0
local BASE

local sidesFront = 3
local sidesBack = 2
local sidesRight = 4
local sidesLeft = 5
local sidesDown = 0
local sidesUp = 1
local startDirection = ""
local currentDirection = ""

function getBase()
  local waypoints = nav.findWaypoints(settings.waypoint_radius)
  for i = 1, waypoints.n do
    if waypoints[i].label == settings.base then
      return { ["x"] = waypoints[i].position[1], ["y"] = waypoints[i].position[2], ["z"] = waypoints[i].position[3] }
    end
  end
end

function getFarmRows()
  local waypoints = nav.findWaypoints(settings.waypoint_radius)
  local rows = {}
  for i = 1, waypoints.n do
    if waypoints[i].label:find(settings.base .. "%-row%-") == 1 then
      local tmpRow = { ["name"] = waypoints[i].label, ["x"] = waypoints[i].position[1], ["y"] = waypoints[i].position[2], ["z"] = waypoints[i].position[3], ["length"] = waypoints[i].label:match(settings.base .. "%-row%-(%d+)") }
      table.insert(rows, tmpRow)
    end
  end
  return rows
end

function saveTable(tbl, filename)
  local charS, charE = "   ", "\n"
  local file, err
  if not filename then
    file =  { write = function(self, newstr) self.str = self.str .. newstr end, str = "" }
    charS, charE = "", ""
  elseif filename == true or filename == 1 then
    charS, charE, file = "", "", io.tmpfile()
  else
    file, err = io.open(filename, "w")
    if err then return _, err end
  end
  
  local tables, lookup = { tbl }, { [tbl] = 1 }
  file:write("return {" .. charE)
  for idx, t in ipairs(tables) do
    if filename and filename ~= true and filename ~= 1 then
      file:write("-- Table: {" .. idx .. "}" .. charE)
    end
    file:write("{" .. charE)
    local thandled = {}
    for i, v in ipairs(t) do
      thandled[i] = true
      if type(v) ~= "userdata" then
        if type(v) == "table" then
          if not lookup[v] then
            table.insert(tables, v)
            lookup[v] = #tables
          end
          file:write(charS .. "{" .. lookup[v] .. "}," .. charE)
        elseif type(v) == "function" then
          file:write(charS .. "loadstring(" .. exportstring(string.dump(v)) .. ")," .. charE)
        else
          local value = (type(v) == "string" and exportstring(v)) or tostring(v)
          file:write(charS .. value .. "," .. charE)
        end
      end
    end
    for i, v in pairs(t) do
      if (not thandled[i]) and type(v) ~= "userdata" then
        if type(i) == "table" then
          if not lookup[i] then
            table.insert(tables, i)
            lookup[i] = #tables
          end
          file:write(charS .. "[{" .. lookup[i] .. "}]=")
        else
          local index = (type(i) == "string" and "[" .. exportstring(i) .. "]") or string.format("[%d]", i)
          file:write(charS .. index .. "=")
        end
        if type(v) == "table" then
          if not lookup[v] then
            table.insert(tables, v)
            lookup[v] = #tables
          end
          file:write("{" .. lookup[v] .. "}," .. charE)
        elseif type(v) == "function" then
          file:write("loadstring(" .. exportstring(string.dump(v)) .. ")," .. charE)
        else
          local value =  (type(v) == "string" and exportstring(v)) or tostring(v)
          file:write(value .. "," .. charE)
        end
      end
    end
    file:write("}," .. charE)
  end
  file:write("}")
  if not filename then
    return file.str .. "--|"
  elseif filename == true or filename == 1 then
    file:seek ("set")
    return file:read("*a") .. "--|"
  else
    file:close()
    return 1
  end
end

function loadTable(sfile)
  local tables, err, _
  if string.sub(sfile, -3, -1) == "--|" then
    tables, err = loadstring(sfile)
  else
    tables, err = loadfile(sfile)
  end

  if err then return _, err end

  tables = tables()
  for idx = 1, #tables do
    local tolinkv, tolinki = {}, {}
    for i, v in pairs(tables[idx]) do
      if type(v) == "table" and tables[v[1]] then
        table.insert(tolinkv, { i, tables[v[1]] })
      end
      if type(i) == "table" and tables[i[1]] then
        table.insert(tolinki, { i, tables[i[1]] })
      end
    end
    for _, v in ipairs(tolinkv) do
      tables[idx][v[1]] = v[2]
    end
    for _, v in ipairs(tolinki) do
      tables[idx][v[2]], tables[idx][v[1]] =  tables[idx][v[1]], nil
    end
  end
  return tables[1]
end

function init()
  BASE = getBase()
  robot.move(sidesFront)
  local nBase = getBase()
  robot.move(sidesBack)

  print(BASE.x .. " - " .. BASE.y .. " - " .. BASE.z)
  print(nBase.x .. " - " .. nBase.y .. " - " .. nBase.z)

  if nBase.x < BASE.x then
    currentDirection = "x+"
  elseif nBase.x > BASE.x then
    currentDirection = "x-"
  elseif nBase.z < BASE.z then
    currentDirection = "z+"
  elseif nBase.z > BASE.z then
    currentDirection = "z-"
  end

  startDirection = currentDirection
end

function rotate()
  robot.turn(true)
  if currentDirection == "z+" then
    currentDirection = "x+"
  elseif currentDirection == "x+" then
    currentDirection = "z-"
  elseif currentDirection == "z-" then
    currentDirection = "x-"
  elseif currentDirection == "x-" then
    currentDirection = "z+"
  end
end

function rotateToDirection(dir)
  if currentDirection == dir then
    return true
  end

  if currentDirection == "z+" then
    robot.turn(true)
    currentDirection = "x+"
    rotateToDirection(dir)
  elseif currentDirection == "x+" then
    robot.turn(true)
    currentDirection = "z-"
    rotateToDirection(dir)
  elseif currentDirection == "z-" then
    robot.turn(true)
    currentDirection = "x-"
    rotateToDirection(dir)
  elseif currentDirection == "x-" then
    robot.turn(true)
    currentDirection = "z+"
    rotateToDirection(dir)
  end
end

function moveForward()
  robot.move(sidesFront)
  if currentDirection == "z+" then
    cz = cz + 1
  elseif currentDirection == "z-" then
    cz = cz - 1
  elseif currentDirection == "x+" then
    cx = cx + 1
  elseif currentDirection == "x-" then
    cx = cx - 1
  end
end

function moveDown()
  robot.move(sidesDown)
  cy = cy - 1
end

function moveUp()
  robot.move(sidesUp)
  cy = cy + 1
end

function moveTo(x, y, z)
  local moving = true
  while moving do
    print("Current Location: " .. cx .. " - " .. cy .. " - " .. cz)
    print("Move To Location: " .. x .. " - " .. y .. " - " .. z)

    if y > cy then
      print("move up")
      robot.move(sidesUp)
      cy = cy + 1
    elseif y < cy then
      print("move down")
      robot.move(sidesDown)
      cy = cy - 1
    elseif currentDirection == "z+" then
      if z > cz then
        robot.move(sidesFront)
        cz = cz + 1
      elseif z < cz then
        robot.turn(true)
        robot.turn(true)
        currentDirection = "z-"
      elseif cz == z and cx ~= x then
        robot.turn(true)
        currentDirection = "x+"
      end
    elseif currentDirection == "z-" then
      if z < cz then
        robot.move(sidesFront)
        cz = cz - 1
      elseif z > cz then
        robot.turn(true)
        robot.turn(true)
        currentDirection = "z+"
      elseif cz == z and cx ~= x then
        robot.turn(true)
        currentDirection = "x-"
      end
    elseif currentDirection == "x-" then
      if x > cx then
        robot.move(sidesFront)
        cx = cx + 1
      elseif x < cx then
        robot.turn(true)
        robot.turn(true)
        currentDirection = "x+"
      elseif cx == x and cz ~= z then
        robot.turn(true)
        currentDirection = "z-"
      end
    elseif currentDirection == "x+" then
      if x < cx then
        robot.move(sidesFront)
        cx = cx - 1
      elseif x > cx then
        robot.turn(true)
        robot.turn(true)
        currentDirection = "x-"
      elseif cx == x and cz ~= z then
        robot.turn(true)
        currentDirection = "z+"
      end
    end

    if cx == x and cy == y and cz == z then
      moving = false
    end
  end
end

function moveHome()
  moveTo(BASE.x, BASE.y + 1, BASE.z)
  moveTo(BASE.x, BASE.y, BASE.z)
  rotateToDirection(startDirection)
end

function checkSeeds() 
  for i = 1, 4 do
    robot.select(i)
    if robot.count() > 1 then
      return true
    end
  end
  return false
end

function moveFarming()
  local rows = getFarmRows()
  local seedsEmpty = false

  for i, row in pairs(rows) do
    print(row.name)
    print(row.x .. " - " .. row.y .. " - " .. row.z)
    moveTo(row.x, row.y + 1, row.z)
    rotateToDirection(startDirection)
    -- start checking row
    for l = 1, row.length do
      moveForward()
      local t = geo.analyze(sidesDown)
      if checkSeeds() then
        if t.name == "minecraft:air" then
          robot.use(sidesDown)
          robot.place(sidesDown)
        elseif t.name == "minecraft:wheat" then
          if t.properties.age == 7 then
            robot.use(sidesDown)
            robot.place(sidesDown)
          end
        end
      else
        seedsEmpty = true
        break
      end
    end
    moveTo(row.x, row.y + 1, row.z)
    if seedsEmpty then
      print("Move to base (Seeds empty)")
      break
    end
  end

  moveHome()
  -- empty inventory
  rotate()
  for i = 5, 16 do
    robot.select(i)
    robot.drop(sidesFront)
  end
  robot.select(1)
  rotateToDirection(startDirection)
end

print("Start Farming Program")
init()
print(currentDirection)
while true do
  moveHome()
  moveFarming()
  print("Sleep for " .. settings.sleep_time .. " seconds")
  os.sleep(settings.sleep_time)
end