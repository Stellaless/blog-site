+++
date = '2026-08-11T10:35:40+08:00'
draft = false
title = '使用hugo和cloudflare搭建blog'
+++

# Hugo + cloudflare 个人博客完整搭建

## 一、前期准备：环境安装

1. 安装 Hugo Extended 版本
   - 踩坑：只装普通hugo，缺少样式编译依赖，PaperMod页面会完全乱码、无样式
   - 校验命令：`hugo version`，输出必须带 `extended` 字样

## 二、新建 Hugo 站点

执行创建命令：

```
hugo new site blog-site
```

- 可选参数 `--format yaml`：新建时直接生成 `hugo.yaml` 配置文件
- 踩坑：创建时没加yaml参数，默认生成 `config.toml`，后续PaperMod官方示例全是yaml，两种配置文件共存会冲突； 解决：备份旧toml `mv config.toml config.toml.bak`，单独新建 `hugo.yaml`

## 三、挑选 Hugo 主题 PaperMod

### 1. 主题挑选渠道

1. 官方主题站：https://themes.gohugo.io/ 筛选博客类、高更新、轻量化主题
2. GitHub 搜索 `hugo blog theme`，优先选Star高、近半年有提交的项目，废弃老主题不要用

### 2. PaperMod 主题获取（3种方式，对应踩坑）

#### 方式1 Git Clone / Git Submodule（命令拉取）

```
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

- 踩坑1：国内直连GitHub报 `Connection was reset` 连接重置，浏览器能访问但Git不走代理； 解决：1. Windows Git不支持socks5代理，不要配置socks；2. ghproxy镜像地址后来服务器失效不可用；3. 最终改用SSH协议彻底绕开HTTPS网络问题
- 踩坑2：直接git clone主题后执行`git add .`，提示`embedded git repository`嵌套仓库； 解决：`git rm --cached themes/PaperMod && rm -rf themes/PaperMod/.git`，把主题转为普通文件提交仓库，新手不推荐submodule

#### 方式2 手动下载ZIP解压（网络差首选）

从GitHub下载master压缩包，解压到`themes/`

- 踩坑：解压文件夹名是`hugo-PaperMod`，配置里写`theme: PaperMod`名称不匹配，报`module not found`； 解决：`mv themes/hugo-PaperMod themes/PaperMod` 重命名文件夹

## 四、站点核心配置 hugo.yaml

### 1. 基础必配模板

```
baseURL: "/"
title: "个人技术博客"
theme: "PaperMod"
locale: zh-CN # 踩坑：旧写法languageCode会弹出废弃警告
hasCJKLanguage: true
outputs:
  home: ["HTML", "RSS", "JSON"] # 站内搜索必备，缺失搜索页面失效
menu:
  main:
    - name: 首页
      url: /
      weight: 1
params:
  defaultTheme: auto
  search: true
```

- 踩坑：保留`languageCode: zh-cn`，启动持续弹出deprecated警告；替换为`locale: zh-CN`消除提示

### 2. 强制创建搜索页面（PaperMod依赖）

```
hugo new content/search.md
```

search.md标准内容：

```
---
title: "站内搜索"
layout: search
url: /search/
draft: false
---
```

- 踩坑1：frontmatter写错符号、用中文冒号/破折号，启动报`toml unmarshal failed`解析失败； 规范：必须用英文`---`包裹头部，冒号后带空格
- 踩坑2：不创建search.md，网页点击搜索按钮404

## 五、创建博客文章 & Frontmatter 语法坑

### 新建文章命令

```
hugo new content/posts/xxx.md
```

1. 踩坑：文章头部TOML语法拼写错误 `draft = flase`，少字母s，直接站点构建失败； 修正：`draft = false`
2. 踩坑：混用TOML(+++)与YAML(---)两种头部格式，解析不稳定；统一全站使用YAML格式
3. 踩坑：启动不加`-D`，草稿文章`draft: true`不会渲染；预览草稿用 `hugo server -D`

## 六、Git 版本管理全流程踩坑

### 1. 本地文件提交流程

1. 新增/修改文件后 `git add .` 加入暂存区
2. 后续新增单个文件，单独 `git add 文件名` 或重复 `git add .` 无副作用
3. `git commit -m "备注信息"` 生成本地版本

- 踩坑：只add不commit，文件仅在缓存，无法推送远程仓库

### 2. 推送GitHub远程仓库（重点网络大坑）

1. 初始问题：HTTPS推送报 

   ```
   Recv failure: Connection was reset
   ```

   - 尝试方案1：配置socks5代理 → Windows Git ServicePointManager不支持socks5，直接报错
   - 尝试方案2：ghproxy镜像中转地址 → 镜像服务器超时无法连接
   - 最终稳定方案：切换SSH协议推送，全程无需代理

2. SSH 完整配置避坑步骤

```
# 1. 清理所有代理、删除旧origin
git config --unset-all http.proxy
git config --unset-all https.proxy
git remote remove origin
# 2. 生成ssh密钥
ssh-keygen -t ed25519
# 3. 配置SSH走443端口（国内22端口封禁）
echo "Host github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519" >> ~/.ssh/config
# 4. 绑定SSH远程仓库
git remote add origin git@github.com:Stellaless/blog-site.git
# 5. 首次推送绑定分支
git push -u origin main
```

- 踩坑：之前执行`git remote remove origin`后直接push，提示`origin不是git仓库`；解决重新add origin绑定地址
- 踩坑：HTTPS推送弹窗输入密码，GitHub不支持账号密码，必须在GitHub后台生成Personal access token令牌填写

### 3. 换行符警告

```
LF will be replaced by CRLF
```

- 说明：Windows Git自动转换换行，无害，可忽略；想关闭执行 `git config core.autocrlf true`

## 七、本地预览启动命令

```
# 正常发布文章预览
hugo server
# 包含草稿文章预览
hugo server -D
```

- 访问地址：[http://127.0.0.1:1313](http://127.0.0.1:1313/)
- 踩坑：主题文件夹名称和配置`theme: xxx`大小写、字符不一致，直接提示主题不存在构建失败

## 八、搭建完成后成果

1. 完整Hugo博客项目，PaperMod极简黑白主题，支持明暗切换、站内搜索、文章目录、代码复制
2. 本地可实时预览写文章，Git完整托管代码，成功推送到GitHub远程仓库
3. 后续拓展：GitHub Pages自动部署、增加评论系统、自定义首页Profile头像模式、配置导航栏与社交图标
