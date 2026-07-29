---
name: gamekee-ba-download
description: 批量下载 gamekee.com 碧蓝档案(BA)图鉴「实装学生」全角色的「回忆大厅」「官方介绍」图片到本地子文件夹。Use when the user wants to batch-download Blue Archive character images from gamekee wiki, or says 回忆大厅 / 官方介绍 图下载, 碧蓝档案图鉴抓图, gamekee ba 图片批量下载, 实装学生图片. Triggers: gamekee, 碧蓝档案, BA图鉴, 回忆大厅, 官方介绍, 角色立绘下载.
---

# GameKee 碧蓝档案图鉴 · 图片批量下载

从 `https://www.gamekee.com/ba/second/23941`（实装学生图鉴）批量下载每个角色详情页「影画鉴赏」标签下的指定类型图片。

## 参数（向用户确认缺失项，有默认值则用默认）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `target` | `hydt` | `hydt`=回忆大厅 ｜ `gfjs`=官方介绍 ｜ `both`=两者都下 |
| `output_dir` | 当前工作目录 | 图片落盘的根目录 |
| `list_url` | `https://www.gamekee.com/ba/second/23941` | 角色花名册入口（实装学生） |

## target 选择器映射表（核心，已实测验证）

| target 值 | 子标签中文名 | 选择器 | 形态 | 备注 |
|---|---|---|---|---|
| `hydt` | 回忆大厅 | `img.hydt` | 单图 | `hydt`=回忆大厅拼音首字母，gamekee 专属 class，全局唯一 |
| `gfjs` | 官方介绍 | `.role-img-box img.item`（取第一个） | 单图 | `item` 是共享 class，必须取首个；该图在 DOM 中永远位于 `img.hydt` 之前 |

> 未来扩展「设定集-日文/繁中」「本家画」「表情」需走图集模式（按 `.header-container` 文本切块取该节区全部 img），本 skill v1 不覆盖。

## 关键约束（踩坑实证，不可违反）

1. **MCP 浏览器进程完全沙箱化**：`browser_run_code_unsafe` 里没有 `require`/`process`/`global`，不能 `import('node:fs')`，`navigator.clipboard` 是 undefined。**唯一数据通道 = 该函数的 return 字符串**。流程必须分两段：浏览器抽 URL（return JSON 字符串）→ 将返回的 JSON 写入文件 → PowerShell 下载。
2. **content JSON 走不通捷径**：`api-cdn.gamekee.com/wiki2.0/pro/829/content/{id}.json` 被 Tencent EdgeOne WAF 拦（PowerShell 直请返回 567；页面内 `fetch` 被 CORS 拦）。但**图片 host `cdnimg-v2.gamekee.com` 不拦**——只能「浏览器跑 JS 拿 img.src → PowerShell 下载图」，无法走「读 API JSON 解析图 URL」。
3. **PowerShell 5.1 编码**：含中文字面量（路径名、`回忆大厅`）的 `.ps1` 脚本**必须存成 UTF-8 with BOM**，否则 PS 5.1 按 GBK 解析，中文路径变乱码，所有下载报「路径不存在」。用 `[IO.File]::ReadAllBytes` 检测后 prepend `EF BB BF` 可补 BOM。
4. **批大小默认 12，仍超时降 6**：实测 15 个角色/批会因单个页面慢加载把单次 MCP 调用拖到 10 分钟超时。12 是大多数时段稳定上限。**若 12 仍触发 MCP 超时**（服务端负载时段波动会导致），降为 6 个/批；拆批结果存为 `batchNNa.json` / `batchNNb.json`，download.ps1 的 `batch*.json` 通配符自动覆盖，无需改脚本。
5. **图片 URL 清洗**：页面 `img.src` 含 `?x-image-process=.../format,webp` 转换参数，**必须 `src.split('?')[0]`** 去掉转换参数；头部补 `https:`（页面里是 `//cdnimg-v2...` 协议相对 URL）。**注意：`split('?')[0]` 不保证拿到原始格式**——CDN 对个别资源做内容协商（content negotiation），无论 URL 带不带 webp 参数，响应体本身可能是 webp（实测 id=677904 律的 URL 是 `.jpg` 但响应是 webp）。脚本必须兼容 PNG/JPG/WEBP 三种格式，不能假设「去参数 = 原始 PNG/JPG」。
6. **文件名取自 `<title>` 而非 list 页卡片名**：`page.title().split('_碧蓝档案')[0]`。list 页用半角括号，title 用全角括号，后者更规范。例：id=86656 list 卡片名「瞬(小)」，title 是「瞬（幼女）」，落盘为 `瞬（幼女）.png`。
7. **扩展名按下载后实际文件头决定，不按 URL 后缀**：CDN 内容协商会导致 URL 后缀与实际格式不一致（实测 267 个里 8 个 png/jpg 后缀互换 + 1 个 webp，共 9 个错配；另有 URL 大写 `.JPG` 的情况）。download.ps1 下载到临时文件后读首 12 字节判定真实格式（`Get-ImageFormat`），再改名成正确扩展名，**完全不依赖 URL 后缀**。

