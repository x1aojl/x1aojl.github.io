### UI管理
#### 目标
UI管理系统主要负责统一打开或关闭游戏窗口，并维护好互斥和层级关系。

#### 流程
1. 打开窗口：
	1. 关闭互斥窗口。
    2. 打开游戏窗口。
    3. 进行窗口排序。

2. 关闭窗口：
	1. 隐藏。
	2. 销毁。

UIManager.lua
```
-- 定义UI模块
module("UI");

local _windows;      -- UI window list
local _canvas_2d;    -- Canvas 2d's rect transform
local _canvas_3d;    -- Canvas 3d's transform
local _ui_camera_2d; -- Canvas 2d's renderer camera
local _ui_camera_3d; -- Canvas 3d's renderer camera

-- window的排序方法, 获取window group的方法
-- NOTE: require只会返回一个值, 故使用loadscript
local _sort_window, _is_exclusive = system.loadscript("Visualization/Manager/UIWindowGroup.lua");

local function _create_window(name, callback, is_3d)
    -- 默认2d window
    if (is_3d == nil) then
        is_3d = false;
    end

    -- 如果window已存在, 直接返回window
    local window = _windows[name];
    if (window ~= nil) then
        -- 通知显示已有UI
        if (callback ~= nil) then
            callback(window, false);
        end
        return window;
    end

    -- 创建对应的window对象
    window = create_object(name, name);

    -- 将window挂到canvas下面
    if (is_3d) then
        window:set_parent(_canvas_3d);
    else
        window:set_parent(_canvas_2d);
    end

    -- 缓存window对象
    _windows[name] = window;

    -- 通知创建UI事件
    if (callback ~= nil) then
        callback(window, true);
    end

    -- 通知打开ui事件
    EventSystem.send(EventID.WINDOW_OPEN, name);

    -- 加载成功
    return window;
end

local function _destroy_window(window)
    
    -- 获取window名字
    local name = window:get_name();
    -- 析构window
    window:dispose();

    -- 从window列表中移除
    _windows[name] = nil;

    -- 通知主动关闭ui事件
    EventSystem.send(EventID.WINDOW_CLOSE, name);
end

function init()
    -- 初始化window打开列表
    _windows = { };

    -- 创建canvas 2d
    Resource.load_sync("Assets/resource/prefab/ui/ui_root_2d.prefab", function (req)
        local ui_root_2d = req:GetInstantiateObject();
        _canvas_2d = ui_root_2d:GetComponent(RectTransform.class_id);

        _ui_camera_2d = ui_root_2d:GetComponent(Canvas.class_id).worldCamera;
    end);

    -- 创建canvas 3d
    Resource.load_sync("Assets/resource/prefab/ui/ui_root_3d.prefab", function (req)
        local ui_root_3d = req:GetInstantiateObject();
        _canvas_3d = ui_root_3d:GetComponent(RectTransform.class_id);

        _ui_camera_3d = ui_root_3d:GetComponent(Canvas.class_id).worldCamera;
    end);
end

function shutdown()
    for k, v in pairs(_windows) do
        v:dispose();
        _windows[k] = nil;
    end

    -- 析构canvas 2d & 3d
    if (_canvas_2d ~= nil) then
        UnityObject.Destroy(_canvas_2d.gameObject);
    end

    if (_canvas_3d ~= nil) then
        UnityObject.Destroy(_canvas_3d.gameObject);
    end

    -- 重置引用
    _windows      = nil;
    _canvas_2d    = nil;
    _canvas_3d    = nil;
end

function suspend()
    -- Suspend所有窗口
    for _, v in pairs(_windows) do
        v:suspend();
    end
end

function resume()
    -- Resume所有窗口
    for _, v in pairs(_windows) do
        v:resume();
    end
end

function update(time)
    -- 更新所有window
    for k, v in pairs(_windows) do
        v:update(time);
    end
end

function open(name, callback)
    -- 打开之前, 先关闭互斥窗口
    close_exclusion(name);

    -- 打开window(无缓存则创建)
    local window = _create_window(name, callback);

    -- 显示window
    window:show();

    -- 置顶window
    window:set_top();

    -- window排序
    _sort_window(_windows);

    -- 返回打开的window
    return window;
end

function open_exclude(name, callback, hide_all)
    -- 打开之前, 先关闭互斥窗口
    close_exclusion(name);

    -- 默认隐藏其余window
    if (hide_all == nil) then
        hide_all = true;
    end

    -- 打开window
    local window = _create_window(name, callback);

    -- 显示window
    window:show();

    -- 置顶window
    window:set_top();

    -- window排序
    _sort_window(_windows);

    -- 关闭或隐藏其余window
    for k, v in pairs(_windows) do
        -- 跳过自己以及pin住的window
        if (window ~= v and not v:is_pinned()) then
            if (hide_all) then
                v:hide();
            else
                _destroy_window(v);
            end
        end
    end

    -- 返回window
    return window;
end

-- 将缓存window变为显示状态
function show(name)
    -- 获取缓存的window
    local window = _windows[name];

    -- Window未缓存
    if (window == nil) then
        system.log_warning("Window(%s) was not found when show window.", name);
        return;
    end

    -- 显示window
    window:show();

    -- 置顶window
    window:set_top();

    -- window排序
    _sort_window(_windows);
end

-- 将缓存window隐藏
function hide(name)
    -- 获取缓存的window
    local window = _windows[name];

    -- Window未缓存
    if (window == nil) then
        system.log_warning("Window(%s) was not found when hide window.", name);
        return;
    end

    -- 隐藏window
    window:hide();
end

function hide_all(name)
    -- 隐藏所有window
    for k, v in pairs(_windows) do
        v:hide();
    end
end

function close(name)
    -- 获取缓存的window
    local window = _windows[name];

    -- 析构window
    if (window ~= nil) then
        _destroy_window(window);
    end
end

-- 1. close_active == true  (关闭active/non-active window)
-- 2. close_active == false (关闭non-active window, 保留active window)
function close_all(close_active)
    -- 默认关闭所有window
    if (close_active == nil) then
        close_active = true;
    end

    -- 遍历所有的window
    for k, v in pairs(_windows) do
        -- 关闭window
        if (close_active or v:is_hide() == false) then
            _destroy_window(v);
        end
    end
end

-- 关闭互斥窗口
function close_exclusion(name)
    -- 遍历所有的window
    for k, v in pairs(_windows) do
        -- 查询是否互斥
        if (_is_exclusive(name, k)) then
            -- 关闭window
            _destroy_window(v);
        end
    end
end

function query(name)
    -- 获取缓存的window
    return _windows[name];
end

function get_canvas_2d()
    return _canvas_2d;
end

function get_canvas_3d()
    return _canvas_3d;
end

function get_ui_camera_2d()
    return _ui_camera_2d;
end

function get_ui_camera_3d()
    return _ui_camera_3d;
end

function get_canvas_scale()
    return Vector2(Screen.width, Screen.height) / _canvas_2d.rect.size;
end

-- 获取所有的全屏窗口
function get_full_screen_windows()
    local list = {};

    -- 遍历打开的窗口
    for _, v in pairs(_windows) do
        if (v:is_hide() == false and v:is_full_screen()) then
            table.insert(list, v);
        end
    end

    return list;
end
```

