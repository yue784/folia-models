# folia-models

Folia 的原生推理权重。这些文件太大，不适合跟应用源码放在一起，所以单独托管在这里，由构建脚本按**大小 + sha256** 校验后下载。

## 内容

| 文件 | 大小 | 用途 | sha256 |
|---|---|---|---|
| `scnet.int8.onnx` | 12,900,277 B | 四轨分离（人声 / 伴奏），Android 原生 ONNX Runtime | `98228931494151762a1c4ab1ec7899a894b1f81fd4a509921a9d2f9bacc50845` |

## 直接下载

不跑构建脚本、只想手动拿文件时：

```bash
# 国内推荐：走加速镜像（2026-11 实测 3-4MB/s，直连 raw 只有 ~9KB/s）
curl -L -o scnet.int8.onnx \
  https://gh-proxy.com/https://raw.githubusercontent.com/yue784/folia-models/main/scnet.int8.onnx

# 备用镜像
curl -L -o scnet.int8.onnx \
  https://ghfast.top/https://raw.githubusercontent.com/yue784/folia-models/main/scnet.int8.onnx

# 直连（海外网络，或上面两个都不可用时）
curl -L -o scnet.int8.onnx \
  https://raw.githubusercontent.com/yue784/folia-models/main/scnet.int8.onnx
```

下载后必须校验，大小和 sha256 都对上才算拿到真文件：

```bash
ls -l scnet.int8.onnx          # 必须是 12900277
sha256sum scnet.int8.onnx      # 必须是 98228931...bacc50845
```

然后放进 `android/app/src/main/assets/models/`，文件名保持 `scnet.int8.onnx`（Kotlin 侧的 `AutomixModels` 只认这个路径）。

## 来源与许可

`scnet.int8.onnx` 由 [starrytong/SCNet](https://github.com/starrytong/SCNet) 的官方 checkpoint 导出为 ONNX，并做 int8 量化。SCNet 以 **MIT** 许可发布，权重随其许可分发；本仓库只做格式转换（PyTorch → ONNX）与量化，模型本身未做再训练。

导出要点（复现用）：输入 `mix_spec` 为 `float32 [1, 4, 2049, 476]`（4 声道 = L 实 / L 虚 / R 实 / R 虚），输出 `stems_spec` 为 `float32 [1, 16, 2049, 476]`（16 = 4 轨 × 2 声道 × 实虚）；STFT 为 `N_FFT=4096`、`HOP=1024`、**矩形窗**、正交归一化（`1/√4096`）、`center=true` 且两端 **reflect** 补边；单窗 485100 样本（11 秒）按模型规则补到 486400。

## 被谁使用

`scripts/fetchAndroidModels.mjs`（Folia 仓库）从本仓库的 `main` 分支下载，校验大小与 sha256 后写入 `android/app/src/main/assets/models/`。下载顺序是**镜像优先**：`gh-proxy.com` → `cdn.jsdelivr.net` → `ghfast.top` → `raw.githubusercontent.com`（直连只作兜底），任一源 20 秒没有新字节就自动换下一个。校验不通过时构建直接失败 —— 宁可打不出包，也不要打一个分轨结果全错的包。

## 更新约定

替换权重时必须同时更新：本文件的大小 / sha256 表格、Folia 仓库 `scripts/fetchAndroidModels.mjs` 的 `bytes` 与 `sha256`、以及 `android/app/build.gradle` 里的体积校验表。三处对不上，构建会拒绝继续。
