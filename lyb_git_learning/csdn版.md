# Git文件的三种状态与工作模式

使用Git操作文件，文件的状态有三种：

<img src="https://img-blog.csdnimg.cn/20210928202204651.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASmFudWFyeUZN,size_20,color_FFFFFF,t_70,g_se,x_16" alt="img" style="zoom:50%;" />



针对Git 文件的三种状态，这里需要了解Git项目的三个工作区域:**工作区、暂存区和Git仓库。**

<img src="https://img-blog.csdnimg.cn/20210928202311163.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASmFudWFyeUZN,size_20,color_FFFFFF,t_70,g_se,x_16" alt="img" style="zoom:50%;" />

**基本的Git工作流程描述如下：**

1. **在工作区中修改某些文件。**
2. **对修改后的文件进行快照，然后添加到暂存区。**
3. **提交更新，将保存在暂存区域的文件快照永久转储到 Git 仓库中。**



创建仓库并提交文件的步骤:

1. `git init`命令在本地初始化一个本地仓库(空仓库)
2. 创建文件,并使用` git status` 查看**文件状态**

![img](https://img-blog.csdnimg.cn/20210928203817524.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASmFudWFyeUZN,size_20,color_FFFFFF,t_70,g_se,x_16)

此时红色的文件仍在暂存区



3. 执行 `git add 文件名` 命令添加文件到暂存区

通常是通过`git add <path>`的形式把`<path>`添加到索引库中，`<path>`可以是**文件**也可以是目录。

 git不仅能判断出`<path>`中，修改(不包括已删除)的文件，还能判断出新添的文件，并把它们的信息添加到索引库中。![img](https://img-blog.csdnimg.cn/20210928204505316.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASmFudWFyeUZN,size_20,color_FFFFFF,t_70,g_se,x_16)

此时可以看到有一个git 已tracked 到新文件git01.txt，文件被成功存放到暂存区



4. 文件被添加到暂存区后，执行 `git commit` 命令提交暂存区文件到本地版本库中。

`git commit -m +注释`  加入注释便于查看版本更新和回退  

5.  `git log `命令用于显示提交日志信息。

而接下来修改文件本身,再次添加,提交

6. 此时执行 `git diff HEAD -- git01.txt `

将暂存区的内容与版本库内容进行比较,结果如下:

![img](https://img-blog.csdnimg.cn/20210928210456637.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASmFudWFyeUZN,size_20,color_FFFFFF,t_70,g_se,x_16)

HEAD版本通过`git checkout 分支`   来改变(类似于一个指针)

 ---：表示变动前的文间

 +++：表示变动后的文件

7. `git reset HEAD` 撤销操作

撤销将文件添加到暂存区的操作(git add)

```
git reset HEAD^            # 回退所有内容到上一个版本  
git reset HEAD^ hello.php  # 回退 hello.php 文件的版本到上一个版本  
git  reset  052e           # 回退到指定版本
```