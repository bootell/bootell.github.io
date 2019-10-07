---
title: '自动化编译 Hexo 博客并部署在 Github Page'
date: 2019-10-03 20:00:00
tags: 
- Blog
---

一直使用 [Travis CI](https://travis-ci.org) 来自动生成和部署 Hexo。最近 Github 推出了自己的持续集成服务 [Github Actions](https://github.com/features/actions)，于是改用它以方便管理。在此记录一下两种方法。

<!--more-->

### Hexo 配置

将主题文件使用 git submodule 跟踪：`git submodule add https://github.com/hexojs/hexo-theme-landscape themes/landscape`；
先随意推送一个提交到远端的 master 分支进行初始化，源码文件不要使用 master 分支提交到远端；

### 使用 Github Actions

目前使用 [Github Actions](https://github.com/features/actions) 需要申请，申请后会在 Repo 里出现 Actions 的标签。
使用时，直接在根目录下创建 `.github/workflow/deploy.yaml` 即可，注意修改 git 的信息。

``` yaml
name: Build and Deploy

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
      with:
        ref: hexo
        submodules: true
    - name: Use Node.js
      uses: actions/setup-node@v1
      with:
        node-version: '10.x'
    - name: Install dependencies
      run: npm install
    - name: Get old version
      run: |
        npx hexo clean
        git clone "https://github.com/${GITHUB_REPOSITORY}" -b master public
    - name: Generate pages
      run: npx hexo generate
    - name: Deploy to GitHub Page
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      run: |
        cd $GITHUB_WORKSPACE/public
        git config user.name $GITHUB_ACTOR
        git add .
        git commit -m "github actions auto build"
        git push https://${GITHUB_ACTOR}:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git master:master
```

### 使用 Travis CI

首先需要创建 [GitHub Token](https://github.com/settings/tokens)，用于最后推到 `Github Pages`；然后登录 [Travis CI](https://travis-ci.org) 并授权后，在设置里添加变量 `GITHUB_TOKEN`，值为上面创建的 token。
在根目录下添加 `.travis.yml` 文件：

``` yaml
language: node_js

node_js: stable

cache:
  directories:
    - node_modules

install:
  - npm install

script:
  - hexo clean
  - git clone https://github.com/${TRAVIS_REPO_SLUG}.git -b master public/
  - hexo generate

after_script:
  - cd ./public
  - git checkout master
  - git config user.name "Travis CI"
  - git add .
  - git commit -m "travis-ci auto build"
  - git push "https://${GITHUB_TOKEN}@github.com/${TRAVIS_REPO_SLUG}.git" master:master
```