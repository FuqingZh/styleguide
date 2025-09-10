### WDL
## WDL 任务找不到目录：最常见原因与一招解决

**一句话结论**
报错“目录不存在/不可访问”多半不是路径写错，而是：**流程把输入目录做成“快捷方式”（软链接）指向 `/cephfs_data/...`，但容器里没把整块 `/cephfs_data` 挂进来**，所以链接指向了“空气”。

---

### 你会看到的现象

* 日志里类似：`FileNotFoundError: .../lims 不存在或不可访问`
* 调试时：`inputs/.../lims -> /cephfs_data/.../lims`（这是软链接）
* 容器里 `ls -ld /cephfs_data` 有时只见空目录，继续进入就断了

> 可把软链接理解为“快捷方式”。快捷方式指向的真实位置，容器必须“看得见”。

---

### 标准修法（推荐其一）

#### 方案 A：让容器“看得见”整块数据盘（适合大目录，如参考库）

把宿主机的 `/cephfs_data` **挂到** 容器里的同一路径。

**Cromwell（本地 Docker backend）**：`backend.conf`

```hocon
backend {
  default = Local
  providers { Local {
    actor-factory = "cromwell.backend.impl.sfs.config.ConfigBackendLifecycleActorFactory"
    config {
      runtime-attributes = """
        String? docker
        Int cpu = 32
        String memory = "64G"
      """
      submit-docker = """
        docker run --rm -i \
          --cpus ${cpu} --memory ${memory} --shm-size=8g \
          -v /cephfs_data:/cephfs_data:ro \
          ${docker} /bin/bash -c "~{script}"
      """
    }
  }}
}
```

**miniwdl：**

```bash
miniwdl run flow.wdl -i inputs.json \
  --runtime-option 'docker_run_options=["-v","/cephfs_data:/cephfs_data:ro","--shm-size=8g"]'
```

**Kubernetes（简版示例）**：把 PVC 挂到 `/cephfs_data`

```hocon
kubernetes { pod {
  volumeMounts = [{ name: "cephfs", mountPath: "/cephfs_data", readOnly: true }]
  volumes = [{ name: "cephfs", persistentVolumeClaim: { claimName: "cephfs-pvc" } }]
}}
```

#### 方案 B：把目录**复制**进执行目录（只适合小目录，如 `lims`）

改“本地化策略”为 **copy**，这样不再用软链接。

```hocon
backend { providers { Local { config {
  filesystems { local { localization = ["copy"] } }  # 默认会用硬/软链接
}}}}
```

---

### WDL 侧的小改动（更稳）

* 把目录输入声明为 **`Directory`**（而不是字符串），引擎才会自动本地化：

  ```wdl
  input { Directory limsinfodir }
  command { python ... --limsinfodir "~{limsinfodir}" ... }
  ```
* 命令里的占位符用 `~{var}`，路径加引号，避免空格/中文导致解析失败。

---

### 排错三行（放在 task 开头）

```bash
echo "[debug] pwd=$(pwd)"
ls -ld /cephfs_data || echo "[debug] /cephfs_data NOT mounted"
ls -l "~{limsinfodir}" || echo "[debug] limsdir not visible"
```

如果 `/cephfs_data` 没挂，选 **方案 A**；若是小目录，选 **方案 B** 即可。

