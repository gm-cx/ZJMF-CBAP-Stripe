# Stripe 支付插件安装与配置指南

**适用系统**：智简魔方（IDCsmart）财务系统  
**插件版本**：1.0.0  
**功能描述**：集成 Stripe 支付网关，支持信用卡、支付宝等多种支付方式，采用 Stripe Checkout 托管页面完成收款。

---

## 目录

- [环境要求](#环境要求)
- [文件结构](#文件结构)
- [安装步骤](#安装步骤)
  - [上传插件文件](#上传插件文件)
  - [安装 Stripe PHP SDK](#安装-stripe-php-sdk)
- [后台配置](#后台配置)
  - [启用插件](#启用插件)
  - [填写 API 密钥](#填写-api-密钥)
- [Stripe Webhook 配置](#stripe-webhook-配置)
- [测试支付流程](#测试支付流程)
- [常见问题排查](#常见问题排查)

---

## 环境要求

| 项目 | 要求 |
|------|------|
| PHP 版本 | ≥ 7.4 |
| PHP 扩展 | curl, json, openssl |
| Stripe 账户 | 已激活的生产或测试账户 |
| 系统版本 | 智简魔方 v3.x |

---

## 文件结构

插件目录位于 `/public/plugins/gateway/stripe_pay/`，上传后结构如下：


stripe_pay/
├── StripePay.php # 插件入口文件（必须）
├── StripePay.png # 支付方式图标（200×100px）
├── config/
│ └── config.php # 后台配置表单定义
├── controller/
│ └── IndexController.php # 同步/异步回调处理器
└── lib/ # Stripe PHP SDK（需手动放置）
├── init.php
├── lib/
│ └── Stripe/
└── data/



## 安装步骤

### 上传插件文件

1. 将 `stripe_pay` 整个文件夹上传至服务器目录：/public/plugins/gateway/
2. 确保目录权限为 `755`，文件权限为 `644`。

### 安装 Stripe PHP SDK

由于系统环境限制，推荐手动下载 SDK 源码包。

1. 访问 [Stripe PHP GitHub Releases](https://github.com/stripe/stripe-php/releases) 下载最新版 `Source code (zip)`，例如 `stripe-php-13.x.x.zip`。
2. 解压后得到文件夹，内部包含 `init.php`、`lib`、`data` 等。
3. 将**所有内容**复制到插件目录下的 `lib/` 文件夹内，确保路径 `lib/init.php` 存在。

> 若使用 Composer，可在插件目录执行 `composer require stripe/stripe-php`，然后将生成的 `vendor` 目录重命名为 `lib`。

---

## 后台配置

### 启用插件

1. 登录魔方管理后台，进入「系统设置」→「插件管理」。
2. 刷新页面，在支付网关列表中找到「Stripe支付」。
3. 点击「安装」，然后点击「设置」进入配置界面。

### 填写 API 密钥

| 配置项 | 说明 |
|--------|------|
| Secret Key | Stripe 生产环境私钥，以 `sk_live_` 开头。在 [Stripe Dashboard](https://dashboard.stripe.com/apikeys) 获取。 |
| Publishable Key | Stripe 生产环境公钥，以 `pk_live_` 开头。 |
| Webhook 签名密钥 | 用于验证异步通知签名，稍后在 Webhook 配置中获取。 |
| 结算货币 | 选择支付使用的货币（支持 USD、EUR、GBP、JPY、CNY、HKD）。 |

> **测试建议**：若需沙箱测试，请暂时使用测试密钥（`sk_test_`、`pk_test_`），测试完成后切换为生产密钥。

---

## Stripe Webhook 配置

Webhook 是 Stripe 异步通知魔方系统订单支付成功的关键，**必须正确配置**。

### 1. 获取 Webhook URL

您的回调地址为：https://您的域名/gateway/stripe_pay/index/notifyHandle
请将 `https://您的域名` 替换为实际网站地址，**必须使用 HTTPS**（测试环境可临时使用 HTTP）。

### 2. 在 Stripe Dashboard 中添加 Endpoint

1. 登录 Stripe Dashboard，进入 **Developers** → **Webhooks**。
2. 点击 **Add endpoint**。
3. 在 **Endpoint URL** 填入上述回调地址。
4. **Events to send** 选择 `checkout.session.completed`（或同时勾选 `payment_intent.succeeded`）。
5. 点击 **Add endpoint** 完成创建。

### 3. 获取签名密钥并填入后台

1. 在刚创建的 Endpoint 详情页，点击 **Reveal** 按钮复制 `whsec_xxx` 开头的密钥。
2. 返回魔方后台插件设置页面，将密钥粘贴至「Webhook 签名密钥」字段。
3. 保存配置。

> **验证**：在 Dashboard 中点击 Endpoint 右侧的「Send test webhook」，若返回 200 状态码则配置成功。

---

## 测试支付流程

1. 前台提交一个测试订单，支付方式选择「Stripe支付」。
2. 页面显示「前往 Stripe 支付页面」按钮，点击后跳转至 Stripe 托管支付页。
3. 使用测试卡号完成支付：
   - 卡号：`4242 4242 4242 4242`
   - 有效期：任意未来日期
   - CVC：任意 3 位数字
   - 持卡人姓名：任意
4. 支付成功后，Stripe 将跳回您的网站，Webhook 会更新订单状态为「已支付」。

> 若订单未更新，请参考下方「常见问题排查」中的第 2 点。

---

## 常见问题排查

### 1. 支付页面提示「Stripe SDK 未安装」

**原因**：`lib/init.php` 文件未找到或路径错误。  
**解决**：检查 `/public/plugins/gateway/stripe_pay/lib/init.php` 是否存在，并确保文件有读取权限。

### 2. 支付成功但订单状态未更新

**原因**：Webhook 未正确触发或签名验证失败。  
**排查步骤**：
- 在 Stripe Dashboard → Webhooks 中查看最近发送记录，确认 HTTP 状态码为 `200`。
- 检查魔方后台「Webhook 签名密钥」是否与 Dashboard 中的完全一致。
- 查看服务器错误日志（`/runtime/log/`），搜索 Stripe 相关报错。
- 确认服务器防火墙未拦截 Stripe 的回调 IP（Stripe 官方 IP 列表见其文档）。

### 3. 支付页面提示「密钥未配置」

**原因**：插件设置中 Secret Key 或 Publishable Key 为空。  
**解决**：重新进入插件设置页面，填写完整密钥后保存。

### 4. 支付时货币显示异常或汇率错误

**原因**：Stripe 账户不支持所选货币，或未开启相应货币结算。  
**解决**：确保 Stripe 账户已激活所选的货币类型，或在后台切换为账户支持的货币（如 USD）。

### 5. 浏览器控制台报错 `Stripe is not defined`

**原因**：支付页面未能加载 Stripe.js 库。  
**解决**：检查服务器能否正常访问 `https://js.stripe.com/v3/`，或是否有前端广告拦截插件干扰。

### 6. 支付完成跳转后显示空白页

**原因**：同步返回地址（`returnHandle`）未设置正确的展示逻辑。  
**解决**：可自定义 `controller/IndexController.php` 中的 `returnHandle` 方法，添加友好的提示或重定向。

---

## 注意事项

- **生产环境务必使用 HTTPS**，否则 Webhook 可能被 Stripe 拒绝。
- 若切换为生产模式，请务必替换为 `sk_live_` 和 `pk_live_` 开头的密钥。
- 退款功能需在魔方后台订单详情页发起，插件已实现 `StripePayHandleRefund` 方法，支持原路退回。

---

## 获取帮助

如有其他问题，请联系开发团队或查阅以下资源：
- [Stripe 官方文档](https://stripe.com/docs)
- [智简魔方支付接口开发文档](http://doc.idcsmart.com/)