## 完整流程

### 阶段 1 — 抓角色花名册（target 无关，只跑一次）

1. `browser_navigate` 到 `list_url`
2. `browser_wait_for` 等 5 秒（SPA 首屏渲染需要时间，否则 `.item-wrapper` 为空）
3. `browser_run_code_unsafe` 执行（若返回 `error:'no wrap'`，说明 SPA 首屏渲染失败——`page.reload()` 后再等 5 秒重试，最多 2 次；仍空则报错让 agent 重新 `browser_navigate`，gamekee CDN 偶发不稳）：
   ```js
   async (page) => {
     const data = await page.evaluate(() => {
       // 第一个 .model-tab-content 即「实装学生」标签页
       const wrap = document.querySelectorAll('.model-tab-content')[0];
       if(!wrap) return {error:'no wrap', count:0};
       const cards = Array.from(wrap.querySelectorAll('.item-wrapper a.item'));
       return { count: cards.length, items: cards.map(a => ({
         id: (a.getAttribute('href')||'').match(/\d+/) ? RegExp.lastMatch : '',
         name: a.querySelector('.name') ? a.querySelector('.name').textContent.trim() : ''
       })) };
     });
     return JSON.stringify(data);
   }
   ```
   → 返回约 272 个 `{id, name}`。list 页的 name 仅交叉校验，**真正文件名在阶段 2 从 title 取**。

### 阶段 2 — 逐角色抽取图片 URL（按 target 切换选择器）

对 roster 按**每 12 个一批**调用 `browser_run_code_unsafe`，模板如下（batch_size=12）：

```js
async (page) => {
  const ids = ["ID1","ID2",...,"ID12"];  // 本批 12 个 id
  const out = [];
  for (const id of ids) {
    const rec = {id, name:null, img:null, err:null};
    try {
      const u = 'https://www.gamekee.com/ba/tj/'+id+'.html?tab=3';  // ?tab=3 = 预选影画鉴赏
      const poll = async (ms) => {
        let t0=Date.now();
        while(Date.now()-t0<ms){
          // ★ 选择器按 target 切换这一行 ★
          //   hydt: document.querySelector('img.hydt')
          //   gfjs: document.querySelector('.role-img-box img.item')
          const h = await page.evaluate(() => {
            const i = document.querySelector('img.hydt');  // ← hydt 改 gfjs 时换这行
            return i ? i.getAttribute('src') : null;
          });
          if(h) return h;
          await page.waitForTimeout(400);
        }
        return null;
      };
      await page.goto(u,{waitUntil:'domcontentloaded',timeout:30000});
      let h = await poll(8000);
      // 兜底：没拿到则点「回忆大厅/官方介绍」子标签再轮询
      if(!h){
        try{
          await page.evaluate(() => {
            // 子标签文本节点（回忆大厅/官方介绍等都在 .tab-box 内）
            const cs = Array.from(document.querySelectorAll('*'))
              .filter(e => e.children.length===0
                && (e.textContent.trim()==='回忆大厅'||e.textContent.trim()==='官方介绍')
                && e.closest('.tab-box,[class*=sub-tab],[class*=tab-nav]'));
            if(cs[0]) cs[0].click();
          });
        }catch(ce){}
        h = await poll(5000);
      }
      const title = await page.title();
      let name = title.split('_碧蓝档案')[0];
      // ★ 「编辑中」页面：title 前缀带【编辑中】，剥离前缀并标记（见「已知数据特点」）
      //   实测 id=714029 千秋(泳装) 等 wiki 未完成编辑的角色，title 为「【编辑中】千秋(泳装)_碧蓝档案...」
      //   这类页面回忆大厅图通常缺失（img.hydt=null，非占位图），官方介绍图可能正常存在。
      //   不剥离会导致落盘文件名带【编辑中】前缀污染。
      if(name.startsWith('【编辑中】')){
        rec.editing = true;
        name = name.replace(/^【编辑中】/, '').trim();
      }
      rec.name = name;
      let url = h ? h.split('?')[0] : null;        // ★ 去掉 webp 转换参数
      if(url && url.indexOf('http')!==0){ url = 'https:'+url; }  // ★ 补协议
      rec.img = url;
    } catch(e) { rec.err = e.message; }
    out.push(rec);
  }
  return JSON.stringify(out);
}
```