UIWindow.lua
```
-- 定义UIWindow类
local UIWindow = class("UIWindow", nil);

function UIWindow:__awake(name)
    self.m_name           = name;
    self.m_enabled        = false;
    self.m_gameobject     = nil;
    self.m_transform      = nil;
    self.m_is_pinned      = false; -- 标记window在打开后只能显示调用close系列函数去关闭
    self.m_is_full_screen = false; -- 标记window是不是全屏窗口
    self.m_red_dots       = {};    -- 收集所有红点
    self.m_redraw_callback = function ()
        self:redraw();
    end

    self.m_update_red_dot_callback = function(specified_id, enable)
        self:on_update_red_dot(specified_id, enable);
    end

    EventSystem.subscribe(EventID.RECONNECT_OK, self.m_redraw_callback);
    EventSystem.subscribe(EventID.UI_UPDATE_RED_DOT, self.m_update_red_dot_callback);
end

function UIWindow:dispose()
    EventSystem.unsubscribe(EventID.UI_UPDATE_RED_DOT, self.m_update_red_dot_callback);
    EventSystem.unsubscribe(EventID.RECONNECT_OK, self.m_redraw_callback);

    -- 析构unity object
    UnityObject.Destroy(self.m_gameobject);

    -- 重置引用
    self.m_gameobject      = nil;
    self.m_transform       = nil;
    self.m_redraw_callback = nil;
end

function UIWindow:update(delta_time)
end

-- Suspend
function UIWindow:suspend()
end

-- Resume
function UIWindow:resume()
end

-- 绘制整个界面
function UIWindow:redraw()
    -- NOTE:
    --     此函数不做任何处理, 有派生类来实现
end

-- 界面脏处理
function UIWindow:sync_data(entity, prev_dbase, curr_dbase)
    -- NOTE:
    --     此函数不做任何处理, 有派生类来实现
end

function UIWindow:show()
    -- 显示window
    if (not self.m_enabled) then
        self.m_enabled = true;
        self.m_gameobject:SetActive(true);
    end
end

function UIWindow:hide()
    -- 隐藏window
    if (self.m_enabled) then
        self.m_enabled = false;
        self.m_gameobject:SetActive(false);
    end
end

function UIWindow:set_top()
    self.m_transform:SetAsLastSibling();
end

function UIWindow:is_top()
    -- 获取window的transform
    local transform  = self.m_transform;

    --(排除event_system & ui_camera)
    local childCount = transform.parent.childCount - 3;

    -- 开启了gm ui,数量-1
    if (UserD.gm_enabled() == true) then
        childCount = childCount -1;
    end

    -- 是否是top window
    return transform:GetSiblingIndex() == childCount;
end

function UIWindow:set_sibling_index(index)
    self.m_transform:SetSiblingIndex(index);
end

function UIWindow:get_sibling_index()
    return self.m_transform:GetSiblingIndex();
end

function UIWindow:set_parent(parent)
    self.m_transform:SetParent(parent, false);
end

function UIWindow:get_parent()
    return self.m_transform.parent;
end

function UIWindow:get_name()
    return self.m_name;
end

function UIWindow:is_hide()
    return self.m_enabled == false;
end

function UIWindow:mark_pinned()
    self.m_is_pinned = true;
end

function UIWindow:is_pinned()
    return self.m_is_pinned;
end

-- 标记全屏界面
function UIWindow:mark_full_screen()
    self.m_is_full_screen = true;
end

function UIWindow:is_full_screen()
    return self.m_is_full_screen;
end

function UIWindow:load(name)
    -- 生成window的prefab路径
    local path = string.format("Assets/resource/prefab/ui/window/%s.prefab", name);

    -- 同步加载资源
    Resource.load_sync(path, function (req)
        -- 实例化game object
        local gameobject = req:GetInstantiateObject();

        -- 绑定数据
        self.m_gameobject = gameobject;
        self.m_transform  = gameobject.transform;

        -- 通知加载完成(由派生类实现)
        self:on_loaded();
    end);
end

function UIWindow:on_loaded()
end
```
