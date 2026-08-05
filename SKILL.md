---
name: gamekee-ba-download
description: 批量下载 gamekee.com 碧蓝档案(BA)图鉴「实装学生」全角色的「回忆大厅」「官方介绍」「角色立绘」图片到本地子文件夹。Use when the user wants to batch-download Blue Archive character images from gamekee wiki, or says 回忆大厅 / 官方介绍 / 角色立绘 图下载, 碧蓝档案图鉴抓图, gamekee ba 图片批量下载, 实装学生图片. Triggers: gamekee, 碧蓝档案, BA图鉴, 回忆大厅, 官方介绍, 角色立绘下载.
---

# GameKee 碧蓝档案图鉴 · 图片批量下载

从 `https://www.gamekee.com/ba/second/23941`（实装学生图鉴）批量下载每个角色详情页的指定类型图片。

## 参数（向用户确认缺失项，有默认值则用默认）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `target` | `hydt` | `hydt`=回忆大厅 ｜ `gfjs`=官方介绍 ｜ `lihun`=角色立绘 |
| `output_dir` | 当前工作目录 | 图片落盘的根目录 |
| `list_url` | `https://www.gamekee.com/ba/second/23941` | 角色花名册入口（实装学生） |

## target 获取方式映射表（核心，已实测验证）

| target 值 | 中文名 | 获取方式 | 形态 | 备注 |
|---|---|---|---|---|
| `hydt` | 回忆大厅 | `img.hydt` 选择器 | 单图 | `hydt`=回忆大厅拼音首字母，gamekee 专属 class，全局唯一 |
| `gfjs` | 官方介绍 | `.role-img-box img.item`（取第一个）选择器 | 单图 | `item` 是共享 class，必须取首个；该图在 DOM 中永远位于 `img.hydt` 之前 |
| `lihun` | 角色立绘 | `button.action-item` 文本匹配「切换立绘」「下载图片」 | 单图 | ★ **不走选择器/URL 路线**：点击「切换立绘」→ 点击「下载图片」→ Playwright MCP 自动下载到 `.playwright-mcp/`；按键数 **3-5 个因角色而异**（实测：日奈 5 键、泳装 4 键、礼服 3 键），**必须按文本匹配，不可按顺序索引** |

> 未来扩展「设定集-日文/繁中」「本家画」「表情」需走图集模式（按 `.header-container` 文本切块取该节区全部 img），暂不覆盖。

## 关键约束（踩坑实证，不可违反）

1. **MCP 浏览器进程完全沙箱化**：`browser_run_code_unsafe` 里没有 `require`/`process`/`global`，不能 `import('node:fs')`，`navigator.clipboard` 是 undefined。**数据通道有两条**：(a) 该函数的 return 字符串（hydt/gfjs 用——返回 URL JSON → PowerShell 下载）；(b) Playwright MCP 的浏览器下载自动保存到 `.playwright-mcp/`（lihun 用——点击下载按钮后图片自动落到此目录，agent 再移动改名）。
2. **content JSON 走不通捷径**（hydt/gfjs 路线）：`api-cdn.gamekee.com/wiki2.0/pro/829/content/{id}.json` 被 Tencent EdgeOne WAF 拦（PowerShell 直请返回 567；页面内 `fetch` 被 CORS 拦）。但**图片 host `cdnimg-v2.gamekee.com` 不拦**——只能「浏览器跑 JS 拿 img.src → PowerShell 下载图」，无法走「读 API JSON 解析图 URL」。（lihun 路线不涉及此问题——立绘图是页面 JS 生成的 blob，不从 CDN 拉。）
3. **PowerShell 5.1 编码**：含中文字面量（路径名、`回忆大厅`）的 `.ps1` 脚本**必须存成 UTF-8 with BOM**，否则 PS 5.1 按 GBK 解析，中文路径变乱码，所有下载报「路径不存在」。用 `[IO.File]::ReadAllBytes` 检测后 prepend `EF BB BF` 可补 BOM。
4. **批大小默认 12，仍超时降 6**：实测 15 个角色/批会因单个页面慢加载把单次 MCP 调用拖到 10 分钟超时。12 是大多数时段稳定上限。**若 12 仍触发 MCP 超时**（服务端负载时段波动会导致），降为 6 个/批；拆批结果存为 `batchNNa.json` / `batchNNb.json`，download.ps1 的 `batch*.json` 通配符自动覆盖，无需改脚本。（hydt/gfjs 路线适用；lihun 路线每批约 1.5 分钟，不涉及此问题。）
5. **图片 URL 清洗**：页面 `img.src` 含 `?x-image-process=.../format,webp` 转换参数，**必须 `src.split('?')[0]`** 去掉转换参数；头部补 `https:`（页面里是 `//cdnimg-v2...` 协议相对 URL）。**注意：`split('?')[0]` 不保证拿到原始格式**——CDN 对个别资源做内容协商（content negotiation），无论 URL 带不带 webp 参数，响应体本身可能是 webp（实测 id=677904 律的 URL 是 `.jpg` 但响应是 webp）。脚本必须兼容 PNG/JPG/WEBP 三种格式，不能假设「去参数 = 原始 PNG/JPG」。
6. **文件名取自 `<title>` 而非 list 页卡片名**：`page.title().split('_碧蓝档案')[0]`。list 页用半角括号，title 用全角括号，后者更规范。例：id=86656 list 卡片名「瞬(小)」，title 是「瞬（幼女）」，落盘为 `瞬（幼女）.png`。
7. **扩展名按下载后实际文件头决定，不按 URL 后缀**（hydt/gfjs 路线）：CDN 内容协商会导致 URL 后缀与实际格式不一致（实测 272 个里 8 个 png/jpg 后缀互换 + 1 个 webp，共 9 个错配；另有 URL 大写 `.JPG` 的情况）。download.ps1 下载到临时文件后读首 12 字节判定真实格式（`Get-ImageFormat`），再改名成正确扩展名，**完全不依赖 URL 后缀**。（lihun 路线下载的总是 PNG，不涉及此问题。）

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