要点：
- `target=both` 时跑两遍此阶段（一遍用 hydt 选择器，一遍用 gfjs 选择器），分别落盘到不同 batch 文件夹。
- 返回的 JSON 中 `img:null` 的角色（偶发未渲染）收集起来，全部批次跑完后**统一二次重试**（同样的 goto+poll，可加一次 reload 兜底）。实测约 5/267 会偶发 null，重试基本都能成功。
- 角色名含 `*` 等非法字符（如「白子*恐怖」）由阶段 4 脚本统一转义，这里原样保留。

### 阶段 3 — 浏览器→文件系统桥

每批返回的 JSON 写入 `batches/batchNN.json` 文件（每文件一批，NN 从 01 起）。结构：
```json
[
  {"id":"59934","name":"日奈","img":"https://cdnimg-v2.gamekee.com/.../674225.png"},
  {"id":"714029","name":"千秋(泳装)","img":null,"editing":true},
  ...
]
```

> **⚠ batch*.json 一律用文件写入工具（Write / write 等）生成或全量重写，不要用 PowerShell 的 `Set-Content`/`Out-File` 修改这些 JSON。** PS 5.1 的 `-Encoding UTF8` 默认带 BOM，BOM 会污染文件头，导致后续基于文本匹配的编辑工具（edit 等）匹配失败报 "No match found"。如需修改 batch JSON，用写入工具整文件重写。PowerShell 的 `-replace` 在复杂 JSON（含引号/反斜杠/Unicode）上转义也极脆弱，切勿用来批量改 JSON。

### 阶段 4 — PowerShell 批量下载（脚本模板）

在 `output_dir` 下建子文件夹（Plan A：按 target 分子文件夹）：
- `hydt` → `{output_dir}/回忆大厅/`
- `gfjs` → `{output_dir}/官方介绍/`
- `both` → 两个子文件夹都建

把下面脚本存为 `download.ps1`（**必须 UTF-8 with BOM**，见约束 3）。脚本零硬编码，全部通过 `param()` 显式传参，调用方式见脚本下方：

