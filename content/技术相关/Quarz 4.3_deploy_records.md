
> “Quartz 是一款快速、具有丰富插件的静态站点生成器，可将 Markdown 内容转换为功能齐全的网站。成千上万的学生、开发人员和教师已经在使用 Quartz 将个人笔记、网站和数字花园发布到网络上。
> 
> - Obsidian 兼容性，全文搜索，关系图谱，注释嵌入，Wiki 链接，反向链接，Latex，语法高亮，
> - 悬浮预览窗，支持 Docker，以及更多开箱即用的功能
> - 配置文件和内容的热重载
> - 简单的 JSX 布局和页面组件
> - 极快的页面加载速度和极小的捆绑包大小
> - 通过插件可以完全定制内容的处理、过滤和页面生成”

官方文档：[Welcome to Quartz 4](https://quartz.jzhao.xyz/)   
非官方中文翻译：[Quartz 4](https://cmbill.github.io/quartz-doc-cn/#-get-started) 

## Node.js

Quartz 至少需要 [Node](https://nodejs.org/)   v18.14 和 `npm` v9.3.1 才能正常运行。  
参考文章 → [Node.js安装及环境配置超详细教程【Windows系统】](https://blog.csdn.net/Nicolecocol/article/details/136788200) 

Win + R ，输入 `cmd` 打开命令提示符。  
输入 `D:` 进入 D 盘，依次输入下面命令，会自动在 D 盘创建 【quartz】 文件夹

```bash
git clone https://github.com/jackyzha0/quartz.git
cd quartz
npm i
npx quartz create
```



初始化完成后，输入下面命令，即可本地预览 http://localhost:8080/

```bash
npx quartz build --serve
```



本地预览时如端口被占用，可先查询 8080 端口使用情况，记住数字 id，执行第二行命令即可。

```bash
netstat -ano | findstr :8080
taskkill -pid <id> -f
```


打开 Obsidian —— Manage vaults —— 打开本地仓库，选择【D:\\git_quartz\\quartz】文件夹打开。  
主页的内容编辑在 `content/index.md` ，所有想发布的文章都应该在【content】文件夹内。

## Github

GitHub 创建新仓库，不要使用 `README` 、license 或 `gitignore` 文件初始化新存储库。

在 GitHub 存储库的 Quick Setup 页面顶部，单击剪贴板以复制远程存储库 SSH 链接

进入文件夹的根目录【D:\quartz】运行以下命令

```bash
# git remote add origin SSH 链接
 git remote add origin git@github.com:vcdog/quartz.git
# 如果报错 error: remote upstream already exists.
# 那么就删除现有远程仓库 git remote remove origin
# git remote add origin SSH 链接
git remote add origin git@github.com:vcdog/quartz.git
git remote add upstream https://github.com/jackyzha0/quartz.git
git remote -v
```


同步您的内容到 Github 存储库

```bash
npx quartz sync --no-pull
```



> 这步可能报错，原因是没正确配置 SSH 密钥：  
> [git@github.com](mailto:git@github.com) : Permission denied (publickey). fatal: Could not read from remote repository. Please make sure you have the correct access rights and the repository exists. An error occurred above while pushing to remote origin.

### SSH 密钥

同步内容到 GitHub 时，会要求 SSH 密钥。

默认情况下，SSH 密钥存储在 【~/.ssh】 目录

#### 1. 检查是否存在

首先，检查系统上是否已经生成了 SSH 密钥。默认情况下，SSH 密钥存储在 `~/.ssh` 目录下。可以通过以下命令检查是否存在 SSH 密钥：

```sh
ls -al ~/.ssh
```

Copy

应该能看到类似 `id_rsa` 和 `id_rsa.pub` 的文件（或者 `id_dsa` 和 `id_dsa.pub`，取决于你使用的密钥类型）。

#### 2. 生成新的

如果没有找到 SSH 密钥，或者想生成一个新的密钥，使用以下命令：

```sh
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```



按照提示操作，你可以选择默认的文件名和位置，或者指定一个不同的文件名和位置。

#### 3. 添加到 ssh-agent

确保 `ssh-agent` 正在运行，并将 SSH 密钥添加到 `ssh-agent` 中：

```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```



#### 4. 添加到 GitHub 账户

1. 复制 SSH 公钥到剪贴板。可以使用以下命令：

```sh
cat ~/.ssh/id_rsa.pub
```


2. 登录 GitHub 账户，进入 Settings —— SSH and GPG keys —— New SSH key，粘贴复制的公钥，并添加一个描述性的标题。

#### 5. 测试 SSH 连接

```sh
ssh -T git@github.com
```


如果一切正常，应该会看到类似以下的消息：  
Hi username! You’ve successfully authenticated, but GitHub does not provide shell access.

## Cloudflare 托管

> 我选择了 Cloudflare，还有 GitHub Pages、Vercel、Netlify、GitLab Pages 多种选择。

1. 登录 [Cloudflare 仪表板](https://dash.cloudflare.com/) 
2. 在帐户主页中，选择 Workers & Pages —— Create —— Pages —— Connect to Git
3. 选择刚才创建的 GitHub 存储库 —— Begin setup，并在设置构建和部署部分中按照下列表格填写信息 —— Save and Deploy —— Custom domains 绑定子域名即可

|配置选项|值|
|---|---|
|生产分支|`v4`|
|框架预设|`无`|
|构建命令|`npx quartz build`|
|构建输出目录|`public`|

![](https://pic.yelleis.top/try/202407022120605.webp)

## 样式美化

配色参考网站：[AI Color Palette Generator](https://aicolors.co/ "可以根据你的描述生成贴心的配色方案，同时你也可以参考提供的配色案例，每一种配色方案都有对应的效果展示，让你能够快速找到想要自己的配色方案，提供四种识图案例，分别是移动应用、仪表盘、落地页、作品集，你可以单击复制颜色或者直接导出配色方案，不喜欢的颜色还可以手动修改。") 

滚动条、代码块限制高度等修改可以参考我的这篇文章 → [hugo stack 主题美化](https://yelleis.top/p/hugo-theme-stack-beautification/) 

网站图标更换 quartz/static/icon.png  
把原图片重命名为 icon1.png，把想用的图标文件重命名为 icon.png，并放到上面的路径中。更换后强制刷新页面（Ctrl + F5 或 Ctrl + Shift + R）查看效果。