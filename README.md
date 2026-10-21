# GARVEE 三站 SEO 与 SSR 审计 · 完整证据包

对 GARVEE 美国站、英国站、加拿大站测试环境的页面类型与 SSR 证据审计。
本仓库包含**汇总视图**与**全部原始证据**，可离线复核每一条结论。

## 快速开始

| 入口 | 用途 |
|---|---|
| `index.html` | 汇总看板：逐页结果、按市场/类型统计、问题分类 |
| `pages.csv` | 逐页明细（Excel 可直接打开） |
| `detail/index.html` | 明细看板：每条规则的实际值 / 预期值 / 判定依据 |
| `detail/checks.csv` | 全部规则判定明细（约 12,000 行） |
| `detail/samples.csv` | 样本级汇总 |

用浏览器直接打开 `index.html` 即可，无需服务器。

## 原始证据

| 路径 | 内容 |
|---|---|
| `raw/*.html.gz` | 原始 HTTP 响应（未渲染源码），文件名即样本 ID |
| `raw/*.nojs.html.gz` | 禁用 JavaScript 时的浏览器快照 |
| `raw/*.js.html.gz` | 执行 JavaScript 后的浏览器快照 |
| `raw/*.interaction.html.gz` | 下滚交互后的浏览器快照 |
| `audit.sqlite` | 完整审计数据库：样本、判定、Jev 调用、事件日志 |
| `manifest.json` | 冻结的样本清单与采集策略 |
| `data.json` / `data.js` | 汇总看板的结构化数据 |

`audit.sqlite` 是单一事实来源，可用任意 SQLite 工具查询：

```sql
-- 查看某页全部判定
SELECT json_extract(spec,'$.url') AS url
FROM samples
WHERE json_extract(spec,'$.url') LIKE '%/blogs/news%';

-- 统计各规则失败次数
SELECT json_extract(c.value,'$.rule') AS rule, COUNT(*) AS n
FROM samples, json_each(result,'$.checks') c
WHERE json_extract(c.value,'$.status')='fail'
GROUP BY rule ORDER BY n DESC;
```

## 覆盖

- **266 个页面**：取自三站自身 sitemap，每类型至少 5 个样本（站点实际不足时如实记录）
- **功能状态**：排序、分页、库存筛选来自真实控件点击，非猜测参数
- **三层证据**：原始 HTTP、无 JS / JS / 交互浏览器快照、Jev 语义判断
- **263 页**完成浏览器取证，**134 页**完成 Jev 判断

## 主要发现

| 问题 | 页面数 |
|---|---|
| 缺少分享图片标签（OG / Twitter） | 180+ |
| 服务端直出内容被 `hidden` 隐藏 | 155 |
| 缺少 canonical 标签 | 122 |
| 缺少 meta description | 101 |
| 缺少 title 标签 | 72 |
| Sitemap 提交页面返回 404 | 35 |

## 方法说明

- 预期值来自项目已确认的 canonical / robots / 分享标签契约，**不取自页面自身观测值**。
- SSR 结论基于源码与渲染结果对照；缺少证据的项标为待确认，不按通过处理。
- Jev 仅回答窄范围语义问题（标题是否匹配、内容是否同主题、证据是否充分），
  不参与 HTTP、canonical、robots 等确定性判断。
- 同一规则在原始 HTML 和三种浏览器状态中会各计一次，明细表中浏览器条目约为源码条目的三倍。

## 已知边界

- 测试环境结果，不等同生产环境现状。
- 35 个 4xx 是站点 sitemap 主动提交但实际不可访问的 URL（含中文内部集合名）。
- 未覆盖：hreflang 实体互返、完整 Schema 资格、HTML5 语义、Core Web Vitals。
- 部分页面区域文本为空，Jev 返回「证据不足」属预期行为，不记为缺陷。

## 复现

证据由 `jev-seo` Skill 的 matrix 审计能力采集，可用同一套规则离线复核：

```bash
<skill>/run matrix recheck <run-dir> --policy <policy.json>
```

`recheck` 不发起网络请求、不调用模型，仅重算已保存证据。
