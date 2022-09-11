---
title: 'macOS Dictionary 词典编译'
date: 2022-08-24 00:15:00
tags:
- macOS
---


macOS 自带的 Dictionary.app 相比于其他词典应用，支持触摸板重按取词，十分方便。但自带词典随够用但较少，可以从其他词典转换成支持的词典，以丰富词库。
环境搭建及编译难度较小，重点在于各个词典的优化，以达到更加的显示效果。下面是整个词典制作的过程。

<!--more-->

### 环境部署

官方文档：[pyglossary](https://github.com/ilius/pyglossary/blob/master/doc/apple.md)

##### 1. Python环境搭建

``` shell
# 下载项目
git clone https://github.com/ilius/pyglossary.git
cd pyglossary

# python环境
python3 -m venv venv
source venv/bin/activate
pip3 install lxml beautifulsoup4 html5lib

# 报错 "LZO compression support is not available" 时安装
brew install lzo
pip3 install python-lzo
```

##### 2. Apple 词典工具安装

下载地址：[Additional Tools for Xcode](http://developer.apple.com/downloads)
下载与自身 xcode 匹配的 "Additional Tools for Xcode"
将其中 `Utilities/Dictionary Development Kit` 解压放到 `/Applications/Utilities` 下

##### 3. 音频转换需要安装speex

brew直接安装的话，发现报错 "speexdec: command not found",不带 bin 可执行文件，需要从源码编译 [issues/184](https://github.com/ilius/pyglossary/issues/184)
下载源码：[speex.org](https://www.speex.org/downloads/)

``` shell
# 报错 "autoreconf: error: aclocal failed with exit status: 2"，需安装依赖
brew install automake
# 报错 "make[4]: Nothing to be done for `all'."，安装依赖
brew install pkg-config

./configure
make
make install
```

经测试，spx文件不需要转换，直接变更后缀为`.mp3`也可正常使用

### 词典选择

资源比较丰富且整理较好的有以下几个：
- 牛津高阶英汉双解词典（第四版）/（第八版）/（第九版）/（第十版）
- 朗文当代高级词典（第五版）
- 柯林斯COBUILD双解词典
- 韦氏高阶英汉双解词典

资源获取的网站常用的有以下几个：
- [https://freemdict.com/](https://freemdict.com/)
- [https://forum.freemdict.com/c/12-category/12](https://forum.freemdict.com/c/12-category/12)
- [https://www.pdawiki.com/forum/forum-4-1.html](https://www.pdawiki.com/forum/forum-4-1.html)

### 转换过程

##### 1. 资源转换

``` shell
# 移动资源文件
mkdir dist
mv "~/Downloads/LDOCE 5++ V2.15" ./dist/ldoce5
cd dist/ldoce5

# 转换为XML文件
python3 ../../main.py --write-format=AppleDict "LDOCE5++ V 2-15.mdx" LDOCE5++

cd LDOCE5++

# 转换音频
find OtherResources -name "*.spx" -execdir sh -c 'spx={};speexdec $spx  ${spx%.*}.wav' \;

# 资源地址替换
sed -i "" 's:src="/:src=":g' LDOCE5++.xml
sed -i "" 's|sound://\([/_a-zA-Z0-9]*\).spx|\1.wav|g' LDOCE5++.xml
```

将`LM5style.css`文件内容，替换到 `objects/LDOCE5++.dictionary/Contents/DefaultStyle.css`

##### 2. 优化

为了让词典在 Dictionary.app 中显示效果更好，需要进行优化。

```
sed -i "" "s|> <\/a>|>🔈</a>|g" LDOCE5.xml
```

##### 3. 编译与安装

``` shell
make
make install
```


### 示例

由于 Dictionary.app 自带牛津词典，这里转换朗文和柯林斯作为补充。

##### 1. LDOCE 5++ V2.15

``` shell
# 转换为XML文件
python3 ../../main.py --write-format=AppleDict "LDOCE5++ V 2-15.mdx" LDOCE5

cd LDOCE5

# 转换音频，直接将后缀改为mp3
find OtherResources -name "*.spx" -execdir sh -c 'spx={};mv $spx ${spx%.*}.mp3' \;

# 资源地址替换
sed -i "" 's:src="/:src=":g' LDOCE5.xml
sed -i "" 's|sound://\([/_a-zA-Z0-9]*\).spx|\1.mp3|g' LDOCE5.xml
```

修改 `LDOCE5.plist`
CFBundleDisplayName：朗文当代高级词典（第五版）
CFBundleName：朗文当代

将 `LM5style.css` 内容复制到 `LDOCE5.css`，并修改内容
```css
# 显示隐藏
.Sense .corpus .title {
    /* display: none; */
}
.LDOCEVERSION_new .BoxPanel {
    /* display: none; */
}
.dictionary_intro {
    display: none;
    /*background-color: #314089;*/
    /*color: #fff;*/
    padding-left: 10px;
    margin: 5px 0 10px -7px;
}
.wordfams .LDOCE5pp_sensefold {
    display: none;
}
.topics_container {
    display: none;
}

# 发音图标
@font-face {
    font-family: 'lm5pp_icomoon';
    src: url("lm5pp_icomoon.ttf");
    font-weight: normal;
    font-style: normal;
}
@font-face {
    font-family: 'cjkextent';
    src: url("cjkextent.ttf");
    font-weight: normal;
    font-style: normal;
}

```

##### 2. CollinsCOBUILDOverhaul v2.30

``` shell
# 转换为XML文件
python3 ../../main.py --write-format=AppleDict "CollinsCOBUILDOverhaul V 2-30.mdx" COLLINS

cd COLLINS

# 资源地址替换
sed -i "" 's:src="/:src=":g' COLLINS.xml
```

修改 `COLLINS.plist`
CFBundleDisplayName：柯林斯英汉双解词典
CFBundleName：柯林斯双解

自带CSS效果显示不好，这里查找到柯林斯完美复原版的样式文件，进行替换。将内容复制到 `COLLINS.css`，并修改内容
```css
# 展开隐藏项
.hidden {
    /* display: none */
}

# 隐藏单词用量
.trend.folded {
    display: none;
}

# 发音图标
@font-face {
    font-family: 'icomoon';
    src: url("icomoon.ttf");
    font-weight: normal;
    font-style: normal;
}
```

### 参考资料

> [https://github.com/ilius/pyglossary/blob/master/doc/apple.md](https://github.com/ilius/pyglossary/blob/master/doc/apple.md)
> [https://kaihao.io/2018/mdict-to-macos-dictionary/](https://kaihao.io/2018/mdict-to-macos-dictionary/)
> [https://10382.github.io/post/Mdict转Mac词典小记/](https://10382.github.io/post/Mdict%E8%BD%ACMac%E8%AF%8D%E5%85%B8%E5%B0%8F%E8%AE%B0/)
> [https://www.zhihu.com/question/20428599](https://www.zhihu.com/question/20428599)
> [https://www.pdawiki.com/forum/thread-42822-1-1.html](https://www.pdawiki.com/forum/thread-42822-1-1.html)
> [https://www.pdawiki.com/forum/thread-13014-1-1.html](https://www.pdawiki.com/forum/thread-13014-1-1.html)
> [https://placeless.net/blog/macos-dictionaries](https://placeless.net/blog/macos-dictionaries)
> [http://qunwang6.github.io/blog/OSXDictionary/](http://qunwang6.github.io/blog/OSXDictionary/)
> [https://www.readern.com/convert-mdict-to-macos-dictionary.html](https://www.readern.com/convert-mdict-to-macos-dictionary.html)