```powershell
param(
  [Parameter(Mandatory=$true)]  [string]$OutputDir,   # 图片落盘根目录（绝对路径）
  [Parameter(Mandatory=$true)]  [ValidateSet('hydt','gfjs')][string]$Target,  # hydt=回忆大厅 gfjs=官方介绍
  [string]$BatchesDir = (Join-Path $OutputDir 'batches')   # batch*.json 所在目录，默认 {OutputDir}\batches
)
$ErrorActionPreference = 'Continue'

# target → 子文件夹名映射（Plan A：按 target 分子文件夹）
$subMap = @{ 'hydt' = '回忆大厅'; 'gfjs' = '官方介绍' }
$dest = Join-Path $OutputDir $subMap[$Target]
if(-not (Test-Path -LiteralPath $dest)){ New-Item -ItemType Directory -Force -Path $dest | Out-Null }

$headers = @{
  'Referer'    = 'https://www.gamekee.com/'
  'User-Agent' = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36'
}

# 读首 12 字节判定真实图片格式（CDN 做内容协商，URL 后缀不可信，见约束 5/7）
# PNG=89504E47  JPG=FFD8FF  WEBP=RIFF(52494646)????????WEBP(57454250)
function Get-ImageFormat($p){
  $fs=$null
  try{
    $fs=[IO.File]::OpenRead($p); $b=New-Object byte[] 12; $n=$fs.Read($b,0,12)
    if($n -ge 4 -and $b[0]-eq0x89 -and $b[1]-eq0x50 -and $b[2]-eq0x4E -and $b[3]-eq0x47){return 'png'}
    if($n -ge 3 -and $b[0]-eq0xFF -and $b[1]-eq0xD8 -and $b[2]-eq0xFF){return 'jpg'}
    if($n -ge 12 -and $b[0]-eq0x52 -and $b[1]-eq0x49 -and $b[2]-eq0x46 -and $b[3]-eq0x46 -and $b[8]-eq0x57 -and $b[9]-eq0x45 -and $b[10]-eq0x42 -and $b[11]-eq0x50){return 'webp'}
    return $null
  }catch{return $null}
  finally{ if($fs){ $fs.Close() } }   # Read 抛异常也释放句柄，否则后续 rename/remove 会"文件被占用"
}
function Test-ValidImage($p){ return ([string](Get-ImageFormat $p)) -ne '' }

$files = Get-ChildItem (Join-Path $BatchesDir 'batch*.json') | Sort-Object Name
$ok=0; $skip=0; $fail=0; $nullc=0; $redl=0
foreach($f in $files){
  $arr = Get-Content $f.FullName -Raw -Encoding UTF8 | ConvertFrom-Json
  foreach($e in $arr){
    $name = $e.name; if(-not $name){$name=$e.id}
    $safe = ($name -replace '[\\/:*?"<>|]','_').Trim()   # 非法字符→_，如「白子*恐怖」→「白子_恐怖」
    if(-not $e.img){ $nullc++; continue }
    # 跳过检查：该名字任意扩展名的有效文件已存在则跳过（扩展名按实际内容定，不靠 URL 后缀）
    $existing = @('png','jpg','webp') | ForEach-Object {
      $p = Join-Path $dest ($safe + '.' + $_)
      if((Test-Path -LiteralPath $p) -and (Test-ValidImage $p)){ $p }
    }
    if($existing){ $skip++; continue }   # 断点续传：已下且有效则跳过
    # 删除同名损坏文件（任意扩展名）
    @('png','jpg','webp') | ForEach-Object {
      $p = Join-Path $dest ($safe + '.' + $_)
      if(Test-Path -LiteralPath $p){ Remove-Item -LiteralPath $p -Force; $redl++ }
    }
    # 下载到临时文件，按实际内容决定扩展名（CDN 内容协商可能返回与 URL 后缀不同的格式，见约束 7）
    $tmp = Join-Path $dest ($safe + '.tmp')
    try{
      # ★ 下载重试：CDN 偶发超时（见故障排查表），最多 3 次线性退避
      $dlOk = $false
      for($dr=0; $dr -lt 3; $dr++){
        try{
          Invoke-WebRequest -Uri $e.img -Headers $headers -OutFile $tmp -UseBasicParsing -TimeoutSec 120 -ErrorAction Stop
          $dlOk = $true; break
        }catch{
          if($dr -eq 2){ throw }
          Start-Sleep -Milliseconds (1000 * ($dr + 1))
        }
      }
      $realFmt = Get-ImageFormat $tmp
      if($realFmt){
        # 用 Move-Item -Force（不用 Rename-Item）：实测 PS5.1 的 Rename-Item -Force 不覆盖已存在目标，
        # 而 Move-Item -Force 会覆盖。若前一步 Remove-Item 损坏文件失败导致同名残留，Move 仍能成功。
        # ★ Move 重试：杀软实时扫描可能锁定刚写完的 .tmp 句柄，最多 3 次退避（见故障排查表）
        $destPath = Join-Path $dest ($safe + '.' + $realFmt)
        $moved = $false
        for($mr=0; $mr -lt 3; $mr++){
          try{
            Move-Item -LiteralPath $tmp -Destination $destPath -Force -ErrorAction Stop
            $moved = $true; break
          }catch{
            if($mr -eq 2){ throw }
            Start-Sleep -Milliseconds (500 * ($mr + 1))
          }
        }
        if($moved){ $ok++ }
      } else {
        Remove-Item -LiteralPath $tmp -Force -ErrorAction SilentlyContinue
        $fail++; Write-Output "BAD  $safe (无法识别图片格式)"
      }
    }catch{ Remove-Item -LiteralPath $tmp -Force -ErrorAction SilentlyContinue; Write-Output "FAIL $safe $($_.Exception.Message)"; $fail++ }
  }
}
Write-Output "---"
Write-Output "OK=$ok SKIP=$skip NULL=$nullc FAIL=$fail REDL=$redl"
```

