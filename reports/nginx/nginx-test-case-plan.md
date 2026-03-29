# Nginx 功能测试用例方案

**版本**: 1.0  
**QA**: Agent (QA Engineer)  
**日期**: 2026-03-29  
**项目**: V2Ray Nginx + Certbot 自动 TLS 配置模块

---

## 1. 测试方案概述

### 1.1 测试目标

| 目标 | 说明 |
|------|------|
| **功能正确性** | 确保所有协议（WebSocket/HTTP2/gRPC）配置生成正确 |
| **证书管理** | 验证 Certbot 证书申请、续期、软链接创建流程，包括版本验证 |
| **配置冲突处理** | 检查重复配置时的备份与覆盖逻辑，支持多协议配置合并 |
| **Nginx 重载/重启** | 验证服务管理函数的可靠性，包括配置回滚机制 |
| **集成稳定性** | 确保本地测试与生产环境一致性 |
| **高可用性** | 验证 HTTPS 跳转、证书过期告警、DNS 解析失败处理 |
| **并发控制** | 验证多进程并发配置操作的安全性 |
| **版本验证** | 确保 Certbot 版本 ≥1.21.0 |

### 1.2 测试范围

| 测试类型 | 覆盖内容 |
|----------|---------|
| **协议配置** | `nginx_config *ws*` (WebSocket), `nginx_config *h2*` (HTTP2), `nginx_config *grpc*` (gRPC) |
| **基础操作** | `nginx_config new`, `nginx_config del`, `nginx_reload`, `nginx_restart` |
| **证书管理** | `nginx_certbot issue` (首次申请/续期), `nginx_certbot renew` |
| **配置冲突** | 多协议共存、重复域名配置处理、配置合并逻辑 |
| **边界条件** | 空路径、特殊字符域名、端口边界、DNS 解析失败 |
| **高可用性** | HTTPS 跳转、证书过期告警、配置回滚、并发控制 |
| **版本验证** | Certbot 版本检查（≥1.21.0） |

### 1.3 测试方法

- **单元测试**: BATS (Bash Automated Testing System) + Mock 环境
- **集成测试**: VPS 环境 (`proxy.yourdie.com`) 真实 Nginx/Certbot
- **边界测试**: 极端输入（空字符串、超长域名、端口超出范围）
- **端到端测试**: 完整业务流（配置生成→证书申请→服务启动→连接测试）

---

## 2. 详细测试用例

### 2.1 单元测试用例（本地环境）

