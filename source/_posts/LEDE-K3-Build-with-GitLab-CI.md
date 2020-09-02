---
title: '自动化编译 LEDE K3'
date: 2019-02-28 10:18:00
updated: 2019-10-10 10:08:00
tags: 路由器
---


本地编译一次 LEDE/OpenWrt 固件花了近3个小时，下载依赖文件因为网络问题也比较慢，考虑可以利用各种免费的 CI 自动集成工具来编译需要的固件，目前可选的有 Github Actions 和 GitLab CI。



### 编译配置

##### 生成编译配置文件

编译使用 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) 项目，首先对要编译的固件进行配置：

``` bash
# 克隆项目
git clone https://github.com/coolsnowwolf/lede.git

cd lede

# 下载软件源
./scripts/feeds update -a
./scripts/feeds install -a

# 选择配置，生成配置文件
make menuconfig
```

##### 固化原固件配置

将需要原固件需要保存的文件，添加到编译目录的 `/files` 内，即可保存之前固件的配置。
常见的配置地址如下所示

``` directory
/
└── etc
    ├── config
    │   ├── system    主机名，时区等
    │   ├── network   网络
    │   ├── dhcp
    │   └── ddns
    └── lib\lua\luci\view\admin_status\index.htm  主页样式
```



### 自动化编译

自动化编译需要先配置好配置文件，把 `make menuconfig` 生成的配置文件保存为 `defconfig`，一并提交到远程仓库。

##### GitHub Actions

[Github Actions](https://github.com/features/actions) 免费账户每月有2000分钟时长，单次最高运行6小时，非常充足。目前测试阶段需要申请开通。
编译需要添加配置文件 `.github/workflow/build.yaml`:

``` yml
name: Build

on: [push]

jobs:
  k3:
    runs-on: ubuntu-16.04
    steps:
      - uses: actions/checkout@master
      - name: Set Environment
        run: |
          sudo apt-get -yqq update
          sudo apt-get -yqq install build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint
      - name: Prepare Build
        run: |
          ./scripts/feeds update -a
          ./scripts/feeds install -a
          cp k3config .config
          make defconfig
          sed -i 's|^TARGET_|# TARGET_|g; s|# TARGET_DEVICES += phicomm-k3|TARGET_DEVICES += phicomm-k3|' target/linux/bcm53xx/image/Makefile
      - name: Build Image
        run: make -j$(nproc) V=sc
      - name: Upload Image
        uses: actions/upload-artifact@v1
        with:
          name: lede-k3-${{ github.sha }}
          path: bin
```


##### GitLab CI

GitLab CI 的 Shared Runners 每个 Job 的时间限制为3小时，每月有2000分钟的免费编译时长，足够日常编译自己的固件了。
编译需要在 GitLab 后台 Setting/CI/General 中设置 `Timeout` 时间为 `3h`，并添加配置文件 `.gitlab-ci.yml` 如下：

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
    - make -j$(nproc) V=sc > ./build.log 2>&1
  only:
    - ci
```



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
  驱动下载地址：https://github.com/Hill-98/phicommk3-firmware
  驱动源码包位置：`/package/lean/k3-brcmfmac4366c-firmware/files/lib/firmware/brcm/brcmfmac4366c-pcie.bin`
  驱动固件位置：`/lib/firmware/brcm/brcmfmac4366c-pcie.bin`

- mwan3 ipset
  使用 mwan3 的 ipset 功能，可以对域名进行匹配
  ``` bash
  echo 'ipset create baidu hash:ip family inet' >> /etc/rc.local
  echo 'conf-dir=/etc/dnsmasq.d' >> /etc/dnsmasq.conf
  echo 'ipset=/baidu.com/baidu' >> /etc/dnsmasq.d/baidu.conf
  ```



### 参考资料
https://www.right.com.cn/forum/thread-257677-1-1.html
https://www.right.com.cn/forum/thread-419328-1-1.html

<!--more-->