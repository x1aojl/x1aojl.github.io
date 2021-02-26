### 聚合组件
```
-- 聚合函数
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
