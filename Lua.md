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
