### 从 Table 中查询字段
1. 直接在表中查找，找到则返回 res，无则继续。
2. 判断是否有元表，没有则返回 nil，有则继续。
3. 查询元表中的 __index 字段：
    * 不存在或者值等于 nil 则返回 nil，
    * 如果是一个函数则调用函数
    * 如果是一张表则重复上述步骤。
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
