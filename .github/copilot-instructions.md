# Copilot Agent 运行环境

## 环境配置
- **可用工具**: `jq`, `xvfb`, `tree`, `ripgrep`, `socat`, `sshpass`, `qemu-user-static`
- **Playwright**: 已预装，优先用于 E2E 测试和动态页面验证。
- **OpenWrt**:
  - 镜像: `openwrt/rootfs:x86-64-24.10.4` (含完整 opkg)
  - 源码: `openwrt/luci` (与仓库代码同级)
  - 编译链: `build-essential`, `gettext`, `libssl-dev`, `zlib1g-dev`

## ⚠️ 关键流程：LuCI 服务启动
**必须严格遵守启动顺序：ubusd -> procd -> rpcd -> uhttpd**
(见下文“一键启动脚本”，否则会导致 `Failed to connect to ubus` 或 `left-hand side expression is null`)
这是压缩后的版本。我移除了冗余的解释性文字（如“为什么要按顺序启动”的原理说明，改为直接的指令），合并了相似的命令，并将多行脚本压缩为单行或紧凑块。

核心逻辑（启动顺序、路径映射、关键命令、排查表、测试选择器）完全保留。

**一键启动脚本 (安装依赖+配置+启动):**
```bash
docker run -d --name openwrt-luci -p 8080:80 openwrt/rootfs:x86-64-24.10.4 /bin/ash -c '
mkdir -p /var/lock /var/run && opkg update && opkg install luci luci-base luci-compat ucode-mod-log
/sbin/ubusd & sleep 1 && /sbin/procd & sleep 2 && /sbin/rpcd & sleep 1
/usr/sbin/uhttpd -f -h /www -r OpenWrt -x /cgi-bin -u /ubus -t 60 -T 30 -A 1 -n 3 -N 100 -R -p 0.0.0.0:80 &
echo -e "password\npassword" | passwd root
uci set luci.themes.Bootstrap=/luci-static/bootstrap && uci commit luci
tail -f /dev/null'
# 访问: http://localhost:8080 (root/password)
```

## 3. 开发工作流

### 文件部署映射
| 类型 | 本地源码路径 (`luci-app-tailscale-community/...`) | 容器路径 (`openwrt-luci:...`) |
|---|---|---|
| **JS** | `htdocs/luci-static/resources/view/tailscale.js` | `/www/luci-static/resources/view/tailscale.js` |
| **Menu** | `root/usr/share/luci/menu.d/*.json` | `/usr/share/luci/menu.d/` |
| **ACL** | `root/usr/share/rpcd/acl.d/*.json` | `/usr/share/rpcd/acl.d/` |
| **Ucode**| `root/usr/share/rpcd/ucode/*.uc` | `/usr/share/rpcd/ucode/` |

### 部署命令 (示例)
```bash
# 1. 复制文件
docker cp [LocalPath] openwrt-luci:[ContainerPath]
# 2. 只有首次需创建 UCI 配置
docker exec openwrt-luci sh -c '[ ! -f /etc/config/tailscale ] && printf "config settings settings\n\toption enabled 0\n\toption port 41641\n\toption fw_mode nftables\n" > /etc/config/tailscale'
# 3. 重载 rpcd (更新 ACL/Ucode 后必须)
docker exec openwrt-luci sh -c "kill -9 \$(pgrep rpcd) && /sbin/rpcd &"
# 4. 验证 RPC
docker exec openwrt-luci ubus call tailscale get_status
```

### 代码检查与测试
- **Lint/Fmt**: `npx tsc -b`, `npm run lint`, `npx prettier --write .`, `npx knip`
- **Test**: `npx vitest`
- **i18n**: `cd luci && ./build/i18n-sync.sh applications/luci-app-tailscale-community`
- **Ucode调试**: `docker exec openwrt-luci ucode /usr/share/rpcd/ucode/tailscale.uc`

### Playwright 登录片段
```js
await page.goto('http://localhost:8080/cgi-bin/luci/');
await page.getByRole('textbox', { name: 'Password' }).fill('password');
await page.getByRole('button', { name: 'Log in' }).click();
```

## 4. 故障排查

### 常见问题

| 现象 | 原因 | 修复 |
|---|---|---|
| `Failed to connect to ubus` | ubusd 挂了 | `/sbin/ubusd &` |
| 500 / `left-hand side expression is null` | procd 挂了 | `/sbin/procd &` |
| 插件 RPC 错误 | rpcd 未重载或 ACL 错 | 重启 rpcd / 查 ACL |
| 主题渲染错 | 缺少主题配置 | `uci set luci.themes.Bootstrap=/luci-static/bootstrap` |

### 2FA 插件排查

