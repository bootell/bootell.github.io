---
title: '利用 GitLab CI 编译 LEDE K3'
date: 2019-02-28 10:18:00
tags:
---

本地编译一次 LEDE/OpenWrt 固件花了近3个小时，下载依赖文件因为网络问题也比较慢，考虑可以利用各种免费的 CI 自动集成工具来编译需要的固件。

GitLab CI 的 Shared Runners 每个 Job 的时间限制为3小时，每月有2000分钟的免费编译时常，足够日常编译自己的固件了。

<!--more-->

编译采用 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) 项目，编译流程见项目文档。

### 编译配置

首次需要先选择编译的配置，可以在本地下载整个项目，或是在远程主机上完成，保存配置文件为 `defconfig`

然后创建 GitLab CI 执行配置文件 `.gitlab-ci.yml` 如下

``` yml
image: tossp/lede:latest

stages:
  - build
  - deploy

make:
  stage: build
  variables:
    GIT_SUBMODULE_STRATEGY: recursive
    FORCE_UNSAFE_CONFIGURE: '1'
  cache:
    key: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
    paths:
      - dl/
      - feeds/
      - staging_dir/
      - build_dir/
      - key-*
  artifacts:
      name: "${CI_JOB_STAGE}_${CI_JOB_NAME}_${CI_COMMIT_REF_NAME}"
      when: always
      paths:
        - bin/
        - build.log
  script:
    - ./scripts/feeds update -a
    - ./scripts/feeds install -a
    - cp defconfig .config
    - make defconfig
    - sed -i 's|^TARGET_|# TARGET_|g; s|# TARGET_DEVICES += phicomm-k3|TARGET_DEVICES += phicomm-k3|' target/linux/bcm53xx/image/Makefile
    - make -j1 V=sc > ./build.log 2>&1
  only:
    - ci
```

编译前需要在 GitLab 后台 Setting/CI/General 中设置 `Timeout` 时间为 `3h`

### 其他问题

- 原固件为LEDE时升级
  将固件传输到路由器 `/tmp/`
  ``` bash
  mtd -r write /tmp/openwrt-bcm53xx-phicomm-k3-squashfs.trx firmware
  ```

- 开启隐藏功能（ssr） 

  ``` bash
  echo 0xDEADBEEF > /etc/config/google_fu_mode
  ```

- 无线驱动

  可以更换 K3 的无线驱动，直接替换文件即可

  https://github.com/Hill-98/phicommk3-firmware

  驱动源码包位置

  `/package/lean/k3-brcmfmac4366c-firmware/files/lib/firmware/brcm/brcmfmac4366c-pcie.bin`

  驱动固件位置

  `/lib/firmware/brcm/brcmfmac4366c-pcie.bin`


### 参考资料
https://www.right.com.cn/forum/thread-257677-1-1.html
https://www.right.com.cn/forum/thread-419328-1-1.html
