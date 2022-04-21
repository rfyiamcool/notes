## git规范的必要性

#### 分支管理

- 代码提交在应该提交的分支
- 随时可以切换到线上稳定版本代码
- 多个版本的开发工作同时进行

#### 记录的可读性

- 所有commit必须有注释，**内容必须按照注释格式严格执行！**
- 正确为每个项目设置Git提交用到的user.name和user.email信息，以公司邮箱为准，不可随意设置以影响无法正确识别。 查看当前项目配置信息的命令：`git config -l`

#### 版本号(tag)

- 版本号(tag)命名规则
  主版本号.次版本号.修订号，如2.1.13。(遵循GitHub语义化版本命名规范)
- 版本号仅标记于master分支，用于标识某个可发布/回滚的版本代码
- 对master标记tag意味着该tag能发布到生产环境
- 对master分支代码的每一次更新(合并)必须标记版本号
- 仅项目管理员有权限对master进行合并和标记版本号

## 语义化版本

![img](https://miro.medium.com/proxy/1*X0AvlnrTy5Vt2-cGiJCSvg.png)

语义化版本:

- 主版本号，重大升级改动，一般 API 的兼容性变化时，包括但不限于新增特性、修改机制、删除功能， 一般不兼容上一个主版本号.

- 次版本号 (minor)，新增或修改功能时，必须是向下兼容，不影响软件的整体流程或概念的更新.

- 修订号(patch)，当你做了向下兼容的问题修正，可以理解为Bug fix版本。

版本例子:

```
v1.0.0-alpha.0
v1.0.0-alpha.1
v1.0.0-beta.0
v1.0.0-rc.0
v1.0.0
```

label:

- alpha：内部测试版

- beta：公开测试版

- rc: 候选版本, 不再增加新功能.

## 分支管理规范

### Git-Flow

![](https://xiaorui.cc/image/2020/20210416171419.png)

### 分支说明

| 名称    | 说明                                     | 命名规范       | 命名示例          | 合并目标 | 合并操作      |
| ------- | ---------------------------------------- | -------------- | ----------------- | -------- | ------------- |
| master  | 线上稳定版本                             | master         | master            | --       | --            |
| release | 预发布分支                               | release/xxx    | release/v1.0.0    | master   | merge request |
| develop | 当前正在开发的分支                       | develop        | develop           | master   | merge request |
| feature | 功能分支, 每个功能需分别建立自己的子分支 | feature/功能名 | feature/login     | develop  | merge request |
| hotfix  | 紧急修复分支                             | hotfix/xxx     | hotfix/v1.0.1-xxx | master   | merge request |

###  master 分支

**使用规范:**

- master分支存放的是随时可供在生产环境中部署的稳定版本代码
- 使用 tag 标记一个版本用于发布或回滚
- master分支是保护分支，不可直接push到远程仓master分支

### develop 分支

**使用规范:**

- develop分支是保存当前最新开发成果的分支
- develop分支衍生出各个feature分支
- 小的改动可以直接在 develop 分支进行，改动较多时切出新的 feature 分支进行

**注:**  更好的做法是develop 分支作为开发的主分支，也 `不允许直接提交代码`。小改动也应该以 feature 分支提 merge request 合并，目的是保证每个改动都经过了强制代码 review，降低代码风险。

### feature 分支

**使用规范:**

- 分支的命名格式建议 `feature/login`.
- 以功能为单位从 develop 拉一个 feature 分支
- 每个 feature 分支颗粒要尽量小，以利于快速迭代和避免冲突
- 当其中一个feature分支完成后，需合并回 develop 分支，另需删除 `feauter/xxx` 分支
- feature分支只与 develop 分支交互，不能与 master 分支直接交互

**流程**

feature分支做完后，必须合并回Develop分支, 合并完分支后一般会删点这个Feature分支，毕竟保留下来意义也不大。

![img](https://img-blog.csdnimg.cn/img_convert/42c0c3148c91822537746727a6a4a1b5.png)

### release 分支

使用规范：

- 分支的命名格式建议 `release/*`, `*` 以本次发布的版本号为标识. 

- release主要用来为发布新版的测试、预发布、修复的分支.
- release分支可以从develop分支上指定commit派生出.
- release分支测试通过后，合并到master分支并且给master标记一个版本号.
- 如在 release 分支发现 `bug` 可在 release 中修复，测试完成后合并到 develop 和 master 分支.
- release分支一旦建立就将独立，不可再从其他分支pull代码.
- 必须合并回develop分支和master分支.

**流程:**

Release分支基于Develop分支创建，打完Release分支之后，我们可以在这个Release分支上测试，修改Bug等。同时，其它开发人员可以基于Develop分支新建Feature (记住：一旦打了Release分支之后不要从Develop分支上合并新的改动到Release分支)发布Release分支时，合并Release到Master和Develop， 同时在Master分支上打个Tag记住Release版本号，然后可以删除Release分支了。

![img](https://img-blog.csdnimg.cn/img_convert/51a0a3ca863eeb314e86e4c239caa9ff.png)

**注意:**  如项目不大可省略该分支，避免繁琐的 gitflow 过程.

### hotfix 分支

**使用规范：**

- 命名规则：`hotfix/*`，`*` 以本次发布的版本号为标识.
- hotfix分支用来快速给已发布产品修复bug或微调功能.
- 只能从master分支指定tag版本衍生出来.
- 一旦完成修复bug，必须合并回master分支和develop分支.
- master被合并后，应该被标记一个新的版本号.
- hotfix分支一旦建立就将独立，不可再从其他分支pull代码.

**流程:**

hotfix 分支基于 master 分支创建，开发完后需要合并回 master 和 develop 分支，同时在 master 上打一个tag.

![img](https://img-blog.csdnimg.cn/img_convert/c79155a29aa3e4dc3e8e3029b50f6e66.png)

##  分支操作流程示例

这部分内容结合日常项目的开发流程，涉及到开发新功能、分支合并、发布新版本以及发布紧急修复版本等操作，展示常用的命令和操作。

1. 切到 develop 分支，更新 develop 最新代码

   ```
   git checkout develop
   git pull --rebase
   ```

2. 新建 feature 分支，开发新功能

   ```
   git checkout -b feature/xxx
   ...
   git add <files>
   git commit -m "feat(xxx): commit a"
   git commit -m "feat(xxx): commit b"
   # 其他提交
   ...
   ```

   如果此时 develop 分支有一笔提交，影响到你的 feature 开发，可以 rebase develop 分支，前提是 该 feature 分支只有你自己一个在开发，如果多人都在该分支，需要进行协调：

   ```
   # 切换到 develop 分支并更新 develop 分支代码
   git checkout develop
   git pull --rebase
   
   # 切回 feature 分支
   git checkout feature/xxx
   git rebase develop
   ```
   
3. 完成 feature 分支，合并到 develop 分支

   ```
   # 切到 develop 分支，更新下代码
   git checkout develop
   git pull --rebase
   
   # 合并 feature 分支
   git merge feature/xxx --no-ff
   
   # 删除 feature 分支
   git branch -d feature/xxx
   
   # 推到远端
   git push origin develop
   ```

4. 当某个版本所有的 feature 分支均合并到 develop 分支，就可以切出 release 分支，准备发布新版本，提交测试并进行 bug fix

   ```
   # 当前在 develop 分支
   git checkout -b release/xxx
   
   # 在 release/xxx 分支进行 bug fix
   git commit -m "fix(xxx): xxxxx"
   ...
   ```

5. 所有 bug 修复完成，准备发布新版本

   ```
   # master 分支合并 release 分支并添加 tag
   git checkout master
   git merge --no-ff release/xxx --no-ff
   # 添加版本标记，这里可以使用版本发布日期或者具体的版本号
   git tag 1.0.0
   
   # develop 分支合并 release 分支
   git checkout develop
   git merge --no-ff release/xxx
   
   # 删除 release 分支
   git branch -d release/xxx
   ```

   至此，一个新版本发布完成。

6. 线上出现 bug，需要紧急发布修复版本

   ```
   # 当前在 master 分支
   git checkout master
   
   # 切出 hotfix 分支
   git checkout -b hotfix/xxx
   
   ... 进行 bug fix 提交
   
   # master 分支合并 hotfix 分支并添加 tag(紧急版本)
   git checkout master
   git merge --no-ff hotfix/xxx --no-ff
   # 添加版本标记，这里可以使用版本发布日期或者具体的版本号
   git tag 1.0.1
   
   # develop 分支合并 hotfix 分支(如果此时存在 release 分支的话，应当合并到 release 分支)
   git checkout develop
   git merge --no-ff hotfix/xxx
   
   # 删除 hotfix 分支
   git branch -d hotfix/xxx
   ```

   至此，紧急版本发布完成。

## 提交信息规范

git commit 格式 如下：

```
<type>(<scope>): <subject>
```

各个部分的说明如下：

- **type 类型，提交的类别**

  - **feat**: 新功能
  - **fix**: 修复 bug
  - **docs**: 文档变动
  - **style**: 格式调整，对代码实际运行没有改动，例如添加空行、格式化等
  - **refactor**: bug 修复和添加新功能之外的代码改动
  - **perf**: 提升性能的改动
  - **test**: 添加或修正测试代码
  - **chore**: 构建过程或辅助工具和库（如文档生成）的更改

- **scope 修改范围**

  主要是这次修改涉及到的部分，简单概括，例如 login、train-order

- **subject 修改的描述**

  具体的修改描述信息

- **范例**

  ```
  feat(detail): 详情页修改样式
  fix(login): 登录页面错误处理
  test(list): 列表页添加测试代码
  ```

这里对提交规范加几点说明：

1. `type + scope` 能够控制每笔提交改动的文件尽可能少且集中，避免一次很多文件改动或者多个改动合成一笔。
2. `subject` 对于大部分国内项目而已，如果团队整体英文不是较高水平，比较推荐使用中文，方便阅读和检索。
3. 避免重复的提交信息，如果发现上一笔提交没改完整，可以使用 `git commit --amend` 指令追加改动，尽量避免重复的提交信息。

##  hotfix

使用 `checkout` 命令，直接退回到最近一次上线的tag位置，然后以此为基准创建一个新的fix分支:

```shell
git checkout -b hotfix/fix-login v1.1.0
```

执行完以后就已经在新创建的 `hotfix/fix-login` 分支了，而且代码已经回到了最近一次上线的状态。完成修复以后直接commit并打上新的tag, 比如`v1.1.0`, 最后切回master分支，将`fix-bug`合并到master即可：

```sh
git commit '修复xxx问题'
git tag v1.1.1

git checkout master
git merge v1.1.1
```

## 回滚代码

直接 `git revert` 当前代码，然后通过 `cicd` 的特性对上一个 git tag号进行回滚.

## merge vs rebase

同一个分支使用rebase

不同分支使用merge