#### 问题：LuCI dispatcher 报错 `Unable to resolve path for module 'log'`
**原因**: OpenWrt 容器可能缺少 `ucode-mod-log` 包
**修复方案**:
1. 尝试安装包：`opkg install ucode-mod-log`
2. 如果网络不可用，改进代码使 log 模块可选：
   ```ucode
   let log_mod = null;
   try { log_mod = require('log'); } catch {}
   let syslog = log_mod?.syslog ?? function() {};
   ```

#### 问题：容器多个 uhttpd 进程导致 `bind(): Address in use`
**原因**: 旧进程未完全退出，端口仍被占用
**修复方案**:
```bash
# 逐个杀死所有 uhttpd 进程
for pid in $(docker exec openwrt-luci pgrep uhttpd); do
  docker exec openwrt-luci kill -9 $pid
done
# 重启服务
docker exec openwrt-luci /sbin/rpcd &
docker exec openwrt-luci /usr/sbin/uhttpd -f -h /www -r OpenWrt -x /cgi-bin -u /ubus -t 60 -T 30 -A 1 -n 3 -N 100 -R -p 0.0.0.0:80 &
```

#### 问题：修改文件后 LuCI 仍显示旧内容
**原因**: 缓存文件未清除，或 rpcd 未重新加载
**修复方案**:
```bash
# 清除所有缓存
docker exec openwrt-luci rm -f /tmp/luci-indexcache* /tmp/luci-modulecache/*
# 重启 rpcd
RPCD_PID=$(docker exec openwrt-luci pgrep rpcd)
docker exec openwrt-luci kill -9 $RPCD_PID
docker exec openwrt-luci /sbin/rpcd &
```

#### 问题：登录页面不显示 2FA OTP 字段
**原因**: 
1. sysauth 模板未更新为支持 `auth_fields`
2. dispatcher.uc 的 `get_auth_challenge()` 未返回所需字段
**排查步骤**:
```bash
# 验证模板是否支持 auth_fields
grep -c 'auth_fields' /usr/share/ucode/luci/template/themes/bootstrap/sysauth.ut

# 验证 dispatcher 是否加载插件
grep -c 'load_auth_plugins' /usr/share/ucode/luci/dispatcher.uc

# 验证 UCI 配置是否正确
cat /etc/config/luci_plugins

# 测试插件是否正常加载
docker exec openwrt-luci wget -q -O - http://127.0.0.1/cgi-bin/luci/ | grep -E 'luci_otp|test_otp'
```

### Plugin 架构迁移注意事项

#### 配置映射
- **旧架构** (luci-app-2fa): `2fa` UCI config with 节点 `settings` 和 per-user 节点
- **新架构** (master plugin): `luci_plugins` UCI config with plugin UUID 节点，per-user keys: `key_<user>`, `type_<user>`, `step_<user>`, `counter_<user>`

#### 文件路径映射
| 层级 | 旧路径 | 新路径 |
|---|---|---|
| **后端插件** | `/usr/share/luci/auth.d/*.uc` | `/usr/share/ucode/luci/plugins/auth/login/<uuid>.uc` |
| **UI 插件** | `/system/2fa` (独立页面) | `/view/plugins/<uuid>.js` (System > Plugins 集成) |
| **OTP 生成器** | 仅支持 2fa config | 支持 `--plugin=<uuid>` 参数 |

#### OTP 生成器的双配置支持
```bash
# 旧 2fa 配置方式
/usr/libexec/generate_otp.uc username

# 新插件配置方式 (带 UUID)
/usr/libexec/generate_otp.uc username --plugin=bb4ea47fcffb44ec9bb3d3673c9b4ed2 --no-increment
```

### 测试工作流

1. **单元测试（OTP 生成器）**:
   ```bash
   docker exec openwrt-luci /usr/libexec/generate_otp.uc root --no-increment --plugin=bb4ea47fcffb44ec9bb3d3673c9b4ed2
   ```

2. **集成测试（登录流程）**:
   ```bash
   # 获取当前 OTP
   OTP=$(docker exec openwrt-luci /usr/libexec/generate_otp.uc root --no-increment --plugin=bb4ea47fcffb44ec9bb3d3673c9b4ed2)
   
   # 测试登录
   curl -X POST http://localhost:8080/cgi-bin/luci/ \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "luci_username=root&luci_password=password&luci_otp=$OTP" \
     -i
   # 期望: HTTP 302 Found + Set-Cookie 头
   ```

3. **错误场景**:
   ```bash
   # 错误的 OTP - 应返回 403
   curl -X POST http://localhost:8080/cgi-bin/luci/ \
     -d "luci_username=root&luci_password=password&luci_otp=000000" \
     -H "Content-Type: application/x-www-form-urlencoded" | grep -i "alert"
   ```

**注**: 参考 `openwrt/luci` 源码保持一致性；LLM 写 ucode 易有幻觉，请参考官方文档。
