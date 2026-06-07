# Harness 性能优化实战手册

> **5 个隐藏坑 + 7 个修复模式 + 1 套 Checklist**
>
> 来源：StepFun Harness P4 联邦同步模块的实战优化
> 时间：2026-06-07
> 性能提升：4-500x

## 📊 实战数据

| 场景 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 完整 e2e (17 tests) 冷启 | 60-117s | 15.5s | 4-7x |
| 完整 e2e (17 tests) 热缓存 | ~60s | 8.3s | 7-12x |
| Federation sync 冷启 | 12-24s | 1.5s | 8-16x |
| Federation sync 缓存命中 | 12-24s | 0.05-0.18s | 100-500x |
| 离线场景 | 12-24s (反复超时) | 0.7s (探测后跳走) | 17-34x |

## 🎯 5 个隐藏坑

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
def fetch_with_hard_timeout(url, soft=3, hard=5):
    result = [None]
    def worker():
        result[0] = _fetch_url(url, timeout=soft)
    t = threading.Thread(target=worker, daemon=True)
    t.start()
    t.join(timeout=hard)
    return result[0]  # 超过 hard 一定返回 None
```

### 坑 #4：无缓存导致重复执行放大痛苦

```
test_e2e.py:
  test_17_federation():
    add × 2:           0.2s
    list:              0.05s
    sync (cold):       12-24s   ← 痛点
    discover:          0.2s
    remove:            0.05s
    ─────────────────────
    总计:              12.5-24.5s

如果开发者跑 5 次 e2e 调试:  60-120s 纯 federation 浪费
```

**避免**：写盘缓存 + TTL
```python
# .harness/federation/<name>/.sync_meta.json
{
  "last_sync": "2026-06-07T18:50:31",
  "status": "ok",
  "skills": 88,
  "failed": 0
}

def _should_skip_sync(name, ttl=300):
    meta = read_sync_meta(name)
    if not meta: return False
    age = (now() - parse(meta["last_sync"])).total_seconds()
    return age < ttl  # 5 分钟内直接跳过
```

### 坑 #5：测试全串行 + 子进程启动开销

```
17 tests × subprocess.run() × Python 冷启动 (~0.5s) = 8.5s 纯启动
+ federation sync 12s
+ queue worker 3次 = 4.5s
+ economics run 6次 = 6s
─────────────────────────────────────────
总计 ≈ 31s（理想） 到 117s（慢网）
```

**避免**：
```python
# P0-P3 独立测试 → 并行
import concurrent.futures
with ThreadPoolExecutor(max_workers=4) as pool:
    futures = {pool.submit(test_fn): name for name, fn in p0_p3_tests}
    for f in as_completed(futures):
        check(f.result())

# P4 状态耦合 → 串行
for name, fn in p4_tests:
    test(name, fn)
```

---

## 🛠️ 7 个修复模式（按 ROI 排序）

### 模式 #1：网络探测（必做，1秒省钱）
```python
_NET_OK = None
def _probe_network(timeout=1):
    global _NET_OK
    if _NET_OK is not None: return _NET_OK
    _NET_OK = _fetch_url("https://api.github.com/rate_limit", timeout=timeout) is not None
    return _NET_OK
```

### 模式 #2：TTL 缓存（必做，重复运行救星）
```python
# 5min 内重复 sync 走缓存
# 见坑 #4 修复代码
```

### 模式 #3：并发 I/O（N>3 必做）
```python
import concurrent.futures
with ThreadPoolExecutor(max_workers=6) as pool:
    futures = {pool.submit(fetch_one, x): x for x in items}
    for fut in as_completed(futures, timeout=remaining):
        handle(fut.result())
```

### 模式 #4：整体总超时（必做，防止单源拖死全流程）
```python
# 单请求 3s × 单源 5 个请求 = 15s 总预算
# 超出预算后所有未完成项走 stub
```

### 模式 #5：离线快速路径（必做，无网环境友好）
```python
if not _probe_network():
    for sn in skill_names:
        write_stub(sn)  # 跳过实际拉取
    return {"status": "offline"}
```

### 模式 #6：可配置参数（强烈推荐）
```python
# --filename SKILL.md / skill.yaml / manifest.json
# --timeout 15 / 30 / 60
# --max-skills 20 / 50
# --force 绕过缓存
```

### 模式 #7：早退警告（推荐，引导用户修正）
```python
# 当 404 率 > 80% 时
if len(failed) / len(total) > 0.8:
    print(f"[WARN] {failed}/{total} files not found. "
          f"Try --filename SKILL.md (case-sensitive)")
```

---

## ✅ Checklist：写新模块前逐项检查

每次写**涉及外部 HTTP 调用**的模块：

| # | 检查项 | 命令/代码 |
|---|--------|----------|
| 1 | 单请求有 timeout？ | `urllib.request.urlopen(url, timeout=3)` |
| 2 | **整体操作有总超时？** | `--timeout 15` + `as_completed(timeout=remaining)` |
| 3 | N>3 时并发？ | `ThreadPoolExecutor(max_workers=4-8)` |
| 4 | 结果缓存 / TTL？ | `.sync_meta.json` + 5min TTL |
| 5 | 离线 fallback / stub？ | 探测失败 → 写 stub 返回 |
| 6 | 返回数量有上限？ | `MAX_ITEMS = 20` 截断 |
| 7 | 文件名/参数可配置？ | `--filename / --name-pattern` |
| 8 | 高失败率早退警告？ | `if failed/total > 0.8: print(WARN)` |
| 9 | urllib 替代评估过？ | 频繁调用考虑 httpx / aiohttp |
| 10 | 测试可并行？ | P0-P3 `ThreadPoolExecutor` + P4 串行 |

---

## 📂 实际修复代码片段

`harness_federation.py` 的 `sync_github_source()` 完整实现见：
- `references/harness_federation_sync_patch.py`

关键点：
```python
def sync_github_source(source, total_timeout=15, force=False):
    # 1. 缓存检查
    if not force and _should_skip_sync(name):
        return {"status": "cached"}
    
    # 2. 网络探测
    start = time.monotonic()
    if not _probe_network():
        write_stubs_and_return("offline")
    
    # 3. 并发拉取
    skill_names = _fetch_github_skill_list(repo, ref, subpath)
    remaining = total_timeout - (time.monotonic() - start)
    
    with ThreadPoolExecutor(max_workers=6) as pool:
        futures = {pool.submit(_fetch_one, sn): sn for sn in skill_names}
        try:
            for fut in as_completed(futures, timeout=remaining):
                handle(fut.result())
        except TimeoutError:
            for fut, sn in futures.items():
                if not fut.done():
                    write_stub(sn)  # 超时部分走 stub
    
    # 4. 早退警告
    if len(skill_names) >= 5 and len(failed)/len(skill_names) > 0.8:
        print(f"[WARN] {failed}/{total} files not found. Try --filename SKILL.md")
```

---

## 🧪 验证

完整测试通过率：**17/17 PASS**

```bash
cd stepfun-harness
python scripts/test_e2e.py

# 冷启: 15.5s   (优化前 60-117s)
# 热缓存: 8.3s  (优化前 ~60s)
```

---

## 📚 延伸阅读

- [Python urllib 真实超时行为分析](https://stackoverflow.com/q/22352045)
- [GitHub API Rate Limiting](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting)
- [ThreadPoolExecutor 最佳实践](https://docs.python.org/3/library/concurrent.futures.html)

---

## 📜 License

MIT
