# git基本原理和操作

## references

- <https://www.ruanyifeng.com/blog/2018/10/git-internals.html>
- <https://shalithasuranga.medium.com/how-does-git-work-internally-7c36dcb1f2cf>
- <https://git-scm.com/book/zh/v2>

## .git目录

### objects目录

- object type: blob, tree, commit, tag
- blob 是添加到object database中的文件
- tree 代表目录，每个tree对象记录了本级目录所包含object的文件权限，type, hash, 文件名
- commit 指向一个root tree对象，并记录parent_commit, author, committer, commit_content
- tag(annotated) 指向一个commit,及额外的tag信息。(普通tag是纯粹的指针不包含其它信息，所以不会生成object，只在refs/tags中记录)
- git是一个内容寻址文件系统，所有的内容都存储在objects目录(object database)
- git为每个object生成一个40位的hash值,相当于一个唯一地址
  >sha1sum (object_type + 1_space + object_byte_count + 1_null_byte)
- 之后调用zlib对object进行压缩
- 在一个git项目中，git最开始会以松散（loose）形式存储object,压缩后的object文件存储路径为
  >obj_hash[0,2] / obj_hash[2,38]
- 在松散形式中git不会对文件进行差异存储，也就是说保留所有版本的完整内容
- 当自动（例如git push时）或手动执行git gc操作时，git会对松散存储中的文件进行pack/repack，index
- 在pack目录中生成pack文件和对应的idx文件
- pack文件为差异存储后的包文件
- idx为包文件的索引，记录包文件中object的hash,type,size,offset,diff pointer
- git不会对悬空（dangling）的object打包,比如staged但没有commit的文件
- info/packs文件中记录了所有的pack文件

```shell
git verify-pack -v .git/objects/pack/pack-xxx.idx
```

### refs目录

- ref类似于快捷方式或者dns,用来指向object
- `refs/heads`目录中记录所有的分支所指向的commit
- `refs/tags`目录中记录所有的tag所指向的commit
- `refs/remotes`目录中同步了远程repo中的HEAD和refs/heads
- 实际上除了commit,ref也可以指向其它任何类型的object
- 这个目录中的文件可以被打包成`.git/packed-refs`一个文件
  - 通过`git gc`或`git pack-refs`触发

### HEAD文件

HEAD文件指向一个ref,表示当前checkout的分支
>ref: refs/heads/main

### index文件

- index文件记录新生成的tree对象,也就是stage的状态

## git add

```shell
git init
touch test.txt

git hash-objects test.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391
# 计算文件哈希值

git hash-objects -w test.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391
# 计算文件哈希值，二进制压缩后写入object database
# 默认类型为blob，存储路径为：
# .git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391

# 可以使用git cat-file命令解压查看
git cat-file e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 -p

# 此时test.txt被写入object database,但还没有添加到stage area

git update-index --add --cacheinfo 100644 \
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 test.txt
# 使用git update-index 将文件名，哈希值，文件系统权限写入stage area
# 实际上是在.git/index文件中生成了记录test.txt信息的tree对象

# 以上`hash-objects -w`和`update-index --add`两条命令相当于执行了git add text.txt的操作
# 这两条命令也可以使用git update-index --add test.txt简化替代

# 此时test.txt被写入object database,并添加到stage area

echo 'hello world' > test.txt
# 修改文件

git hash-object -w test.txt
3b18e512dba79e4c8300dd08aeb37f8e728b8dad
# 执行写入object database，因为文件改变，哈希值也改变了，生成的二进制文件路径为：
# .git/objects/3b/18e512dba79e4c8300dd08aeb37f8e728b8dad

git update-index --add --cacheinfo 100644 \
3b18e512dba79e4c8300dd08aeb37f8e728b8dad test.txt
# 使用git update-index 将更新后的信息写入stage area
# 在.git/index文件中记录test.txt信息的tree对象被更新

# 上两条命令相当于在文件修改之后又执行了一次git add test.txt的操作

git ls-files --stage
# 查看stage area中的记录

mkdir docs
echo 1 >docs/help.txt
git add docs/help.txt
# 再添加一个文件

git ls-files --stage
100644 d00491fd7e5bb6fa28c517a0bb32b8b506539d4d 0       docs/help.txt
100644 3b18e512dba79e4c8300dd08aeb37f8e728b8dad 0       test.txt
# 此时stage area中的信息

```

