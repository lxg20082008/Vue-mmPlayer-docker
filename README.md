# Vue-mmPlayer-docker

基于 [Vue-mmPlayer](https://github.com/maomao1996/Vue-mmPlayer) 前端 + [NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi) 后端的网易云音乐网页播放器，打包为**单容器** Docker 镜像。

## 源仓库选择（改造说明）

| 部分 | 使用仓库 | 说明 |
| :--- | :--- | :--- |
| 前端 | [lxg20082008/Vue-mmPlayer](https://github.com/lxg20082008/Vue-mmPlayer) | fork 自上游 maomao1996，紧跟最新版，仅多一处 `src/App.vue` 标题改动（「听吧!唱吧!」提示语） |
| 后端 | [wujiyu115/NeteaseCloudMusicApi](https://github.com/wujiyu115/NeteaseCloudMusicApi) | fork 自上游 Binaryify，维护最勤，**默认端口已改为 5001** |

> 为什么这么选：原来的 `wujiyu115/Vue-mmPlayer` 前端是旧 webpack 4 架构、与上游严重分叉（ahead 131 / behind 125），故前端改用 lxg 的最新版；后端方面 lxg 与 wujiyu 是父子关系、只差 1~2 个提交，选维护更勤的 wujiyu。

## 架构：为什么是单容器

单个容器内同时运行三部分，由 `docker-entrypoint.sh` 启动：

- **nginx**：提供前端静态文件（`dist`），并把 `/api/*` 反代给后端
- **NeteaseCloudMusicApi**：`node app.js`，默认监听 `127.0.0.1:5001`
- **check.sh**：每 30 秒检测后端进程，挂了自动拉起

关键原因：后端默认绑定 `127.0.0.1`，在**同一个容器**里 nginx 用 `localhost:5001` 就能直连；若拆成两个容器，反而要额外设 `HOST=0.0.0.0` 并改 `proxy_pass` 为服务名。因此单容器最省事、最稳。

## 请求链路与端口

```
浏览器 ──> nginx(:80，宿主机映射 32108) ──> 后端(容器内 localhost:5001)
```

只暴露前端 nginx 端口，后端 5001 不对外。

## 配置说明

| 配置点 | 位置 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| 前端 API 地址 | 前端 `.env` → `VUE_APP_BASE_API_URL` | `/api` | 相对路径 = 同源走 nginx 反代；也可填绝对地址 `http://<IP>:5001` 直连后端 |
| 后端反代端口 | `default.conf` → `proxy_pass` | `http://localhost:5001/` | 需与后端端口一致 |
| 后端端口 | 环境变量 `PORT` | `5001` | fork 已把上游默认 3000 改成 5001 |
| 后端监听地址 | 环境变量 `HOST` | `127.0.0.1` | 单容器内保持默认即可 |

> ⚠️ `.env` 是**构建期**配置：`VUE_APP_BASE_API_URL` 在 `npm run build` 时就编译进前端 JS，容器跑起来后改 `.env` 文件不会生效，要改需重新 build。运行期可调的是 nginx `default.conf`（挂载覆盖）和后端 `PORT`/`HOST`（环境变量）。

## 构建

```bash
docker build -t vue-mmplayer .
```

## 运行

```bash
# 映射到宿主机 32108（80 端口被封时用非 80 端口）
docker run --name mm_player --restart always -d -p 32108:80 vue-mmplayer
```

访问 `http://<服务器IP>:32108`。

- 若 80 端口可用，可直接 `-p 80:80`。
- `-v /mnt/music:/data` 用于挂载本地音乐，纯在线播放不需要。

## 架构兼容性

- 基础镜像 `node:lts-alpine` 是多架构镜像，**x86_64 与 ARM（树莓派）均可**构建运行。
- 前端使用 `@vue/cli-service ~5.0.8`（webpack 5），在新版 Node 上构建**不会**遇到老 webpack4 的 `openssl-legacy-provider` 报错。

## 版本

1.3
