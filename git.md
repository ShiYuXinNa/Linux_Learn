# 关于GitHub的指令 #

## 上传GIT仓库

1. 本地创建ssh key，在GitHub的设置-SSH中添加ssh-key

   - ```shell
     ssh-keygen -t rsa # 生成ssh-key
     cat  /c/Users/[用户名]/.ssh/id_rsa.pub # 查看ssh-key
     ssh -T git@github.com # 验证是否成功
     ```

2. 设置用户名和邮件

   - ```shell
     git config --global user.name "Your Name"
     git config --global user.email "email@example.com"
     ```

3. 初始化git

   - ```shell
     git init
     ```

4. 将当前文件夹下所有文件添加至暂存区

   - ```shell
     git add .
     ```
     

5. 提交文件至仓库（`git status` 可查看文件状态，红色为没有提交，绿色表示添加到本地仓库）

   - ```shell
     git commit -m "提交描述"
     ```

6. 将分支名更改为main（如果默认分支名为master）

   - ```shell
     git branch -M main
     ```

7. 将本地仓库与远程仓库关联起来

   - ```shell
     git remote add origin git@github.com:用户名/仓库名.git
     ```

8. 将本地的main分支推送到远程的origin仓库

   - ```shell
     git push -u origin main
     ```



## 克隆仓库 ##

    git clone <个人的仓库地址>



## 查看仓库状态 ##

    git status


## 文件操作 ##

### 添加文件 ###

    git add <文件名/文件路径>

### 删除文件 ###

    git rm <文件名/文件路径>

## 提交暂存区 ##

    git commit -m [提交信息]

## 查看提交日志 ##

	git log

## 上传远程仓库 ##

	git push <远程主机名> <本地分支名>:<远程分支名>

