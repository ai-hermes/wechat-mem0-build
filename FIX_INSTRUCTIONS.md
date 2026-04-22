# CI 构建错误修复说明

## 问题分析

### 错误现象
```
Found mac update metadata: release/alpha-mac.yml
No arm64-specific metadata found, expecting universal build in release/alpha-mac.yml
release/alpha-mac.yml does not contain arm64 artifact reference
```

### 根本原因

1. **CI 配置**：有两个独立的 mac 构建任务
   - `macos-15` + `x64`
   - `macos-latest` + `arm64`

2. **electron-builder 配置**：`electron-builder.yml` 中配置了同时构建两个架构
   ```yaml
   mac:
     target:
       - target: zip
         arch:
           - x64
           - arm64
   ```

3. **package.json 脚本**：只有一个通用的 `build:mac` 脚本
   ```json
   "build:mac": "npm run build && electron-builder --mac --config"
   ```

4. **冲突**：
   - 两个 CI 任务都运行同一个 `build:mac` 脚本
   - 每个任务都尝试构建所有架构（x64 和 arm64）
   - 但由于运行在不同的 runner 上，实际只能构建对应架构
   - 两个任务生成的更新元数据文件名相同（`alpha-mac.yml`）
   - 最后上传的文件覆盖了之前的，导致只有 x64 的元数据

### 验证逻辑期望

CI 的验证步骤期望以下两种情况之一：

**方案 A：分离的架构特定文件**
- `alpha-mac.yml` - 包含 x64 构建产物
- `alpha-mac-arm64.yml` - 包含 arm64 构建产物

**方案 B：Universal Build**
- 只有 `alpha-mac.yml`，但包含 x64 和 arm64 两个架构的引用

## 解决方案

需要在源代码仓库 `ai-hermes/wechat-mem0` 的 `package.json` 中添加架构特定的构建脚本：

```json
{
  "scripts": {
    "build:mac": "npm run build && electron-builder --mac --config",
    "build:mac:x64": "npm run build && electron-builder --mac --x64 --config",
    "build:mac:arm64": "npm run build && electron-builder --mac --arm64 --config"
  }
}
```

### 为什么这样修复

1. **`--x64` 和 `--arm64` 标志**：
   - 告诉 electron-builder 只构建指定架构
   - 生成架构特定的更新元数据文件
   - x64 构建生成 `alpha-mac.yml`
   - arm64 构建生成 `alpha-mac-arm64.yml`

2. **CI 自动选择**：
   - CI 配置已更新，会优先查找 `build:mac:x64` 或 `build:mac:arm64`
   - 如果找不到，会回退到 `build:mac`（向后兼容）

## 实施步骤

### 1. 修改源代码仓库

在 `ai-hermes/wechat-mem0/package.json` 中添加：

```json
"build:mac:x64": "npm run build && electron-builder --mac --x64 --config",
"build:mac:arm64": "npm run build && electron-builder --mac --arm64 --config"
```

### 2. 提交并推送

```bash
cd /Users/dingwenjiang/workspace/codereview/ai-hermes/wechat-mem0
git add package.json
git commit -m "fix(ci): add arch-specific mac build scripts for split builds"
git push origin main
```

### 3. 触发新的构建

重新触发 CI 构建，验证修复是否生效。

## 验证

构建成功后，`release/` 目录应该包含：

```
release/
├── alpha-mac.yml              # x64 更新元数据
├── alpha-mac-arm64.yml        # arm64 更新元数据
├── rethink-ai-0.1.2-alpha.44-x64.zip
├── rethink-ai-0.1.2-alpha.44-x64.dmg
├── rethink-ai-0.1.2-alpha.44-arm64.zip
└── rethink-ai-0.1.2-alpha.44-arm64.dmg
```

## 备选方案

如果不想分离构建，可以改为在单个 runner 上构建 universal binary：

1. 只保留一个 mac 构建任务（使用 `macos-latest`）
2. 使用 `build:mac` 脚本（构建所有架构）
3. 生成的 `alpha-mac.yml` 会包含两个架构的引用

但这种方案的缺点是：
- 构建时间更长
- 需要 runner 支持交叉编译
- better-sqlite3 等原生模块可能有兼容性问题
