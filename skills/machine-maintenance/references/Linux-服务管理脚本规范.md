# Linux 服务管理脚本规范
> 适用于 Alibaba Cloud Linux 3 / RHEL 8 / CentOS 8，Bash 4.4+

## 定位
对 `systemctl` 和服务专属 CLI 的便捷封装，统一交互风格，降低运维记忆负担。脚本存放于 `~/bin/` 并加入 `PATH`。

## 目录结构
```
~/bin/
├── README.md           # 项目说明与版本记录
├── openresty.sh        # Web/反向代理服务
├── postgresql.sh       # 关系型数据库
├── redis.sh            # 缓存/消息队列
└── <service>.sh        # 按需扩展
```

## 必须支持的子命令
所有脚本应至少实现以下子命令，保持用户体验一致：

| 子命令 | 说明 |
|:---|:---|
| `status` | 查看服务运行状态（systemctl status） |
| `start` | 启动服务 |
| `stop` | 停止服务 |
| `restart` | 重启服务 |
| `reload` | 重载配置（不中断服务） |
| `config` | 显示关键路径（二进制、配置、日志、数据目录） |
| `help` | 显示帮助信息 |

### 可选子命令
| 类型 | 子命令 | 适用场景 |
|:---|:---|:---|
| Web 服务 | `logs [error\|access] [--lines N]` | 实时跟踪或查看最近日志 |
| 数据库 | `connect [-c "SQL"]` | 快速进入客户端交互终端 |
| 缓存 | `cli`, `info`, `monitor`, `ping` | Redis 专属操作 |
| 通用 | `flush`, `clear` | 清空缓存/临时数据 |

## 脚本模板
```bash
#!/bin/bash
# <服务名> 管理脚本（基于 systemctl）
set -euo pipefail

# ========== 服务信息 ==========
SERVICE_NAME="<systemd-service-name>"
BIN_PATH="<可执行文件路径>"
CONF_PATH="<主配置文件路径>"
LOG_DIR="<日志目录>"
DATA_DIR="<数据目录>"

# ========== 帮助信息 ==========
usage() {
    cat <<USAGE_EOF
用法: $0 <subcommand> [options]
子命令: status | start | stop | restart | reload | config | help
...
USAGE_EOF
}

# ========== 前置检查 ==========
check_service_exists() {
    if ! systemctl cat "${SERVICE_NAME}" >/dev/null 2>&1; then
        echo "错误: ${SERVICE_NAME}.service 不存在" >&2
        exit 1
    fi
}

# ========== 核心函数 ==========
do_status()  { sudo systemctl status "${SERVICE_NAME}"; }
do_start()   { check_service_exists; sudo systemctl start "${SERVICE_NAME}"; }
do_stop()    { check_service_exists; sudo systemctl stop "${SERVICE_NAME}"; }
do_restart() { check_service_exists; sudo systemctl restart "${SERVICE_NAME}"; }
do_reload()  { check_service_exists; sudo systemctl reload "${SERVICE_NAME}"; }
do_config()  {
    echo "二进制: ${BIN_PATH}"
    echo "配置:   ${CONF_PATH}"
    echo "日志:   ${LOG_DIR}"
    echo "数据:   ${DATA_DIR}"
}

# ========== 命令分发 ==========
case "${1:-}" in
    status)  do_status  ;;
    start)   do_start   ;;
    stop)    do_stop    ;;
    restart) do_restart ;;
    reload)  do_reload  ;;
    config)  do_config  ;;
    help|-h|--help) usage ;;
    *) usage; exit 1 ;;
esac
```

## 关键约定
1. **set -euo pipefail**：所有脚本必须启用严格模式。
2. **sudo 显式调用**：systemctl 命令前加 `sudo`，不假设用户以 root 运行。
3. **服务存在性检查**：start/stop/restart/reload 前用 `systemctl cat` 验证服务文件存在，避免模糊报错。
4. **路径变量置顶**：所有关键路径定义为顶部常量，便于版本切换时修改。
5. **颜色输出（可选）**：Redis 等交互式脚本可使用 ANSI 颜色；纯运维脚本保持朴素输出。
6. **环境变量覆盖**：敏感信息（密码、端口）支持环境变量注入，默认值仅用于开发环境。
7. **README 同步更新**：新增/修改脚本后，同步更新 `~/bin/README.md` 的脚本列表、详细说明和版本记录。

## 与 infra-care 的关系
- 本规范是 **公共参考知识**，不包含具体机器的实际路径或凭据。
- 实际部署记录写入 `local/维护档案.md`，路径和验证结果以本机事实为准。
- 脚本本身存放在 `~/bin/`，不纳入 infra-care 仓库；仓库仅保存规范和脱敏示例。

## 验证清单
新增或修改脚本后，依次确认：
- [ ] `<script>.sh help` 输出完整且路径正确
- [ ] `<script>.sh status` 能正确反映服务状态
- [ ] `<script>.sh config` 显示的路径与实际一致
- [ ] `shellcheck <script>.sh` 无严重警告
- [ ] README.md 已同步更新
