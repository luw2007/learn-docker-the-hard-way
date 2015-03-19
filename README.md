learn docker the hard way
=========================
**Project documentation with Markdown.**
[![Docs Status][readthedocs-image]][readthedocs-link]

从老版本读docker, 深入了解基础实现。  
为什么从老版本开始。 因为代码少，基本都是核心内容。使用以下脚本

	# 统计测试以外的golang 代码行数和单词数
	find . -name "*.go" -exec wc {} \; |grep -v "_test" |sort -u |awk -F" " '{a+=$1;b+=$2}END{print a, b}'
得到`v0.1.0`版本中golang代码**4499 15630**，
`v1.0.0`中则是**101549 376671**

##阅读
[在线阅读][readthedocs-link]  
下载**[源代码][source-link]**，
采用[mkdocs][mkdocs-link]生成html版本阅读体验较佳。  
也可以在[马克飞象][maxiang-link]等[markdown][markdown-link]阅读md源码。


##注意事项
###备注
代码中的原有注释沿用'//'一般为英文注释。
为了区别中文解释使用'//#'中文解释。如：

	// Running in init mode
	//# `SysInit`在`sysinit.go`中。
上面是源代码中的备注， 后面是中文解释

###docker版本
老版采用tag`v0.1.0`, 
对照版采用tag`v1.0.0`

###版权
`luw2007`个人所有，转载请保留版权声明。
如果有其他参与者考虑使用开发版权, 如 Creative Commons licenses。

[readthedocs-image]: https://readthedocs.org/projects/learn-docker-the-hard-way/badge/?version=latest
[readthedocs-link]: http://learn-docker-the-hard-way.readthedocs.org
[source-link]: https://github.com/luw2007/learn-docker-the-hard-way
[mkdocs-link]: http://www.mkdocs.org/
[markdown-link]: http://zh.wikipedia.org/zh-cn/Markdown
[maxiang-link]: http://maxiang.info