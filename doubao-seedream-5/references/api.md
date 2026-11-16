# 官方 API 对照

核对日期：2026-11-16。仅针对 `doubao-seedream-5-5-261028`。

来源：[图片生成 API](https://console.volcengine.com/ark/region:cn-beijing/docs/image-generation-api)、[官方 Markdown](https://ark.volcengine.com/region:cn-beijing/docs/image-generation-api.md)。仅采用 Seedream 5.5 对应条目，不沿用其他型号限制。

## 请求协议

`POST https://ark.cn-beijing.volces.com/api/v3/images/generations`

头部为 `Authorization: Bearer <SEEDREAM_API_KEY>`、`Content-Type: application/json`。CLI不允许覆盖端点或模型，不自动重试POST。

| 字段 | 支持的值与含义 | CLI |
|---|---|---|
| model | 固定 doubao-seedream-5-5-261028 | 自动填入，拒绝其他值 |
| prompt | 普通生成/编辑必填；拆图可省略；建议中文≤300字、英文≤600词，非硬性截断限制 | --prompt |
| image | 单个字符串或数组；公开URL / data:image/小写格式;base64,... | --image，可重复 |
| layer_decomposition | boolean，默认false；true返回底图与最多99个透明层 | layers命令置true |
| layer_decomposition_options | 仅拆图生效；max_images 为 2–100 整数（含底图），默认30；text_recognition 为布尔值，默认false | --max-images / --text-recognition / --no-text-recognition |
| size | 普通默认2K；拆图默认auto，详见下表 | --size |
| optimize_prompt_options | 仅支持 {"mode":"standard"}，不支持fast | --optimize |
| output_format | png默认 / jpeg；拆图只控制底图格式，图层固定PNG | --output-format |
| background | auto默认 / opaque / transparent；透明与JPEG输出冲突；拆图禁止传入此字段 | --background |
| response_format | url默认，24小时有效 / b64_json | --response-format |
| watermark | boolean，默认true，右下角“AI 生成”水印 | --watermark / --no-watermark |

完整JSON使用 `--request`；仍校验模式和所有字段，未知字段直接拒绝。Seedream 5.5不支持 `sequential_image_generation`（包括disabled）、`sequential_image_generation_options`、`tools` 或 `stream`。

`watermark=false` 控制本次是否添加水印，不保证消除参考图中已有的水印。

拆图按 `max_images` 预扣 IPM，实际生成不足时释放多扣额度；它是上限，不保证实际张数。CLI 仅在 layers 模式接受拆图配置，普通模式携带配置会报错，避免参数无效却被误认为已生效。

## 尺寸

| 模式 | 可选值 | 约束 |
|---|---|---|
| 普通生成/编辑 | 1K、1.5K、2K、WIDTHxHEIGHT | 精确像素总数921600–4624220；宽高比1/16–16 |
| 图层拆分 | auto、1K、1.5K、2K | 不接受精确尺寸；底图保持输入比例 |

普通模式按官方建议采用“档位 + prompt比例描述”，需要确切尺寸时指定WIDTHxHEIGHT。档位示例（不是完整列表或比例保证）：

| 档位 | 1:1 | 16:9 | 9:16 |
|---|---|---|---|
| 1K | 1024×1024 | 1424×800 | 800×1424 |
| 1.5K | 1536×1536 | 2048×1152 | 1152×2048 |
| 2K | 2048×2048 | 2816×1584 | 1584×2816 |

费用以当前模型定价为准，不沿用其他型号的价格说明。

## 输入图片

| 模式 | 数量 | 格式 | 每图限制 |
|---|---|---|---|
| 文生图 | 0 | — | — |
| 图生图/多图参考 | 1–10 | JPEG、PNG、WebP、BMP、TIFF、GIF、HEIC、HEIF | ≤30MB；宽高均>14；像素196–36000000；比例1/16–16 |
| 拆图 | 1 | JPEG、PNG | ≤30MB；像素262144–36000000；比例1/16–16 |

5.5 原生透明图支持无参考图的文生图，或最多 10 张参考图；输入无需透明通道，可用 JPEG。`generate --image` 用于参考创作，`edit --image` 用于保留原图的编辑，二者使用同一官方生成协议。脚本检查本地文件和Data URI的真实解码结果；URL原样交给服务端，不假称已检查。HEIC/HEIF本地解码需要可选 `pillow-heif` 插件，安装后自动注册。所有本地图保持原始内容，调整尺寸或转码需明确作为额外处理。

## 响应与完整性

响应包含 `model`、`created`、`data[]`、`usage`。保留完整JSON及未知字段，不把HTTP200当作业务成功。普通响应一张图片；拆图包含底图及所有图层。

| 字段 | 含义 |
|---|---|
| url / b64_json | 可下载URL或Base64图片内容 |
| size、output_format | 实际尺寸与格式；脚本对照真实文件校验 |
| background | 实际透明度类型 transparent / opaque，可能与请求值不同；原样保留并检查透明产物的 Alpha 通道 |
| z_index | 底图为0，图层1–99，较大值覆盖较小值 |
| bounding_box.absolute | [left, top, right, bottom]，相对于输出底图画布的像素坐标 |
| bounding_box.normalized | 同上顺序，相对输出底图归一化到0–1000 |
| name、description | 图层名称和说明，保存为元数据，不作为文件路径 |
| text_recognition | 开启拆图 OCR 且为规整文字时返回；含 text、lines、行 bounding_box、runs[].text/style；完整保留至 response.json 和 manifest 元数据 |
| usage | input_images、generated_images、output_tokens、total_tokens；原样保留 |

OCR 的 style 可包含 similar_font_family、foreground_color、bold、italic、underline、strikethrough、shadow、translucent。非文字图层、艺术字、手写体不返回 OCR；字体范围见[官方支持字体](https://ark.volcengine.com/region:cn-beijing/docs/ocr-supported-fonts)。

图层恢复先缩放到 `(right-left, bottom-top)`，再在 `(left,top)` 叠加。不能把图层原图尺寸当成画布展示尺寸。

拆图会生成独立元素并补全遮挡区域，不保证按层重组后与输入逐像素一致。交付前仍应查看底图、各层及重组效果。

服务端拆图中任意图层失败，整个请求报错，没有部分服务端成功语义。本地下载可以部分完成：`manifest.complete=false`、非零退出码，原响应和完成文件仍保留，使用download恢复。

脚本本地保护值：JSON响应/文件最大512MB、单个输出文件最大64MB，不是官方输出规格。认证请求禁止重定向；下载仅接受HTTPS，可跟随HTTPS重定向，不携带API密钥。网络操作默认超时300秒，可用 `--timeout` 调整。请求只尝试一次，恢复下载也不创建生成请求。
