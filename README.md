## Hileveps技术博客

### 在windows环境下构建环境

#### 安装Ruby环境

本来一开始打算安装RVM来进行Ruby环境管理，RVM不直接支持windows，但是可以使用cygwin等环境，然后安装cygwin后，`rvm install 2.7` 的时候，没有针对cygwin的现成的ruby binary包，所以需要手动从源码安装。所有没有使用rvm。

windows安装ruby，从如下链接下载即可：

https://rubyinstaller.org/downloads/

> 特别注意：不要追求新版本，我一开始下载了ruby2.7，后来很多Gem要求的ruby环境都小于2.7。所以后来切换到2.6版本。

#### 配置Jekyll环境

按照如下脚本执行：

```
gem install jekyll
```

执行完成之后就可以使用jekyll了。

```
cd blog
# 启动jekyll
jekyll serve -w
```