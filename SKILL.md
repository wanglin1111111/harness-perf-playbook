---
name: harness-perf-playbook
version: 1.0.0
author: wanglin1111111
description: |
  Harness 性能优化实战手册，包含 5 个隐藏坑、7 个修复模式和 1 套 Checklist。来源：StepFun Harness P4 联邦同步模块的实战优化，性能提升 4-500x。涵盖列表 API 优化、文件名硬编码问题、urllib.timeout 不精确等常见性能陷阱。
---

# Harness 性能优化实战手册

> **5 个隐藏坑 + 7 个修复模式 + 1 套 Checklist**
> 来源：StepFun Harness P4 联邦同步模块的实战优化
> 时间：2026-06-07
> 性能提升：4-500x

## 实战数据

| 场景 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 完整 e2e (17 tests) 冷启 | 60-117s | 15.5s | 4-7x |
| 完整 e2e (17 tests) 热缓存 | ~60s | 8.3s | 7-12x |
| Federation sync 冷启 | 12-24s | 1.5s | 8-16x |
| Federation sync 缓存命中 | 12-24s | 0.05-0.18s | 100-500x |
| 离线场景 | 12-24s (反复超时) | 0.7s (探测后跳走) | 17-34x |

## 5 个隐藏坑

### 坑 #1：列表 API 返回数量不可控（最致命）

```python
# 期望
skill_names = list_github_dir(repo, "skills")  # 假设 5-10 个

# 现实
skill_names = list_github_dir(repo, "skills")  # 88 个子目录
for sn in skill_names:
    fetch(sn)  # × 88 次 HTTPS = 12-24s
```

**避免**：
```python
MAX_SKILLS_PER_SOURCE = 20
if len(skill_names) > MAX_SKILLS_PER_SOURCE:
    print(f"[WARN] capping {len(skill_names)} → {MAX_SKILLS_PER_SOURCE}")
    skill_names = skill_names[:MAX_SKILLS_PER_SOURCE]
```

### 坑 #2：文件名硬编码（87/88 次 404 浪费）

```python
# 代码假设
text = fetch(f"{subpath}/{name}/skill.yaml")  # 全 404

# 实际仓库可能是
skills/xxx/SKILL.md           # 大写 .md
skills/xxx/manifest.json      # 不同命名
skills/xxx/package.json       # npm 风格
```

**避免**：
```bash
# 1. 添加源前先探测目录结构
curl -s https://api.github.com/repos/{owner}/{repo}/contents/{path}/{sample} | jq '.[].name'

# 2. 工具支持 --filename 参数（不要硬编码）
harness federate add --filename SKILL.md
```

### 坑 #3：urllib.timeout 不精确

```python
# 误以为
urllib.request.urlopen(url, timeout=3)  # 一定 3s 内返回

# 实际行为（DNS 失败/连接被 RST/TLS 卡住）
TCP 连接建立: 受 OS SYN_RETRANSMISSION 影响, 可能 30s+
DNS 解析:     不受 timeout 控制, 依赖系统 resolver
TLS 握手:     额外 1-30s
只有数据传输: 严格受 timeout=3 控制
```

**实测延迟**：
| 场景 | timeout=3 实际耗时 |
|------|-------------------|
| 正常可达 | 0.35-0.50s ✅ |
| DNS 解析失败 | **5-30s** ❌ |
| 连接被 RST | **3-10s** ❌ |
| TLS 握手卡住 | **10-30s** ❌ |

**避免**：
```python
# 方案A: socket 全局默认超时
import socket
socket.setdefaulttimeout(3)

# 方案B: 双重超时（soft + hard）
def fetch(url, soft=3, hard=10):
    ...
```

## 7 个修复模式

1. **列表截断**：限制每次获取的技能数量
2. **动态探测**：先探测目录结构再决定文件名
3. **双重超时**：socket 全局超时 + 单次请求超时
4. **连接复用**：使用 requests.Session 或 urllib3 PoolManager
5. **缓存策略**：TTL 缓存 +  stale-while-revalidate
6. **渐进式加载**：先加载元数据，再按需加载详情
7. **离线降级**：探测失败时快速跳走，不阻塞

## Checklist

- [ ] 限制列表 API 返回数量（MAX_SKILLS_PER_SOURCE）
- [ ] 探测目录结构而非硬编码文件名
- [ ] 设置 socket 全局超时
- [ ] 使用连接池复用 HTTP 连接
- [ ] 实现缓存层减少重复请求
- [ ] 添加离线降级逻辑
- [ ] 监控关键路径耗时

## 许可证

MIT License
