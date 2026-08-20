# V10 Vue Admin 部署说明

本项目由林汐制作 好用就来个star 本产品无后门 放心食用

## 1. 架构说明

| 组件 | 说明 |
|------|------|
| 本仓库前端 | Vue 3 + Vben Admin，构建为静态资源（`apps/web-antd/dist`） |
| V10 后端 | 既有 ThinkPHP 服务，提供 `/{DIR_ADMIN}/v1` REST API |
| 鉴权 | 登录返回 JWT，请求头 `Authorization: Bearer <jwt>` |

前端不包含 PHP 业务逻辑。连接参数（后台目录、接口站点、登录账号）可在登录页右上角「连接设置」中配置并保存到浏览器本地。

默认后台目录：`rjhyyqbu`  
默认接口路径：`/{后台目录}/v1`

---

## 2. 环境要求

- Node.js `^22.18` 或 `^24.12`
- pnpm `>= 11`
- 可访问的 V10 实例（PHP + MySQL）
- 生产环境建议 Nginx / Caddy / Apache 反向代理

---

## 3. 获取与构建

```bash
git clone <your-repo-url>
cd vue-vben-admin-main
pnpm install
pnpm build:antd
```

构建产物：

- `apps/web-antd/dist/` — 静态站点目录
- `apps/web-antd/dist.zip` — 压缩包（若开启归档）

生产环境变量见 `apps/web-antd/.env.production`：

```env
VITE_BASE=/
VITE_GLOB_API_URL=/rjhyyqbu/v1
VITE_ROUTER_HISTORY=hash
```

说明：运行时「连接设置」可覆盖实际请求的后台目录与接口站点；`VITE_GLOB_API_URL` 仅作构建期默认值。

若静态资源部署在子路径（例如 `/vben/`），需设置：

```env
VITE_BASE=/vben/
```

后重新执行 `pnpm build:antd`。

---

## 4. 生产部署（Nginx 同源）

假设：

- 静态文件目录：`/var/www/v10-admin`（内容为 `dist`）
- V10 PHP 监听：`127.0.0.1:9212`
- 后台目录：`rjhyyqbu`（以实际为准）

```nginx
server {
    listen 80;
    server_name admin.example.com;

    root /var/www/v10-admin;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~ ^/[^/]+/v1/ {
        proxy_pass http://127.0.0.1:9212;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Authorization $http_authorization;
        proxy_read_timeout 120s;
        client_max_body_size 50m;
    }

    location ~ ^/[^/]+/(plugin|template)/ {
        proxy_pass http://127.0.0.1:9212;
        proxy_set_header Host $host;
        proxy_set_header Authorization $http_authorization;
        proxy_set_header Cookie $http_cookie;
    }

    location ~ ^/[^/]+/.*\.htm$ {
        proxy_pass http://127.0.0.1:9212;
        proxy_set_header Host $host;
        proxy_set_header Authorization $http_authorization;
        proxy_set_header Cookie $http_cookie;
    }

    location /upload/ {
        proxy_pass http://127.0.0.1:9212;
        proxy_set_header Host $host;
    }

    location /plugins/ {
        proxy_pass http://127.0.0.1:9212;
        proxy_set_header Host $host;
    }
}
```

发布步骤：

1. `pnpm build:antd`
2. 将 `apps/web-antd/dist/*` 同步到 `/var/www/v10-admin/`
3. `nginx -t && systemctl reload nginx`（或等价命令）
4. 浏览器打开站点 → 右上角「连接设置」确认后台目录 → 登录

HTTPS 请自行配置证书（Let’s Encrypt / 云厂商证书）。

---

## 5. GitHub 发布建议

推荐仓库内容：

```text
vue-vben-admin-main/
  apps/web-antd/          # 业务前端
  packages/               # 框架依赖
  pnpm-workspace.yaml
  package.json
  README.md
  apps/web-antd/DEPLOY.md
```

不建议提交：

- `node_modules/`
- `apps/web-antd/dist/`（可用 GitHub Actions 构建产物）
- `.env.development.local`、含真实密码的本地文件

`.gitignore` 至少包含：

```gitignore
node_modules
dist
dist.zip
*.local
.DS_Store
```

首次推送示例：

```bash
git init
git add .
git commit -m "feat: V10 admin frontend based on vue-vben-admin"
git branch -M main
git remote add origin git@github.com:<user>/<repo>.git
git push -u origin main
```

可选 CI（`.github/workflows/build.yml`）在 push 时执行 `pnpm install && pnpm build:antd`，并将 `dist` 上传为 Artifact。

---

## 6. 连接设置说明

登录页 / 后台右上角「连接设置」：

| 字段 | 含义 |
|------|------|
| 后台目录 | V10 的 `DIR_ADMIN`，如 `rjhyyqbu` |
| 接口站点 | 留空表示与当前页面同源；跨机调试可填 `http://127.0.0.1:9212` |
| 登录账号 / 密码 | 可选保存，勾选「记住密码」后写入浏览器 localStorage |

配置保存在浏览器本地（`localStorage`），不会写入服务器。更换电脑或清除站点数据后需重新填写。

---

## 7. 验收清单

- [ ] 静态资源可访问，登录页正常
- [ ] 「连接设置」中后台目录与 V10 一致
- [ ] 登录成功，侧边栏菜单加载
- [ ] 用户列表等业务接口返回 200
- [ ] `/upload` 图片可显示（若使用上传资源）

---

## 8. 常见问题

| 现象 | 排查 |
|------|------|
| 登录网络错误 | 反代是否匹配 `/{目录}/v1`；接口站点是否可达 |
| 401 反复跳登录 | JWT 失效或 Authorization 头被代理剥离 |
| 子路径页面空白 | `VITE_BASE` 是否与部署路径一致并已重新构建 |
| 开发改目录无效 | 确认 Vite 代理含 `^/[^/]+/v1`，并已重启 `pnpm dev:antd` |

本地开发：

```bash
pnpm dev:antd
```

默认地址：`http://localhost:5666`