| ID | 测试名称 | 测试目的 | 测试步骤 | 预期结果 | 环境 |
|----|----------|----------|----------|----------|------|
| **UT-001** | `nginx_config new` 创建基本配置 | 验证新域名首次配置生成 | 1. 调用 `nginx_config new "VLESS" "/v2ray" "8080"`<br>2. 检查配置文件 `VLESS-${HOST}.conf` 内容 | 1. 配置文件正确生成<br>2. 包含 80→443 跳转<br>3. WebSocket 配置（Upgrade/Connection 头） | 本地 |
| **UT-002** | WebSocket 协议配置完整性 | 验证 WebSocket 特定配置项 | 1. 调用 `nginx_config ws "VLESS" "/ws" "10080"`<br>2. 检查配置文件中 `proxy_set_header Upgrade` 和 `Connection` | 1. `proxy_http_version 1.1`<br>2. `proxy_set_header Upgrade $http_upgrade`<br>3. `proxy_set_header Connection "upgrade"` | 本地 |
| **UT-003** | HTTP/2 协议配置验证 | 验证 HTTP/2 监听配置 | 1. 调用 `nginx_config h2 "VLESS" "/h2" "20080"`<br>2. 检查 `listen 443 ssl http2` | 1. 监听语句包含 `http2`<br>2. SSL 配置正确 | 本地 |
| **UT-004** | gRPC 协议配置验证 | 验证 gRPC 特定配置项 | 1. 调用 `nginx_config grpc "VLESS" "/grpc" "30080"`<br>2. 检查 `grpc_pass` 和 `grpc_set_header` | 1. 使用 `grpc_pass grpc://`<br>2. 包含 `grpc_set_header` 指令 | 本地 |
| **UT-005** | 证书申请模拟（NXDOMAIN） | 验证失败场景处理 | 1. Mock `certbot` 返回非 0<br>2. Mock DNS 解析失败（模拟）<br>3. 调用 `nginx_config *` 后检查返回值 | 1. 返回非 0<br>2. 记录错误日志<br>3. 配置文件存在但 Not TLS<br>4. 指数退避重试 3 次 | 本地 |
| **UT-006** | 证书申请模拟（成功） | 验证成功场景处理 | 1. Mock `certbot` 返回 0<br>2. Mock `.add` 文件创建<br>3. 检查软链接创建<br>4. 检查版本验证（Certbot ≥1.21.0） | 1. 返回 0<br>2. 软链接 `/etc/nginx/ssl/{HOST}` 存在<br>3. 版本检查通过 | 本地 |
| **UT-007** | 重复配置冲突处理 | 验证配置覆盖逻辑 | 1. 创建现有配置文件<br>2. 调用相同域名配置<br>3. Mock 用户输入 "1"（覆盖） | 1. 备份为 `.bak`<br>2. 覆盖新配置 | 本地 |
| **UT-008** | `nginx_config del` 删除配置 | 验证配置清理 | 1. 创建多个协议配置<br>2. 调用 `nginx_config del`<br>3. 检查文件删除 | 所有 `*-${HOST}.conf` 及 `.add` 文件被删除 | 本地 |
| **UT-009** | `nginx_reload` 重载配置 | 验证 Nginx 重载流程 | 1. Mock `pgrep` 返回 Nginx PID<br>2. Mock `nginx -s reload`<br>3. 调用 `nginx_reload` | 1. 返回 0<br>2. 执行 `nginx -s reload` | 本地 |
| **UT-010** | `nginx_reload` Nginx 未运行 | 验证自动启动逻辑 | 1. Mock `pgrep` 返回失败<br>2. Mock `systemctl start nginx`<br>3. 调用 `nginx_reload` | 1. 启动 Nginx<br>2. 返回 0 成功 | 本地 |
| **UT-011** | `nginx_test` 配置测试 | 验证 Nginx 配置语法检查 | 1. Mock `nginx -t` 返回 0/1<br>2. 调用 `nginx_test` | 1. 返回对应退出码 | 本地 |
| **UT-012** | `nginx_certbot renew` 续期流程 | 验证证书续期自动化 | 1. Mock 证书过期（<30天）<br>2. 调用 `nginx_certbot renew` | 1. 执行 `certbot renew`<br>2. 触发 `systemctl reload nginx` | 本地 |
| **UT-013** | HTTPS 跳转验证 | 验证 HTTP→HTTPS 自动跳转 | 1. 调用 `nginx_config new`<br>2. 访问 `http://domain.com`<br>3. 检查响应码 | 1. 返回 301/302<br>2. Location 头指向 HTTPS | 本地 |
| **UT-014** | 并发配置操作测试 | 验证多进程竞争保护 | 1. 并发调用 `nginx_config ws` 5 次<br>2. 检查配置一致性 | 1. 所有配置正确生成<br>2. 无冲突或损坏 | 本地 |

---

### 2.2 集成测试用例（VPS 环境）

