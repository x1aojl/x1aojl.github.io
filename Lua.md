### 从 Table 中查询字段
1. 直接在表中查找，找到则返回 res，无则继续。
2. 判断是否有元表，没有则返回 nil，有则继续。
3. 查询元表中的 __index 字段：
    * 不存在或值为 nil，返回 nil;
    * 如果是一个函数，调用函数;
    * 如果是一个表，重复步骤1、2、3。
```
void luaV_gettable (lua_State *L, const TValue *t, TValue *key, StkId val) {
  int loop;
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    const TValue *tm;
    if (ttistable(t)) {  /* `t' is a table? */
      Table *h = hvalue(t);
      const TValue *res = luaH_get(h, key); /* do a primitive get */
      if (!ttisnil(res) ||  /* result is no nil? */
          (tm = fasttm(L, h->metatable, TM_INDEX)) == NULL) { /* or no TM? */
        setobj2s(L, val, res);
        return;
      }
      /* else will try the tag method */
    }
    else if (ttisnil(tm = luaT_gettmbyobj(L, t, TM_INDEX)))
      luaG_typeerror(L, t, "index");
    if (ttisfunction(tm)) {
      callTMres(L, val, tm, t, key);
      return;
    }
    t = tm;  /* else repeat with `tm' */ 
  }
  luaG_runerror(L, "loop in gettable");
}
```

### 给 Table 的字段赋值
1. 查询表中字段，字段存在且值不为 nil 则赋值并返回，否则继续。
2. 判断是否有元表，没有则赋值并返回，有则继续。
3. 查询元表中的 __newindex 字段：
    * 不存在或值为 nil，赋值并返回；
    * 如果是一个函数，调用函数但不赋值；
    * 如果是一个表，则只给 __newindex 这个表进行赋值（无论 __newindex 表中是否存在该字段）。

```
void luaV_settable (lua_State *L, const TValue *t, TValue *key, StkId val) {
  int loop;
  TValue temp;
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    const TValue *tm;
    if (ttistable(t)) {  /* `t' is a table? */
      Table *h = hvalue(t);
      TValue *oldval = luaH_set(L, h, key); /* do a primitive set */
      if (!ttisnil(oldval) ||  /* result is no nil? */
          (tm = fasttm(L, h->metatable, TM_NEWINDEX)) == NULL) { /* or no TM? */
        setobj2t(L, oldval, val);
        h->flags = 0;
        luaC_barriert(L, h, val);
        return;
      }
      /* else will try the tag method */
    }
    else if (ttisnil(tm = luaT_gettmbyobj(L, t, TM_NEWINDEX)))
      luaG_typeerror(L, t, "index");
    if (ttisfunction(tm)) {
      callTM(L, tm, t, key, val);
      return;
    }
    /* else repeat with `tm' */
    setobj(L, &temp, tm);  /* avoid pointing inside table (may rehash) */
    t = &temp;
  }
  luaG_runerror(L, "loop in settable");
}
```

### class
```
-- 类
function class(class_name, super)
    local cls = {};
    cls.name = class_name;

    if (super) then
        setmetatable(cls, { __index = super });
        cls.super = super;
    else
        cls.ctor = function() end;
    end

    function cls.new(...)
        local instance = {};
        setmetatable(instance, { __index = cls });
        instance.ctor(...);

        return instance;
    end

    return cls;
end

-- 基类
local A = class("A");
A.ctor = function()
    print("A");
end

-- 派生类
local B = class("B", A)
B.ctor = function()
    -- 主动调用基类构造函数
    B.super.ctor();
    print("B");
end

-- 测试
local a = A.new();
local b = B.new();

-- 输出
-- A
-- A
-- B
```

### component
```
-- 聚合组件
function component(symbol, component)
    for k, v in pairs(component) do
        symbol[k] = v;
    end
end

-- 模块 A
local A = { };
function A:func_a()
    print("a");
end

-- 模块 B
local B = { };
function B:func_b()
    print("b");
end

-- 聚合组件 A -> B
component(B, A);

-- 测试
B:func_a();
B:func_b();

-- 输出
-- a
-- b
```

### 表的遍历 ipairs 和 pairs
```
-- table
-- 数组部分 + 哈希部分
-- array = { 1, "3", nil, 5 }
-- node = { a : 2, b : 4 }
local t = { 1, a = 2, "3", b = 4, nil, 5}

