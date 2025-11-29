# ConvertEasy 小程序云托管配置说明

## 云托管实例信息

- **云环境 ID**: `convert-easy-9gy01nt7e03d9579`
- **服务名称**: `convert-easy`
- **公网访问地址**: `https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com`

## 已配置文件

### 1. `app.js` - 云开发初始化

```javascript
wx.cloud.init({
  env: "convert-easy-9gy01nt7e03d9579",
  traceUser: true,
});
```

✅ 已正确配置

### 2. `utils/api.js` - API 配置

```javascript
const CLOUD_CONFIG = {
  env: "convert-easy-9gy01nt7e03d9579",
  serviceName: "convert-easy",
};

const PUBLIC_BASE_URL =
  "https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com";
```

✅ 已更新配置

## API 调用方式

### 云调用方式（文档转换）

使用 `wx.cloud.callContainer` API：

```javascript
wx.cloud.callContainer({
  config: {
    env: "convert-easy-9gy01nt7e03d9579",
  },
  path: "/convert/upload",
  header: {
    "X-WX-SERVICE": "convert-easy",
  },
  method: "POST",
  data: {
    downloadUrl: tempUrl,
    cloudPath: cloudPath,
    category: "document",
    target: targetFormat,
    source: sourceFormat,
  },
});
```

**使用场景**：

- 文档转换（`pages/document/document.js`）
- 格式查询
- 健康检查

**优点**：

- 无需配置服务器域名白名单
- 自动处理鉴权
- 与云开发深度集成

### HTTP 方式（音频转换）

使用 `wx.request` 和 `wx.uploadFile` API：

```javascript
wx.uploadFile({
  url: "https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com/convert/upload",
  filePath: localFilePath,
  name: "file",
  formData: {
    category: "audio",
    target: targetFormat,
  },
});
```

**使用场景**：

- 音频转换（`pages/audio/audio.js`）
- 需要直接上传文件的场景

**注意事项**：

- 需要在小程序后台配置服务器域名白名单
- 需要手动处理文件上传

## 小程序后台配置

### 服务器域名配置

登录 [微信公众平台](https://mp.weixin.qq.com/)，进入"开发" > "开发管理" > "开发设置" > "服务器域名"：

#### request 合法域名

```
https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com
```

#### uploadFile 合法域名

```
https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com
```

#### downloadFile 合法域名

```
https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com
```

### 云开发环境配置

确保在云开发控制台正确配置：

1. 进入 [云开发控制台](https://console.cloud.tencent.com/tcb)
2. 选择环境：`convert-easy-9gy01nt7e03d9579`
3. 云托管 > 服务列表，确认服务 `convert-easy` 正在运行
4. 检查容器配置和公网访问设置

## 测试连接

### 方法 1: 在小程序中测试

打开"文档转换"页面，会自动执行健康检查：

```javascript
async testCloudConnection() {
  try {
    await healthCheckByCloud();
    console.log('✅ 云调用连接成功');
    showToast('云服务连接正常', 'success');
  } catch (err) {
    console.error('❌ 云调用连接失败:', err);
    showToast('云服务连接失败', 'none');
  }
}
```

### 方法 2: 使用 curl 测试公网接口

```bash
# 健康检查
curl https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com/health

# 查询支持的格式
curl https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com/supported-formats?category=document
```

## API 端点说明

| 端点                     | 方法 | 说明               | 调用方式    |
| ------------------------ | ---- | ------------------ | ----------- |
| `/health`                | GET  | 健康检查           | 云调用      |
| `/supported-formats`     | GET  | 查询支持的格式     | 云调用/HTTP |
| `/detect-targets`        | POST | 检测文件可转换格式 | 云调用      |
| `/convert/upload`        | POST | 上传并转换文件     | 云调用/HTTP |
| `/convert/task/{taskId}` | GET  | 查询任务状态       | 云调用/HTTP |
| `/public/{filename}`     | GET  | 下载转换后的文件   | HTTP        |

## 工作流程

### 文档转换流程（云调用）

1. 用户选择文件 → `wx.chooseMessageFile()`
2. 上传到云存储 → `wx.cloud.uploadFile()`
3. 获取临时 URL → `wx.cloud.getTempFileURL()`
4. 云调用转换接口 → `wx.cloud.callContainer()`
5. 轮询任务状态 → 云调用 `/convert/task/{taskId}`
6. 下载结果文件 → HTTP 下载

### 音频转换流程（HTTP）

1. 用户选择文件 → `wx.chooseMessageFile()`
2. 直接上传到服务器 → `wx.uploadFile()`
3. 轮询任务状态 → HTTP `/convert/task/{taskId}`
4. 下载结果文件 → HTTP 下载

## 常见问题

### Q1: 提示"云服务连接失败"

**可能原因**：

- 云环境 ID 配置错误
- 服务名称配置错误
- 云托管服务未启动
- 网络连接问题

**排查步骤**：

1. 检查 `app.js` 和 `utils/api.js` 中的配置
2. 在云开发控制台确认服务状态
3. 查看小程序开发工具控制台的详细错误信息

### Q2: HTTP 请求失败

**可能原因**：

- 服务器域名未配置白名单
- 公网地址配置错误
- 后端服务未启动

**排查步骤**：

1. 在小程序后台配置服务器域名白名单
2. 检查 `utils/api.js` 中的 `PUBLIC_BASE_URL`
3. 使用 curl 测试公网地址是否可访问

### Q3: 文件上传失败

**可能原因**：

- 云存储配置问题
- 文件大小超限
- 临时 URL 获取失败

**排查步骤**：

1. 确认云存储已开通并配置权限
2. 检查文件大小（建议小于 10MB）
3. 查看控制台详细错误信息

## 开发建议

### 1. 本地调试

在微信开发者工具中：

- 开启"不校验合法域名"（仅开发时）
- 查看 Console 和 Network 标签的详细日志
- 使用云调用测试工具测试 API

### 2. 错误处理

所有 API 调用都应包含 try-catch：

```javascript
try {
  const result = await createDocumentConvertTask({
    filePath,
    targetFormat,
    sourceFormat,
  });
  // 处理成功
} catch (error) {
  console.error("转换失败:", error);
  showToast(error.message || "转换失败");
}
```

### 3. 日志监控

在云开发控制台 > 云托管 > 日志查询中：

- 查看服务运行日志
- 监控错误和异常
- 分析性能问题

## 更新日志

- **2024-11-29**: 初始配置
  - 配置云环境 ID: `convert-easy-9gy01nt7e03d9579`
  - 配置服务名称: `convert-easy`
  - 更新公网地址: `https://convert-easy-203720-5-1389303207.sh.run.tcloudbase.com`

## 相关文档

- [微信云开发文档](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/basis/getting-started.html)
- [云托管文档](https://cloud.weixin.qq.com/cloudrun)
- [wx.cloud.callContainer API](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/reference-sdk-api/utils/Cloud.callContainer.html)
- [后端 API 文档](../backend/README.md)