| ID | 测试名称 | 测试目的 | 测试步骤 | 预期结果 | 环境 |
|----|----------|----------|----------|----------|------|
| **IT-001** | WebSocket 协议端到端 | 验证真实环境 WebSocket 代理 | 1. 部署 Nginx + V2Ray<br>2. 使用 `nginx_config ws` 生成配置<br>3. 使用 `nginx_certbot issue` 申请证书<br>4. `curl -H "Upgrade: websocket" -H "Connection: Upgrade" https://proxy.yourdie.com/ws` | 1. 连接建立<br>2. 转发到后端 8080 端口 | VPS |
| **IT-002** | HTTP/2 协议端到端 | 验证 HTTP/2 连接 | 1. 部署 V2Ray HTTP/2 模式<br>2. 使用 `nghttp` 测试 `curl --http2 https://proxy.yourdie.com/h2` | 1. 请求成功（HTTP/2）<br>2. 后端收到请求 | VPS |
| **IT-003** | gRPC 协议端到端 | 验证 gRPC 请求 | 1. 部署 V2Ray gRPC 模式<br>2. 使用 `grpcurl` 发送请求 | 1. 连接建立<br>2. 返回 gRPC 响应 | VPS |
| **IT-004** | 多站点共存 | 验证共享 80/443 端口 | 1. 配置 `proxy.yourdie.com`<br>2. 配置 `api.proxy.yourdie.com`<br>3. 检查 `ss -tlnp | grep ':443'`<br>4. **SNI 验证**：`openssl s_client -connect proxy.yourdie.com:443 -servername proxy.yourdie.com` 和 `openssl s_client -connect api.proxy.yourdie.com:443 -servername api.proxy.yourdie.com` | 1. 两个站点共存<br>2. SSL SNI 正确路由<br>3. 各自返回对应证书 | VPS |
| **IT-005** | 证书申请（首次） | 验证 Certbot 首次申请 | 1. 清理旧证书<br>2. 调用 `nginx_certbot issue "proxy.yourdie.com"`<br>3. 检查 `/etc/letsencrypt/live/` | 1. 证书成功生成<br>2. 软链接创建成功 | VPS |
| **IT-006** | 证书续期（自动） | 验证定时任务与续期 | 1. 检查 `crontab -l` 包含续期任务<br>2. 手动触发 `certbot renew --dry-run` | 1. 任务存在<br>2. 续期流程无错误 | VPS |
| **IT-007** | 证书续期（强制） | 验证 30 天内续期逻辑 | 1. 修改本地系统时间模拟过期<br>2. 调用 `nginx_certbot issue`<br>3. 检查 `--force-renewal` 或续期分支 | 1. 触发续期流程<br>2. 新证书生成 | VPS（模拟） |
| **IT-008** | 配置冲突（生产环境） | 验证 `.bak` 备份命名 | 1. 创建旧配置文件<br>2. 覆盖新配置<br>3. 检查 `.bak.YYYYMMDDHHmmss` 命名 | 1. 备份文件存在<br>2. 时间戳格式正确 | VPS |
| **IT-009** | Nginx 重启/重载 | 验证服务稳定性 | 1. 修改配置<br>2. 调用 `nginx_reload`<br>3. 持续访问 5 分钟，检查 5xx 错误 | 1. 无连接中断<br>2. 配置生效 | VPS |
| **IT-010** | 故障恢复 | 验证配置失败回滚 | 1. 输入错误配置（端口 0）<br>2. 检查 `nginx -t` 是否拦截 | 1. `nginx -t` 返回错误<br>2. 未重载服务 | VPS |
| **IT-011** | 证书过期告警测试 | 验证告警机制 | 1. 设置告警阈值（7天）<br>2. 检查 `/etc/letsencrypt/live/` 过期时间<br>3. 验证告警日志 | 1. 提前告警<br>2. 记录告警日志<br>3. 触发续期通知 | VPS |

---

### 2.3 边界条件测试用例

