# KV 订阅短链接

本分支的 `worker.js` 在原有订阅转换功能上增加 KV 短链接：

- 面板生成订阅链接后，可点击“生成并复制”创建短链接。
- 短链接形如 `https://你的-worker域名/s/随机码`。
- 访问短链接时返回 HTTP 302，跳转到原订阅链接。
- 短链映射存储于 Cloudflare Workers KV；默认不自动过期，删除 KV 对应记录即可失效。
- 原有 `SUB_BUCKET`（KV 或 R2）用途不变，短链接使用单独的 `SHORT_LINKS` 绑定。

## Cloudflare Dashboard 部署

1. 打开 Cloudflare Dashboard → **Workers & Pages** → 你的 psub Worker。
2. 打开 **Settings / 设置 → Bindings / 绑定**。
3. 添加 **KV namespace**，变量名称必须填写 `SHORT_LINKS`。
4. 选择已有 KV namespace，或新建一个，例如 `psub-short-links`。
5. 保存绑定，然后部署本分支的 `worker.js`。
6. 打开 Worker 首页，在订阅链接输入框填写原有订阅，按平时流程点击“生成订阅链接”；下方“订阅短链”区域再点击“生成并复制”。
7. 在浏览器打开生成的短链接，应返回 302 并跳转到原订阅链接。

不要把现有 `SUB_BUCKET` 绑定改名；原有订阅代理逻辑仍然使用它。

## Wrangler 配置示例

如果你通过 Wrangler 部署，在现有配置中追加 KV 绑定（替换为你自己创建的 ID）：

```toml
[[kv_namespaces]]
binding = "SHORT_LINKS"
id = "替换成你自己的KV_NAMESPACE_ID"
```

保留原有 `SUB_BUCKET` 与 `BACKEND` 配置。

## 路由与数据

- `POST /short`：接受面板兼容的 `multipart/form-data` 字段 `longUrl`（Base64 编码），校验后写入 KV。
- `GET /s/{code}`：读取 KV 并返回 302 跳转。
- KV 键使用 `psub-short:{code}`，值为原始的 HTTP/HTTPS 订阅 URL。

此版本默认不设置过期时间，以免已分享的订阅链接自行失效。需要失效时，可在 Cloudflare KV 中删除对应 `psub-short:` 记录。

## 安全建议

短链创建接口为便于原面板调用，没有登录鉴权，因此任何知道 Worker 域名的人都可以尝试创建短链。建议在 Cloudflare WAF / Rate Limiting 中限制 `POST /short` 的请求频率，并关注 KV 使用量。短链只是重定向地址，不会加密或隐藏最终目标，也不会替代订阅本身的访问权限控制。

## 前端兼容说明

项目会在请求首页时取得上游的 `frontend.html`，并注入短链按钮与接口地址。若上游前端更新了压缩后的 Vue 渲染代码，界面注入点可能需要同步更新；Worker 的 `/short` 与 `/s/{code}` API 本身独立于这部分面板注入。
