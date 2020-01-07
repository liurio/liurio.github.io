## jkell搭建github.io博客

### 1 安装工具

推荐 https://rubyinstaller.org/downloads/，下载 Ruby 2.6.1-1(x64)，点击安装即可，并将路径加入环境变量。

```shell
gem install jekyll
gem install jekyll bundler
```

### 2 liurio.github.io的仓库

### 3 启动jekll

```
bundle exec jekyll serve
```

### 4 测试博客

新增文章应该在 `_posts` 目录下操作，而且注意命名要按照 `年-月-日-标题.后缀名` 的格式。

写好后，执行

```bash
git add .
git commit -m ''
git push origin master
```

模板地址：链接: https://pan.baidu.com/s/1sDupkKEYzoKWfvVAbmw8Cw  提取码: ajnp 