| ID | 测试名称 | 测试目的 | 测试步骤 | 预期结果 | 环境 |
|----|----------|----------|----------|----------|------|
| **BT-001** | 空域名输入 | 验证域名合法性校验 | 1. 调用 `nginx_config new "" "" ""` | 1. 返回非 0<br>2. 错误提示"域名无效" | 本地 |
| **BT-002** | 超长域名（>253字符） | 验证域名长度限制 | 1. 生成 256 字符域名<br>2. 调用配置函数 | 1. 返回非 0<br>2. 拒绝处理 | 本地 |
| **BT-003** | 特殊字符域名 | 验证转义处理 | 1. 域名包含 `@#$%^&*()`<br>2. 调用配置函数 | 1. 拒绝或转义<br>2. Nginx 配置有效 | 本地 |
| **BT-004** | 端口 0 / 负数端口 | 验证端口范围检查 | 1. 端口 `-1` / `0`<br>2. 调用配置函数 | 1. 返回非 0<br>2. 错误"端口无效" | 本地 |
| **BT-005** | 端口 65535+1 | 验证端口上限 | 1. 端口 `65536`<br>2. 调用配置函数 | 1. 返回非 0<br>2. 错误"端口超出范围" | 本地 |
| **BT-006** | URL 路径 `/` | 验证根路径处理 | 1. `URL_PATH="/"`, `PORT="80"` | 1. 配置生成无错误<br>2. Nginx `location /` 正确 | 本地 |
| **BT-007** | URL 路径空字符串 | 验证空路径处理 | 1. `URL_PATH=""`<br>2. 调用配置函数 | 1. 默认 `/`<br>2. 或拒绝处理 | 本地 |
| **BT-008** | Certbot 执行超时 | 验证 `--connect-timeout` | 1. Mock 网络延迟 >30s<br>2. 调用 `certbot` | 1. 返回错误代码<br>2. 输出提示"网络超时" | 本地 |
| **BT-009** | 同一域名多协议 | 验证配置合并逻辑 | 1. 配置 `VMess-ws` + `VLESS-grpc`<br>2. 检查配置文件生成规则<br>3. 验证 `.add` 文件包含所有配置 | 1. `VMess-${HOST}.conf` 和 `VLESS-grpc-${HOST}.conf` 分别生成<br>2. `.add` 文件包含两个配置引用（`include conf.d/VMess-*.conf; include conf.d/VLESS-grpc-*.conf;`）<br>3. 配置不冲突 | 本地 |
| **BT-010** | `.add` 文件缺失 | 验证 Nginx 启动保护 | 1. 删除 `.add` 文件<br>2. 调用 `nginx_reload` | 1. 自动创建空 `.add`<br>2. 不报错 | 本地 |
| **BT-011** | DNS 解析失败 | 验证域名解析失败处理 | 1. Mock DNS 解析失败<br>2. 调用 `nginx_config new`<br>3. 检查返回值和日志 | 1. 返回非 0<br>2. 错误"DNS 解析失败"<br>3. 不创建配置 | 本地 |

---

### 2.4 端到端测试用例

| ID | 测试名称 | 测试目的 | 测试步骤 | 预期结果 | 环境 |
|----|----------|----------|----------|----------|------|
| **E2E-001** | 完整部署流程 | 验证从零到运行 | 1. 新 VPS<br>2. 执行 `install_nginx_certbot`<br>3. 执行 `nginx_config ws`<br>4. 执行 `nginx_certbot issue`<br>5. 执行 `nginx_reload`<br>6. `curl https://proxy.yourdie.com/v2ray` | 1. 无错误<br>2. 站点 HTTPS 可访问<br>3. HTTP→HTTPS 跳转 | VPS |
| **E2E-002** | 升级保留配置 | 验证新旧版本升级 | 1. 部署旧版本配置<br>2. 更新脚本<br>3. 执行新配置 | 1. `.bak` 备份保留<br>2. 新配置生效 | VPS |
| **E2E-003** | 故障注入恢复 | 验证容错能力 | 1. 中断网络（iptables drop）<br>2. 调用 `nginx_certbot issue`<br>3. 恢复网络<br>4. 重试 | 1. 失败提示<br>2. 不破坏现有配置 | VPS |
| **E2E-004** | 证书过期自动续期 | 验证 crontab 行为 | 1. 检查 `crontab -l`<br>2. 强制 `certbot renew --dry-run` | 1. 3 AM 任务存在<br>2. `systemctl reload nginx` 触发 | VPS |
| **E2E-005** | 配置回滚功能测试 | 验证配置回滚机制 | 1. 创建配置 A<br>2. 创建配置 B<br>3. 触发回滚到 A<br>4. 检查配置文件 | 1. 配置文件恢复为 A<br>2. `.bak` 备份保留<br>3. 服务正常 | VPS |

---

## 3. 测试环境要求

### 3.1 本地环境要求

| 依赖 | 版本 | 说明 |
|------|------|------|
| **BATS** | ≥1.6.0 | Bash 测试框架 |
| **bash** | ≥4.4 | 支持 `[[ ]]`、`local` |
| **Mock 工具** | `bats-mock` 或自定义 | Mock `certbot`, `nginx`, `systemctl` |
| **OpenSSL** | ≥1.1.1 | 证书验证 |

**最小命令环境**：
```bash
apt-get install -y bats coreutils openssl
```

### 3.2 VPS 环境要求

