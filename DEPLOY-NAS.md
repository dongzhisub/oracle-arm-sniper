# NAS (群晖 DSM) 定时触发器部署指南

> 目的：用家里群晖的 DSM 任务计划每 15 分钟精准触发一次抢机 workflow，
> 绕开 GitHub 自带 cron 的 2~4 小时调度延迟。GitHub 侧 cron 保留作兜底。

## 一、生成 Fine-grained PAT（细粒度令牌）

1. GitHub → 右上头像 → **Settings** → 左栏最底 **Developer settings**
2. **Personal access tokens → Fine-grained tokens** → **Generate new token**
3. 填写：
   - Token name: `sniper-dispatcher`
   - Expiration: **90 days**（到期后需重新生成并更新 DSM 任务计划）
   - Repository access: **Only select repositories** → 只选 `dongzhisub/oracle-arm-sniper`
   - Permissions → Repository permissions → **Actions: Read and write**
     （Metadata: Read-only 会自动带上，别的不用勾）
4. Generate token → 复制以 `github_pat_` 开头的值（只显示一次）

泄漏影响评估：该 PAT 只能触发这一个仓库的 workflow，看不到代码、读不到
Secrets（Secrets 只写不读）、碰不到 OCI 凭证。最坏情况是被人反复触发抢机
脚本刷 Actions 分钟数（$0.008/分钟），吊销即止损。

## 二、DSM 任务计划配置（DSM 7.x）

1. DSM → **控制面板 → 任务计划 → 新增 → 计划的任务 → 用户定义的脚本**
2. **常规** 标签：
   - 任务名称: `sniper-trigger`
   - 用户: **root**（必须，否则可能写不了日志）
   - 已启用: ✅
3. **计划** 标签：
   - 运行日期: 每天
   - 频率: **每 15 分钟**
   - ⚠️ 检查"最后运行时间"覆盖到 23:59，保证全天覆盖
4. **任务设置** 标签 → 用户定义的脚本，粘贴下面脚本，并把
   `PASTE_YOUR_FINE_GRAINED_PAT_HERE` 换成第一步的 PAT：

```sh
#!/bin/sh
# Oracle ARM Sniper dispatcher (DSM Task Scheduler, every 15 min)
PAT="PASTE_YOUR_FINE_GRAINED_PAT_HERE"
LOG="/volume1/sniper_trigger.log"
TS=$(date '+%F %T')
CODE=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 15 --max-time 60 \
  -X POST \
  -H "Authorization: Bearer ${PAT}" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/dongzhisub/oracle-arm-sniper/actions/workflows/sniper.yml/dispatches \
  -d '{"ref":"main"}')
echo "${TS} dispatch=${CODE}" >> "${LOG}"
```

5. 确定。选中任务点 **运行** 手动试一次。

## 三、验证

1. DSM：任务计划 → 选中任务 → 操作 → **查看结果**，应无报错；
   File Station 里看 `/volume1/sniper_trigger.log` 出现一行
   `2026-xx-xx xx:xx:xx dispatch=204`（204 = 成功）
2. GitHub：`gh run list` 或 Actions 页面应立即多出一条 run
3. 状态码对照：`204` 成功 / `401` PAT 无效或过期 / `422` workflow 路径不对 /
   `000` 网络不通（检查 NAS 出网/DNS）

## 四、运维备忘

- **日志**：`/volume1/sniper_trigger.log` 每次触发追加一行，隔段时间清一下即可
- **双触发兜底**：GitHub 自带 cron（有 2~4h 延迟）保留未删，NAS 断电/断网时
  仍有随机触发；两边并发由 workflow 的 concurrency 自动排队合并，无冲突
- **PAT 到期**（90 天）：日志连续出现 `401` 就是过期信号 → 重新生成 PAT →
  编辑任务计划里的脚本替换 → 保存
- **抢到机器后**：workflow 会自动推送 commit 注释 cron 自退役，但 NAS 触发器
  是外部调用，**不会自动停**！抢到后请到 DSM 把任务 `sniper-trigger` 停用
  （workflow 哨兵会拦住重复创建，但建议干脆停掉省心）
