# 二维码登录流程说明

更新时间：2026-07-09

这份文档说明 Tovigor 设备端 App、认证后端、微信小程序之间如何配合完成扫码登录。

## 一句话版本

设备端先向后端申请一个临时登录会话，后端生成二维码。用户用微信扫码进入小程序，小程序把微信登录 `code` 和二维码里的 `sessionId` 发回后端。设备端持续轮询这个 `sessionId`，发现后端确认登录后，拿到 `token/user`，再进入首页。

## 涉及三方

设备端 App：

```text
tovigor_yei_unIappX
```

负责显示屏保、显示二维码、轮询登录状态、保存 `token/user`、进入首页。

认证后端：

```text
tovigor-auth-service
```

负责创建二维码登录会话、生成二维码/小程序码、接收小程序确认登录、返回 `token/user`。

微信小程序：

```text
tovigor-user-miniprogram
```

负责扫码后打开页面、读取 `scene/sessionId`、调用 `wx.login`、把 `code` 发给后端确认登录。后续这个项目还可以继续承载用户训练信息、训练报告、历史记录等功能。

## 设备端流程

入口一：首页首屏屏保：

```text
IdleOverlay -> LoginQrOverlay -> 首页内容
```

入口二：独立屏保路由：

```text
pages/idle/idle.uvue -> LoginQrOverlay -> pages/index/index
```

完整步骤：

1. 用户点击屏保唤醒。
2. `LoginQrOverlay.uvue` 挂载。
3. `LoginQrOverlay.uvue` 调用 `createQrSession()`。
4. `utils/auth-api.uts` 请求后端：

```http
POST http://122.51.155.6:10086/api/auth/qr-sessions
```

5. 后端返回二维码登录会话：

```json
{
  "sessionId": "xxx",
  "qrContent": "xxx",
  "qrImageBase64": "data:image/png;base64,...",
  "expiresAt": "2026-07-09T17:00:00+08:00"
}
```

6. `LoginQrOverlay.uvue` 显示这张二维码图片。
7. 设备端每 2 秒轮询一次：

```http
GET http://122.51.155.6:10086/api/auth/qr-sessions/{sessionId}
```

8. 如果返回 `PENDING`，继续等待。
9. 如果返回 `EXPIRED`，提示二维码过期，并显示“重新获取二维码”。
10. 如果返回 `CONFIRMED`，保存 `token/user`，关闭登录层，进入首页。

登录成功返回示例：

```json
{
  "status": "CONFIRMED",
  "token": "jwt-token",
  "user": {
    "id": "user-id",
    "openid": "wechat-openid",
    "nickname": ""
  }
}
```

设备端保存位置：

```text
uni storage:
tovigor_auth_token
tovigor_auth_user
```

## 后端流程

后端接口：

```http
GET  /api/health
POST /api/auth/qr-sessions
GET  /api/auth/qr-sessions/{sessionId}
POST /api/auth/wechat/confirm
```

### 创建设备登录会话

设备端请求：

```http
POST /api/auth/qr-sessions
```

后端做这些事：

1. 生成一个 `sessionId`。
2. 在 Redis 里保存一个待登录会话，状态是 `PENDING`。
3. 生成二维码图片。
4. 把 `sessionId`、二维码内容、二维码图片 base64、过期时间返回给设备端。

### 二维码怎么生成

后端有两种生成方式：

配置了 `WECHAT_APP_ID` 和 `WECHAT_APP_SECRET` 时：

```http
POST https://api.weixin.qq.com/wxa/getwxacodeunlimit
```

后端生成真正的微信小程序码。小程序码里的 `scene` 就是：

```text
sessionId
```

微信扫码后会进入后端配置的页面：

```text
pages/login/scan-login
```

没有配置微信凭据时：

后端退回普通二维码，内容类似：

```text
tovigor-auth://qr-login?sessionId=xxx
```

这个主要用于本地开发和兜底调试。它不是微信官方小程序码，微信扫码后通常只会打开一段文本或链接，不会自动进入我们的小程序页面。

### 小程序确认登录

用户扫设备端二维码后，小程序打开页面并拿到 `scene`。

小程序会做三件事：

1. 从 `options.scene` 里取 `sessionId`。
2. 调用 `wx.login()` 获取 `code`。
3. 请求后端确认登录：

```http
POST http://122.51.155.6:10086/api/auth/wechat/confirm
```

请求体示例：

```json
{
  "sessionId": "xxx",
  "code": "wechat-login-code",
  "nickname": "",
  "avatarUrl": ""
}
```

后端做这些事：

1. 校验 `sessionId` 是否存在、是否过期、是否已经确认。
2. 用 `code` 调微信 `jscode2session` 换取 `openid/unionid`。
3. 查找或创建用户。
4. 把 Redis 里的二维码会话状态改成 `CONFIRMED`。
5. 设备端下一次轮询时就能拿到 `token/user`。

