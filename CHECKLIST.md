# HTTP I/O 模块性能 Checklist

> 适用：任何写"循环拉取 + 写文件"的网络同步模块
> 来源：StepFun Harness federation sync 优化实战

## 🛑 上线前必过（否则必踩坑）

### 1. 超时
- [ ] **单请求有 timeout？**（推荐 2-3s）
- [ ] **整体操作有总超时？**（推荐 10-30s）
- [ ] **DNS / TLS / TCP 各阶段都有保护？**（socket.setdefaulttimeout）

### 2. 并发
- [ ] **N>3 时用 ThreadPoolExecutor？**（推荐 max_workers=4-8）
- [ ] **as_completed 带 timeout？**（防止总耗时失控）
- [ ] **超时后未完成项有兜底？**（写 stub / 跳过 / 报错）

### 3. 缓存
- [ ] **结果有 TTL 缓存？**（5min 适合配置同步，1h 适合数据同步）
- [ ] **缓存 key 含版本/时间戳？**（避免脏数据）
- [ ] **有 force 选项绕过缓存？**（调试/数据更新场景）

### 4. 离线 / 降级
- [ ] **网络探测有吗？**（1s 快速判断可达性）
- [ ] **离线有 stub 数据？**（不阻塞上层业务）
- [ ] **降级路径有明确日志？**（便于排查）

### 5. 配置
- [ ] **文件名/参数可配置？**（不同仓库命名规范不同）
- [ ] **列表数量有上限？**（MAX_ITEMS 防爆）
- [ ] **优先级可配？**（多源冲突时手动覆盖）

### 6. 早退 / 警告
- [ ] **404 率/失败率高时警告？**（引导用户修正配置）
- [ ] **长时间无响应有进度提示？**（避免"假死"困惑）
- [ ] **错误信息含可操作建议？**（"Try --filename X"）

### 7. 测试
- [ ] **冷启动时间 < 30s？**
- [ ] **热缓存时间 < 5s？**
- [ ] **离线场景可降级通过？**
- [ ] **测试间无状态泄漏？**

---

## 🎯 性能数字参考

| 指标 | 目标 | 警戒线 | 失败线 |
|------|------|--------|--------|
| 单次 e2e 测试 | < 30s | 30-60s | > 60s |
| 单次 API 调用 | < 1s | 1-3s | > 3s |
| 单源 federation sync | < 15s | 15-30s | > 30s |
| 缓存命中响应 | < 0.2s | 0.2-1s | > 1s |
| 离线降级 | < 2s | 2-5s | > 5s |
| 子进程启动 | < 0.5s | 0.5-1s | > 1s |

---

## 🔍 常见问题速查

| 症状 | 可能原因 | 修复 |
|------|---------|------|
| sync 永远卡住 | urllib.timeout 不生效 | 加 hard timeout + 线程 |
| sync 12-24s | 串行 + 88 个文件 | ThreadPoolExecutor(6) |
| sync 完 0 个文件 | 文件名错（SKILL.md vs skill.yaml） | 加 --filename 参数 + 早退警告 |
| 重跑同样慢 | 无缓存 | .sync_meta.json + 5min TTL |
| 离线时同样慢 | 总是试到超时 | 1s 探测 + 离线 stub |
| 第二次跑 60s | 17 测试全串行 + Python 冷启 | P0-P3 并行 + P4 串行 |
| 列表返回 1000+ | 没设上限 | MAX_SKILLS = 20 截断 |

---

## 📋 实施模板

```python
def sync_external_source(source, total_timeout=15, force=False):
    """标准模板：探测 + 缓存 + 并发 + 超时 + stub"""
    
    # 1. 缓存
    if not force and _should_skip(name, ttl=300):
        return {"status": "cached"}
    
    start = time.monotonic()
    
    # 2. 探测
    if not _probe():
        write_stubs(items, source)
        return {"status": "offline"}
    
    # 3. 列表（设上限）
    items = list_items(source)[:MAX_ITEMS]
    
    # 4. 并发
    remaining = total_timeout - (time.monotonic() - start)
    with ThreadPoolExecutor(max_workers=6) as pool:
        futures = {pool.submit(fetch_one, x): x for x in items}
        try:
            for fut in as_completed(futures, timeout=remaining):
                process(fut.result())
        except TimeoutError:
            for fut in futures:
                if not fut.done():
                    process((futures[fut], None))  # 走 stub
    
    # 5. 早退
    if failed/total > 0.8:
        warn("Try different --filename")
    
    # 6. 写缓存
    _write_cache_meta(name, ...)
    return {"status": "ok"}
```

---

## 🚨 千万不要做

1. ❌ **不要假设返回列表小** — 真实仓库可能 100+ 子目录
2. ❌ **不要硬编码文件名** — 不同仓库用 SKILL.md / manifest.json / package.json
3. ❌ **不要无超时循环** — 一次网络失败 = 整个 sync 卡住
4. ❌ **不要忽略 4xx 响应** — 404 率高通常不是网络问题
5. ❌ **不要无脑串行** — N>3 必须并发
6. ❌ **不要每次重跑都重做** — 5min 缓存能省 99% 重复工作
7. ❌ **不要相信 urllib.timeout=3 真的 3s** — DNS/TLS 不受其控制
