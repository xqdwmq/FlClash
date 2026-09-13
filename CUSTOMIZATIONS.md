# 定制记录

本 fork（Xlclash）相对上游 [chen08209/FlClash](https://github.com/chen08209/FlClash) 的个性化定制台账。
每次新的个性化定制，在「定制清单」末尾按模板追加一节；同步上游新版本时，对照本清单逐项核对冲突。

## 条目模板

```markdown
### YYYY-MM-DD · 标题
- 基线：所在的分支 / 基于的上游 tag
- 类型：修复 / 功能 / 视觉 / 流水线 / 文档
- 文件：涉及文件（相对路径）
- 背景：为什么上游的默认行为不满足
- 方案：做了什么，怎么实现的
- 回归保护：锁行为的测试或校验
- 注意：与上游合并时的冲突点、失效条件
```

## 分支与版本约定

- `graphics` 分支：当前活跃基线，直接切自上游稳定 tag（当前 v0.8.97），定制在其上叠加。
- 版本号：tag 用 `v<上游版本>.<本地补丁号>`（如 `v0.8.97.1`）；pubspec 用三段式
  `0.8.97+<YYYYMMDD><序号>`（上游限制 pubspec 只能三段，补丁号由 tag 承载）。
- 发版：改 pubspec 版本号 → `dart run tool/changelog.dart release --version <版本>` → 提交
  `chore(release): vX.Y.Z.N` → 推 tag 触发上游工作流构建并发布 Release。
- 本机跑 Flutter 命令带 `PUB_HOSTED_URL=https://pub.dev`，否则 CN 镜像会静默重解析
  pubspec.lock，造成依赖漂移。

## 定制清单

### 2026-09-12 · 可变字体 wght 轴钉住，修复中文整体变粗

> 详细修改思路、复用 runbook 与排错表见 [FONT_WEIGHT_PINNING.md](FONT_WEIGHT_PINNING.md)。

- 基线：graphics @ v0.8.97
- 类型：修复（渲染）
- 文件：`lib/common/text.dart`、`lib/application.dart`、`test/common/text_test.dart`、`test/application_test.dart`
- 背景：Flutter 3.41 起 `fontWeight` 会隐式写入可变字体的 `wght` 轴；红米 K70
  （HyperOS）的中文回退字体 MiSans 是可变字体，导致 v0.8.97 基线（需 Flutter 3.47.1
  构建）相对旧构建（Flutter 3.35.7）所有 w500/w600 文字角色真实变粗。
- 方案：`lib/common/text.dart` 新增 `toDefaultWeight` 扩展（TextStyle/TextTheme/
  Typography 三级），显式钉住 `FontVariation('wght', 400)`；`lib/application.dart` 的
  `buildAppTheme()` 通过 `typography:` 参数注入，组件主题在构造时派生自带钉住。
- 回归保护：`test/common/text_test.dart`（钉住语义）、`test/application_test.dart`
  （端到端断言真实主题的 fontVariations）。
- 注意：静态字体（Roboto、JetBrainsMono、Icons）不支持 wght 轴，完全不受影响；若上游
  重构 `buildAppTheme` 或 Typography 注入方式，需保留钉住链路。

### 2026-09-12 · 发版流水线适配 fork

- 基线：graphics @ v0.8.97.1
- 类型：流水线
- 文件：`.github/workflows/build.yaml`、`tool/src/changelog/git.dart`、
  `tool/src/changelog/builder.dart`、`test/tool/changelog_e2e_test.dart`
- 背景：上游工作流假定其仓库的 secrets 与 3 段式版本号；fork 缺
  `SERVICE_JSON`/`TELEGRAM_*`/部署密钥，且本地补丁版本是 4 段式。
- 方案：
  - 上游工作流加最小守卫：无 `SERVICE_JSON` 时保留仓库内 google-services.json（空文件
    会让 google-services 插件挂掉）；telegram / Homebrew / fdroid 分发步骤在对应
    secret 缺失时跳过（GitHub Release 创建与 APK 签名不受影响，签名用 fork 的
    KEYSTORE 四件套）。
  - changelog 工具 `VersionTag` 支持 `vX.Y.Z.N` 本地补丁段：排序在基线 tag 之后、
    下一个 patch 之前，渲染独立小节。
- 回归保护：`test/tool/changelog_e2e_test.dart` 新增两条 4 段式用例。
- 注意：发版前确认 fork 上存在 changelog 依赖的上游边界 tag（当前至少
  `v0.8.96`、`v0.8.97`，否则 CI 校验报 unknown revision）；同步新上游版本后把新的
  上游 tag 也推到 fork。

### 2026-09-13 · 加载动画恢复 v7.0.33 样式（卡片细圆圈 + 通用旋转星形）

- 基线：graphics @ v0.8.97.2
- 类型：视觉
- 文件：`lib/widgets/loading.dart`、`test/widgets/loading_test.dart`
- 背景：用户要的动画是 v7.0.33 的样式——一颗持续旋转（3s/圈）、角数在 3↔9 连续
  往复（1s easeInOut）的星形（StarBorder 的 points 浮点连续变形，innerRadius 0.8 /
  齿圆角 0.5 / 谷圆角 0.1 / squash 0.5）。上游 v0.8.97 改为 M3E 离散形状序列
  （MaterialShapes 七形状 Morph），观感完全不同；曾按七形状参数复刻并随
  v0.8.97.2 发布，真机确认仍不是目标样式，本条目取代之。
- 方案：分两层恢复，与 v7.0.33 逐点对齐：
  - 代理卡片测延迟 pending：`CircularProgressIndicator(strokeWidth: 2)`（细线
    圆圈 spinner，"一圈一圈转"的那个）——`lib/views/proxies/card.dart`；
  - 其余加载点（资源页/面板/授权页等）：`CommonCircleLoading` 恢复为 v7.0.33
    的 StarBorder 旋转星形（3s/圈、角数 3↔9 连续往复、innerRadius 0.8、
    齿圆角 0.5、谷圆角 0.1、squash 0.5），外壳沿用新版尺寸语义（默认 48、
    被父约束钳制取短边、Loose 约束收缩、RepaintBoundary）。
  M3E 的离散顶点 API（RoundedPolygon.star 角数只能整数）无法实现连续点数
  变形，经评估后明确不采用。
- 回归保护：`test/widgets/loading_test.dart`（尺寸解析 + 旋转/点数持续动画）。
- 注意：`CommonCircleLoading` API 收敛为 `{color}`（variant/polygons/
  semanticLabel 等无调用点使用，已移除）；material_new_shapes 依赖仍被
  null_status.dart 使用故保留；上游重构 loading.dart 时按本条目重新对位。

## 未迁移（旧 v7.0.x / optimize 分支，按需迁移）

以下定制留在 `optimize` 分支，尚未带到新基线；迁移时移入上方清单并注明迁移提交：

- 应用名 Xlclash 与品牌层（`f40d01a3`、`9b0dee79`）；applicationId
  `com.xqd922.flclash`（dev 后缀 `.dev`）。
- 内置主题色扩展与主题页自定义色功能（`735797f6`、`75f158b3`）。
- 主题选项精简：移除上游独有的主题选项（`69c45c8b`）。
- 纯黑模式移除（旧 application.dart 内「本地定制」注释）。
- 测试套件对 Xlclash 定制与 Windows 宿主的适配（`71565c8a`）。
