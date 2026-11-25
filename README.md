# vibe-time-sound

[Vibe Time](https://github.com/limengwei/VibeTime) 的声音库仓库：主程序不内嵌任何声音文件，客户端运行时按 manifest 增量下载并缓存到本地（`%AppData%\vibe-time\pack\`）。

## 仓库结构

```
vibe-time-sound/
├── sounds/<分类>/<base>.<ext>   # 声音文件（m4a/mp3/wav/ogg，原样入库不转码）
├── manifest.json                # 索引：version + 每文件 id/path/size/sha256 —— packgen 生成，勿手改
├── labels.json                  # 中文名/分类名/署名 —— 手工可编辑，packgen 会保留手工修改
└── sources.json                 # 可选：下载源下发 —— 纯手工维护（未创建则客户端只用内置源）
```

## 命名规范

- 分类目录名 = 分类 ID（英文小写，如 `rain`、`animals`）；新分类未在 labels.json 配置中文名时，客户端显示目录名
- 文件 base 名（去扩展名）= 声音短名（如 `heavy-rain`）
- 声音 ID = `分类/base`（如 `rain/heavy-rain`），是客户端引用的稳定键
- 重命名目录/文件 = 旧声音下架 + 新声音上架：客户端自动删除旧文件、下载新文件

## labels.json

| 节 | 结构 | 说明 |
|---|---|---|
| `categories` | `分类ID → {title, emoji}` | 中文分类名与 emoji；改名会传播到客户端 |
| `labels` | `base名 → 中文名` | 未命名的显示英文派生名；packgen 输出会提示待命名数量 |
| `attribution`（可选） | `base名 → {origin, license, url}` | 外部整理预留：来源页、许可、外部直链 |

## 更新流程

### A. 增/删/换声音文件

在主程序仓库（`D:\workspace\go\src\vibe-time`）运行：

```powershell
$env:GOROOT=$null
go run ./cmd/packgen -src <整理目录> -repo D:\workspace\py\vibe-time-sound
```

- 新文件拷入，源目录没有的删除，内容变化的重算 sha256，**version 自动递增**
- 之后编辑 labels.json 给新声音补中文名（packgen 输出会提示「待命名」数量）
- 推送（**双远端都要**）：

```powershell
git push origin master   # gitee
git push github master   # github
```

客户端下次启动自动增量：新增下载、换字节重下、下架删盘并停播、改名生效。version 只是日志信息，无需手工管理。

### B. 换下载源链接（sources.json）

镜像挂了想换源时，在仓库根维护 `sources.json`：

```json
{"sources": ["https://raw.githubusercontent.com/limengwei/vibe-time-sound/master"]}
```

- 值为 **raw 基址**（能拼 `/manifest.json` 访问到的根）
- 客户端优先用下发源、内置源兜底；manifest 的拉取永远保留内置多源竞速，sources.json 写坏也不会死锁

### C. 外部直链/署名（attribution）

labels.json 的 `attribution` 节示例：

```json
{
  "beehive": {
    "origin": "https://sound-effects.bbcrewind.co.uk/...",
    "license": "RemArc",
    "url": "https://外站直链/beehive.m4a"
  }
}
```

- `url` 作为该文件的额外下载候选参与多源竞速；**字节必须与 manifest 哈希一致**（即同字节的备用位置，不是放不同内容的地方——内容仍以仓库为准）
- packgen 重写 labels.json 时保留该节，声音下架自动清理
- `origin` / `license` 客户端已解析，暂未做展示 UI

## 客户端下载行为（参考）

- 启动拉 manifest（多源并发竞速，HTML/text 响应判负），按 state.json 记录的哈希做增量更新
- 冷启动（零缓存）显示欢迎页引导，用户确认后整库 zip 单请求拉齐
- 缓存目录：`%AppData%\vibe-time\pack\`（`sounds/` + `manifest.json` + `state.json`）

## 许可注意

- [Moodist](https://github.com/remvze/moodist) 声音：CC 许可，相对宽松
- BBC Rewind（RemArc 许可）：仅个人/教育/研究用途，**限制商用**——整理时逐条记录 license
- 外站资源入库前自行确认许可，并在 attribution 记录来源