-- ipairs
-- 只遍历数组部分的整数元素，遇到空值就中断
for i, v in ipairs(t) do
    print(string.format("i = %s, v = %s", i, v));
end

-- 输出
-- i = 1, v = 1
-- i = 2, v = 3

-- pairs
-- 先遍历数组部分，再遍历哈希部分
for k, v in pairs(t) do
    print(string.format("k = %s, v = %s", k, v));
end

-- 输出
-- k = 1, v = 1
-- k = 2, v = 3
-- k = 4, v = 5
-- k = a, v = 2
-- k = b, v = 4
```

### 计算表的长度 # 和 pairs
```
local t = { 1, nil, a = 2, "3", nil, b = 4 }

-- # 统计表的数组部分最后一个有效元素前所有元素的数量
print(string.format("#t = %s", #t)); -- #t = 3

-- pairs 会遍历表的数组部分和哈希部分所有的有效元素
local length = 0;
for __, __ in pairs(t) do
    length = length + 1;
end

print(string.format("length = %s", length)); -- length = 4
```

### C++ 调用 Lua
1. lua_open 创建 Lua 环境 </br>
```
#define lua_open()	luaL_newstate()
```
2. luaL_openlibs 打开 Lua 标准库 </br>
```
LUALIB_API void (luaL_openlibs) (lua_State *L); 
```
3. luaL_dofile 加载运行 Lua 脚本 </br>
```
#define luaL_dofile(L, fn) (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
```
4. lua_getglobal 获取 Lua 脚本中的函数 </br>
```
#define lua_getglobal(L,s)	lua_getfield(L, LUA_GLOBALSINDEX, (s))
```
5. push functions (C -> stack) 传递参数给 Lua 函数 </br>
``` 
LUA_API void  (lua_pushnil) (lua_State *L);
LUA_API void  (lua_pushnumber) (lua_State *L, lua_Number n);
LUA_API void  (lua_pushinteger) (lua_State *L, lua_Integer n);
LUA_API void  (lua_pushlstring) (lua_State *L, const char *s, size_t l);
LUA_API void  (lua_pushstring) (lua_State *L, const char *s);
LUA_API const char *(lua_pushvfstring) (lua_State *L, const char *fmt, va_list argp);
LUA_API const char *(lua_pushfstring) (lua_State *L, const char *fmt, ...);
LUA_API void  (lua_pushcclosure) (lua_State *L, lua_CFunction fn, int n);
LUA_API void  (lua_pushboolean) (lua_State *L, int b);
LUA_API void  (lua_pushlightuserdata) (lua_State *L, void *p);
LUA_API int   (lua_pushthread) (lua_State *L);
``` 
6. call functions (run Lua code) 调用 Lua 函数 </br>
``` 
LUA_API void  (lua_call) (lua_State *L, int nargs, int nresults);
LUA_API int   (lua_pcall) (lua_State *L, int nargs, int nresults, int errfunc);
LUA_API int   (lua_cpcall) (lua_State *L, lua_CFunction func, void *ud);
```
7. access functions (stack -> C) 获取 Lua 函数的返回值 </br>
``` 
LUA_API lua_Number      (lua_tonumber) (lua_State *L, int idx);
LUA_API lua_Integer     (lua_tointeger) (lua_State *L, int idx);
LUA_API int             (lua_toboolean) (lua_State *L, int idx);
LUA_API const char     *(lua_tolstring) (lua_State *L, int idx, size_t *len);
LUA_API size_t          (lua_objlen) (lua_State *L, int idx);
LUA_API lua_CFunction   (lua_tocfunction) (lua_State *L, int idx);
LUA_API void	       *(lua_touserdata) (lua_State *L, int idx);
LUA_API lua_State      *(lua_tothread) (lua_State *L, int idx);
LUA_API const void     *(lua_topointer) (lua_State *L, int idx);
```

### Lua 调用 C++
1. lua_open 创建 Lua 环境 </br>
```
#define lua_open()	luaL_newstate()
```
2. luaL_openlibs 打开 Lua 标准库 </br>
```
LUALIB_API void (luaL_openlibs) (lua_State *L); 
```
3. lua_register 注册 C++ 函数
``` 
#define lua_register(L,n,f) (lua_pushcfunction(L, (f)), lua_setglobal(L, (n)))
```
4. luaL_dofile 加载运行 Lua 脚本 </br>
```
#define luaL_dofile(L, fn) (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
```