### 阶段 2 — 逐角色抽取图片（按 target 切换方式）

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
- 返回的 JSON 中 `img:null` 的角色（偶发未渲染）收集起来，全部批次跑完后**统一二次重试**（同样的 goto+poll，可加一次 reload 兜底）。实测约 5/272 会偶发 null，重试基本都能成功。
- 角色名含 `*` 等非法字符（如「白子*恐怖」）由阶段 4 脚本统一转义，这里原样保留。

#### `target=lihun` 专用模板（与 hydt/gfjs 路线完全不同）

lihun（角色立绘）**不走 URL 提取 + PowerShell 下载**路线——立绘图是页面 JS 实时生成的 blob，不是 CDN 静态图。必须「点击切换立绘 → 点击下载图片 → Playwright 自动下载」，再由 agent 移动改名。模板：

```js
async (page) => {
  const ids = ["ID1","ID2",...,"ID12"];
  const out = [];
  for (const id of ids) {
    const rec = {id, name:null, downloadFile:null, err:null};
    try {
      // ★ lihun 不加 ?tab=3，留在默认「角色技能」页（立绘按钮在此页）
      const u = 'https://www.gamekee.com/ba/tj/'+id+'.html';
      await page.goto(u,{waitUntil:'domcontentloaded',timeout:30000});
      await page.waitForTimeout(3500);  // 等 SPA 渲染按钮
      // ★ 文本匹配点「切换立绘」（按键 3-5 个因角色而异，不可按顺序索引）
      //   若 btnCount=0（首次导航冷缓存，SPA 未渲染），reload + 等 5s 重试一次
      let clicked = await page.evaluate(() => {
        const btns = Array.from(document.querySelectorAll('button.action-item'));
        const lihun = btns.find(b => (b.textContent||'').trim() === '切换立绘');
        if(lihun){ lihun.click(); return true; }
        return false;
      });
      if(!clicked){
        await page.reload({waitUntil:'domcontentloaded',timeout:30000});
        await page.waitForTimeout(5000);
        await page.evaluate(() => {
          const btns = Array.from(document.querySelectorAll('button.action-item'));
          const lihun = btns.find(b => (b.textContent||'').trim() === '切换立绘');
          if(lihun) lihun.click();
        });
      }
      await page.waitForTimeout(2500);  // 等立绘渲染
      // ★ hook a.click 捕获 download 文件名，然后点「下载图片」
      await page.evaluate(() => {
        window.__dlFile = null;
        const origClick = HTMLAnchorElement.prototype.click;
        HTMLAnchorElement.prototype.click = function(){
          if(this.download){ window.__dlFile = this.download; }
          HTMLAnchorElement.prototype.click = origClick;  // 用完即还原
          return origClick.call(this);
        };
        const btns = Array.from(document.querySelectorAll('button.action-item'));
        const dl = btns.find(b => (b.textContent||'').trim() === '下载图片');
        if(dl) dl.click();
      });
      await page.waitForTimeout(2000);  // 等下载完成（Playwright MCP 自动存到 .playwright-mcp/）
      const dlFile = await page.evaluate(() => window.__dlFile);
      const title = await page.title();
      let name = title.split('_碧蓝档案')[0];
      // 「编辑中」检测与剥离（同 hydt/gfjs 模板）
      if(name.startsWith('【编辑中】')){
        rec.editing = true;
        name = name.replace(/^【编辑中】/, '').trim();
      }
      rec.name = name;
      rec.downloadFile = dlFile;  // 如 "291700.png"（数字美术资源 ID，不是角色名）
    } catch(e) { rec.err = e.message; }
    out.push(rec);
  }
  return JSON.stringify(out);
}
```

