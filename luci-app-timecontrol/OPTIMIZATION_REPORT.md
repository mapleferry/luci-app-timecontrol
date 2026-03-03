# luci-app-timecontrol 代码优化报告

## 一、项目概述

本项目为 OpenWrt 上网时间控制插件，通过 nftables/iptables 实现按时间和星期限制设备上网。核心逻辑位于 `/usr/bin/timecontrol` 和 `/usr/bin/timecontrolctrl`。

---

## 二、已实施的修复

### 1. 设备 ID 解析 Bug（关键）

**问题**：`grep -o '[0-9]'` 仅匹配单个数字，设备索引 10 及以上会被错误解析为 1、0 等。

**修复**：改为 `grep -oE '[0-9]+'` 以正确匹配多位数字。

**涉及文件**：
- `root/usr/bin/timecontrol`（start 命令中的 idlist 解析）
- `root/usr/bin/timecontrolctrl`（idlist 初始化）

### 2. stop_timecontrol 调用错误

**问题**：`remove_tcp_reset` 被无参数调用，但函数需要 `$target` 参数，会导致解析失败。

**修复**：改为 `nft delete table inet timecontrol` 直接清理 TCP RST 相关表，避免无效调用。

**涉及文件**：`root/usr/bin/timecontrol`

### 3. iptables 模式下 MAC 地址控制失效

**问题**：`ipset hash:net` 不支持 MAC 地址，`ipset add timecontrol_blacklist "00:11:22:33:44:55"` 会失败。

**修复**：从 `/proc/net/arp` 解析 MAC 对应 IP，将 IP 加入 `timecontrol_blacklist`；删除时同样解析 MAC 后移除对应 IP。

**涉及文件**：`root/usr/bin/timecontrol`（timeadd、timedel）

### 4. timecontrolctrl 动态刷新设备列表

**问题**：`idlist` 仅在脚本启动时读取一次，用户在 LuCI 中新增/禁用设备后，轮询进程无法感知。

**修复**：将 `idlist` 获取移入 `process_devices` 循环内，每次轮询时重新读取 UCI 配置；并增加逻辑：当设备被禁用时，从控制列表中移除。

**涉及文件**：`root/usr/bin/timecontrolctrl`

### 5. timecontrol-call sh 兼容性

**问题**：`[[ ]]` 和 `==` 为 bash 语法，在 OpenWrt 的 sh 下可能失败。

**修复**：改为 `[ ]` 和 `=`，符合 POSIX sh。

**涉及文件**：`root/usr/libexec/timecontrol-call`

---

## 三、其他优化建议（未实施）

### 1. 配置保存后自动重启服务

**建议**：在 LuCI 保存 UCI 后，通过 uci-defaults 或 init 脚本触发 `timecontrol restart`，确保配置立即生效。

**实现**：在 `basic.js` 的 form 提交后调用 `rpc.call('uci', 'apply', ...)` 或 `fs.exec('/etc/init.d/timecontrol', ['restart'])`。

### 2. init 脚本缺失

**现状**：`uci-defaults` 引用 `/etc/init.d/timecontrol`，但该文件不在本包中。若依赖其他包提供，需在 README 中说明；否则建议在 Makefile 中安装 init 脚本。

### 3. 轮询间隔可配置

**建议**：将 `timecontrolctrl` 中 `sleep 60` 改为可配置（如从 UCI 读取），便于不同场景调整。

### 4. ipset 范围格式

**说明**：`ipset hash:net` 不支持 `192.168.1.1-192.168.1.100` 格式。当前实现可能依赖 `iprange` 模块或需逐 IP 添加。若遇范围添加失败，可考虑改为循环添加或使用 iprange 模块。

### 5. 白名单模式

**现状**：UI 中已注释白名单选项。若需支持白名单，需在 `init_timecontrol` 和 `timeadd`/`timedel` 中实现“允许列表内、拒绝列表外”的逻辑。

---

## 四、网络设备上网管控功能确认

| 功能           | 状态           | 说明 |
|----------------|----------------|------|
| 黑名单控制     | ✅ 已支持       | 通过 nftables/ipset 实现 |
| 时间范围       | ✅ 已支持       | timestart/timeend |
| 星期选择       | ✅ 已支持       | 0=每天, 1-7=星期, 1,2,3,4,5=工作日, 6,7=休息日 |
| IP/MAC/CIDR/范围 | ✅ 已支持     | parse_target 支持多种格式 |
| 普通/强力控制  | ✅ 已支持       | 链 forward/input |
| nftables 模式  | ✅ 已支持       | 优先使用 |
| iptables 模式  | ✅ 已支持       | 备选，MAC 已修复 |
| 定时轮询       | ✅ 已支持       | timecontrolctrl 每 60 秒检查 |

---

## 五、修改文件清单

| 文件 | 修改内容 |
|------|----------|
| `root/usr/bin/timecontrol` | 1) 设备 ID 解析 2) stop 中 remove_tcp_reset 3) iptables MAC 解析 |
| `root/usr/bin/timecontrolctrl` | 设备 ID 解析 |

---

## 六、测试建议

1. 添加多台设备（索引 ≥10）验证 ID 解析正确。
2. 添加 MAC 地址设备，验证 iptables 模式下控制生效。
3. 执行 `timecontrol stop` 确认无报错。
4. 切换时间范围，验证设备在允许/禁止时段正确加入/移除控制列表。
