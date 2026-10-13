下面是一份**可直接用于发布前 / 回归测试的完整测试清单（Checklist）**，我按照**真实运维与开源项目发布标准**来设计，而不是“功能点罗列”。
你可以把这份清单直接放进 README 的 *Testing / Validation* 章节，或者作为你自己发布前的 SOP。

------

# Terraria Docker Server – 测试清单（Release Checklist）

## 一、镜像构建与基础环境

### \☐Docker 镜像可正常构建

- `docker compose build` 无错误
- 不依赖本地缓存（可 `--no-cache` 构建）
- 构建时间在可接受范围内

### ☐ 基础镜像最小化

- 镜像体积符合预期
- 无多余 build 依赖残留（apt cache 已清理）

------

## 二、启动与首次运行

### \☐ 容器可正常启动

- `docker compose up -d` 成功
- 容器状态为 `Up`

### \☐ Terraria Server 成功启动

- 日志中无 error / crash
- 控制台显示监听端口

```bash
docker compose logs -f terraria
```

------

## 三、配置与参数注入（.env → server.conf）

### \☐ server.conf 自动生成

- `/config/server.conf` 在首次启动时创建
- 重启容器不会覆盖已有配置

### ☐ 环境变量生效

逐项验证：

- `WORLD_NAME`
- `WORLD_SIZE`
- `DIFFICULTY`
- `SEED`
- `SERVER_PORT`
- `MAX_PLAYERS`
- `LANGUAGE`
- `AUTOSAVE`

------

## 四、世界数据持久化（关键）

### ☐ 世界文件正确生成

- `/worlds/*.wld` 存在
- 文件大小随游戏进程增长

### ☐ 世界修改不会回滚

- 砍树 / 挖地 / 建筑
- 重启容器后仍然存在

```bash
docker compose restart terraria
```

------

## 五、自动保存（Autosave）

### ☐ autosave 配置生效

- `server.conf` 中存在 `autosave=1`
- 长时间运行无数据丢失

### ☐ 非正常退出恢复

- `docker stop terraria`
- 再次启动世界未回滚

------

## 六、优雅停服（Signal Handling）

### ☐ SIGTERM 触发保存

```bash
docker stop terraria
```

- 日志出现 save 行为
- 世界未损坏

------

## 七、自动下载 Server（版本管理）

### ☐ 首次启动自动下载

- 指定 `TERRARIA_VERSION`
- `/opt/terraria/TerrariaServer.bin.x86_64` 正确生成

### ☐ 重启不重复下载

- 二次启动跳过下载逻辑

------

## 八、备份功能（backup.sh）

### ☐ 备份脚本存在

```bash
which backup.sh
```

### ☐ 手动备份可执行

```bash
docker compose exec terraria backup.sh
```

### ☐ 备份文件生成

- `/backups/*.tar.gz` 存在
- 文件大小合理

------

## 九、恢复功能（restore.sh）

### ☐ 恢复脚本存在

```bash
which restore.sh
```

### ☐ 运行中禁止恢复

```bash
docker compose exec terraria restore.sh xxx.tar.gz
```

- 正确拒绝执行

### ☐ 正常恢复流程

1. `docker compose down`
2. `docker compose run --rm terraria bash`
3. `restore.sh <backup>`
4. `docker compose up -d`

### ☐ 恢复结果验证

- 世界状态回滚到备份时间点
- 自动生成 pre-restore 备份

------

## 十、异常与边界场景

### ☐ 错误版本号

- 无效 `TERRARIA_VERSION` 报错清晰

### ☐ 网络异常

- 下载失败日志明确
- 不导致容器反复 crash

### ☐ 空 worlds / backups 目录

- 首次运行无异常
- 不误删宿主数据

------

## 十一、可维护性与开源合规

### ☐ README 完整

- 快速开始
- 配置说明
- 备份 / 恢复
- 常见问题

### ☐ 可复现

- 新机器 clone → up 成功
- 不依赖本地隐式文件

------

## 十二、发布前最终确认

### ☐ 标签版本号（tag）

- Git tag 已创建

### ☐ 示例 .env 可用

- `.env.example` 与 README 一致

### ☐ 最小化配置启动成功

- 仅 world + port 即可运行

------

## 你已经走到哪一步了？

从你目前的进度来看，你已经 **通过了 80% 以上的关键测试项**。
下一步最自然的事情之一是：

- 写一个 **`.env.example`**
- 或做 **自动备份 cron + 保留策略**
- 或直接 **发 v0.1.0 Release**

如果你愿意，我可以帮你把这份测试清单直接整理成 **README 的 Testing 章节** 或 **GitHub Release Checklist**。