要点（lihun 专属）：
- 每个角色约 8s（导航 3.5s + 切换立绘 2.5s + 下载 2s），12 个/批约 1.5 分钟，不超时。
- **每批结束后立即移走 `.playwright-mcp/` 里的文件**（见阶段 3 lihun 落盘说明），避免下批文件混淆。移走后 agent 在新会话或下批开始前清理 `.playwright-mcp/` 残留。
- `downloadFile` 是数字 ID 文件名（如 `291700.png`），**不是角色名**——agent 必须按 `name` 字段改名落盘。
- `downloadFile:null` 表示下载按钮没触发或 hook 失败，收集起来批次结束后统一二次重试。
- 「编辑中」角色的立绘按钮**通常正常存在**（实测 id=714029 千秋泳装有 4 个按钮且成功下载立绘），与回忆大厅图缺失不同——不要因 `editing:true` 就跳过 lihun 下载。

### 阶段 3 — 浏览器→文件系统桥

每批返回的 JSON 写入 `batches/batchNN.json` 文件（每文件一批，NN 从 01 起）。结构因 target 而异：

hydt/gfjs 路线：
```json
[
  {"id":"59934","name":"日奈","img":"https://cdnimg-v2.gamekee.com/.../674225.png"},
  {"id":"714029","name":"千秋(泳装)","img":null,"editing":true},
  ...
]
```

lihun 路线：
```json
[
  {"id":"59934","name":"日奈","downloadFile":"291700.png"},
  {"id":"714029","name":"千秋(泳装)","downloadFile":"41023487ms66n3ox.png","editing":true},
  ...
]
```

> **⚠ batch*.json 一律用文件写入工具（Write / write 等）生成或全量重写，不要用 PowerShell 的 `Set-Content`/`Out-File` 修改这些 JSON。** PS 5.1 的 `-Encoding UTF8` 默认带 BOM，BOM 会污染文件头，导致后续基于文本匹配的编辑工具（edit 等）匹配失败报 "No match found"。如需修改 batch JSON，用写入工具整文件重写。PowerShell 的 `-replace` 在复杂 JSON（含引号/反斜杠/Unicode）上转义也极脆弱，切勿用来批量改 JSON。

### 阶段 4 — PowerShell 批量下载（脚本模板）

在 `output_dir` 下建子文件夹（按 target 分）：
- `hydt` → `{output_dir}/回忆大厅/`
- `gfjs` → `{output_dir}/官方介绍/`

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

断点续传特性：可反复重跑，已下载的有效文件自动跳过，单次超时（10 分钟）也不丢进度，下次接着来。

#### `target=lihun` 的落盘（不走 download.ps1）

lihun 的图片已在阶段 2 由 Playwright MCP 自动下载到 `.playwright-mcp/` 目录，**不跑 download.ps1**。落盘流程由 agent 直接执行：

1. 在 `output_dir` 下建子文件夹 `{output_dir}/角色立绘/`
2. 逐个读取阶段 2 返回的 JSON 记录，对每条 `{name, downloadFile}`：
   - 源文件：`.playwright-mcp/{downloadFile}`（如 `.playwright-mcp/291700.png`）
   - 目标文件：`{output_dir}/角色立绘/{name}.png`（`name` 已剥离 `【编辑中】` 前缀、转义非法字符）
   - 用文件移动/拷贝工具把源文件移到目标路径并改名
3. `downloadFile:null` 的角色（下载失败/按钮缺失）跳过，收集起来统一重试
4. 每批处理完后清理 `.playwright-mcp/` 里的残留文件，避免下批混淆

断点续传：目标路径已存在有效 PNG 则跳过（读首 4 字节验 `89 50 4E 47`）。

### 阶段 5 — 校验 + 重试 + 收尾