## git commit

```shell
git write-tree
9763b20c60edd9f2f6349969fca1e516a0ee39ca
# 把.git/index文件中的tree对象写入object database

git cat-file 9763b20c60edd9f2f6349969fca1e516a0ee39ca -p
040000 tree dbe2d41cad560c499f1a2ffe7247a30bdf2b6607    docs
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    test.txt
# 这个是根目录tree对象，记录本级目录包括的内容

git cat-file -p dbe2d41cad560c499f1a2ffe7247a30bdf2b6607
100644 blob d00491fd7e5bb6fa28c517a0bb32b8b506539d4d    help.txt
# 还有docs子目录tree对象，记录本级目录包括的内容

echo 2 >docs/help2.txt
git add docs/help2.txt
# 在子目录再添加一个文件
# .git/index文件中生成新的tree对象

git write-tree
388c24f2e453aa0e2a42b9ad85bcf87a644a5556
# 把.git/index文件中的tree对象写入object database

git cat-file -p 388c24f2e453aa0e2a42b9ad85bcf87a644a5556
040000 tree abc1e5c27008b89f5288c31b5710a2666931f8d3    docs
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    test.txt

git cat-file -p abc1e5c27008b89f5288c31b5710a2666931f8d3
100644 blob d00491fd7e5bb6fa28c517a0bb32b8b506539d4d    help.txt
100644 blob 0cfbf08886fca9a91cb753ec8734c84fcbe52c9f    help2.txt
# 由于子目录增加了文件，导致子目录和根目录哈希值发生变化

# 由此看出，根目录的哈希值可以推导出整个目录的结构和文件信息，可以代表一次快照或commit
# 现在一次commit所需的信息都准备好了，就差临门一脚

echo "first commit" | git commit-tree 388c24f2e453aa0e2a42b9ad85bcf87a644a5556
e395e5df5ecf013c8ef97ba2e47f6e3c4fa8388d
# 生成commit对象

git cat-file -p e395e5df5ecf013c8ef97ba2e47f6e3c4fa8388d
tree 388c24f2e453aa0e2a42b9ad85bcf87a644a5556
author emptyteeth <76892982+emptyteeth@users.noreply.github.com> 15 +0800
committer emptyteeth <76892982+emptyteeth@users.noreply.github.com> 15 +0800
first commit
# commit对象包含了对root tree的引用，对parent commit的引用和commit信息
# 相当于给tree加了个外包装，由于这里是第一次提交，所以并没有引用parent commit
# 一般情况下commit对象和root tree对象总是成对出现的，可以理解为一套班子两块牌子
# commit对象负责操作记录和历史关系，root tree则负责文件的组织工作

# 此时commit已经完成，其实只是生成了一个指向的tree的commit对象
# 但是git status看不出有任何变化，因为branch的HEAD还是空的，需要把它指向这个commit对象

echo e395e5df5ecf013c8ef97ba2e47f6e3c4fa8388d >.git/refs/heads/main
# 完成最后一步

# 以上相当于git commit -m "first commit"的操作

echo "hello world again" > test.txt
# 修改文件

git hash-object -w test.txt
c90c5155ccd6661aed956510f5bd57828eec9ddb
# 写入object database

git update-index test.txt
# 更新.git/index tree记录

git write-tree
72b46a2df8b175f0d7c58d8064d680011a20817d
# 生成新的tree对象，写入object database

echo "second commit" | git commit-tree 72b46a2df8b175f0d7c58d8064d680011a20817d -p e395e5df5ecf013c8ef97ba2e47f6e3c4fa8388d
4446e393a4927a7dbcadc143f500a1a48541edf7
# 生成新的commit对象，这次多了一个-p参数指定父级commit对象
# 指定父级commit关系就是git log回溯变更的信息来源
# 如果不指定父级commit，并且移动了HEAD，那么git log就只有当前这一条commit

git cat-file 4446e393a4927a7dbcadc143f500a1a48541edf7 -p
tree 72b46a2df8b175f0d7c58d8064d680011a20817d
parent e395e5df5ecf013c8ef97ba2e47f6e3c4fa8388d
author emptyteeth <76892982+emptyteeth@users.noreply.github.com> 1619017629 +0800
committer emptyteeth <76892982+emptyteeth@users.noreply.github.com> 1619017629 +0800
second commit
# 查看新的commit对象

echo 4446e393a4927a7dbcadc143f500a1a48541edf7 > .git/refs/heads/main
# 移动main branch的HEAD到最新的commit

# 以上相当于git add test.txt && git commit -m "second commit"
```