| 项目 | 要求 | 示例 |
|------|------|------|
| **操作系统** | Ubuntu 20.04+ / Debian 11+ / CentOS 8+ | Ubuntu 22.04 |
| **Nginx** | ≥1.18.0 | 1.24.0（已确认） |
| **Certbot** | ≥1.21.0 | 2.7.0+（_建议升级_） |
| **端口** | 80, 443 开放 | `ufw allow 'Nginx Full'` |
| **DNS** | 域名解析到 VPS | `proxy.yourdie.com → 72.11.140.248` |
| **内存** | ≥512MB | Certbot 运行需要 |
| **磁盘** | ≥1GB 剩余 | 证书存储 |

**推荐的Certbot版本检查**：
```bash
certbot --version  # 需 ≥ 1.21.0 (ACME v2)
```

**防火墙开放命令**：
```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw reload
```

---

## 4. 风险评估

| ID | 风险描述 | 发生概率 | 影响范围 | 风险等级 | 缓解措施 |
|----|----------|----------|----------|----------|----------|
| **R-001** | `certbot renew` 失败 | 低（自动化） | TLS 过期 → 服务整体不可用 | **高** | 1. 添加监控告警<br>2. 本地备份证书<br>3. 手动续期预案<br>4. 故障转移至备用域名 |
| **R-002** | 配置覆盖导致数据丢失 | 中（用户误操作） | 单一协议失效 | **中** | 1. `.bak` 自动命名<br>2. 覆盖前双确认（UI/日志）<br>3. 版本控制 |
| **R-003** | `nginx -t` 未捕获语法错误 | 低（已实现） | Nginx 工作进程崩溃 | **高** | 1. 使用 `nginx -t` 前缀<br>2. 配置验证在 `nginx_config` 函数中强制调用<br>3. 部署前执行完整检查<br>4. 保存回滚点 |
| **R-004** | 网络波动导致证书申请失败 | 中（公网环境） | TLS 无法建立 | **高** | 1. 自动重试 3 次<br>2. 指数退避（1s, 2s, 4s）<br>3. 降级方案（临时证书） |
| **R-005** | 多协议端口冲突 | 中（用户输入错误） | 部分协议无法连接 | **中** | 1. 随机端口 fallback<br>2. 端口范围检查<br>3. 端口占用检测 |
| **R-006** | `.add` 文件语法错误 | 低（用户手动编辑） | Nginx 启动失败 | **中** | 1. `.add` 为空模板（自动生成）<br>2. `.add` 语法自动检查<br>3. 手动编辑警告 |
| **R-007** | Certbot 版本过低 | 中（旧系统） | 无法申请证书 | **高** | 1. 版本检查（≥1.21.0）<br>2. 升级脚本自动执行<br>3. 兼容模式（旧版支持）<br>4. 版本验证测试（UT-006） |
| **R-008** | 80 端口被占用 | 低（服务冲突） | 证书申请失败 | **中** | 1. `ss -tlnp :80` 检查<br>2. 自动停止占用服务<br>3. 端口切换提示 |
| **R-009** | SSL/TLS 配置过时 | 低（算法更新） | 安全扫描降级 | **低** | 1. 每季度更新加密套件<br>2. 自动化扫描（SSL Labs） |
| **R-010** | DNS 解析失败 | 中（网络问题） | 证书申请和配置失败 | **中** | 1. DNS 解析验证<br>2. 多 DNS 服务器 fallback<br>3. 重试机制（指数退避）<br>4. BT-011 测试用例 |
| **R-011** | 证书过期未告警 | 低（告警失效） | 用户未及时续期 | **高** | 1. 提前 7 天告警<br>2. 多通道通知（邮件/短信）<br>3. IT-011 测试用例验证<br>4. 自动续期备用方案 |

---

## 5. 实施建议

### 5.1 实施步骤

| 阶段 | 步骤 | 时间 | 负责人 |
|------|------|------|--------|
| **Phase-0** | **E2E 冒烟测试**（核心路径验证） | 1 小时 | QA |
| **Phase-1** | 单元测试补充 | 2 小时 | QA Agent |
| **Phase-2** | 本地集成模拟 | 1 小时 | Dev/QA |
| **Phase-3** | VPS 测试环境准备 | 0.5 小时 | DevOps |
| **Phase-4** | E2E 测试执行 | 2 小时 | QA/QA |
| **Phase-5** | 风险问题修复 | 视情况 | Dev |
| **Phase-6** | 文档更新 | 1 小时 | QA |

