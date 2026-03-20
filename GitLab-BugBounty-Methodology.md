# GitLab BugBounty 挖掘方法论

## 一、信息收集

### 1.1 漏洞报告来源
| 来源 | 链接 | 用途 |
|------|------|------|
| GitLab Issues | https://gitlab.com/gitlab-org/gitlab/-/issues |
| HackerOne 报告 | https://hackerone.com/groups/gitlab/reports |
| 漏洞作者 GitHub | https://gitlab.com/gitlab-org/gitlab/-/issues/?sort=created_date&state=all&search=msaleiko |

### 1.2 新功能跟踪
| 资源 | 链接 | 说明 |
|------|------|------|
| 官方文档 | https://docs.gitlab.com/ee/user/permissions.html | 权限配置参考 |
| YouTube 更新 | https://www.youtube.com/@Gitlab/videos | 功能演示 |
| 更新日志 | https://about.gitlab.com/releases/#upcoming-releases | 版本发布计划 |
| 仓库 releases | https://gitlab.com/gitlab-org/gitlab/-/releases | 具体版本变更 |

### 1.3 版本发布日程
| Version | Release Date |
|---------|--------------|
| 17.8 | January 16th, 2025 |
| **17.9** | **February 20th, 2025** |
| 17.10 | March 20th, 2025 |
| 17.11 | April 17th, 2025 |
| 18.0 | May 15th, 2025 |

---

## 二、环境搭建

### 2.1 Docker 安装 (Ubuntu)

```bash
# 卸载旧版本
sudo apt-get remove docker docker-engine docker.io containerd runc

# 安装必要支持
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release

# 添加阿里云 GPG key
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 添加阿里云 apt 源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

# 配置镜像加速
sudo tee /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "https://docker.m.daocloud.io/",
        "https://huecker.io/",
        "https://docker.nju.edu.cn",
        "https://xx4bwyg2.mirror.aliyuncs.com"
    ]
}
EOF

sudo systemctl restart docker
```

### 2.2 GitLab 部署
| 部署方式 | 资源 |
|----------|------|
| Docker Compose | https://github.com/sameersbn/docker-gitlab |
| 官方镜像 | https://hub.docker.com/r/gitlab/gitlab-ee |
| Vulhub 漏洞环境 | https://github.com/vulhub/vulhub/tree/master/gitlab |

```bash
# 快速启动示例
docker run -d \
  --hostname gitlab.example.com \
  -p 8080:80 -p 2222:22 \
  --name gitlab \
  gitlab/gitlab-ee:17.2
```

---

## 三、漏洞挖掘方法论

### 3.1 核心思路：版本对比法

```
新版本发布 → 对比文档差异 → 发现新增功能/权限变更 → 测试权限绕过
```

### 3.2 权限研究框架

| 角色 | 权限等级 | 关注点 |
|------|----------|--------|
| Guest | 最低 | 是否有未授权访问 |
| Reporter | 只读 | 是否可读敏感信息 |
| Developer | 中等 | 是否存在越权操作 |
| Maintainer | 较高 | 是否可修改关键配置 |
| Owner | 最高 | 是否有账户接管风险 |

### 3.3 文档差异对比方法

1. **Wayback Machine 对比**
   - 访问：`https://web.archive.org/web/20250000000000*/https://docs.gitlab.com/ee/user/permissions.html`
   - 选择两个时间点进行对比

2. **HTML Diff 工具**
   - 使用 `htmldiff.js` 对比页面变化
   - 重点关注：新增功能、权限变更

### 3.4 对比记录模板

| 功能 | Guest | Reporter | Developer | Maintainer | Owner | 风险评估 |
|------|-------|----------|-----------|------------|-------|----------|
| 新增功能名 | | | | | | ⭐级别 |

---

## 四、漏洞案例分析

### 案例：Group Developers 可以查看 group runners

| 项目 | 信息 |
|------|------|
| 影响版本 | GitLab Enterprise Edition 17.2.0-pre |
| 严重程度 | 中等 |
| 问题类型 | 权限绕过 |
| 报告链接 | https://gitlab.com/gitlab-org/gitlab/-/issues/472012 |

#### 复现步骤

1. **文档监控**
   - 访问 https://docs.gitlab.com/ee/user/permissions.html#cicd-group-permissions
   - 发现 `group runners` 查看权限仅限 Main 和 Owner

2. **环境准备**
   - Firefox：Admin 账户
   - Chrome：Developer 账户
   - 创建 group runners

3. **漏洞验证**
   - Developer 账户无法看到 runners（正常）
   - 抓包获取 Admin 的 API 请求
   - 用 Developer 身份重放请求 → 成功访问（越权）

#### 关键发现

```
文档声明：Developer 无权查看 group runners
实际情况：通过 API 参数可绕过前端限制
结论：后端权限校验缺失
```

---

## 五、功能测试 Checklist

### 5.1 新功能测试项

- [ ] Import issues from CSV（Reporter+ 可操作）
- [ ] Export issues to CSV（Reporter+ 可操作）
- [ ] Manage Feature flags（Maintainer+ 可操作）
- [ ] Convert to another item type（所有角色）
- [ ] Edit & delete models/versions（Maintainer+）
- [ ] Edit & delete experiments（Maintainer+）

### 5.2 权限绕过测试点

| 测试场景 | 预期结果 | 实际结果 |
|----------|----------|----------|
| 低权限用户访问高权限 API | 拒绝访问 | ? |
| 修改请求参数提升权限 | 拒绝访问 | ? |
| 未授权访问隐藏功能 | 拒绝访问 | ? |

---

## 六、学习资源

### 6.1 HackerOne 热门报告作者
| 作者 | 主页 |
|------|------|
| msaleiko | GitLab Issues 搜索 |
| ashish_r_padelkar | https://hackerone.com/ashish_r_padelkar |

### 6.2 学习方法
1. 阅读前 20 个 HackerOne 报告
2. 总结报告中的常见漏洞模式
3. 对照官方文档验证漏洞根因
4. 在本地环境复现

---

## 七、版本对比记录

### 2024-12-26 vs 2025-01-26

| 变更类型 | 功能 | 风险 |
|----------|------|------|
| 新增 | Item locking resolving threads | ⭐ |
| 新增 | Edit epic | ⭐ |

### 2024-12-05 vs 2024-12-26

| 新增功能 | Guest | Reporter | Developer | Maintainer | Owner | 漏洞级别 |
|----------|-------|----------|-----------|------------|-------|----------|
| Edit & delete models, versions, artifacts | - | - | - | ✓ | ✓ | 权限扩大 ⭐ |
| Edit & delete experiments, candidates | - | - | - | ✓ | ✓ | 权限扩大 ⭐ |
| Import issues from CSV | - | ✓ | - | ✓ | ✓ | 权限问题 ⭐⭐⭐⭐ |
| Export issues to CSV | - | ✓ | ✓ | ✓ | ✓ | 新功能 ⭐⭐ |
| Manage Feature flags | - | - | - | ✓ | ✓ | 权限问题 ⭐⭐⭐⭐ |
| Convert to another item type | ✓ | ✓ | ✓ | ✓ | ✓ | 新功能 ⭐⭐ |

### 2024-11-05 vs 2024-12-05
- 新增 Planner 角色

---

## 八、快捷命令

### GitLab Docker 环境
```bash
# 启动
docker start gitlab

# 停止
docker stop gitlab

# 查看日志
docker logs -f gitlab

# 进入容器
docker exec -it gitlab bash
```
