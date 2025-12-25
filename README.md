# GAS Network

GASNetwork 用于与 DLRS GAS (Game Account System) API 进行交互，提供用户认证、云存档、版本管理等功能。

## 目录

- [文档地址](#文档地址)
- [系统架构](#系统架构)
- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [API 使用指南](#api-使用指南)
- [配置说明](#配置说明)
- [错误处理](#错误处理)
- [示例代码](#示例代码)
- [安全说明](#安全说明)
- [注意事项](#注意事项)
- [API 端点](#api-端点)
- [鸣谢](#鸣谢)

## 文档地址

 [>> 点击此处跳转 <<](https://api.chinadlrs.com/category/应用接口)

## 系统架构

```
GASNetwork/
├── GAS/
│   ├── Common/          # 通用类
│   │   ├── GASCommonResp.cs      # 通用响应模型
│   │   ├── GASException.cs        # 异常处理
│   │   ├── GASResponseChecker.cs # 响应检查
│   │   └── GASResponseLogger.cs  # 日志记录
│   ├── Config/          # 配置管理
│   │   ├── GASConfig.cs          # 配置文件 ScriptableObject
│   │   └── GASConfigManager.cs   # 配置管理器
│   ├── Enum/            # 枚举定义
│   │   └── GASLang.cs            # 语言枚举（zh/en）
│   ├── Handler/         # 处理器
│   │   └── GASBrowserHandler.cs  # 浏览器处理
│   ├── Models/          # 数据模型
│   │   ├── ...
│   ├── Network/         # 网络层
│   │   ├── GASApiRoute.cs       # API 路由定义
│   │   ├── GASEncryption.cs     # AES 加密解密
│   │   └── GASHttpClient.cs     # HTTP 客户端
│   └── Service/         # 业务服务层
│       ├── ...
└── Resources/           # 资源文件
    └── GASConfig.asset  # 配置文件
```

## 环境要求

### 必需依赖

- **UniTask**: 用于异步操作（[GitHub](https://github.com/Cysharp/UniTask)）
- **Newtonsoft.Json**: JSON 序列化库 Unity 专用版（[Github](https://github.com/applejag/Newtonsoft.Json-for-Unity) | [Unity](http://docs.unity3d.com/Packages/com.unity.nuget.newtonsoft-json@3.0/manual/index.html)）

### 安装 UniTask

如果项目中尚未安装 UniTask，可以通过以下方式安装：

1. 使用 Unity Package Manager
2. 或从 [UniTask GitHub](https://github.com/Cysharp/UniTask) 下载

### 安装 Newtonsoft.Json

⚠️ 需要安装 Unity 专用版以支持 `IL2CPP` 运行

步骤：  
1. 使用 Unity Package Manager
2. Add package by name... -> 输入包名`com.unity.nuget.newtonsoft-json`，版本`3.2.2`


## 快速开始

### 1. 创建配置文件（已包含在Resources，无需重复创建）

在 Unity 编辑器中：

1. 右键点击 `Assets/GASNetwork/Resources/` 目录
2. 选择 `Create > GAS > Config`
3. 创建 `GASConfig` 资源文件
4. 在 Inspector 中设置：
   - **App Id**: 你的应用 ID
   - **App Token**: 你的应用令牌

### 2. 设置接口语言（可选）

```csharp
using GAS.Config;
using GAS.Enum;

// 设置为中文
GASConfigManager.Lang = GASLang.zh;

// 设置为英文
GASConfigManager.Lang = GASLang.en;
```

### 3. 基本使用示例

```csharp
using Cysharp.Threading.Tasks;
using GAS.Service;
using GAS.Config;

public class MyGameManager : MonoBehaviour
{
    private OAuthService _oauth = new OAuthService();
    private ProfileService _profile = new ProfileService();
    
    async void Start()
    {
        // 获取 appId 和 appToken
        int appId = GASConfigManager.AppId;
        string appToken = GASConfigManager.AppToken;
        
        // 执行 OAuth 登录流程...
    }
}
```

## API 使用指南

### OAuth 授权登录

OAuth 登录流程分为三个步骤：

1. **获取 auth_token**
2. **打开浏览器进行授权**
3. **轮询获取 access_token**

```csharp
using GAS.Service;
using GAS.Handler;

// 1. 获取 auth_token
var oauthService = new OAuthService();
var authTokenResp = await oauthService.GetAuthTokenAsync();
string authToken = authTokenResp.Data.AuthToken;

// 2. 打开浏览器进行授权
GASBrowserHandler.OpenAuthBrowser(GASConfigManager.AppId, authToken);

// 3. 轮询获取 access_token（需要实现轮询逻辑）
// 建议每 1 秒轮询一次，最多轮询 10 次
for (int i = 0; i < 10; i++)
{
    await UniTask.Delay(1000);
    var accessResp = await oauthService.ExchangeAuthTokenAsync(authToken);
    if (accessResp.Code == 200)
    {
        string accessToken = accessResp.Data.AccessToken;
        string email = accessResp.Data.Email;
        // 登录成功，保存 accessToken 和 email
        break;
    }
}
```

### 自动登录

```csharp
var autoLoginService = new AutoLoginService();
var resp = await autoLoginService.AutoLoginAsync(email, accessToken);
// 验证登录状态
```

### 获取用户信息

```csharp
var profileService = new ProfileService();
var resp = await profileService.GetProfileAsync(email, accessToken);
string nickname = resp.Data.Nickname;
// 使用用户信息...
```

### 云存档

#### 保存存档

```csharp
var archiveService = new ArchiveService();

// 准备存档内容（JSON 格式）
string archiveContent = "{\"level\":1, \"score\":1000, \"items\":[]}";
string version = Application.version;

// 保存存档（内容会自动加密）
var resp = await archiveService.SaveAsync(email, accessToken, version, archiveContent);
```

#### 读取存档

```csharp
var archiveService = new ArchiveService();

// 读取存档
var resp = await archiveService.ReadAsync(email, accessToken);

// 解密存档内容
string decryptedContent = archiveService.DecryptArchiveContent(resp.Data.ContentEncrypted);

// 解析 JSON
var archiveData = JsonConvert.DeserializeObject<YourArchiveType>(decryptedContent);
```

#### 删除存档

```csharp
var archiveService = new ArchiveService();
var resp = await archiveService.DeleteAsync(email, accessToken);
```

### 版本管理

```csharp
var versionService = new VersionService();

// 查询版本（sequence 会被自动加密）
string sequence = "1.0.0";
var resp = await versionService.GetVersionAsync(sequence);

// 获取版本列表
string[] versions = resp.Data.Versions;
```

### 兑换码

#### 单次兑换码

```csharp
var redeemService = new RedeemService();
var resp = await redeemService.RedeemAnonymousAsync("REDEEM_CODE_123");

// 解密兑换内容
string content = redeemService.DecryptRedeemContent(resp.Data.ContentEncrypted);
```

#### 全局兑换码

```csharp
var redeemService = new RedeemService();
var resp = await redeemService.RedeemWithAccountAsync(email, accessToken, "REDEEM_CODE_123");

// 解密兑换内容
string content = redeemService.DecryptRedeemContent(resp.Data.ContentEncrypted);
```

### 退出登录

```csharp
var oauthService = new OAuthService();
var resp = await oauthService.LogoutAsync(email, accessToken);
// 请清除本地保存的 accessToken
```

## 配置说明

### GASConfig 配置

配置文件位于 `Resources/GASConfig.asset`，包含以下字段：

- **App Id** (`int`): 应用 ID，从 GAS 平台获取
- **App Token** (`string`): 应用令牌，从 GAS 平台获取

### 语言配置

Api 接口支持两种语言：

- `GASLang.zh` - 中文（默认）
- `GASLang.en` - 英文

可以通过 `GASConfigManager.Lang` 动态切换：

```csharp
GASConfigManager.Lang = GASLang.en; // 切换到英文
```

## 错误处理

### 异常类型

系统定义了三种异常类型：

1. **GASException** - 通用异常
   ```csharp
   catch (GASException ex)
   {
       Debug.LogError($"错误代码: {ex.Code}, 消息: {ex.Message}");
   }
   ```

2. **GASNetworkException** - 网络异常
   ```csharp
   catch (GASNetworkException ex)
   {
       Debug.LogError($"网络错误: {ex.Code}, {ex.Message}");
       Debug.LogError($"原始响应: {ex.RawText}");
   }
   ```

3. **GASParseException** - JSON 解析异常
   ```csharp
   catch (GASParseException ex)
   {
       Debug.LogError($"解析错误: {ex.Message}");
   }
   ```

### 响应检查

所有服务方法都会自动检查响应状态码，如果 `code != 200`，会抛出 `GASException`。

### 日志记录

系统会自动记录所有 HTTP 请求和响应，包括：
- 请求方法、URL、请求体
- 响应状态码、响应体、耗时

日志通过 `GASResponseLogger` 输出到 Unity Console。

## 示例代码

完整的使用示例请参考 `Demo/Scripts/GASDemo.cs`，其中包含：

- OAuth 完整登录流程
- 用户信息获取
- 云存档的保存和读取
- 版本查询
- 兑换码使用
- 退出登录

### 关键代码片段

```csharp
// OAuth 登录流程
private async void RunOAuth()
{
    // 1. 获取 auth_token
    var resp = await _oauth.GetAuthTokenAsync();
    _authToken = resp.Data.AuthToken;
    
    // 2. 打开浏览器
    GASBrowserHandler.OpenAuthBrowser(appId, _authToken);
    
    // 3. 轮询获取 access_token
    OnAuthCallbackPolling(_authToken);
}

// 保存云存档
private async void RunSaveArchive()
{
    string content = "{\"level\":1, \"percent\":100}";
    var resp = await _archive.SaveAsync(_email, _accessToken, Application.version, content);
    Debug.Log(resp.Msg);
}

// 读取云存档
private async void RunReadArchive()
{
    var resp = await _archive.ReadAsync(_email, _accessToken);
    string decrypted = _archive.DecryptArchiveContent(resp.Data.ContentEncrypted);
    Debug.Log("存档内容: " + decrypted);
}
```

## 安全说明

- **加密算法**: 使用 AES-256-CBC 加密敏感数据
- **密钥派生**: 使用 SHA256 从 AppToken 派生加密密钥
- **传输安全**: 所有 API 请求通过 HTTPS 传输
- **令牌管理**: 请妥善保管 `accessToken`，建议存储在安全的位置

## 注意事项

1. **配置文件位置**: `GASConfig.asset` 必须放在 `Resources/` 目录下
2. **异步操作**: 所有网络请求都是异步的，使用 `await` 等待结果
3. **OAuth 轮询**: 建议实现超时机制，避免无限轮询
4. **存档格式**: 云存档内容应为 JSON 字符串格式
5. **版本号**: 保存存档时需要提供版本号，建议使用 `Application.version`

## API 端点

所有 API 端点定义在 `GASApiRoute.Endpoints` 中：

- `Oauth` - OAuth 授权登录
- `AutoLogin` - 自动登录
- `Profile` - 用户信息
- `Archive` - 云存档
- `Version` - 版本管理
- `Redeem` - 兑换码
- `Config` - 远程配置获取

## 鸣谢

幻影の刄  
DL RS 同人群官方网站  
DL RS 同人社区全体开发者

## 开源协议

This package is licensed under The MIT License (MIT)

---

此文档使用 Cursor 辅助生成  

Author: BluesDawn  
Date: 2025-11-29  
Edit: 2025-12-23