# Oracle ARM Instance Sniper（甲骨文 ARM 抢机 · GitHub Actions 云端版）

来源帖子：https://www.nodeloc.com/t/topic/98502

全自动云端抢甲骨文 `VM.Standard.A1.Flex` 实例，无需本地挂机：
- 手动触发：仓库 → Actions → **Run workflow**
- 自动触发：每 15 分钟一次（cron `*/15 * * * *`），单次运行 45 分钟自动安全退出

## 部署步骤

1. 在你的 GitHub 账号下新建一个 **私有仓库（Private）**，把本目录内容（`.github/workflows/sniper.yml`）推上去。
2. 进入仓库 **Settings → Secrets and variables → Actions**，点 `New repository secret`，逐一添加以下 **7 个 Secrets**（变量名必须完全一致）：

| Secret 名称 | 说明 | 获取位置 |
|---|---|---|
| `TENANCY_OCID` | Oracle 租户 ID | 控制台 → 配置文件/租户信息，`ocid1.tenancy.oc1..xxx` |
| `USER_OCID` | Oracle 用户 ID | `ocid1.user.oc1..xxx` |
| `REGION` | 服务器所在区域 | 如 `us-phoenix-1`、`ap-seoul-1` |
| `FINGERPRINT` | API 密钥指纹 | 控制台 → 用户设置 → API 密钥 |
| `PRIVATE_KEY` | API 私钥**完整文本** | 生成 API Key 时下载的 pem 文件内容（含 `-----BEGIN PRIVATE KEY-----` 等行，保留真实换行） |
| `SUBNET_ID` | 目标 VCN 的子网 ID | 网络 → 虚拟云网络 → 子网，`ocid1.subnet.oc1..xxx` |
| `USER_SSH_PUB_KEY` | 你本机的 SSH 公钥 | 如 `~/.ssh/id_ed25519.pub` 的内容；**必须配对，否则抢到也登录不了** |

3. 到 Actions 面板手动 `Run workflow` 试跑一次，确认日志里 `[AD]` / `[IMG]` 正常输出、循环报 `No stock`（说明配置正确，只是在等库存）。

## 注意事项

- **机型**：默认真 2核12G（`ocpus:2 / memoryInGBs:12`，A1 免费额度 4C24G 的一半，以后还能再开一台小配置）。想单实例拿满全量额度，把 `sniper.yml` 里 `SHAPE_CONFIG` 改成 `'{"ocpus":4,"memoryInGBs":24}'`。
- **系统镜像**：默认 Ubuntu 24.04，改 `OS_NAME` / `OS_VERSION` 两个 env 即可。
- **防重复抢（抢到即自毁）**：每轮开头先检查租户里是否已有存活的 A1 实例——有则说明之前抢到了，自动禁用本 workflow 并退出；创建实例成功的瞬间也会自动禁用 workflow（通过 GitHub API），cron 从此不再触发，无需手动干预。若自动禁用失败（权限异常），日志会打印 `[GUARD][WARN]` 提醒手动去 Settings → Actions 关闭。
- **GitHub 定时任务不精确**：`cron` 触发常有几分钟延迟；且仓库 **60 天无任何活动**时 GitHub 会自动停用 schedule，偶尔去手动跑一次或提交保持活跃。
- **安全**：仓库务必保持 Private，所有凭证走 Secrets，不要写进代码。
