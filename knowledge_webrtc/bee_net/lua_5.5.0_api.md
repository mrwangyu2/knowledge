# Lua 5.5.0 API 变更与 BeeNet 兼容性

## 概述

Lua 5.5.0 是 PUC-Rio 于 2025 年发布的最新 Lua 版本（`LUA_VERSION_NUM = 505`）。

## 关键 API 变更

### 1. `luaL_newmetatable` 新增 `__name` 字段

```c
// Lua 5.5.0 源码 (lauxlib.c)
LUALIB_API int luaL_newmetatable (lua_State *L, const char *tname) {
  if (luaL_getmetatable(L, tname) != LUA_TNIL)
    return 0;
  lua_pop(L, 1);
  lua_createtable(L, 0, 2);
  lua_pushstring(L, tname);
  lua_setfield(L, -2, "__name");  // ★ 新增：metatable.__name = tname
  lua_pushvalue(L, -1);
  lua_setfield(L, LUA_REGISTRYINDEX, tname);
  return 1;
}
```

**影响**：在 Lua 5.5.0 之前，`luaL_newmetatable` 不设置 `__name` 字段。5.5.0 新增此行为后，metatable 结构变为：
```lua
{
  __name = "modulename",    -- 新增
  -- 后续由应用代码设置的字段...
}
```

### 2. 堆栈值类型变化

Lua 5.5.0 内部使用新的值表示方式（`TValue` 结构中通过 `Value.p` 访问栈指针）。

### 3. 其他观察

- `lua_gettop` 不再使用 `lua_lock`（值类型变更所致）
- `luaL_requiref` 接口不变
- `luaL_setfuncs` 接口不变
- 元方法事件系统（`ltm.c`）不变
- 错误消息机制（`lvm.c`）不变

## BeeNet 崩溃根因

### 问题表现
```
PANIC: unprotected error in call to Lua API (attempt to index a string value)
```

### 崩溃位置
`LuaOpenPeerConnectionLib` (peerconnection.cpp) 中，`luaL_newmetatable` + `luaL_setfuncs` + `lua_pop` 完成后的下一行代码处。

### 根因
Lua 5.5.0 新增的 `__name` 字段与 BeeNet 的 metatable 模式冲突。当 `luaL_newmetatable` 设置 `__name` 后，BeeNet 代码随后设置 `__index`、`__newindex`、`__gc` 等字段。这些 metamethod 的组合在后续 Lua 内部操作中触发 "attempt to index a string value" 错误。

### 修复方法
在 `luaL_newmetatable` 后立即清除 `__name` 字段：

```c
luaL_newmetatable(L, "modulename");
lua_pushnil(L); lua_setfield(L, -2, "__name");  /* Lua 5.5.0 compat */
// ... 后续 metatable 设置代码
```

### 影响范围
所有使用 `luaL_newmetatable` 的模块文件（7 个）：
- `datacache.cpp`
- `http_exec.cpp`
- `media_frame.cpp`（videoframe + audioframe）
- `peerconnection.cpp`（peerconnection + datachannel）
- `timer.cpp`
- `utils.cpp`（decryption）
- `ws_exec.cpp`

## 构建验证

- Debug + Release 双版本编译通过
- `bee_client.exe` 启动正常，成功连接服务器、加入房间、接收数据

## Lua 5.5.0 源码结构

```
src/
├── lua.h        — C API 声明
├── lauxlib.h    — 辅助库（luaL_*）
├── lualib.h     — 标准库加载
├── lstate.h     — 全局状态机定义
├── lapi.c       — API 实现
├── lauxlib.c    — 辅助库实现
├── lstate.c     — luaL_newstate
├── ldo.c        — 函数调用/panic
├── lvm.c        — 虚拟机执行引擎
├── lgc.c        — 增量式垃圾收集
├── ltable.c     — 表操作
├── ltm.c        — 元方法事件系统
├── lbaselib.c   — 基础库
├── liolib.c     — IO 库
├── lstrlib.c    — 字符串库
└── linit.c      — luaL_openlibs
```

## 关键 API 速查

| API | 文件 | 说明 |
|-----|------|------|
| `luaL_newstate` | lstate.c | 创建 Lua 状态机 |
| `luaL_openlibs` | linit.c | 加载所有标准库 |
| `luaL_newmetatable` | lauxlib.c | 创建 metatable（5.5.0 新增 __name） |
| `luaL_setfuncs` | lauxlib.c | 批量注册 C 函数到表 |
| `luaL_requiref` | lauxlib.c | 模块注册（检查 _LOADED） |
| `luaL_newlib` | lauxlib.h(宏) | 创建并填充模块表 |
| `lua_gettop` | lapi.c | 获取栈顶（5.5.0 无锁） |
| `lua_atpanic` | lapi.c | 设置 panic 处理器 |