**Phase-0 说明**：E2E 冒烟测试优先执行，确保核心部署流程可用，避免在基础功能故障时浪费时间在详细测试上。

### 5.2 验收标准

| 标准 | 指标 | 检查方式 |
|------|------|----------|
| **通过率** | ≥95% 单元测试 | `bats tests/` |
| **覆盖率** | `nginx.sh` ≥80% 函数 | `bashcov` |
| **E2E 冒烟** | 首次部署成功率 100% | VPS 真实部署 |
| **E2E 全量** | 全部 E2E 场景通过（包括回滚） | VPS 真实部署 |
| **证书申请** | 成功率 ≥99%（自动重试后） | 成功/重试 |
| **无回归** | 旧功能不被破坏 | 每次 PR 回归测试 |

### 5.3 自动化建议

**GitHub Actions 工作流**：
```yaml
name: Nginx Tests
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: |
          apt-get install -y bats
          bats tests/unit/nginx_test.bats
```

**VPS 集成测试计划**（cron 每周日）：
```bash
0 3 * * 0 /home/user/qa/run_e2e_tests.sh >> /var/log/qa_nginx.log 2>&1
```

---

## 附件

### A. 测试脚本示例（补充单元测试）

```bash
# BATS: nginx/wss/boundary.bats
@test "nginx_config ws - 空路径处理" {
    source "$IS_SH_DIR/src/nginx.sh"
    export HOST="test.com"
    
    run nginx_config ws "VLESS" "" "8080"
    [[ "$status" -eq 0 ]]
    
    # 验证默认路径
    grep -q 'location /' "$IS_NGINX_CONF/VLESS-${HOST}.conf"
}
```

### B. 常见问题 (FAQ)

| 问题 | 解决方案 |
|------|----------|
| 如何模拟证书过期？ | 修改系统时间 `date -s "2026-01-01"` |
| 如何查看 Certbot 日志？ | `tail -f /var/log/letsencrypt/letsencrypt.log` |
| Nginx 配置错误排查？ | `nginx -t` → `journalctl -u nginx -f` |
| 如何验证 TLS 1.3？ | `openssl s_client -connect proxy.yourdie.com:443 -tls1_3` |
| 如何测试指数退避重试？ | 模拟网络故障，检查重试间隔（1s, 2s, 4s） |

### C. `.add` 文件模板

**模板内容**（自动生成，无需手动编辑）：
```nginx
# Auto-generated file: /etc/nginx/conf.d/.add
# This file includes all protocol-specific configurations
# Generated by: nginx_config <protocol> <app> <path> <port>

# Include all protocol configurations
include conf.d/VMess-*.conf;
include conf.d/VLESS-*.conf;
include conf.d/SSR-*.conf;
include conf.d/Trojan-*.conf;

# Custom user configuration (optional)
# include conf.d/*.custom.conf;
```

**生成逻辑**：
```bash
# 自动生成 .add 文件
generate_add_file() {
    local add_file="/etc/nginx/conf.d/.add"
    
    # 创建空文件
    echo "# Auto-generated file: ${add_file}" > "$add_file"
    echo "# This file includes all protocol-specific configurations" >> "$add_file"
    echo "# Generated by: nginx_config <protocol> <app> <path> <port>" >> "$add_file"
    echo "" >> "$add_file"
    echo "# Include all protocol configurations" >> "$add_file"
    
    # 动态生成 include 语句
    for conf in /etc/nginx/conf.d/*.conf; do
        if [[ -f "$conf" && ! "$(basename $conf)" == ".add" ]]; then
            echo "include ${conf};" >> "$add_file"
        fi
    done
    
    echo "" >> "$add_file"
    echo "# Custom user configuration (optional)" >> "$add_file"
    echo "# include conf.d/*.custom.conf;" >> "$add_file"
}
```

### D. 网络重试逻辑（指数退避）

