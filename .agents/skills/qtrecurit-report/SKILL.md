# 量潮招聘日报流程

使用 `qtadmin qtrecurit status` 命令生成量潮招聘日报，保存到 `data/report/qtrecurit/daily/`。

## 前置条件

- `qtadmin` 已安装
- `data/report` 子模块已初始化

## 流程

### 1. 生成日报

```bash
qtadmin qtrecurit status > data/report/qtrecurit/daily/$(date +%Y-%m-%d).md
```

### 2. 提交到子模块

```bash
git -C data/report add qtrecurit/daily/
git -C data/report commit -m "docs: update recruitment daily report for $(date +%Y-%m-%d)"
git -C data/report push
```

### 3. 更新主仓库引用

```bash
git add data/report
git commit -m "chore: update data/report submodule (recruitment daily report)"
git push
```

## 命令参考

`qtadmin qtrecurit status` 支持的参数：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--days N` | 统计最近 N 天 | — |
| `--start YYYY-MM-DD` | 起始日期 | 当月 1 日 |
| `--end YYYY-MM-DD` | 结束日期 | 当天 |

不传参数时默认统计当月至今数据。

### 示例

```bash
# 本月数据（默认）
qtadmin qtrecurit status

# 最近 7 天
qtadmin qtrecurit status --days 7

# 指定日期范围
qtadmin qtrecurit status --start 2026-06-01 --end 2026-06-17
```

## 日报内容

命令输出包含以下章节：

- **标题**：`# 量潮招聘数据统计 (开始日期 至 结束日期)`
- **总量**：投递总数 + 可识别岗位数及比例
- **岗位分布**：各岗位投递人数排序表
- **投递趋势**：每日投递数 + 趋势箭头（↑/↓/-）

## 分类规则

内置分类规则位于 `apps/qtadmin/src/cli/src/human/config.rs`，匹配逻辑：

1. 提取主题中 `【】` 或 `岗位：` 后的关键词
2. 按优先级匹配关键词列表，排除命中 exclude 列表的项
3. 不匹配任何规则的归入"未识别"

可通过环境变量 `QTRECURIT_CONFIG` 或运行目录下的 `qtrecurit.toml` 覆盖分类规则。
