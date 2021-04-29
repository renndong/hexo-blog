---
title: Github Pull Request的提出与采纳
tags: git
categories: git
abbrlink: c97c9d53
date: 2020-01-24 13:19:54
---

这一文来讲解一下Github Pull Request（以下简称PR）的使用方法：
    
- 作为PR的提出者，如何对某个仓库提交PR，如何根据仓库管理者对所提交PR的反馈对PR进行完善
- 作为PR的接收者，如何对PR进行测试，对提出者进行反馈以及合并PR到仓库中。

这里我使用两个GitHub账户来进行说明，PR接收者账户为lml256，PR提出者账户为rikkaii。并以lml256账户中的learngit仓库进行试验。

## 如何提出PR

如果你对Github上的某个开源项目非常感兴趣，想贡献自己的力量为其添加新功能，或者发现了代码中的一些bug，想为其进行修复。那么可以将该开源仓库Fork到你的仓库列表中，并对其进行修改后，向原仓库提交PR，请求仓库的管理员合并你的代码。下面假设我们的账号为rikkaii，并向lml256账号中的learngit项目提交PR为例，详细的说明一下该流程：

### Fork仓库

Fork仓库非常简单，只需要到仓库首页的右上角点一下Fork按钮即可，Github会自动在你的仓库列表中创建该仓库的一个副本。

{% img /images/Github-Pull-Request的提出与采纳1.png %}

如下图，GitHub自动在你的账户上创建了一个副本，并在仓库名的下方指明了该仓库是Fork来的：

{% img /images/Github-Pull-Request的提出与采纳2.png %}

### 添加新功能

现在我们为该仓库添加一些新功能：

首先将该仓库clone到本地

```
$ git clone git@rikkaii:rikkaii/learngit.git
Cloning into 'learngit'...
remote: Enumerating objects: 65, done.
remote: Total 65 (delta 0), reused 0 (delta 0), pack-reused 65
Receiving objects: 100% (65/65), 5.44 KiB | 928.00 KiB/s, done.
Resolving deltas: 100% (17/17), done.
```

我们从master分支上新建一个**特性分支**，并在特性分支上面实现我们的新功能。这里简要说明下什么是特性分支：在主流的git开发模式中并不会直接在master上开发，而是在开发某个功能时新建一个分支，叫做特性分支，在其上进行开发，等到开发完成并测试完成后，再合并到master分支中，然后删除该分支。并且特性分支的名称要简明扼要，方便别人明白其作用。这里我们创建一个`python-div`分支用于开发新功能。

```
$ git checkout -b python-div
Switched to a new branch 'python-div'
```

我们在目录中创建一个`div.py`文件并在其编写一些代码：

```python
# div.py

# div.py

def div(a, b):
    ans = a / b
    return ans

```

然后进行提交，并推送到GitHub：

```bash
$ git add div.py

$ git commit -m "add div.py"
[python-div 7aad1de] add div.py
 1 file changed, 5 insertions(+)
 create mode 100644 div.py

$ git push --set-upstream origin python-div
...
To rikkaii:rikkaii/learngit.git
 * [new branch]      python-div -> python-div
Branch 'python-div' set up to track remote branch 'python-div' from 'origin'.
```

在上面的命令中，由于python-div分支是我们本地新建的，在GitHub远程仓库中并没有该分支的相关信息，为了能够将该分支的修改推送到GitHub远程仓库上，在推送时需要添加`--set-upstream`参数，这样会在远程仓库中也创建一个python-div分支，并将该分支的修改推送上去。

{% img /images/Github-Pull-Request的提出与采纳3.png %}

可以看到，我们的仓库中已经有了python-div分支以及本次commit添加的div.py文件。

### 发起PR

接下来，你可以点击上图中分支名称旁边的的New pull Request按钮来发起一个PR。这会自动跳转到原仓库下面，并选择python-div作为要推送的分支，如下：

{% img /images/Github-Pull-Request的提出与采纳4.png %}

至于要将python-div分支上的修改推送到原仓库的什么分支上，Github在这里自动为我们选择了master分支，这里需要注意的是分支的选择与原仓库使用的什么开发模式有关，一般正规的开源仓库都会说明PR的提交规则，包括提交到什么分支以及如何描述commit等等，这需要我们仔细阅读相关的要求并遵循。这里为了演示方便，将python-div分支的修改请求推送到原仓库的master上，并且在上图下面的说明栏里也只是使用GitHub的默认说明。

接下来便可以点击上图下面的Create pull request按钮来创建一个分支了。

{% img /images/Github-Pull-Request的提出与采纳5.png %}

当出现上图的样子后，我们的PR就创建成功了。我们只需要静静等待仓库管理员的审核就行了。

### 修正PR

但是，有些事总是事与愿违，有时候我们认为我们的代码没有问题，但是却被仓库管理员反馈（或者被其他开发者指出）我们的代码有问题，需要我们修复。管理员通过对我们的代码进行审计后，指出了我们的错误，Github也会将管理员与我们之间及其他开发者之间（这里并没有其他开发者）对该次PR的讨论反映在了时间轴上，并会以邮件的形式通知我们：

