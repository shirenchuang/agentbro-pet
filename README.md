# AgentBroPet

[中文](#中文) | [English](#english)

## 中文

AgentBroPet 是一个面向 [AgentBro](https://github.com/shirenchuang/agentbro) 和 OpenAI Codex Desktop 宠物体系的 Codex Skill。

它可以把一个角色概念、品牌线索、参考图或已有生成图，制作成可在 AgentBro / Codex 中使用的动态宠物包：

```text
pet.json
spritesheet.webp
```

生成结果遵循 [AgentBro 宠物市场](https://www.agentbro.net/pets) 使用的同一套格式：透明背景、8 x 9 动画图集、每格 `192x208`，并包含 AgentBro/Codex 标准状态。

### 相关链接

- AgentBro 开源项目：[github.com/shirenchuang/agentbro](https://github.com/shirenchuang/agentbro)
- 作者开源主页：[github.com/shirenchuang](https://github.com/shirenchuang)
- AgentBro 宠物市场：[agentbro.net/pets](https://www.agentbro.net/pets)
- 当前 Skill 仓库：[github.com/shirenchuang/agentbro-pet](https://github.com/shirenchuang/agentbro-pet)

### 宠物市场

在 AgentBro 宠物市场里可以浏览、安装、分享社区宠物：

![AgentBro pet market desktop view](assets/agentbro-pets-market-desktop.png)

![AgentBro pet market full page](assets/agentbro-pets-market-web.png)

### 这个 Skill 做什么

AgentBroPet 保留了 `hatch-pet` 的确定性宠物图集流程，但不再强绑定 Codex 原生 `$imagegen`。

它会优先寻找当前 Agent 环境里可用的图像生成能力，例如：

- Codex `$imagegen`
- 其他已安装的生图 Skill
- OpenAI 图像 CLI/API，例如 `gpt-image-2`
- Nano Banana / Gemini / Flux / ComfyUI / Midjourney 等外部工作流
- 手动外部后端：由另一个 Agent 或模型生成 PNG，再把本地路径交回流程

核心思路是：**生图后端可以替换，但图集组装、验证和 QA 保持确定性。**

### 安装这个 Skill

把仓库克隆到 Codex skills 目录：

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/shirenchuang/agentbro-pet.git ~/.codex/skills/agentbro-pet
```

然后在 Codex 里调用：

```text
$AgentBroPet create a Teemo-inspired scout pet for AgentBro
```

非 Codex Agent 也可以使用这个目录作为工作流包：只需要能运行脚本、读取 `imagegen-jobs.json`、生成需要的图片，并把选中的输出复制到 `decoded/` 路径。

### 用 npx 安装宠物市场里的宠物

AgentBro 宠物市场由 [`abpets`](https://www.npmjs.com/package/abpets) CLI 驱动。你不需要全局安装，直接用 `npx`：

```bash
# 推荐：使用完整作者/宠物名
npx abpets install shirenchuang/teemo

# 如果 slug 不冲突，也可以只写宠物名
npx abpets install teemo
```

`abpets` 会把宠物同时安装到：

```text
~/.codex/pets/<slug>/
~/.agentbro/pets/<slug>/
```

所以同一个宠物可以被 Codex Desktop 和 AgentBro 里的 Claude Code、Codex CLI、Cursor、GitHub Copilot、Cline、Gemini CLI 等 Agent 使用。

常用命令：

```bash
# 搜索市场宠物
npx abpets search
npx abpets search teemo

# 查看本机已安装宠物
npx abpets list

# 卸载宠物
npx abpets uninstall teemo

# 登录并提交自己制作的宠物
npx abpets login
npx abpets submit ~/.codex/pets/my-pet
```

要求：Node.js 18 或更高版本。

### 输出格式

最终宠物包包含：

```text
<pet-id>/
├── pet.json
└── spritesheet.webp
```

图集规格：

- `1536x1872`
- 8 列 x 9 行
- 每格 `192x208`
- 透明背景
- 未使用格子必须完全透明
- WebP 或 PNG 图集

动画行：

```text
0 idle
1 running-right
2 running-left
3 waving
4 jumping
5 failed
6 waiting
7 running
8 review
```

### 基础工作流

准备一次宠物生成任务：

```bash
SKILL_DIR="$HOME/.codex/skills/agentbro-pet"

python3 "$SKILL_DIR/scripts/prepare_pet_run.py" \
  --pet-name "My Pet" \
  --pet-notes "a tiny friendly coding companion" \
  --output-dir ./output/agentbro-pet/my-pet \
  --style-preset auto \
  --force
```

查看待生成图片任务：

```bash
jq '.jobs[] | {id, status, depends_on, prompt_file, input_images, output_path}' \
  ./output/agentbro-pet/my-pet/imagegen-jobs.json
```

使用当前可用的图像后端生成每个 ready job，把选中的结果复制到 job 的 `output_path`，再在 `imagegen-jobs.json` 中标记完成。

所有图片任务完成后，运行确定性图集流程：

```bash
RUN_DIR=./output/agentbro-pet/my-pet

python3 "$SKILL_DIR/scripts/extract_strip_frames.py" \
  --decoded-dir "$RUN_DIR/decoded" \
  --output-dir "$RUN_DIR/frames" \
  --states all \
  --method auto

python3 "$SKILL_DIR/scripts/inspect_frames.py" \
  --frames-root "$RUN_DIR/frames" \
  --json-out "$RUN_DIR/qa/review.json" \
  --require-components

python3 "$SKILL_DIR/scripts/compose_atlas.py" \
  --frames-root "$RUN_DIR/frames" \
  --output "$RUN_DIR/final/spritesheet.png" \
  --webp-output "$RUN_DIR/final/spritesheet.webp"

python3 "$SKILL_DIR/scripts/validate_atlas.py" \
  "$RUN_DIR/final/spritesheet.webp" \
  --json-out "$RUN_DIR/final/validation.json"

python3 "$SKILL_DIR/scripts/make_contact_sheet.py" \
  "$RUN_DIR/final/spritesheet.webp" \
  --output "$RUN_DIR/qa/contact-sheet.png"

python3 "$SKILL_DIR/scripts/render_animation_previews.py" \
  --frames-root "$RUN_DIR/frames" \
  --output-dir "$RUN_DIR/qa/previews"
```

### 后端契约

每个图像后端都必须：

- 读取每个 job 的 prompt 文件
- 在支持参考图时使用 job 中列出的 `input_images`
- 为 base 或 row strip 生成一个选中的 PNG
- 返回本地 `selected_source` 路径
- 不得把 layout guide 当作最终输出
- 不得把蓝色安全框、中心线、标签或引导网格画进最终 row
- 保持所有行中的宠物身份一致

`imagegen-jobs.json` 保留历史命名，但在 AgentBroPet 中它表示「视觉生成任务」，并不绑定 Codex `$imagegen`。

### 质量检查

不要接受未通过以下检查的宠物：

- `qa/review.json` 没有错误
- `final/validation.json` 通过
- 已视觉检查 `qa/contact-sheet.png`
- 已视觉检查 `qa/previews/*.gif`
- 所有行保持相同宠物身份、风格、色板、轮廓和道具
- 方向行动画朝向正确
- idle 不是视觉静止
- 没有引导线、分离特效、白色格子背景、裁切 sprite 或明显尺寸跳动

## English

AgentBroPet is a Codex skill for creating animated pets that work with [AgentBro](https://github.com/shirenchuang/agentbro) and OpenAI Codex Desktop.

It turns a concept, brand cue, reference image, or existing generated artwork into an app-ready pet package:

```text
pet.json
spritesheet.webp
```

Generated pets follow the same format used by the [AgentBro Pet Market](https://www.agentbro.net/pets): a transparent 8 x 9 animation atlas with `192x208` cells and the standard AgentBro/Codex states.

### Links

- AgentBro open-source project: [github.com/shirenchuang/agentbro](https://github.com/shirenchuang/agentbro)
- Author / open-source home: [github.com/shirenchuang](https://github.com/shirenchuang)
- AgentBro Pet Market: [agentbro.net/pets](https://www.agentbro.net/pets)
- This skill repo: [github.com/shirenchuang/agentbro-pet](https://github.com/shirenchuang/agentbro-pet)

### Pet Market

Browse, install, and share community pets in the AgentBro pet market:

![AgentBro pet market desktop view](assets/agentbro-pets-market-desktop.png)

![AgentBro pet market full page](assets/agentbro-pets-market-web.png)

### What This Skill Does

AgentBroPet keeps the deterministic hatch-pet pipeline, but removes the hard dependency on Codex's native `$imagegen` skill. It can use whatever image-generation backend is available in the current agent environment:

- Codex `$imagegen`
- another installed image-generation skill
- OpenAI image CLI/API such as `gpt-image-2`
- Nano Banana / Gemini / Flux / ComfyUI / Midjourney-style external workflows
- a manual backend where another agent or model generates the PNGs and returns local file paths

The key idea is simple: image generation is replaceable, but atlas assembly and QA stay deterministic.

### Install This Skill

Clone this repo into your Codex skills folder:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/shirenchuang/agentbro-pet.git ~/.codex/skills/agentbro-pet
```

Then invoke the skill in Codex:

```text
$AgentBroPet create a Teemo-inspired scout pet for AgentBro
```

For non-Codex agents, use the same folder as a workflow package. The agent only needs to run the scripts, read `imagegen-jobs.json`, generate the requested images, and copy selected outputs into the decoded paths.

### Install Market Pets With npx

The AgentBro pet market is powered by the [`abpets`](https://www.npmjs.com/package/abpets) CLI. You do not need a global install; use `npx`:

```bash
# Recommended: full author/pet reference
npx abpets install shirenchuang/teemo

# Bare slug works when it is not ambiguous
npx abpets install teemo
```

`abpets` installs each pet into both:

```text
~/.codex/pets/<slug>/
~/.agentbro/pets/<slug>/
```

That makes the same pet available to Codex Desktop and AgentBro-hosted agents such as Claude Code, Codex CLI, Cursor, GitHub Copilot, Cline, and Gemini CLI.

Useful commands:

```bash
# Search the market
npx abpets search
npx abpets search teemo

# List locally installed pets
npx abpets list

# Uninstall a pet
npx abpets uninstall teemo

# Log in and submit your own pet
npx abpets login
npx abpets submit ~/.codex/pets/my-pet
```

Requirement: Node.js 18 or newer.

### Output Format

Final pet packages contain:

```text
<pet-id>/
├── pet.json
└── spritesheet.webp
```

The atlas contract:

- `1536x1872` pixels
- 8 columns x 9 rows
- `192x208` pixels per cell
- transparent background
- unused cells fully transparent
- WebP or PNG atlas

Animation rows:

```text
0 idle
1 running-right
2 running-left
3 waving
4 jumping
5 failed
6 waiting
7 running
8 review
```

### Basic Workflow

Prepare a pet run:

```bash
SKILL_DIR="$HOME/.codex/skills/agentbro-pet"

python3 "$SKILL_DIR/scripts/prepare_pet_run.py" \
  --pet-name "My Pet" \
  --pet-notes "a tiny friendly coding companion" \
  --output-dir ./output/agentbro-pet/my-pet \
  --style-preset auto \
  --force
```

Inspect ready visual jobs:

```bash
jq '.jobs[] | {id, status, depends_on, prompt_file, input_images, output_path}' \
  ./output/agentbro-pet/my-pet/imagegen-jobs.json
```

Generate each ready job with your available image backend, copy the selected result into the job's `output_path`, then mark the job complete in `imagegen-jobs.json`.

When all jobs are complete, run the deterministic pipeline:

```bash
RUN_DIR=./output/agentbro-pet/my-pet

python3 "$SKILL_DIR/scripts/extract_strip_frames.py" \
  --decoded-dir "$RUN_DIR/decoded" \
  --output-dir "$RUN_DIR/frames" \
  --states all \
  --method auto

python3 "$SKILL_DIR/scripts/inspect_frames.py" \
  --frames-root "$RUN_DIR/frames" \
  --json-out "$RUN_DIR/qa/review.json" \
  --require-components

python3 "$SKILL_DIR/scripts/compose_atlas.py" \
  --frames-root "$RUN_DIR/frames" \
  --output "$RUN_DIR/final/spritesheet.png" \
  --webp-output "$RUN_DIR/final/spritesheet.webp"

python3 "$SKILL_DIR/scripts/validate_atlas.py" \
  "$RUN_DIR/final/spritesheet.webp" \
  --json-out "$RUN_DIR/final/validation.json"

python3 "$SKILL_DIR/scripts/make_contact_sheet.py" \
  "$RUN_DIR/final/spritesheet.webp" \
  --output "$RUN_DIR/qa/contact-sheet.png"

python3 "$SKILL_DIR/scripts/render_animation_previews.py" \
  --frames-root "$RUN_DIR/frames" \
  --output-dir "$RUN_DIR/qa/previews"
```

### Backend Contract

Every image backend must:

- read each job's prompt file
- use the listed input images whenever references are supported
- generate one selected PNG for the base or row strip
- return a local `selected_source` path
- never use layout guides as final output
- never copy guide boxes, blue safety frames, labels, or center lines into generated rows
- keep the pet identity consistent across all rows

`imagegen-jobs.json` keeps its historical name, but in AgentBroPet it means "visual generation jobs"; it is not tied to Codex `$imagegen`.

### Quality Gates

Do not accept a pet until:

- `qa/review.json` has no errors
- `final/validation.json` passes
- `qa/contact-sheet.png` has been visually checked
- `qa/previews/*.gif` have been visually checked
- all rows preserve the same pet identity, style, palette, silhouette, and props
- directional rows face the correct direction
- idle is not visually inert
- no row contains guide marks, detached effects, white cell backgrounds, cropped sprites, or obvious size popping

## License

Apache-2.0. See [LICENSE.txt](LICENSE.txt).
