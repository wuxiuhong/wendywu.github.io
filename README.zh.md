Wendy Blog
========

### 安装Jekyll 

安装Jekyll之前，要先确保在你的电脑上已经配置好Jekyll运行所需要的环境，。

- Ruby
- RubyGems
- NodeJS或其他 JavaScript 运行环境（如果还没安装NodeJS的，可以参照我写的另一篇文章Mac下安装nvm和NodeJS）
- Python2.7(或2.7以上版本)

安装好以上环境后，使用RubyGems安装Jekyll。执行命令：

```yml

gem install jekyll

```

跟npm一样，所有的 Jekyll 的 gem 依赖包都会被自动安装。如果需要安装指定版本，在上述命令后加上一个-v参数

```yml

gem install jekyll -v '指定版本号'

```

### 快速启动


```yml

jekyll build
jekyll serve

# 直接运行本地可以
npm run dev

```

## License

Code released under the [MIT](https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll/blob/gh-pages/LICENSE) license.