## squash commits

```shell
# let's say we need to squash last 3 commits
git rebase -i HEAD~3

# pick 5555555 5th commit
# squash 4444444 4th commit
# squash 3333333 3rd commit
```

## git remote

```shell
git remote -v
# show remote name and url

git remote show name
# show details of remote

git ls-remote
# show remote HEAD/heads/tags

git remote add nameit https://remote.url/repo
# 添加一个远程repo,并设置同步所有远程ref
```

```ini
#.git/config
[remote "nameit"]
        url = https://remote.url/repo
        fetch = +refs/heads/*:refs/remotes/nameit/*
```

```shell
git remote add nameit2 https://remote.url/repo2 -t b1 -t b2
# 添加一个远程repo,设置只同步远程的b1,b2
```

```ini
#.git/config
[remote "nameit2"]
        url = https://remote.url/repo2
        fetch = +refs/heads/b1:refs/remotes/nameit2/b1
        fetch = +refs/heads/b2:refs/remotes/nameit2/b2
```

```text
[+] <src>:<dst>
src代表远程的ref位置
dst代表远程ref同步到本地的标示位置
加号代表即使不能fast-forward,也强制同步远程ref
```

## git fetch

```shell
git fetch nameit
# 将远程repo nameit中的关注的ref，和这些ref所引用的object同步到本地
# objects直接写入object database,ref写入refs/remotes/nameit

git fetch nameit2 [refs/heads/]b1:refs/remotes/nameit2/b1
# 只同步远程repo nameit2的b1及其引用的objects到本地
```

## git clone/pull

```shell
git clone https://remote.url/reponame [-o nameit] [dirname]
# -o选项设置remote name,默认为origin
# dirname设置要创建的本地目录名，默认为reponame
# 这条命令相当于执行了以下命令,假设远程HEAD当前为main
mkdir dirname && cd dirname
git init
git remote add nameit https://remote.url/reponame
git fetch nameit
cp .git/refs/remotes/nameit/main .git/refs/heads/main
git branch -u nameit/main main
# 设置本地分支从远程同步时的对应关系
# 执行git pull时就会把远程的refs/heads/main merge到本地的refs/heads/main
```

```ini
[branch "main"]
    remote = nameit
    merge = refs/heads/main
```

```shell
git pull
# 相当于执行了以下命令
git fetch nameit
git merge refs/remotes/nameit/main
#or
git merge FETCH_HEAD

```

## git push

```shell
git push [-u] nameit main:main
# 将本地main推送到nameit上的main
# 加上-u参数则为两个分支建立跟踪关系
# 之后在本地main分支只需执行git push命令即可自动推送到nameit上的main

git push origin --delete branchname
# 删除远程ref
```

## git branch

```shell
git branch abc
# 新建abc分支

git branch -d abc
# 删除abc分支

git branch -u origin/abc abc
# 建立跟踪关系

git branch -vv
# 查看所有本地分支详情
```

## git checkout

