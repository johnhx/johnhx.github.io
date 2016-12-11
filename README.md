# ey 中文名伊娃

> fuck me, fuck my dog.

[在线预览](http://blog.ibrother.me/jekyll-theme-ey/)

再简约的主题,再文艺的名字,也抵不过神一样的子标题.

项目名:	JEKYLL THEME EY

基于主题:	[HARMONY](https://github.com/gayanvirajith/harmony)

许可证:	MIT License

主题特性:
* 基于github pages,你可以直接使用github的markdown编辑器发表,编辑文章,把github当作后台来用
* 通过data文件实现同一站点多个作者的功能
* 支持当前页面导航的激活特效
* 支持开启或关闭首页分页器
* 可以在每篇文章单独开关多说评论
* 支持添加谷歌,必应,百度的站长验证头部
* 支持设置谷歌分析,百度统计代码
* 支持 [Open Graph](https://developers.facebook.com/docs/opengraph/) 和 [Twitter Cards](https://dev.twitter.com/docs/cards) 带给你更好的社交分享体验
* 支持chrome color和windows8 color
* 使用模板本身的代码而不是通过插件去实现[分类](http://blog.ibrother.me/jekyll-theme-ey/categories/) 和 [标签](http://blog.ibrother.me/jekyll-theme-ey/tags/) 页面
* 强迫症福利,主题内置了包括solarized-light和monokai等共25套代码高亮样式,默认是solarized-light,你可以在配置文件中快速切换,也可以很方便的加入自己想要的高亮样式
* rss订阅采用更好的atom格式
* 使用标准的sitemap站点地图
* 使用微数据结构化网页
* rakefile方便本地创建新文章和新页面

## 目录结构
这里不是jekyll目录结构的教程与介绍,想了解目录的作用可以看官方文档,这里仅对自定义自己站点的文件做简要说明,大部分自定义选项都暴露在主配置文件`_config.yml`和`_data`数据目录下的数据文件里,作为最终用户,你可以很方便的修改.
```
jekyll-theme-ey
├── 404.md
├── about.md
├── archive.md
├── categories.html
├── _config.yml       // 主配置文件
├── css
│   ├── images
│   │   ├── bullet.png
│   │   ├── bullet.svg
│   │   ├── harmony-blog-page.png
│   │   ├── harmony-home-page.png
│   │   ├── harmony-mobile.jpg
│   │   ├── harmony.png
│   │   ├── harmony-web-2.jpg
│   │   ├── harmony-web-3.jpg
│   │   ├── harmony-web.jpg
│   │   ├── socials-icons.png
│   │   └── socials-icons.svg
│   └── main.scss         // 主css样式表,通过在其中引用其他分开的子样式表
├── _data
│   ├── author.yml        // 在这里设置博客作者的信息
│   ├── navs.yml          // 用于定义导航菜单
│   ├── rolls.yml         // 在这个文件里添加友情链接
│   ├── social.yml        // 用来设置社交帐号链接
│   └── webmaster.yml     // 添加站长验证及网站分析相关的设置
├── feed.xml
├── Gemfile
├── _includes
│   ├── baidu_analytics.html
│   ├── blogroll.html
│   ├── duoshuo_comments.html
│   ├── footer.html
│   ├── google_analytics.html
│   ├── header.html
│   ├── head.html
│   ├── scripts.html
│   └── social.html
├── index.html
├── _layouts
│   ├── default.html
│   ├── page.html
│   └── post.html
├── LICENSE
├── _posts
│   ├── 2014-12-18-code-highlighting.md
│   ├── 2015-07-23-welcome-to-jekyll.markdown
│   ├── 2015-07-27-example-content.md
│   └── 2015-07-29-sample-mathjax-post.md
├── rakefile
├── README.md
├── robots.txt
├── _sass
│   ├── _layout.scss
│   ├── _mixins.scss
│   ├── _normalize.scss
│   ├── syntax-highlighting
│   │   ├── _autumn.scss
│   │   ├── _borland.scss
│   │   ├── _bw.scss
│   │   ├── _colorful.scss
│   │   ├── _default.scss
│   │   ├── _emacs.scss
│   │   ├── _friendly.scss
│   │   ├── _fruity.scss
│   │   ├── _igor.scss
│   │   ├── _manni.scss
│   │   ├── _monokai.scss
│   │   ├── _murphy.scss
│   │   ├── _native.scss
│   │   ├── _paraiso-dark.scss
│   │   ├── _paraiso-light.scss
│   │   ├── _pastie.scss
│   │   ├── _perldoc.scss
│   │   ├── _rrt.scss
│   │   ├── _solarized-dark.scss
│   │   ├── _solarized-light.scss
│   │   ├── _tango.scss
│   │   ├── _trac.scss
│   │   ├── _vim.scss
│   │   ├── _vs.scss
│   │   └── _xcode.scss
│   └── _variables.scss
├── sitemap.xml
└── tags.html
```

[![Analytics](https://ga-beacon.appspot.com/UA-52116871-4/jekyll-theme-ey/readme)](https://github.com/igrigorik/ga-beacon) [![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/ibrother/jekyll-theme-ey/trend.png)](https://bitdeli.com/free "Bitdeli Badge")
