# Gmeek blog build troubleshooting

> 一篇记录「重新开始维护 2 年没更新的 GitHub Pages 博客」完整排障过程的文章。踩了 4 个坑，其中一个是 GitHub Actions 的隐藏故障。

## 背景

我的博客（Gmeek 框架）最后更新停在 2024-03，之后一直没动。最近想发一篇新文章，结果发现：

1. 直接往 `backup/` 目录丢 Markdown → **没用**，构建时被删掉
2. 创建 Issue 发布文章 → **没生成**，Actions 一直排队
3. 手动触发 workflow → **排队 30 分钟不分配 runner**

整个过程像拆盲盒，最后发现根因是个"时间炸弹"。

## 先搞懂 Gmeek 的发布机制

Gmeek（[Meekdai/Gmeek](https://github.com/Meekdai/Gmeek)）是一个基于 GitHub Issues 的静态博客框架，机制是：

```
写 Issue（标题=文章标题，正文=文章内容）→ 触发 Actions → Gmeek.py 拉取 Issues 生成 HTML → 部署 Pages
```

- **文章的真正来源是 GitHub Issues**，不是仓库里的文件
- `backup/` 目录是**构建产物**（Gmeek.py 从 Issue 同步下来的备份），构建时会先清空重建
- `docs/` 是生成的静态页面

**关键规则**：Issue **必须至少带一个 Label**，Gmeek 的 `addOnePostJson()` 只处理 `len(issue.labels) >= 1` 的 Issue，无标签的直接忽略。

## 坑 1：往 backup/ 丢文件 = 无效

我把写好的 Markdown 直接放进 `backup/` 并 push，以为这样就会变成文章。

结果 Gmeek.py 的 `cleanFile()` 第一步就是：

```python
def cleanFile(self):
    if os.path.exists(self.backup_dir):
        shutil.rmtree(self.backup_dir)  # 直接删掉！
    os.mkdir(self.backup_dir)           # 再重建空目录
```

我手工放的文件被当成垃圾清掉了。正确姿势是：**文章写在 Issue 里**，backup 是构建时自动生成的。

## 坑 2：无 Label 的 Issue 被静默忽略

创建了 Issue（标题、正文都正确），但忘了加 Label。GitHub Actions 触发了，但 Gmeek.py 遍历 Issues 时直接跳过：

```python
def addOnePostJson(self, issue):
    if len(issue.labels) >= 1:   # 无标签 → 整个函数不执行
        ...
```

不报错、不警告，文章就是不出来。**给 Issue 加上任意 Label 即可**（Label 还会作为文章的分类标签显示在博客上）。

## 坑 3（重头戏）：Actions 永远排队，根因是 ubuntu-20.04

所有配置看起来都对，但 workflow 的 job **永远卡在 queued**，30 分钟不分配 runner。

排查过程：
1. 检查 Pages 配置：`build_type: workflow` 正常
2. 检查 Actions 权限：`enabled: true` 正常
3. 检查其他仓库：x-python、pi-mono 的 Actions 都在正常运行 → 排除账户级问题

最后怀疑到 workflow 文件里的 `runs-on: ubuntu-20.04`。一查发现：

> **GitHub 已于 2025-04-15 完全移除 Ubuntu 20.04 runner 镜像**（[actions/runner-images#11101](https://github.com/actions/runner-images/issues/11101)）

指定一个不存在的镜像，runner 永远不会分配，job 就无限排队。而我的 workflow 是 2024 年从模板创建的，从来没更新过。

**修复**：两处 `runs-on: ubuntu-20.04` → `ubuntu-24.04`。

修复前后对比：

| | 修复前 | 修复后 |
|---|---|---|
| runner 分配 | 30 分钟+ 不分配 | **22 秒** |
| job 状态 | queued 卡死 | in_progress → success |

顺带说一句：如果你的博客/项目也用 GitHub Actions 且很久没更新，检查一下 workflow 里的镜像版本。GitHub 对 `ubuntu-20.04`、`macos-12` 等旧镜像都执行过移除计划（N-1 支持策略），超过 2 年的 workflow 大概率中招。

## 备用方案：本地构建 Gmeek

在等 runner 的时候，我准备了一个不依赖 Actions 的备用方案——本地直接跑 Gmeek.py：

```bash
# 1. 准备构建目录（模拟 Actions 工作区）
mkdir -p /tmp/gmeek-build
cp -r <仓库内容> /tmp/gmeek-build/
cp -r <Gmeek源码> /tmp/gmeek-build/

# 2. 安装依赖
pip install PyGithub requests xpinyin feedgen Jinja2 transliterate

# 3. 运行构建（需要 GITHUB_TOKEN 访问 Issues）
export GITHUB_WORKSPACE=/tmp/gmeek-build
cd /tmp/gmeek-build
python Gmeek.py <GITHUB_TOKEN> <owner>/<repo> --issue_number '0'

# 4. 把产物拷回仓库提交
cp -a docs backup blogBase.json <仓库>/
```

关键点：Gmeek.py 的 `cleanFile()` 依赖 `GITHUB_WORKSPACE` 环境变量（Actions 里自动有，本地要手动 export），否则会报错。

这个方案验证了「构建逻辑本身没问题，纯粹是 runner 的问题」，也帮我提前拿到了构建产物。

## 坑 4：中文标题的 URL 会很难看

Gmeek 默认 `urlMode: "pinyin"`，中文标题会被转拼音生成文件名。中英混排的标题转出来很乱：

```
pi agent -duo-mo-xing-jie-ru-shi-zhan-：-guan-fang- API -yu-gong-si-wang-guan-shuang- provider -pei-zhi.html
```

（全角冒号都没清理）

而英文标题的 URL 就很干净：`pi agent multi-provider deepseek config.html`。

**建议**：Issue 标题用英文（或全中文 + 纯空格结构），正文内容用中文不受影响。

## 总结

```
文章 = Issue（必须带 Label）          ← Gmeek 的文章机制
backup/ 是构建产物，不是文章源        ← 别往里丢文件
ubuntu-20.04 已死（2025-04）         ← Actions 排队的隐藏根因
本地跑 Gmeek.py 可绕过 Actions       ← 备用方案
英文标题 → 干净 URL                  ← 发布前想好标题
```

一次"发文章"的小事，牵出了 4 个坑，其中 `ubuntu-20.04` 这个坑属于"GitHub 删了旧镜像但你不知道"的典型时间炸弹。如果你的 GitHub Actions 突然开始无限排队，先查 `runs-on` 是不是用了已移除的镜像。