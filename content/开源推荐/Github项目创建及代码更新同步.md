
# 一、GitHub 创建新仓库

## github上操作：

GitHub 创建新仓库，不要使用 `README` 、license 或 `gitignore` 文件初始化新存储库。

## 在本地进行操作：

Win + R ，输入 `cmd` 打开命令提示符。  
输入 `D:` 进入 D 盘，依次输入下面命令，会自动在 D 盘创建 【DBPatroller】 文件夹

```bash
git clone https://github.com/vcdog/DBPatroller.git
cd DBPatroller
```


在 GitHub 存储库的页面，单击剪贴板以复制远程存储库 SSH 链接
![[images/Pasted-image-20240906120103.png]]

进入文件夹的根目录【D:\\DBPatroller】,右键git bash here，执行如下命令：

```bash
# git remote add origin SSH链接
 git remote add origin git@github.com:vcdog/DBPatroller.git
 
# 如果报错 error: remote upstream already exists.
# 那么就删除现有远程仓库 
# git remote remove origin

# git remote add origin SSH链接
git remote add origin git@github.com:vcdog/DBPatroller.git
git remote -v
```

# 二、将在GitHub上更改的内容更新到本地

1、在GitHub上修改完后记得保存

2、进入你本地建立的文件夹，右键git bash here

3、输入pwd确认位置后，输入git pull origin

4、查看本地是否更新成功

# 三 、将在本地更新后的代码上传到GitHub

1、更改本地文件

2、在本地文件夹下右键git bash here，

3、输入pwd确认位置

4、输入git add .

5、输入git commit -m ‘update’

6、输入git push （或 git push -u origin main）

7、打开GitHub查看是否成功


要删除远程 Git 仓库中的某个分支，可以使用以下命令：
```bash
$ git push origin --delete <branch-name>


$ git push origin --delete master
To github.com:vcdog/DBPatroller.git
 - [deleted]         master

$ git branch -r
  origin/main

```


```bash
# 创建标签
git tag -a v1.0.1 -m "Release version 1.0.1"

# 推送标签到远程仓库
git push origin v1.0.1

# 或者推送所有标签
git push origin --tags

```