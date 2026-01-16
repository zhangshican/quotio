# macOS 14.4 兼容性计划

> **目标**: 将 Quotio 最低系统版本从 macOS 15.0 降至 macOS 14.4，同时保持 macOS 15+ 上的完整功能体验。
>
> **创建日期**: 2026-01-16
>
> **策略**: 渐进增强 (Progressive Enhancement) - 在 14.4 上提供核心功能，在 15.0+ 上启用增强特性。

---

## 一、执行摘要

| 类别 | 改动数量 | 风险等级 | 预计工时 |
|------|----------|----------|----------|
| 配置文件 | 3 处 | 低 | 0.5h |
| Swift API 兼容 | ~5 处 | 中 | 2h |
| 文档更新 | 8+ 处 | 低 | 0.5h |
| 测试验证 | - | 中 | 2h |
| **总计** | | | **~5h** |

---

## 二、配置层改动（必须）

### 2.1 Info.plist - 运行时最低版本

**文件**: `Quotio/Info.plist`  
**行号**: 19-20

```xml
<!-- 改动前 -->
<key>LSMinimumSystemVersion</key>
<string>15.0</string>

<!-- 改动后 -->
<key>LSMinimumSystemVersion</key>
<string>14.4</string>
```

**影响**: 这是 macOS 14.x 无法启动的**硬阻塞点**。不改此项，App 在 14.x 上会直接被系统拒绝运行。

---

### 2.2 project.pbxproj - 编译时部署目标

**文件**: `Quotio.xcodeproj/project.pbxproj`  
**行号**: 311, 359 (Debug 和 Release 配置)

```
// 改动前
MACOSX_DEPLOYMENT_TARGET = 14.6;

// 改动后
MACOSX_DEPLOYMENT_TARGET = 14.4;
```

**影响**: 让编译器按 14.4 的 API 边界做检查，编译时会警告/报错不兼容的 API。

---

### 2.3 验证清单

改动后执行：
```bash
# 验证 Info.plist
/usr/libexec/PlistBuddy -c "Print :LSMinimumSystemVersion" Quotio/Info.plist
# 预期输出: 14.4

# 验证 pbxproj (应该只有 14.4，没有 15.0 或 14.6)
grep -E "MACOSX_DEPLOYMENT_TARGET" Quotio.xcodeproj/project.pbxproj
```

---

## 三、Swift API 兼容性改动（必须）

### 3.1 高风险 API 清单

以下 API 在 macOS 14.4 上可能不可用或行为不同，需要添加 `#available` 保护：

| API | 文件:行号 | macOS 最低版本 | 处理方式 |
|-----|-----------|----------------|----------|
| `.contentTransition(.symbolEffect(.replace))` | `SettingsScreen.swift:2699` | 15.0 | 条件编译，14.x 降级为无动画 |
| `.smooth(duration:)` animation | `QuotaProgressBar.swift:27`, `RingProgressView.swift:33`, `QuotaCard.swift:260` | 14.0 ✅ | 无需改动 |
| `@Observable` | 25 个文件 | 14.0 ✅ | 无需改动 (Observation framework 14.0 可用) |
| `@Bindable` | 10+ 处 | 14.0 ✅ | 无需改动 |
| `.onChange(of:) { _, newValue in }` (新签名) | 33 处 | 14.0 ✅ | 无需改动 |
| `NavigationSplitView` | `QuotioApp.swift:300` | 13.0 ✅ | 无需改动 |
| `SMAppService` | `LaunchAtLoginManager.swift` | 13.0 ✅ | 无需改动 |
| `Task.sleep(for:)` | 4 处 | 13.0 ✅ | 无需改动 |
| `.textSelection(.enabled)` | 10 处 | 12.0 ✅ | 无需改动 |
| `.scrollContentBackground(.hidden)` | `QuotaScreen.swift:184,1400` | 14.0 ✅ | 无需改动 |
| `.searchable(text:prompt:)` | `LogsScreen.swift:69` | 12.0 ✅ | 无需改动 |
| `.confirmationDialog` | 5 处 | 12.0 ✅ | 无需改动 |

---

### 3.2 需要修改的代码

#### 3.2.1 SettingsScreen.swift - symbolEffect

**位置**: `Quotio/Views/Screens/SettingsScreen.swift:2699`

**改动前**:
```swift
.contentTransition(.symbolEffect(.replace))
```

**改动后**:
```swift
// macOS 15+ 使用 symbolEffect 动画，14.x 降级为无过渡
#if swift(>=5.9)
if #available(macOS 15.0, *) {
    .contentTransition(.symbolEffect(.replace))
} else {
    // 14.x: 无 contentTransition，直接切换
}
#endif
```

**简化方案** (推荐):
```swift
// 直接移除此行，对 UI 影响极小（仅影响图标切换的微动画）
// .contentTransition(.symbolEffect(.replace))  // Removed for macOS 14.4 compatibility
```

---

### 3.3 潜在风险点（需关注但大概率无需改动）

| API | 说明 | 备注 |
|-----|------|------|
| `.formatted(.relative(presentation:))` | 日期格式化 | 14.0+ 可用，但 `presentation: .named` 参数需验证 |
| `.monospaced()` font modifier | 字体修饰符 | 14.0+ 可用 |
| `.spring(response:dampingFraction:)` | 弹簧动画 | 13.0+ 可用 |

