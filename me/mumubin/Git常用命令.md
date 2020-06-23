1. 清理本地已经仍存在，远程已删除的分支
```shell script
git branch -vv | grep 'origin/.*: gone]' | awk '{print $1}' | xargs git branch -d
``` 

2. 建议用git fetch,git merge来替代git pull

3. 删除远程分支
```shell script
git push origin --delete serverfix
```

4.抛弃stage区域当前的变更

```shell script
git restore test.rb
```

5. 如果提交存在于你的仓库之外，而别人可能基于这些提交进行开发，那么不要执行变基

6. 如果你习惯使用 git pull ，同时又希望默认使用选项 --rebase，你可以执行这条语句 git config --global pull.rebase true 来更改 pull.rebase 的默认配置
```shell script
git pull --rebase
=> 
git fetch
git rebase teamone/master
``` 
   
7. [Fork工作流怎么用](https://git-scm.com/book/zh/v2/%E5%88%86%E5%B8%83%E5%BC%8F-Git-%E5%90%91%E4%B8%80%E4%B8%AA%E9%A1%B9%E7%9B%AE%E8%B4%A1%E7%8C%AE)
    1. clone 原始库
    2. git remote add myfork <url> /url为私有库
    3. git request-pull 命令然后手动地将输出发送电子邮件给项目的维护者。
    ```shell script
    git request-pull origin/master myfork
    ```
   
8. [通过邮件打补丁,提交code](https://git-scm.com/book/zh/v2/%E5%88%86%E5%B8%83%E5%BC%8F-Git-%E5%90%91%E4%B8%80%E4%B8%AA%E9%A1%B9%E7%9B%AE%E8%B4%A1%E7%8C%AE)
    ```shell script
    git format-patch -M origin/master
    ```
   
9. 查看两个分支上的差异提交
```shell script
git log contrib --not master
git log master..contrib
```

10. - git diff master...contrib
        与从master拉出contrib时的节点进行比较
    - git diff master
        与当前master比较

11. [Git维护者](https://github.com/git/git/blob/master/Documentation/howto/maintain-git.txt)

12. git rerereRerere

13. 准备一次发布
```shell script
git archive master --prefix='project/' | gzip > `git describe master`.tar.gz
```

14. 生成发布的简报
```shell script
git shortlog --no-merges master --not v1.0.1
```

15. Fork库和源库保持更新
```shell script
$ git checkout master (1)
$ git pull https://github.com/progit/progit2.git (2)
$ git push origin master (3)
```
- 如果在另一个分支上，就切换到 master
- 从 https://github.com/progit/progit2.git 抓取更改后合并到 master
- 将 master 分支推送到 origin

```shell script
$ git remote add progit https://github.com/progit/progit2.git (1)
$ git branch --set-upstream-to=progit/master master (2)
$ git config --local remote.pushDefault origin (3)

```
- 添加源仓库并取一个名字，这里叫它 progit
- 将 master 分支设置为从 progit 远端抓取
- 将默认推送仓库设置为 origin


16. 获取分支hash值
```shell script
git rev-parse topic1```

17. 就会显示昨天 master 分支的顶端指向了哪个提交。
```shell script
git show master@{yesterday}
```

18. 祖先提交，HEAD~ 和 HEAD^ 是等价的
```shell script
git show HEAD^
```

19.  查看 在 experiment 分支中而不在 master 分支中的提交
```shell script
git log master..experiment
```

20. 查看所有被 refA 或 refB 包含的但是不被 refC 包含的提交
```shell script
$ git log refA refB ^refC
$ git log refA refB --not refC
```

21. 这个语法可以选择出被两个引用 之一 包含但又不被两者同时包含的提交
```shell script
git log master...experiment
```
## Refrences
- [git-scm](https://git-scm.com/book/en/v2)