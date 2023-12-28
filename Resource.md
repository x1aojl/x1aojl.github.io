### 资源管理
#### 目标
资源管理系统主要是对项目中的资源进行统一调度。

#### 分类
1. unity资源: 场景、预制、材质、贴图、shader
2. 代码资源: c#、lua
3. 配置表资源：json、csv

#### 流程
1. 打包资源：
	1. 将unity资源打成ab包，并生成资源与ab包的映射配置表。
    2. 将ab包、lua代码、配置表拷贝到StreamingAssets目录下。

2. 使用资源：
	1. 加载：资源路径-->ab包名-->ab包路径-->ab包-->资源，添加引用。
	2. 卸载：将没有引用的资源对象全部卸载。

3. 更新资源：
    1. 打补丁包：比对上版本的资源md5清单，将所有差异资源打成ab包，并生成补丁包与版本的映射配置表。
    2. 上传资源：将补丁包、配置表上传到服务器。
    3. 下载资源：与服务器比对版本号，如果不一致则通过版本路径下载各版本资源。

ResourceManager.cs
```
using System;
using System.Collections.Generic;
using UnityEngine;

using Object = UnityEngine.Object;

public class LoadingRequest
{
    public LoadingRequest(string name, AssetBundleRequest request)
    {
        this.m_Name = name;
        this.m_Request = request;
    }

    public LoadingRequest(string name, Object data)
    {
        this.m_Name = name;
        this.m_Data = data;
    }

    public bool IsDone()
    {
        return Synchronize();
    }

    public bool IsAlive()
    {
        return this.m_Data != null ||
               this.m_Request != null;
    }

    public T GetData<T>()
    {
        // Data is loaded already
        if (m_Data != null)
            return (T)(object)m_Data;

        // Synchronize async operate
        Synchronize();

        // Success
        return (T)(object)m_Data;
    }

    public float Progress()
    {
        // Asset is loaded already
        if (m_Data != null)
            return 1.0f;

        // Get async operate progresss
        return m_Request.progress;
    }

    public Object GetRawObject()
    {
        return this.GetRawObject<Object>();
    }

    public Object GetInstantiateObject()
    {
        return this.GetInstantiateObject<Object>();
    }

    public Object GetInstantiateObject(Transform parent)
    {
        return this.GetInstantiateObject<Object>(parent);
    }

    public Object GetInstantiateObject(Vector3 position, Quaternion rotation)
    {
        return this.GetInstantiateObject<Object>(position, rotation);
    }

    public Object GetInstantiateObject(Transform parent, bool worldPositionStays)
    {
        return this.GetInstantiateObject<Object>(parent, worldPositionStays);
    }

    public Object GetInstantiateObject(Vector3 position, Quaternion rotation, Transform parent)
    {
        return this.GetInstantiateObject<Object>(position, rotation, parent);
    }

    public T GetRawObject<T>() where T : Object
    {
        // Data is loaded already
        if (m_Data != null)
            return (T)m_Data;

        // Synchronize async operate
        Synchronize();

        // Success
        return (T)m_Data;
    }

    public T GetInstantiateObject<T>() where T : Object
    {
        // Get raw object
        var rawOb = GetRawObject<T>();

        // Instantiate asset
        return rawOb != null ? Object.Instantiate<T>(rawOb) : null;
    }

    public T GetInstantiateObject<T>(Transform parent) where T : Object
    {
        // Get raw object
        var rawOb = GetRawObject<T>();

        // Instantiate asset
        return rawOb != null ? Object.Instantiate<T>(rawOb, parent) : null;
    }

    public T GetInstantiateObject<T>(Vector3 position, Quaternion rotation) where T : Object
    {
        // Get raw object
        var rawOb = GetRawObject<T>();

        // Instantiate asset
        return rawOb != null ? Object.Instantiate<T>(rawOb, position, rotation) : null;
    }

    public T GetInstantiateObject<T>(Transform parent, bool worldPositionStays) where T : Object
    {
        // Get raw object
        var rawOb = GetRawObject<T>();

        // Instantiate asset
        return rawOb != null ? Object.Instantiate<T>(rawOb, parent, worldPositionStays) : null;
    }

    public T GetInstantiateObject<T>(Vector3 position, Quaternion rotation, Transform parent) where T : Object
    {
        // Get raw object
        var rawOb = GetRawObject<T>();

        // Instantiate asset
        return rawOb != null ? Object.Instantiate<T>(rawOb, position, rotation, parent) : null;
    }

    // Synchronize async loading resource
    private bool Synchronize()
    {
        // Data is loaded already
        if (m_Data != null)
            return true;

        // Check if async operate finished
        if (m_Request.isDone)
            m_Data = m_Request.asset;

        // Success ?
        return m_Data != null;
    }

    private string             m_Name;
    private Object             m_Data;
    private AssetBundleRequest m_Request;
}

public partial class ResourceManager
{
    public delegate void OnResourceLoaded(LoadingRequest request);

    internal static void Init()
    {
        // Init data
        m_Requests        = new Dictionary<string, WeakReference>();
        m_LoadingRequests = new Dictionary<LoadingRequest, OnResourceLoaded>();

        // Load asset map
        LoadAssetMap();
    }

    internal static void Shutdown()
    {
        // Unload asset map
        UnloadAssetMap();

        // Reset reference
        m_Requests        = null;
        m_LoadingRequests = null;
    }

    internal static void Update()
    {
        // No loading callback should handle
        if (m_LoadingRequests.Count == 0)
            return;

        // Requests which was finished
        var loadedReqs = new List<LoadingRequest>();

        foreach (var kvp in m_LoadingRequests)
        {
            var req      = kvp.Key;
            var callback = kvp.Value;

            // Asset request was finished
            if (! req.IsDone())
                continue;

            // Notify
            callback(req);

            // Add loaded request
            loadedReqs.Add(req);
        }

        // Remove finished asset requests
        foreach (var req in loadedReqs)
            m_LoadingRequests.Remove(req);
    }

    public static LoadingRequest LoadSync(string path)
    {
        // Load asset from AssetDatabase in editor mode.
        if (Application.isEditor)
            return LoadFromDatabaseSync(path);

        WeakReference handle;

        // Try get resource request's weak reference
        if (m_Requests.TryGetValue(path, out handle))
        {
            // Resource is already loaded
            if (handle.Target != null && ((LoadingRequest)handle.Target).IsAlive())
                return (LoadingRequest)handle.Target;

            // Weak reference was expired, clean cache
            m_Requests.Remove(path);
        }

        // Try get asset bundle with asset's path
        var ab = ResolveBundle(path);
        if (ab == null)
            Diagnostics.RaiseException("AssetBundle was not found when loading asset({0}).", path);

        // Load asset from asset bundle synchronized
        var asset = ab.LoadAsset(path);
        if (asset == null)
            Diagnostics.RaiseException("Asset({0}) was not found in asset bundle({1}).", path, ab.name);

        // Create resource loading request
        var req = new LoadingRequest(path, asset);

        // Cache request with asset's path
        m_Requests.Add(path, new WeakReference(req));

        // Success
        return req;
    }

    public static LoadingRequest LoadAsync(string path)
    {
        // Load asset from AssetDatabase in editor mode.
        if (Application.isEditor)
            return LoadFromDatabaseAsync(path);

        WeakReference handle;

        // Try get resource request's weak reference
        if (m_Requests.TryGetValue(path, out handle))
        {
            // Resource is already loaded
            if (handle.Target != null && ((LoadingRequest)handle.Target).IsAlive())
                return (LoadingRequest)handle.Target;

            // Weak reference was expired, clean cache
            m_Requests.Remove(path);
        }

        // Try get asset bundle with asset's path
        var ab = ResolveBundle(path);
        if (ab == null)
            Diagnostics.RaiseException("AssetBundle was not found when loading asset({0}).", path);

        // Load asset from asset bundle asynchronized
        var asset = ab.LoadAssetAsync(path);
        if (asset == null)
            Diagnostics.RaiseException("Asset({0}) was not found in asset bundle({1}).", path, ab.name);

        // Create resource loading request
        var req = new LoadingRequest(path, asset);

        // Success
        return req;
    }

    internal static LoadingRequest LoadSync(string path, OnResourceLoaded callback)
    {
        // Load asset synchronized
        var req = LoadSync(path);

        // Append to queue
        AppendToQueue(req, callback);

        // Finish request
        return req;
    }

    internal static LoadingRequest LoadAsync(string path, OnResourceLoaded callback)
    {
        // Load resource asynchronized
        var req = LoadAsync(path);

        // Append to queue
        AppendToQueue(req, callback);

        // Finish request
        return req;
    }

    public static void UnloadUnusedAssets()
    {
        var keys = new List<string>();

        // Find all expired requests
        foreach (var kvp in m_Requests)
        {
            if (kvp.Value.Target == null)
                keys.Add(kvp.Key);
        }

        // Clean expired requests
        for (var i = 0; i < keys.Count; ++i)
            m_Requests.Remove(keys[i]);

        // Unload unused assets
        Resources.UnloadUnusedAssets();
    }

    private static void AppendToQueue(LoadingRequest req, OnResourceLoaded callback)
    {
        // No callback
        if (callback == null)
            return;

        // Resource was loaded
        if (req.IsDone())
        {
            callback(req);
            return;
        }

        OnResourceLoaded callbacks;

        // Save callback with resource request handle
        if (m_LoadingRequests.TryGetValue(req, out callbacks))
            callbacks += callback;
        else
            m_LoadingRequests.Add(req, callback);
    }

    private static Dictionary<string, WeakReference>            m_Requests;
    private static Dictionary<LoadingRequest, OnResourceLoaded> m_LoadingRequests;
}
```