调用方式（`-Target` 只接受 `hydt` 或 `gfjs`，传错会直接报错；`-BatchesDir` 不传则默认 `{OutputDir}\batches`）：

```powershell
# 下回忆大厅
& "...\download.ps1" -OutputDir "E:\some\folder" -Target hydt
# 下官方介绍
& "...\download.ps1" -OutputDir "E:\some\folder" -Target gfjs
# batch json 在别处时显式指定
& "...\download.ps1" -OutputDir "E:\out" -Target hydt -BatchesDir "D:\manifests"
```

`target=both` 时连续调用两次（一次 `-Target hydt` 一次 `-Target gfjs`），分别落到 `{OutputDir}\回忆大厅\` 和 `{OutputDir}\官方介绍\` 两个子文件夹。断点续传特性：可反复重跑，已下载的有效文件自动跳过，单次超时（10 分钟）也不丢进度，下次接着来。

### 阶段 5 — 校验 + 重试 + 收尾

1. **完整性校验**：遍历子文件夹，每个文件读首 12 字节验头（`Test-ValidImage` → `Get-ImageFormat`），统计 PNG/JPG/WEBP/BAD 计数。
2. **文件数核对**：`{子文件夹文件数} == {成功抽取的角色数}`，`0 BAD`，`0 重名`（用 `Group-Object Name` 查重名）。
3. **「编辑中」角色核对**：阶段 2 返回的 JSON 里 `editing:true` 的角色，其回忆大厅图通常缺失（`img:null`，wiki 页面尚未完成编辑）——这是 wiki 数据状态，**不是 skill 失败**。`target=hydt` 时这些角色不会落盘文件（正常）；`target=gfjs` 时官方介绍图可能正常下到（文件名已剥离 `【编辑中】` 前缀）。把这些角色名单列给用户，提示「wiki 编辑中，回忆大厅图暂缺，可日后补下」。
4. **重跑 download.ps1**：因 `Test-Path`+`Test-ValidImage` 跳过有效文件，重跑只会补下 NULL/FAIL/损坏的，安全幂等。
5. **清理测试文件**：阶段验证用的 test PNG、页面截图等临时文件删掉。

## 已知数据特点（非 bug，不要误判为失败）

- `瞬（泳装）` 与 `雪玲（泳装）` 的回忆大厅图在 wiki 上**本就是同一张**（MD5 相同），各自保存一份符合预期。
- 联动 4 角色（初音未来 / 御坂美琴 / 食蜂操祈 / 佐天泪子）也有回忆大厅和官方介绍图，不要当特例排除。
- list 卡片名与 title 名可能不同：id=86656 卡片写「瞬(小)」，title 是「瞬（幼女）」，落盘按 title 为 `瞬（幼女）.png`（约束 6，更规范）。
- CDN 内容协商：个别资源 URL 后缀与实际响应格式不一致。实测 267 个里 8 个 png/jpg 后缀互换 + 1 个 webp（律 id=677904，URL `.jpg` 但响应 webp）+ 部分大写 `.JPG`。download.ps1 按 `Get-ImageFormat` 实际内容定扩展名，这些都会被正确识别并落成 `{名}.webp` / `{名}.jpg` 等，**不是失败**。
- 最小图可能仅 ~64 KB（如艾米临战，wiki 原图就小），不要按文件大小说它是失败的。
- **「编辑中」角色**（实测 272 个里前 5 个：千秋(泳装) id=714029、真琴(泳装) id=714033、皋月(泳装) id=714037、伊吹(泳装) id=714055、伊吕波(泳装) id=714062）：wiki 页面处于草稿态，`<title>` 为 `【编辑中】<角色名>_碧蓝档案...`。这类页面**回忆大厅图缺失**（`img.hydt` 不存在，返回 null，不是占位图），**官方介绍图可能正常存在**。阶段 2 已剥离 `【编辑中】` 前缀并标记 `editing:true`，落盘文件名不会带污染。`target=hydt` 时这 5 个角色不产出文件是预期行为，不是失败。

## 故障排查表

| 症状 | 原因 | 修复 |
|---|---|---|
| PowerShell 全部 FAIL「路径不存在」 | `.ps1` 没 BOM，PS 5.1 按 GBK 解析中文路径 | 给 `.ps1` 补 UTF-8 BOM（`EF BB BF`） |
| 单批 `browser_run_code_unsafe` MCP 超时 (~10min) | batch_size 太大，单页慢加载拖垮整批 | 先试 12 个/批；**仍超时降 6**，拆批存为 `batchNNa.json`/`batchNNb.json`，`batch*.json` glob 自动覆盖 |
| 部分角色 `img:null` | SPA 偶发未渲染子标签内容 | 收集起来批次结束后统一二次重试（goto+poll，可加 reload） |
| PowerShell 直请 content JSON 返回 567 | Tencent EdgeOne WAF 拦 api-cdn host | 别走 JSON 捷径，老老实实浏览器抽 img.src |
| 页面内 `fetch(api-cdn...)` 报 Failed to fetch | 同源/CORS 拦截 | 同上，不 fetch，用 DOM 读 img.src |
| 同一文件反复 BAD、重跑无法收敛（死循环） | CDN 内容协商返回 webp，旧版 `Test-ValidImage` 只认 PNG/JPG → 删除→重下→仍 webp→仍 BAD | 已修复：`Get-ImageFormat` 识别 WEBP（`RIFF...WEBP`），webp 文件判有效，存为 `.webp` |
| URL 是 `.png`/`.jpg` 但落盘成了 `.webp` 或后缀互换 | CDN 内容协商，URL 后缀不可信 | 已是预期行为，脚本按实际内容定扩展名，**非 bug** |
| 下载的图打不开/半截 | 下载中断 | 脚本 `Test-ValidImage` 自动识别并删除重下 |
| `.tmp file being used by another process`（Move-Item 失败） | 杀软（Windows Defender 等）实时扫描锁定刚写完的 `.tmp` 句柄 | 脚本已内置 3 次退避重试（500ms/1000ms/1500ms）；仍失败的可重跑 download.ps1（幂等，已下的跳过）；频繁出现可将输出目录加入杀软白名单 |
| CDN 图片下载偶发 `ERR_TIMED_OUT` / 超时 | gamekee CDN 偶发不稳 | 脚本已内置 3 次下载重试（1s/2s 线性退避）；重跑补下失败项 |
| `edit` 工具改 batch*.json 报 "No match found" | 该 JSON 被 PowerShell `Set-Content` 写过，文件头带 BOM 污染 | 用文件写入工具全量重写该 JSON（见阶段 3 约束）；切勿用 PowerShell 改 batch JSON |
| 部分角色回忆大厅图缺文件（`img:null`）且 title 含「编辑中」 | wiki 页面草稿态，回忆大厅图尚未上传 | 非失败；文件名已自动剥离 `【编辑中】` 前缀；把这些角色列入「待补下」，日后 wiki 编辑完成再重跑 |
