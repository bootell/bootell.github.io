---
title: '利用 GitLab CI 编译 LEDE K3'
date: 2019-02-28 10:18:00
tags:
---

编译一次 LEDE/OpenWrt 固件大概需要2个小时左右，下载依赖文件因为网络问题也比较慢，可以利用公用的各种 CI 自动集成工具来编译需要的固件。

GitLab CI 的 Shared Runners 每个 Job 的时间限制为3小时，每月有2000分钟的免费编译时常，足够日常编译自己的固件了。

<!--more-->


### 编译配置

首次需要先下载整个项目，在本地选择好需要的配置，保存配置文件为 `defconfig`

GitLab CI 执行配置文件如下

``` yml
image: tossp/lede:latest

stages:
  - build
  - deploy

cache:
  key: $(CI_COMMIT_REF_SLUG)
  paths:
    - dl/
    - feeds/
    - staging_dir/
    - build_dir/
    - key-*

make:
  stage: build
  variables:
    GIT_SUBMODULE_STRATEGY: recursive
    FORCE_UNSAFE_CONFIGURE: '1'
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

### 其他问题

- 原固件为LEDE时升级命令
  ``` bash
  mtd -r write /tmp/openwrt-bcm53xx-phicomm-k3-squashfs.trx firmware
  ```

- 开启隐藏功能（ssr） 

  ``` bash
  echo 0xDEADBEEF > /etc/config/google_fu_mode
  ```

- 驱动包

  https://github.com/Hill-98/phicommk3-firmware

  驱动源码包位置

  `/package/lean/k3-brcmfmac4366c-firmware/files/lib/firmware/brcm/brcmfmac4366c-pcie.bin`

  驱动固件位置

  `/lib/firmware/brcm/brcmfmac4366c-pcie.bin`


### 参考资料
https://www.right.com.cn/forum/thread-257677-1-1.html
https://www.right.com.cn/forum/thread-419328-1-1.html