ResourceManager_ResMap.cs
```
using System.IO;
using System.Collections.Generic;

using Core;
using UnityEngine;

public partial class ResourceManager
{
    private static void LoadAssetMap()
    {
        // 创建资源映射表以及资源包映射表
        m_AssetMap  = new Dictionary<string, string>();
        m_BundleMap = new Dictionary<string, AssetBundle>();

        // 解析client.json的磁盘路径
        var path = FileSystem.ResolveDiskPath("/medias/client.manifest");

        // 读取client.json文件
        var content = File.ReadAllText(path);

        // 解析client.json文件
        var dbase = Json.RestoreFrom<Dbase>(content);

        // 读取资源
        var bundles = dbase.Get<Dictionary<string, object>>("bundles");
        var assets  = dbase.Get<Dictionary<string, object>>("assets");

        // 初始化资源包映射表
        foreach (var kvp in bundles)
        {
            // 解析资源包的磁盘路径
            var bundlePath = FileSystem.ResolveDiskPath((string)kvp.Value);

            // 加载资源包
            var bundle = AssetBundle.LoadFromFile(bundlePath);

            // 加入映射表
            m_BundleMap.Add(kvp.Key, bundle);
        }

        // 初始化资源映射表
        foreach (var kvp in assets)
            m_AssetMap.Add(kvp.Key, (string)kvp.Value);
    }

    private static void UnloadAssetMap()
    {
        // 析构已加载的资源包
        foreach (var bundle in m_BundleMap.Values)
            bundle.Unload(true);

        // 重置引用
        m_AssetMap  = null;
        m_BundleMap = null;
    }

    private static AssetBundle ResolveBundle(string path)
    {
        string bundleName;

        // 取资源对应的资源包名
        if (m_AssetMap.TryGetValue(path, out bundleName) == false)
            return null;

        AssetBundle bundle;

        // 获取资源包对象
        if (m_BundleMap.TryGetValue(bundleName, out bundle))
            return bundle;

        // 资源包不存在
        return null;
    }

    private static Dictionary<string, string>      m_AssetMap;  // AssetName  : BundleName
    private static Dictionary<string, AssetBundle> m_BundleMap; // BundleName : AssetBundle
}
```

ResourceManager_Database.cs
```
using UnityEngine;

#if UNITY_EDITOR
using UnityEditor;
#endif

public partial class ResourceManager
{
    private static LoadingRequest LoadFromDatabaseSync(string path)
    {
#if UNITY_EDITOR
        // Load asset from database syncronized
        var asset = AssetDatabase.LoadAssetAtPath<Object>(path);
        if (asset == null)
            Diagnostics.RaiseException("Asset({0}) was not found in asset database", path);

        // Create asset's loading request
        return new LoadingRequest(path, asset);
#else
        Diagnostics.RaiseException("Can not call ResourceManager.LoadFromDatabaseSync outside unity editor.");
        return null;
#endif
    }

    private static LoadingRequest LoadFromDatabaseAsync(string path)
    {
#if UNITY_EDITOR
        // Load asset from database asyncronized
        var asset = AssetDatabase.LoadAssetAtPath<Object>(path);
        if (asset == null)
            Diagnostics.RaiseException("Asset({0}) was not found in asset database.", path);

        // Create asset's loading request
        return new LoadingRequest(path, asset);
#else
        Diagnostics.RaiseException("Can not call ResourceManager.LoadFromDatabaseAsync outside unity editor.");
        return null;
#endif
    }
}
```