## 前端和后端怎么沟通

设备端通过 `uni.request` 请求后端。封装文件：

```text
tovigor_yei_unIappX/utils/auth-api.uts
```

主要函数：

```text
createQrSession()
getQrSessionStatus(sessionId)
saveAuthSession(token, user)
clearAuthSession()
getStoredAuthToken()
```

当前后端地址：

```text
http://122.51.155.6:10086
```

如果后端以后换成域名，只需要把设备端和小程序里的接口地址一起改成新域名即可。

## 为什么之前提到网卡

如果后端跑在公网服务器上，设备端直接请求服务器公网地址即可：

```text
http://122.51.155.6:10086
```

这种部署方式下，设备端不需要关心 Windows 电脑的网卡 IP。

网卡问题只发生在“后端跑在你的 Windows 电脑上，Android 设备要访问这台电脑”的本地联调场景。因为设备端 App 跑在 Android 设备上，后端跑在 Windows 电脑上，它们不是同一个“本机”。

Windows 电脑上的：

```text
127.0.0.1
localhost
```

表示电脑自己。

Android 设备上的：

```text
127.0.0.1
localhost
```

表示 Android 设备自己，不是你的 Windows 电脑。

所以真机访问电脑本地后端时，设备端必须请求电脑在局域网里的 IP，例如：

```text
http://10.0.50.114:10086
```

现在后端已经部署到公网服务器，这个问题就暂时不用管。

## 后端端口

端口是后端服务器进程监听的端口，不是 Android 的端口。

当前后端默认端口：

```text
10086
```

服务地址：

```text
http://122.51.155.6:10086
```

服务器安全组/防火墙需要放行 `10086` 入站流量。正式环境更推荐用 Nginx 反代到 HTTPS 域名，例如：

```text
https://auth.tovigor.com
```

## 后端必须配置什么

基础运行：

```text
MYSQL_URL
MYSQL_USERNAME
MYSQL_PASSWORD
REDIS_HOST
REDIS_PORT
JWT_SECRET
```

生成真正微信小程序码：

```text
WECHAT_APP_ID
WECHAT_APP_SECRET
WECHAT_QR_CODE_PAGE=pages/login/scan-login
WECHAT_QR_CODE_ENV_VERSION=trial
```

`WECHAT_QR_CODE_ENV_VERSION` 常用值：

```text
release
trial
develop
```

`WECHAT_QR_CODE_ENV_VERSION` 必须和微信后台里的版本状态对应：开发版用 `develop`，体验版用 `trial`，正式发布后用 `release`。

## 完整时序图

```text
设备端 App                 认证后端                    微信接口/小程序
   |                         |                          |
   | POST /qr-sessions       |                          |
   |------------------------>|                          |
   |                         | 生成 sessionId            |
   |                         | 保存 PENDING 到 Redis      |
   |                         | 请求微信生成小程序码        |
   |                         |------------------------->|
   |                         |<-------------------------|
   |<------------------------| 返回 qrImageBase64         |
   | 显示二维码               |                          |
   |                         |                          |
   | GET /qr-sessions/{id}   |                          |
   |------------------------>|                          |
   |<------------------------| PENDING                  |
   |                         |                          |
   | 用户微信扫码             |                          |
   |--------------------------------------------------->|
   |                         |                          | 小程序打开，拿到 scene=sessionId
   |                         |<-------------------------| POST /wechat/confirm
   |                         | 调微信 code2session        |
   |                         |------------------------->|
   |                         |<-------------------------| openid/unionid
   |                         | 保存 CONFIRMED             |
   |                         |                          |
   | GET /qr-sessions/{id}   |                          |
   |------------------------>|                          |
   |<------------------------| CONFIRMED + token + user  |
   | 保存 token/user          |                          |
   | 进入首页                 |                          |
```

## 当前已完成

设备端已完成：

```text
屏保唤醒后显示二维码登录层
请求后端创建二维码会话
显示后端返回的二维码 base64
轮询登录状态
登录成功后保存 token/user
登录成功后进入首页
```

## 临时跳过登录开关

二维码链路调试期间，如果需要先进入首页调试其他功能，可以打开设备端临时开关：

```text
tovigor_yei_unIappX/utils/auth-config.uts
```

当前值：

```ts
export const SKIP_QR_LOGIN_FOR_DEV = false
```

`true` 表示屏保唤醒后直接进入首页，不显示二维码登录层。

二维码登录修好后改回：

```ts
export const SKIP_QR_LOGIN_FOR_DEV = false
```

后端已完成：

```text
健康检查
创建二维码会话
查询二维码会话状态
小程序确认登录
配置微信凭据时生成微信小程序码
未配置微信凭据时生成普通二维码兜底
```

小程序已完成：

```text
项目目录：tovigor-user-miniprogram
接收 scene/sessionId
调用 wx.login
请求 POST /api/auth/wechat/confirm
处理确认成功/失败提示
```
