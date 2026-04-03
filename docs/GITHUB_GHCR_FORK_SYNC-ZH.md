# GitHub Actions 自动构建 GHCR 镜像与 Fork 同步说明

这个仓库已经补好了两套工作流：

1. `.github/workflows/docker-publish.yml`
   推送到 `main` 后，自动构建并推送 Docker 镜像到 GitHub Container Registry (`ghcr.io`)。
2. `.github/workflows/sync-upstream.yml`
   每天自动尝试把 fork 的上游更新合并到你当前仓库的默认分支，也支持手动触发。

## 一、镜像发布规则

当你向默认分支 `main` 推送代码时，会自动执行 Docker 构建并推送到：

```text
ghcr.io/<你的-github-用户名>/<你的仓库名小写>
```

以当前仓库为例，镜像地址会是：

```text
ghcr.io/crazycherrys/aiclient-2-api
```

默认会推送这些标签：

- `latest`
- `sha-<7位提交号>`

如果这次 push 里 `VERSION` 文件发生了变更，还会额外：

- 推送版本镜像标签，例如 `2.12.2.2`
- 自动创建并推送 Git Tag，例如 `v2.12.2.2`

## 二、你在 GitHub 上还需要做什么

### 1. 开启 Actions 读写权限

进入仓库：

`Settings -> Actions -> General -> Workflow permissions`

选择：

- `Read and write permissions`

这是必须的，因为：

- 发布 GHCR 镜像需要 `packages: write`
- 自动打 Git Tag 需要 `contents: write`
- fork 自动同步需要把 merge 结果 push 回你的仓库

### 2. 如果你启用了分支保护，需要允许 Actions 推送

如果你的 `main` 分支开启了保护规则，请确认 GitHub Actions 可以推送到该分支，否则：

- 自动同步上游会失败
- 自动创建版本 tag 也可能失败

### 3. 首次检查 Packages

首次成功构建后，可以在这里看到镜像：

`GitHub 仓库首页 -> Packages`

如果仓库是公开的，通常可以把包设为 public，方便直接拉取。

## 三、如何使用 GHCR 镜像

首次登录 GHCR：

```bash
echo <YOUR_GITHUB_TOKEN> | docker login ghcr.io -u <YOUR_GITHUB_USERNAME> --password-stdin
```

拉取镜像：

```bash
docker pull ghcr.io/crazycherrys/aiclient-2-api:latest
```

运行镜像：

```bash
docker run -d \
  -p 3000:3000 \
  -p 8085-8087:8085-8087 \
  -p 1455:1455 \
  -p 19876-19880:19876-19880 \
  --restart=always \
  -v "$(pwd)/configs:/app/configs" \
  --name aiclient2api \
  ghcr.io/crazycherrys/aiclient-2-api:latest
```

## 四、Fork 如何自动跟随母项目更新

新增的 `.github/workflows/sync-upstream.yml` 会做这些事：

1. 自动识别当前仓库是不是 GitHub fork
2. 读取 fork 的上游母仓库信息
3. 拉取上游默认分支
4. 合并到你当前仓库的默认分支
5. 把结果推回你的 fork

触发方式：

- 手动触发：`Actions -> 同步 Fork 上游更新 -> Run workflow`
- 定时触发：每天 UTC `02:17`

## 五、同步策略说明

这个工作流使用的是 `git merge`，不是强制覆盖，所以更适合你这种已经在 fork 上有自定义修改的情况：

- 你的改动会保留
- 上游更新会被合并进来
- 如果上游和你的改动发生真实冲突，工作流会失败，等待你手动解决

这比单纯的 fast-forward 同步更适合长期维护自己的 fork。

## 六、推荐的本地 Git 配置

除了 GitHub Actions 自动同步，也建议你本地把上游 remote 配好：

```bash
git remote add upstream https://github.com/justlovemaki/AIClient-2-API.git
git fetch upstream
git branch --set-upstream-to=origin/main main
```

以后你也可以手动同步：

```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

## 七、常见问题

### 1. 为什么推了代码没有自动构建镜像？

优先检查：

- 你是不是推到了 `main`
- `Actions` 是否已启用
- `Settings -> Actions -> General -> Workflow permissions` 是否是 `Read and write permissions`

### 2. 为什么 GHCR 推送失败？

优先检查：

- 工作流是否具有 `packages: write`
- 仓库名或用户名是否包含大写
- `ghcr.io/<owner>/<repo>` 是否按工作流里那样自动转成了小写

### 3. 为什么 fork 同步失败？

常见原因：

- `main` 分支有保护，Actions 无法 push
- 你的 fork 改动和上游改动冲突，需要手动合并
- 当前仓库并不是 GitHub fork，而是普通 clone 后新建的仓库