建议：降低部署目标后重新编译，观察编译器警告。

---

## 四、文档更新（建议）

以下文件需要同步更新版本声明：

| 文件 | 行号 | 原内容 | 新内容 |
|------|------|--------|--------|
| `README.md` | 75 | `macOS 15.0 (Sequoia) or later` | `macOS 14.4 (Sonoma) or later` |
| `README.zh.md` | 75 | `macOS 15.0（Sequoia）或更高版本` | `macOS 14.4（Sonoma）或更高版本` |
| `README.vi.md` | 75 | `macOS 15.0 (Sequoia) trở lên` | `macOS 14.4 (Sonoma) trở lên` |
| `README.fr.md` | 75 | `macOS 15.0 (Sequoia) ou ultérieur` | `macOS 14.4 (Sonoma) ou ultérieur` |
| `AGENTS.md` | 9 | `macOS 15+` | `macOS 14.4+` |
| `docs/project-overview-prd.md` | 5, 242 | `macOS 15.0+ (Sequoia)` | `macOS 14.4+ (Sonoma)` |
| `docs/codebase-summary.md` | 5, 24 | `macOS 15.0+ (Sequoia)` | `macOS 14.4+ (Sonoma)` |
| `docs/codebase-structure-architecture-code-standards.md` | 5 | `Minimum macOS: 15.0` | `Minimum macOS: 14.4` |

---

## 五、测试验证计划

### 5.1 构建验证

```bash
# 清理并重新构建
xcodebuild clean -project Quotio.xcodeproj -scheme Quotio
xcodebuild -project Quotio.xcodeproj -scheme Quotio -configuration Debug build 2>&1 | tee build.log

# 检查是否有 API 可用性警告
grep -i "available" build.log
grep -i "deprecated" build.log
```

### 5.2 功能测试矩阵

| 测试项 | macOS 14.4 | macOS 15.0+ | 优先级 |
|--------|------------|-------------|--------|
| App 启动 | ⬜ | ⬜ | P0 |
| 主界面渲染 | ⬜ | ⬜ | P0 |
| 菜单栏显示 | ⬜ | ⬜ | P0 |
| OAuth 登录流程 | ⬜ | ⬜ | P0 |
| Proxy 启动/停止 | ⬜ | ⬜ | P0 |
| CLI Agent 配置 | ⬜ | ⬜ | P1 |
| Quota 刷新 | ⬜ | ⬜ | P1 |
| Settings 页面 | ⬜ | ⬜ | P1 |
| Logs 页面 | ⬜ | ⬜ | P2 |
| 开机启动 (SMAppService) | ⬜ | ⬜ | P2 |
| Sparkle 自动更新 | ⬜ | ⬜ | P2 |
| 深色/浅色模式 | ⬜ | ⬜ | P2 |
| symbolEffect 动画 (SettingsScreen) | N/A (降级) | ⬜ | P3 |

### 5.3 测试环境建议

- **macOS 14.4**: 使用 VM (Parallels/VMware) 或备用机器
- **macOS 15.x**: 现有开发环境

---

## 六、执行步骤（按顺序）

### Phase 1: 配置改动 (30 min)

1. [ ] 修改 `Quotio/Info.plist` - LSMinimumSystemVersion → 14.4
2. [ ] 修改 `Quotio.xcodeproj/project.pbxproj` - MACOSX_DEPLOYMENT_TARGET → 14.4 (两处)
3. [ ] 执行验证命令确认改动生效

### Phase 2: 代码兼容 (2 hours)

4. [ ] 修改 `SettingsScreen.swift:2699` - 移除或条件编译 `.contentTransition(.symbolEffect(.replace))`
5. [ ] 清理构建，检查编译器警告
6. [ ] 修复任何新出现的 API 可用性问题

### Phase 3: 文档更新 (30 min)

7. [ ] 更新所有 README 文件的系统要求
8. [ ] 更新 docs/ 目录下的相关文档
9. [ ] 更新 AGENTS.md

### Phase 4: 验证 (2 hours)

10. [ ] 在 macOS 14.4 上完成功能测试矩阵
11. [ ] 在 macOS 15.x 上回归测试，确保功能不变
12. [ ] 验证 Release 构建

---

## 七、回滚方案

如果兼容性改动引入严重问题：

```bash
# 恢复配置文件
git checkout HEAD -- Quotio/Info.plist
git checkout HEAD -- Quotio.xcodeproj/project.pbxproj

# 恢复代码改动
git checkout HEAD -- Quotio/Views/Screens/SettingsScreen.swift
```

---

## 八、附录

### A. 完整 API 可用性参考

| API | 最低 macOS |
|-----|-----------|
| SwiftUI | 10.15 |
| @Observable (Observation) | 14.0 |
| @Bindable | 14.0 |
| NavigationSplitView | 13.0 |
| .onChange(of:) 新签名 | 14.0 |
| .symbolEffect | 14.0 (基础), 15.0 (.replace 增强) |
| SMAppService | 13.0 |
| .textSelection | 12.0 |
| .searchable | 12.0 |
| async/await | 12.0 |
| Sparkle 2.x | 10.13 |

### B. 相关资源

- [Apple Platform Availability](https://developer.apple.com/documentation/swiftui)
- [Swift Evolution: Observation](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md)
- [Sparkle Framework](https://sparkle-project.org)

---

**文档维护者**: AI Assistant  
**最后更新**: 2026-01-16
