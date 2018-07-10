# CoolPlayDruid



## Pull Request 流程

1. 首先 fork 该项目
2. 把 fork 过去的项目也就是你的项目clone到你的本地
3. 运行 `git remote add coolplaydata git@github.com:coolplaydata/coolplaydruid.git` 添加为远端库
4. 运行 `git pull coolplaydata master` 拉取并合并到本地
5. 修改内容
6. commit 后 push 到自己的库（`git push origin master`）
7. 登录 Github 在你首页可以看到一个 `pull request` 按钮，点击它，填写一些说明信息，然后提交即可。

1~3 是初始化操作，执行一次即可。在修改前请先执行步骤 4 同步最新的修改（减少冲突），然后执行 5~7 既可。
