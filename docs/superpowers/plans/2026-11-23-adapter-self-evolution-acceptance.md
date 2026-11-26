# Adapter 自进化一期 · T9 真实验收记录

日期:2026-11-26
验收人:rickhuang(工作台机器)+ Claude(agent-LLM 驱动修复循环)
被验收对象:PR #457(`feat/adapter-self-evolution-dev112` → dev-1.2.2)

## 一、目标与调整

原定目标 wekm 经排查**不成立**:wekm 不在 AdapterHub(404),是用户 v_wbvincentli 机器上自产自用的本地 adapter——其 46 次失败集中在她一人印证了这一点。本地无法取得其代码,验收目标切换为:

> **we123-teamspace**(hub 在售 v1.0.3,kinghao/郝金福,10 个 wewright 命令;线上失败:抽奖类命令 exit-code-1,get-lottery-status 历史成功率仅 18%)

## 二、验收过程实录

| 步骤 | 命令/动作 | 结果 |
|---|---|---|
| 1 修复任务生成 | `webgo repair task --from errors --since 7d --min-calls 3` | 首跑崩溃(**验收发现缺陷 ①**),修复后重新生成 **19 个任务**干净落地 |
| 2 安装目标 | `webgo adapterhub install we123-teamspace` | ✅ v1.0.3 装入用户目录 |
| 3 复现 | `webgo call we123-teamspace.get-lottery-status` | 经浏览器环境修复后**复现真实失败**:页面被重定向到 `/teamspace/login`,命令盲目等待「盲盒抽奖」标题 15s 超时 → exit 1 |
| 4 诊断 | 登录后重跑同命令 | ✅ `ok:true` 返回完整抽奖状态——确认根因为**登录态过期 + 代码缺登录重定向检测**(真实代码缺陷) |
| 5 修复循环 | `runRepairLoop`(agent-LLM 注入补丁) | 第 1 轮:**draft-ready**;四层验证全绿(静态/重放/fixture/冒烟跳过) |
| 6 草稿 | `~/.webcli/repair-drafts/20261126094455/we123-teamspace` | v1.0.3 → **v1.0.4**,diff = 登录重定向检测块(9 行) |
| 7 经验沉淀 | 自动 | `we123-teamspace | fixed | logic-bug | Toolkit Adapter exited with code 1` 已入本地经验库 |

### 浏览器环境排障(过程记录)

复现期间接连遇到:受管浏览器僵尸会话(扩展未连接)→ `service stop` + 清 `~/.webcli/browser/profile/Default/Service Worker` + `service start` 后扩展重连成功(与「升级后清 SW 缓存」既有运维知识一致)。

## 三、验收发现的缺陷(均已修复或有据记录)

| # | 缺陷 | 定级 | 处置 |
|---|---|---|---|
| ① | errdetail 中 `open_browser` 类调用把整段 URL 记进工具名(`open.https:\...`),派生 adapterId 含非法字符 → `saveTask` 写文件 ENOENT,**任务生成中途崩溃、后续任务全部丢失** | 高 | ✅ 已修(`SAFE_ADAPTER_ID` 白名单跳过,commit bf4cb451,已推 PR #457) |
| ② | **写命令被选为重放主命令**:verifyRepair 盲取 failures[0],首条恰为 draw-lottery(写命令)→ 验证轮真实执行了一次抽奖重放(幸被「本周已抽」业务规则拒绝) | 高 | ✅ 验收时以写安全规则规避(只读命令置首 + 关冒烟);**机制性修复列入二期 §5.3**(verifyRepair 主命令选择须跳过写命令) |
| ③ | wekm 类「自产自用」adapter 不在 hub,一期管线无法触达 | 记录 | 二期 hub 经验共享/工作台范畴;此类失败建议 errdetail 报表按「非 hub adapter」分组呈现 |
| ④ | costplus 补贴命令(633 失败/7d)属**桌面内置 extension 命令集**,与 hub v1.0.12(审批流 11 命令)是两套命令集;`status=false` 根因已诊断(e-cology 会话过期返回 HTTP 200 + status:false,代码只映射 HTTP 401/403) | 记录 | builtin 回流通道为二期既定项;诊断结论已录 §五 |
| ⑤ | repo 源码树 `adapterhub-profile.json` 指向外网 weworker.cc,repo checkout 直跑 install 会拿到 HTML | 体验 | 工作台用 `WEBCLI_ADAPTERHUB_ENDPOINT` env 规避;repack 包注入行内 profile 不受影响 |

## 四、验收标准核对(计划 Task 9)

| 标准 | 结果 |
|---|---|
| 1 修复草稿通过四层验证 | ✅ 静态 24 检查全过 / 重放 exit 0 + 数据完整 / fixture 断言过 / 冒烟按写安全规则关闭(替代:登录后 live 调用实测成功) |
| 2 原报错命令实跑 exit 0 | ✅ 登录态下 `replay:get-lottery-status` ✓(验证器真实 Webwright 实跑) |
| 3 经验库出现对应条目 | ✅ `we123-teamspace | fixed | logic-bug` |
| 4 发布后失败数收敛 | ⏳ 待发布后观察(diff 交 kinghao 发布 v1.0.4) |
| 5 验收记录归档 | ✅ 本文档 |

**结论:管线功能验收通过**(触发→生成→复现→诊断→补丁→验证→草稿全链路真实走通);发布环节因 adapter 归属他人(需 kinghao PAT)按实际流程移交,发布后补第 4 条收敛数据。

## 五、移交与后续

1. **we123-teamspace v1.0.4 修复 diff** 交 kinghao 审阅发布(草稿:`~/.webcli/repair-drafts/20261126094455/we123-teamspace`;修复内容:get-lottery-status 增加登录重定向检测,登出态立即返回 NEEDS_LOGIN 引导;建议同 pattern 推广至其余 9 命令)
2. costplus `status=false` 修复(登录失效分级)走 builtin 回流通道,归入二期
3. 二期新增待办:verifyRepair 主命令写安全强制、errdetail 非 hub adapter 分组
