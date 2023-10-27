---
layout: post
title:  "vscode提高效率, 自定义task"
date:   2023-10-27 14:57:00 -0400
categories: vscode task 
---

**问题**

用vscode的时候有需要一些自定义功能, 但是又不想找插件 装插件

例如想要去除trailing whitespace. 

例如想要直接跳转当前repo的github.  

**解决**: vscode custom task


## 上图

vscode 本身自带简单的自定义task功能, 如下图, 按`F1` -> `Tasks: run task` -> 选择自己定义的task 

![vscode-run-task](/assets/images/vscode-run-custom-tasks.gif)



### 如何自定义task

`F1` -> `Tasks: Open User Tasks` 

会打开 `tasks.json`. 

#### 栗子1
定义一个`remove empty line` 的功能, 
```json
{
    "version": "2.0.0",
    "tasks": [

        {
            "label": "Remove empty line",
            "command": "sed -i '' '/^[[:space:]]*$/d' \"${file}\"",
            "type": "shell",
            "problemMatcher": []
        }]
```

#### 栗子2 
在github中打开当前的branch的 pull request. 

```json
        {
            "label": "Currnet PR",
            "command": "open https://$(git config --get remote.origin.url | sed -E 's|^git@(.*):(.*)/(.*)\\.git$|\\1/\\2/\\3|')/pull/$(git rev-parse --abbrev-ref HEAD)",
            "type": "shell",
            "problemMatcher": []
        },
```

#### 栗子3 

我平时有一堆URL想要直接在vscode里面就能跳转, 不想去到chrome里在打开bookmark. 

可以定义一下的task, 使用这个task会跳出一个`quickpick` 选择要去那个url. 

这里用到了 `vscode` `input` 这个功能

```json
    "tasks": [
        {
            "label": "Quick URLs",
            "command": "open ${input:quick-links}",
            "type": "shell",
            "problemMatcher": []
        },
    ] ,
    "inputs": [
        {
            "id": "quick-links",
            "description": "Popular links to go to",
            "type": "pickString",
            "options": [
                "https://....", 
                "https://......"
            ]}
        
```

#### 栗子4 
`vscode` `input` 还能要求用户输入. 

比如以下的task 要求用户输入一个`branch name` 然后用`git`创建一个新的`branch` with custom prefix `whou/<MM-DD>/<user-input>`

```json
    "tasks": [
        {
            "label": "Create Branch",
            "command": "git checkout -b whou/$(date +'%m%d')/${input:branch-base}",
            "type": "shell",
            "problemMatcher": []
        }], 
    "inputs": [
        {
            "id": "branch-base",
            "description": "branch name to create",
            "type": "promptString"
        }]        
```

## 最后
这种快速的定义task来提高效率很像在`emacs`中可以通过custom 整个环境(lisp machine)来达到自己想要的功能. 
