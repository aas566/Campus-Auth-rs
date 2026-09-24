# Campus-Auth 唐山校园网适配包

基于 [Campus-Auth](https://github.com/Misyra/Campus-Auth-rs)（Misyra/Campus-Auth，Windows 单文件版）的校园网 Portal 自动登录适配。

> 仅限本人账号自用，请遵守学校校园网管理规定；禁止批量登录、代登。

## 适配内容

| 文件 | 说明 |
| --- | --- |
| `tasks/browser/ts-asiainfo-portal.json` | 亚信 AAA 门户直连登录任务（页面内 fetch：redirectUri → user/login，密码 MD5，已在线自动跳过） |
| `config/profile.example.json` | 配置方案模板（账号/密码用占位符，请自行替换） |
| `docs/portal-asiainfo-reverse-notes.md` | 门户接口逆向记录（登录接口、报文、校验规则） |

## 快速开始（Windows）

1. 从 [Campus-Auth 官方 Release](https://github.com/Misyra/Campus-Auth/releases) 下载 Windows 单文件版，解压到本地目录（如 `C:\campus-auth`）。
2. 把本适配包 `tasks/browser/ts-asiainfo-portal.json` 放入运行目录 `tasks/browser/` 下。
3. 修改方案配置（见下），把 `active_task` 设为 `ts-asiainfo-portal`。
4. 启动 `campus-auth.exe`，打开本地控制台（默认 `http://127.0.0.1:50721`），确认监测已开启、网络状态为在线。
5. 断线时监测会自动执行该任务登录；开机自启可在控制台「设置 · 系统」开启。

### 方案配置（config/profiles/default.json 示例）

```json
{
  "id": "default",
  "name": "唐山校园网",
  "username": "你的上网账号",
  "password": "ENC:填写控制台加密后的密文（或在控制台账号页直接填写密码，应用会自动加密存储）",
  "auth_url": "http://61.240.139.120:18088/?userip=你的IP&nasip=NAS的IP&user-mac=你的MAC&user-vlan=你的VLAN",
  "trigger_url": "http://www.msftconnecttest.com/connecttest.txt",
  "isp": "",
  "gateway_ip": "",
  "wifi_ssid": "",
  "active_task": "ts-asiainfo-portal"
}
```

说明：
- `auth_url`：校园网门户重定向 URL（浏览器打开门户时的完整地址，含 userip/nasip/user-mac/user-vlan 参数）。
- `trigger_url`：连通性探测地址（触发型门户用明文 http，断线访问会 302 重定向到门户，任务据此拿到最新参数自动登录）。
- 密码无需手工生成 ENC：在控制台账号页输入明文密码保存即可，应用自动加密落盘。

## 工作原理

1. 监测发现断网（探测触发地址无重定向/探测失败）。
2. 执行活跃任务：访问 `trigger_url` → 离线时被 NAS 重定向到门户（带最新接入参数）。
3. 页面内 fetch 依次调用：
   - `POST /aaaselfservice/api/v1/portal/redirectUri`（建立会话，获得 JSESSIONID）
   - `POST /aaaselfservice/api/v1/user/login`（密码 MD5，提交账号与接入参数）
4. 成功判定：`resultCode == "000000"`，或 `600005` 且非 timeout（即已在线，无需重复登录）。
5. 已在线时（触发地址直接响应、未重定向）任务自动跳过登录，返回成功。

## 更新上游

本 fork 是上游的适配分支；上游更新后手动同步：

```bash
git remote add upstream https://github.com/Misyra/Campus-Auth-rs.git
git fetch upstream
git merge upstream/master
```
