### 读表操作调用栈
1. 调用一个函数，传入函数的参数个数和返回值个数。
```
LUA_API void lua_call (lua_State *L, int nargs, int nresults) {
  StkId func;
  func = L->top - (nargs+1);
  luaD_call(L, func, nresults);
}
```

2. 调用一个函数（C 或 Lua）。要调用的函数在指针 func 处。</br>
   参数位于函数之后的堆栈中。</br>
   返回时，所有结果都将从函数的起始位置堆叠到堆栈中。
```
void luaD_call (lua_State *L, StkId func, int nResults) {
  if (luaD_precall(L, func, nResults) == PCRLUA)  /* is a Lua function? */
    luaV_execute(L, 1);  /* call it */
}
```

3. 解析字节码，执行指令
```
void luaV_execute (lua_State *L, int nexeccalls) {
  switch (GET_OPCODE(i)) {
    case OP_SELF: {
      StkId rb = RB(i);
      setobjs2s(L, ra+1, rb);
      Protect(luaV_gettable(L, rb, RKC(i), ra));
      continue;
    }
}
```

4. 从表中读取元素 </br>
   判断索引值
   - 如果值 ~= nil：判断是否存在元表
     - 如果存在元表：判断元表是否存在 "__index" 索引
       - 如果存在 "__index" 索引：判断 "__index" 索引值是否为函数
         - 如果 "__index" 索引值是函数：执行函数
         - 如果 "__index" 索引值非函数：重复上述步骤
       - 如果没有 "__index" 索引：返回 nil
     - 如果没有元表：返回 nil
   - 如果值 == nil：返回索引值

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
