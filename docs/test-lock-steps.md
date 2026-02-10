# 锁释放实测步骤（在本项目内完成）

按顺序执行以下步骤，在本仓库内完成「持锁时 restart → 锁释放 → 另一 runner 拿到锁」的实测。

---

## 第一步：配置 .env

在项目根目录：

```bash
cd ~/os-internship/github-runners
cp .env.example .env
```

编辑 `.env`，至少填写（把下面的值换成你的）：

```env
ORG=你的GitHub用户名或组织名
GH_PAT=你的PAT

# 两个板子共用一个锁，便于观察「一个持锁、一个等锁」
RUNNER_RESOURCE_ID=board-shared
```

**PAT 从哪来**：GitHub 网页 → 右上角头像 → **Settings** → 左侧最下方 **Developer settings** → **Personal access tokens** → **Tokens (classic)** → **Generate new token**。勾选权限：`repo`（若注册到仓库）、或 **admin:org** 下与 runner 相关的权限（若注册到组织）。生成后复制 token，只显示一次，填到 `GH_PAT`。

- **没有组织、用个人账户**：`ORG` 填你的 **GitHub 用户名**，并**必须**设置 `REPO=github-runners`（本仓库名），否则会报 `Failed to fetch registration token!`（个人账户没有 org API，只能用仓库级 Runner）。
- 若仓库是「用户名/github-runners」，则 `ORG=用户名`、`REPO=github-runners`。保存退出。

---

## 第二步：宿主机创建锁目录

```bash
sudo mkdir -p /tmp/github-runner-locks
sudo chmod 1777 /tmp/github-runner-locks
```

---

## 第三步：生成并启动板子 Runner

```bash
cd ~/os-internship/github-runners
chmod +x runner.sh
./runner.sh init -n 0
```

`-n 0` 表示只起板子 runner（phytiumpi、roc-rk3568-pc），不起普通 runner。等待 compose 拉镜像并启动。

---

## 第四步：确认容器在跑

```bash
./runner.sh ps
```

应看到两个容器，例如 `xxx-runner-phytiumpi`、`xxx-runner-roc-rk3568-pc`。记下你的 **RUNNER_NAME_PREFIX**（一般是主机名），后面 restart 要用容器名。

---

## 第五步：注册 Runner 到 GitHub

```bash
./runner.sh register
```

按提示输入或使用 .env 中的 ORG/GH_PAT。完成后打开 GitHub 本仓库 → **Settings → Actions → Runners**，确认两个 runner 显示为 Idle。

---

## 第六步：把测试 workflow 推到 GitHub

本仓库已包含 `.github/workflows/test-lock.yml`，推送到远程即可在 Actions 里看到：

```bash
git add .github/workflows/test-lock.yml docs/test-lock-steps.md
git commit -m "ci: add test-lock workflow for runner-wrapper 锁实测"
git push origin feat/runner-wrapper-multi-org-lock
```

（若当前分支名不同，把最后一行换成你的分支名。）

---

## 第七步：第一次触发（让 Runner A 持锁）

1. 浏览器打开本仓库 **Actions** 页。
2. 左侧点 **test-lock**。
3. 右侧点 **Run workflow**，再点绿色 **Run workflow**。
4. 等约 15 秒，点进刚出现的这次 run。
5. 点进 **hold-lock** job，展开 **Set up job** 或第一步的日志。
6. 确认出现 **`Acquired lock for board-shared`**，说明当前接 job 的 runner 已持锁。

记下这次是哪个 runner 接的：看日志里的 runner 名，或看 **Fourth step** 里两个容器名，能接 phytiumpi/roc 的通常是 `xxx-runner-phytiumpi` 或 `xxx-runner-roc-rk3568-pc`。例如是 `DESKTOP-92DBKKJ-runner-phytiumpi`。

---

## 第八步：第二次触发（让 Runner B 等锁）

1. 仍在 **Actions → test-lock** 页。
2. 再点一次 **Run workflow** → **Run workflow**。
3. 立刻点进**第二次**出现的 run。
4. 点进 **hold-lock** job，看 **Set up job** 日志。
5. 应看到 **`Waiting for lock: board-shared`**，且一直停在这里。

此时：第一次 run 的 job 在持锁跑 `sleep 90`，第二次 run 的 job 在等锁。

---

## 第九步：持锁期间重启 Runner A

在**本机终端**执行（把容器名换成你在第七步记下的、正在跑第一次 job 的 runner 名）：

```bash
cd ~/os-internship/github-runners
./runner.sh restart DESKTOP-92DBKKJ-runner-phytiumpi
```

若你的是 roc 接的第一次 job，则换成：

```bash
./runner.sh restart DESKTOP-92DBKKJ-runner-roc-rk3568-pc
```

（容器名可用 `./runner.sh ps` 或 `docker ps` 查看。）

---

## 第十步：看结果

1. **第一次 run**：对应 job 会因 runner 被重启而**失败**。
2. **第二次 run**：几秒内应从 **Waiting for lock** 变为 **Acquired lock for board-shared**，然后正常跑完约 90 秒的 step。

若第二次 run 的 job 能拿到锁并跑完，说明：**持锁时执行 `./runner.sh restart` 后锁会释放，其他等待同一 RUNNER_RESOURCE_ID 的 runner 能拿到锁并跑 job。**

---

## 可选：看日志确认

- 第一次 run 的 job 日志：有 `Acquired lock for board-shared`，无 `Releasing lock`。
- 第二次 run 的 job 日志：先 `Waiting for lock: board-shared`，再 `Acquired lock for board-shared`，最后 `Releasing lock for board-shared`。