{% img /images/Github-Pull-Request的提出与采纳11.png %}

接下来，我们按照仓库管理员的要求，对代码进行完善，修改后的代码如下：

```python
# div.py

def div(a, b):
    try:
        ans = a / b
    except ZeroDivisionError as e:
        print('error:', e)
        return None
    return ans

```

修改完成后，对代码进行提交并推送到GitHub：

```
$ git add div.py

$ git commit -m "fix zero division error"
[python-div c3b73e7] fix zero division error
 1 file changed, 5 insertions(+), 1 deletion(-)

$ git push
...
To rikkaii:rikkaii/learngit.git
   7aad1de..c3b73e7  python-div -> python-div
```

当我们推送完后，无需再次发起PR，Github会自动的将该次的提交反映到原来PR时间轴上，如下图，并且Github还会向管理员发送邮件通知本次修改：

{% img /images/Github-Pull-Request的提出与采纳12.png %}

当仓库管理者通过了本次代码，将我们的代码合并到了master分支中后，我们也会收到Github发来的关于Merge本次PR的邮件通知。

## 接收PR

说完了如何提交PR，我们再说一说如何接收PR。作为一个开源项目的管理者，学会如何测试别人提出的PR并进行反馈或者合并是很必要的。这里假设我们的账号为lml256，以接收rikkaii账号向lml256账号中的learngit仓库提交的PR为例，详细的说明一下作为仓库的管理者，如何测试和合并PR。

### 审计PR

打开我们的仓库，在Pull Requests列表中可以看到rikkaii向我们的仓库提交了一个PR

{% img /images/Github-Pull-Request的提出与采纳6.png %}

点进去可以看到详细的信息：

{% img /images/Github-Pull-Request的提出与采纳7.png %}


我们可以在Files changed栏里对代码进行审计，如果代码有问题的话，可以对每行代码添加批注，以帮助PR提交者进行完善，写完批注后，点击start a review便可发起一个code review。如下：

{% img /images/Github-Pull-Request的提出与采纳8.png %}

当代码全部都审计完后，点击下图中的submit review便可以提交本次的审计结果。

{% img /images/Github-Pull-Request的提出与采纳9.png %}

提交后，我们对代码的审计结果便会反映在该PR的时间轴上，并且GitHub会向PR发起者发送回复通知邮件。

{% img /images/Github-Pull-Request的提出与采纳10.png %}

### 对PR进行本地测试

只在Github上凭借眼睛看并不能发现所有的问题，在对PR正式合并前，我们还需要对PR进行本地测试，如何使用git进行操作如下图：

{% img /images/Github-Pull-Request的提出与采纳14.png %}

首先我们需要将我们仓库的代码clone到本地（也可能不需要clone，如果本地已经有该仓库的话），然后拉取PR发起者仓库的相关信息，这需要我们先将PR发起者Fork的仓库添加到本地仓库的远程仓库：

```
$ git remote add PR_Sponsor git@rikkaii:rikkaii/learngit.git

$ git fetch PR_Sponsor  # 获取PR发起者仓库的相关信息
...
From rikkaii:rikkaii/learngit
 * [new branch]      dev        -> PR_Sponsor/dev
 * [new branch]      master     -> PR_Sponsor/master
 * [new branch]      python-div -> PR_Sponsor/python-div
```

可以看到，我们从PR发起者仓库中获取到三个分支，其中包括为我们提交PR的python-div分支。

在进行测试之前，我们还需要创建一个特性分支用于测试PR，并将python-div分支的修改merge到特性分支上进行测试：

```
$ git checkout -b test_pr
Switched to a new branch 'test_pr'

$ git merge PR_Sponsor/python-div
Updating 9053d5b..c3b73e7
Fast-forward
 div.py | 9 +++++++++
 1 file changed, 9 insertions(+)
 create mode 100644 div.py
```

### Merge代码

经过一番测试后，我们确信该PR的代码没有问题，可以合并代码到PR发起者要求合并到的分支上（这里是master）。这里有两种方法来处理这个问题，一是直接使用GitHub提供的Merge按钮，这种方法比较直观方便，但是不够灵活，无法处理合并冲突；二是在本地将测试后的特性分支合并到要合并的分支上，然后手动提交。尽管第二种方式处理起来比较复杂，但是却比较灵活，可以处理Github不能胜任的地方。这里我们选择第二种：

```
$ git checkout master
Switched to branch 'master'
Your branch is up to date with 'origin/master'.

$ git merge test_pr
Updating 9053d5b..c3b73e7
Fast-forward
 div.py | 9 +++++++++
 1 file changed, 9 insertions(+)
 create mode 100644 div.py

$ git branch -d test_pr   # 这里我们的特性分支已没作用，可以删除
Deleted branch test_pr (was c3b73e7).

$ git push
...
To lml256:lml256/learngit.git
   9053d5b..c3b73e7  master -> master
```

这里，我们已经处理完了该PR，同时，GitHub也会检测到我们合并了该PR，也将其反映到该PR的时间轴上，并将该PR的状态设置为Merged状态，关闭该次PR：

{% img /images/Github-Pull-Request的提出与采纳13.png %}

---

完

版权所有，未经允许不得转载