**实现代码**：
```bash
# 模拟 certbot 命令（带重试逻辑）
certbot_with_retry() {
    local max_retries=3
    local base_delay=1  # 第一次重试等待 1 秒
    local retry_count=0
    
    while [ $retry_count -lt $max_retries ]; do
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] 尝试执行 certbot 命令 (尝试 $((retry_count + 1))/$max_retries)..."
        
        if certbot "$@" 2>&1; then
            echo "✅ certbot 命令执行成功"
            return 0
        fi
        
        retry_count=$((retry_count + 1))
        
        if [ $retry_count -lt $max_retries ]; then
            local delay=$((base_delay * (2 ** retry_count)))  # 指数退避: 2, 4, 8...
            echo "⚠️ certbot 命令失败，等待 ${delay} 秒后重试..."
            sleep $delay
        fi
    done
    
    echo "❌ certbot 命令执行失败（已重试 $max_retries 次）"
    return 1
}

# DNS 解析验证
verify_dns_resolution() {
    local domain="$1"
    
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] 验证 DNS 解析: $domain"
    
    if host "$domain" > /dev/null 2>&1; then
        echo "✅ DNS 解析成功"
        return 0
    else
        echo "❌ DNS 解析失败"
        return 1
    fi
}
```

**测试用例验证**：
- **UT-005**: 模拟证书申请失败，验证重试逻辑和指数退避
- **BT-011**: 模拟 DNS 解析失败，验证错误处理

### E. Certbot 版本验证测试

**版本检查脚本**：
```bash
# 检查 Certbot 版本（≥1.21.0）
check_certbot_version() {
    local required_version="1.21.0"
    local current_version
    
    if ! command -v certbot &> /dev/null; then
        echo "❌ Certbot 未安装"
        return 1
    fi
    
    # 获取当前版本
    current_version=$(certbot --version 2>&1 | grep -oP '\d+\.\d+\.\d+' | head -1)
    
    if [[ -z "$current_version" ]]; then
        echo "⚠️ 无法解析 Certbot 版本，尝试继续..."
        return 0
    fi
    
    echo "Certbot 当前版本: $current_version (要求 ≥ $required_version)"
    
    # 版本比较（简单语义版本比较）
    if [[ $(compare_versions "$current_version" "$required_version") -lt 0 ]]; then
        echo "❌ Certbot 版本过低，请升级"
        return 1
    fi
    
    echo "✅ Certbot 版本符合要求"
    return 0
}

# 语义版本比较函数
compare_versions() {
    local v1="$1"
    local v2="$2"
    
    if [[ "$v1" == "$v2" ]]; then
        echo 0
        return
    fi
    
    local major1 minor1 patch1 major2 minor2 patch2
    
    IFS='.' read -r major1 minor1 patch1 <<< "$v1"
    IFS='.' read -r major2 minor2 patch2 <<< "$v2"
    
    # 比较主版本
    if [[ $major1 -gt $major2 ]]; then
        echo 1
        return
    elif [[ $major1 -lt $major2 ]]; then
        echo -1
        return
    fi
    
    # 比较次版本
    if [[ $minor1 -gt $minor2 ]]; then
        echo 1
        return
    elif [[ $minor1 -lt $minor2 ]]; then
        echo -1
        return
    fi
    
    # 比较补丁版本
    if [[ $patch1 -gt $patch2 ]]; then
        echo 1
        return
    elif [[ $patch1 -lt $patch2 ]]; then
        echo -1
        return
    fi
    
    echo 0
}
```

**测试用例**：
- **UT-006**: 在成功场景中验证 Certbot 版本检查通过
- **UT-015**（新增）: Certbot 版本过低测试（Mock 旧版本，验证失败）

---

**下一步行动**：  
✅ **确认测试环境** → 分配 VPS 测试窗口时间  
✅ **补充单元测试** → 覆盖边界条件（BT-001 ~ BT-011）和 HTTPS 跳转（UT-013）  
✅ **创建自动化脚本** → `qa/run_all_tests.sh`  
✅ **执行 E2E 冒烟测试** → Phase-0 快速验证核心流程  
✅ **测试配置回滚功能** → E2E-005 验证配置历史管理

**联系 QA**：此方案由 Agent (QA Engineer) 制定，如需调整或补充，请直接提出。
