1. 初始化仓库	

   ```shell
   git init
   ```

2. 查看仓库日志

   ```shell
   git log
   ```

3. 查看仓库状态

   ```shell
   git status
   ```

4. 添加文件

   ```shell
   git add filename
   ```

5. 添加所有文件

   ```shell
   git add -A .
   ```

6. 提交

   ```shell
   git commit -m "<commit message>"
   ```

7. 添加远程仓库地址

   ```shell
   git remote add origin <repository URL>
   ```

8. 推送到远程仓库

   ```shell
   git push -u origin master    // 注释:远程分支为origin，默认的本地分支为master，-u是记住提交参数，下次可以直接使用git push 进行提交
   ```

9. 拉取远程仓库

   ```shell
   git pull origin master     // 注释:同上
   ```

10. 查看最近一次提交的更改

    ```shell
     git diff HEAD		//注释:HEAD为指针,diff的另一个重要用途是查看已经暂存文件内的更改。记住，staged文件是我们告诉git已准备好提交的文件。
    ```

11. 取消暂存

    ```shell
    git reset <filename>	//注释:从暂存区移除指定文件
    ```

12. 撤销

    ```shell
    git checkout -- <filename>	//注释:重新检出某个文件，恢复为最后一次commit的状态
    ```

13. 创建分支

    ```shell
    git branch <new_branch>
    ```

14. 切换分支

    ```shell
    git checkout <branch>
    ```

15. 创建并切换分支

    ```shell
    git checkout -b <new_branch>   //注释:13、14两条命令的合并
    ```

16. 移除文件

    ```shell
    git rm '<filename>'		//注释：此操作会同时从磁盘，暂存区移除指定文件
    ```

17. 合并分支到master

    ```shell
    git merge <branch>
    ```

18. 移除分支

    ```shell
     git branch -d <branch>
    ```

19. 修改已存在仓库的URL

    ```shell
    git remote rm origin
    git remote add origin [url]
    ```

20. 设置提交的用户name和email

    ```shell
    git config user.email personal@example.org
    git config user.name "whatf hobbyist"    //注释：如果需要全局设置可以使用 --global
    ```

21. 相关链接

[try git](https://try.github.io/levels/1/challenges/1)

[Pro Git 中文版](https://git-scm.com/book/zh/v2)