1. **完整性校验**：遍历子文件夹，每个文件读首 12 字节验头（`Test-ValidImage` → `Get-ImageFormat`），统计 PNG/JPG/WEBP/BAD 计数。
2. **文件数核对**：`{子文件夹文件数} == {成功抽取的角色数}`，`0 BAD`，`0 重名`（用 `Group-Object Name` 查重名）。
3. **「编辑中」角色核对**：阶段 2 返回的 JSON 里 `editing:true` 的角色，其回忆大厅图通常缺失（`img:null`，wiki 页面尚未完成编辑）——这是 wiki 数据状态，**不是 skill 失败**。`target=hydt` 时这些角色不会落盘文件（正常）；`target=gfjs` 时官方介绍图可能正常下到；`target=lihun` 时立绘图通常正常存在（实测编辑中角色 714029 有立绘）。把这些角色名单列给用户，提示「wiki 编辑中，回忆大厅图暂缺，可日后补下」。
4. **重跑 download.ps1**：因 `Test-Path`+`Test-ValidImage` 跳过有效文件，重跑只会补下 NULL/FAIL/损坏的，安全幂等。
5. **清理测试文件**：阶段验证用的 test PNG、页面截图等临时文件删掉。

## 已知数据特点（非 bug，不要误判为失败）

- `瞬（泳装）` 与 `雪玲（泳装）` 的回忆大厅图在 wiki 上**本就是同一张**（MD5 相同），各自保存一份符合预期。
- 联动 4 角色（初音未来 / 御坂美琴 / 食蜂操祈 / 佐天泪子）也有回忆大厅和官方介绍图，不要当特例排除。
- list 卡片名与 title 名可能不同：id=86656 卡片写「瞬(小)」，title 是「瞬（幼女）」，落盘按 title 为 `瞬（幼女）.png`（约束 6，更规范）。
- CDN 内容协商：个别资源 URL 后缀与实际响应格式不一致。实测 272 个里 8 个 png/jpg 后缀互换 + 1 个 webp（律 id=677904，URL `.jpg` 但响应 webp）+ 部分大写 `.JPG`。download.ps1 按 `Get-ImageFormat` 实际内容定扩展名，这些都会被正确识别并落成 `{名}.webp` / `{名}.jpg` 等，**不是失败**。
- 最小图可能仅 ~64 KB（如艾米临战，wiki 原图就小），不要按文件大小说它是失败的。
- **「编辑中」角色**（实测 272 个里前 5 个：千秋(泳装) id=714029、真琴(泳装) id=714033、皋月(泳装) id=714037、伊吹(泳装) id=714055、伊吕波(泳装) id=714062）：wiki 页面处于草稿态，`<title>` 为 `【编辑中】<角色名>_碧蓝档案...`。这类页面**回忆大厅图缺失**（`img.hydt` 不存在，返回 null，不是占位图），**官方介绍图可能正常存在**。阶段 2 已剥离 `【编辑中】` 前缀并标记 `editing:true`，落盘文件名不会带污染。`target=hydt` 时这 5 个角色不产出文件是预期行为，不是失败。
- **lihun 下载文件名是数字 ID**（如 `291700.png`），不是角色名——这是 gamekee 美术资源 ID。agent 必须按 `name` 字段（从 `<title>` 取）改名落盘。实测日奈 id=59934 下载文件名 `291700.png`，agent 改名为 `日奈.png`。
- **lihun 按键文本跨角色一致**：实测日奈/日奈泳装/日奈礼服 3 个角色，「切换立绘」和「下载图片」两个按钮的文本完全相同（`button.action-item`，textContent 精确匹配）。但**按键总数 3-5 个因角色而异**（日奈 5 键含「切换时装」「切换动作」、泳装 4 键、礼服 3 键），必须用 `.find(b => textContent === '切换立绘')` 文本匹配，**不可按索引**（如 `btns[2]`）定位。

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
| lihun 模式 `downloadFile:null` | 下载按钮未触发 / hook 失败 / SPA 冷缓存未渲染 | 收集起来批次结束后统一二次重试（重新 goto+切换立绘+下载）；编辑中角色也有立绘，不要因 `editing:true` 跳过 |
| lihun 下载的文件在 `.playwright-mcp/` 找不到 | Playwright MCP 下载目录配置改变 / 文件被杀软拦截 | 检查 `.playwright-mcp/` 目录是否存在且有新文件；确认 Playwright MCP 的 `--browser` 进程未崩溃 |
| lihun 批次间文件混淆 | 上一批的 `.playwright-mcp/` 文件未清理 | **每批处理完后立即移走并清理 `.playwright-mcp/`**，见阶段 3 lihun 落盘说明 |
