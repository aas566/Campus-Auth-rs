# 亚信 AAA 自服务门户接口逆向笔记

唐山校园网门户（Vue SPA，标题 self-care，© AsiaInfo 亚信）自动登录所需接口记录。
逆向依据：门户主包 `js/appdbd597.js` 与登录 chunk `static/js/408.js`（webpack 懒加载模块 408）。

## 登录流程

1. **建立会话**（必须先调用，否则登录报 600005 Login timeout）

```
POST /aaaselfservice/api/v1/portal/redirectUri
Content-Type: application/json
Referer: <门户 URL>（必填，服务端校验 Referer，缺失返回 403 Invalid Referer）

{ "urlParams": { "userip": "...", "nasip": "...", "user-mac": "...", "user-vlan": "..." } }
```

响应（成功）：`{"resultCode":"000000","result":{"instanceId":179,"instanceIdY":183,...}}`
同时服务端下发 `JSESSIONID` Cookie（后续请求需携带）。

2. **登录**

```
POST /aaaselfservice/api/v1/user/login
Content-Type: application/json
Referer: <门户 URL>
Cookie: JSESSIONID=...

{
  "account": "上网账号",
  "password": "<md5(明文密码) 32位小写>",
  "macBindDays": 7,
  "code": "",
  "picCode": "",
  "userName": "上网账号",
  "urlParams": { "userip": "...", "nasip": "...", "user-mac": "...", "user-vlan": "..." },
  "verifyCode": ""
}
```

## 成功 / 失败判定

| resultCode | 含义 | 处理 |
| --- | --- | --- |
| `000000` | 登录成功 | 成功 |
| `600005` + desc 含「已经连接/无需重复登录/already connected」 | 已在线（MAC/IP 已认证） | 视为成功（在线） |
| `600005` + desc 含 timeout/超时 | 会话无效/参数过期 | 失败，需重新 redirectUri 建会话 |
| `400006/400007/400008` 等 | 业务错误（如需要验证码） | 失败 |

注意：同一 resultCode 600005 的中英文提示取决于会话语言，判定时按「非 timeout」处理更稳妥。

## 其他接口

| 方法/路径 | 说明 |
| --- | --- |
| `POST /aaaselfservice/api/v1/user/logout` | 登出（请求体为裸 urlParams 对象；需已登录会话 token，否则 400006） |
| `POST /aaaselfservice/api/v1/user/updatePassword` | 改密 |
| `GET /aaaselfservice/api/v1/portal/instance?instanceId=N` | 获取门户模板实例 |
| `GET /aaaselfservice/api/v1/user/captcha` | 图形验证码（仅部分场景出现） |

## 关键细节

- **Referer 校验**：所有 API 请求必须带门户页 Referer（`http://61.240.139.120:18088/...`），否则 HTTP 403。
- **密码 MD5**：提交前对明文密码做 MD5（小写十六进制）。
- **接入参数**：`userip/nasip/user-mac/user-vlan` 取自门户重定向 URL 的 query；其中 `user-mac` 为 URL 编码形式（如 `fc%3Ab0%3Ade...`），提交时需解码为 `fc:b0:de:...`。
- **无验证码 / 无运营商选择**：该门户当前登录表单仅有账号、密码、7 天免登录，无验证码步骤，无需 OCR。
- **Cookie 会话**：redirectUri 响应设置 JSESSIONID；同一页面内同源 fetch 自动携带。