```shell
git checkout abc
# 切换到分支

git checkut -b abc
# 新建并切换到分支

git checkout -b b2 origin/b2
# 相当于以下命令,拉取远程b2,建立本地b2指向它，然后建立跟踪关系
git fetch origin b2:refs/remotes/origin/b2
git checkout refs/remotes/origin/b2
git branch b2
git checkout b2
git branch -u origin/b2 b2

git checkout --track origin/b2
# 效果同上

git checkout b2
# 如果本地不存在b2，且远程存在b2，效果同上
```

## git config

```shell
git config --global init.defaultBranch branch_name
git config push.default origin
# 存在多个remote时，设置默认push目标
git config "branch.branch_name.remote" origin
git config "branch.branch_name.merge" "refs/heads/branch_name"
# 设置跟踪关系
git config remote.origin.push refs/heads/main:refs/heads/main
git config remote.origin.push refs/heads/main:refs/heads/qa/main
# 设置推动到指定的remote branch
git config --global https.proxy http://127.0.0.1:1080
git config --global https.proxy https://127.0.0.1:1080
git config --global --unset http.proxy
git config --global --unset https.proxy
git config user.email "a@b.c"
git config user.name "abc"
git config --global core.editor "nano -w"
```

## git reset

```shell
git reset [default:--mixed] [default:HEAD]
#discard the stage area
#keep the working tree

git reset --soft
#keep the stage area
#keep the working tree
#means doing nothing

#this is how you cancel commit
git reset HEAD~N
#move HEAD N steps back
#discard the stage area
#keep the working tree

git reset HEAD~N --soft
#move HEAD N steps back
#keep the stage area
#keep the working tree

git reset HEAD~N --hard
#move HEAD N steps back
#discard the stage area
#discard the working tree

hard  - lose changes between W, I and R
mixed - lose changes between W and I
soft  - lose nothing
```

## git restore

```shell
git restore --source/-s [default:HEAD]
git restore [.|file] [default:--worktree/-W]
# restore the working tree from the stage area
#if the file isn't in the stage area, then restore from HEAD

git restore file --staged/-S
# restore the stage area from the HEAD, i.e. empty the stage area
# keep the working tree
# equals to git reset file

git restore file -S -W
# restore the stage area from the HEAD, i.e. empty the stage area
# restore the working tree from the stage area
# equals to
git restore file -S && git restore file
git reset file --hard
```

## git rm

```shell
git rm file
# delete the file from working tree
# stage a delete mark of the file

git rm file --cached
# stage a delete mark of the file
# keep the file in the working tree
```

## git tag

```shell
git tag v1 [default:HEAD]
# add a tag to a ref/hash

git tag [-l]
# list tags

git tag -n
# list tags and anotation

git tag -d v1
# delete a tag

git push origin refs/tags/v1:refs/tags/v1
# push a tag to origin, equals to
git push origin v1

git push origin :refs/tags/v1
# delete a tag from origin, equals to
git push --delete origin v1

git push --tags
# sync all tags to remote

git push --follow-tags
# push commits and their tags
```

## 一些快捷方式和缩略用法

```text
HEAD HEAD~1(HEAD^) HEAD~2 HEAD~3(HEAD~~~) HEAD~N
@{u} 代表当前分支跟踪的目标分支

ref^{tree} 表示ref所指向的commit所指向的tree对象

master...dev

refs/remotes/origin/abc
remotes/origin/abc
origin/abc
后两种写法是第一种的缩略用法
```

## github

### 手动更新fork

```shell
# 方法1
git checkout master
git pull https://github.com/progit/progit2.git
git push origin master
# 方法2
git remote add progit https://github.com/progit/progit2.git
git branch --set-upstream-to=progit/master master
git config --local remote.pushDefault origin
git checkout master
git pull
git push
```

### 关注PR

```ini
fetch = +refs/pull/*/head:refs/remotes/origin/pr/*
```

stash

```shell

Host github.com-abc
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_github

##bfg
https://rtyley.github.io/bfg-repo-cleaner/

git clone --depth 1 -b branch1 url
git fetch --depth 1 origin branch2:branch2

git checkout --orphan empty-branch
git rm -rf .


git commit --allow-empty -m "[ci run] test"
```
