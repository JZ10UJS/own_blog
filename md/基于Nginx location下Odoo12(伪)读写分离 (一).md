---
title: 基于Nginx location下Odoo12(伪)读写分离 (一)
date: 2019-07-04 18:34:30
tags:
---

# 序言
各位odoo的开发应当都知道, odoo上生产环境一般就两台机器, 一台应用服务器, 一台数据库服务器. pg的扩展还可能通过pgpoll实现, 通过中间件的方式达到应用层无感的读写分离. 但是应用服务器, 应当如何扩展, 网上可查的资料寥寥无几. 为此, 我给大家提供一些微小的经验.
> 注意: 即将说的这些东西, 我都没上生产, 也就自己弄着玩了玩, 各位自行取舍

# 背景
odoo应用服务为什么难以扩展, 难以在多台服务器上部署, 主要是在以下两个问题

 - session store的问题
 - 附件(ir.attachment)存放的问题

## Session store
odoo默认使用FilesystemSessionStore, 默认存放session文件路径为`$(data_dir)/sessions`, `data_dir`为启动odoo的配置文件中所配置的参数, 默认是 `~/.local/share/Odoo`
```python
# odoo/http.py line:1299
	@lazy_property
	def session_store(self):
	    # Setup http sessions
	    path = odoo.tools.config.session_dir
	    _logger.debug('HTTP sessions stored in: %s', path)
	    return werkzeug.contrib.sessions.FilesystemSessionStore(
	        path, session_class=OpenERPSession, renew_missing=True)
```
也就是说, 你网页中的cookie, 如果`session_id=26172fdbca61836e6434d5939f036acdf37abe13`, 那么在应用服务器下, 就有`$(data_dir)/sessions/werkzeug_26172fdbca61836e6434d5939f036acdf37abe13.sess` 文件,
可以`python3 -m pickle $(data_dir)/sessions/werkzeug_26172fdbca61836e6434d5939f036acdf37abe13.sess` 查看session文件都放了什么东西.

session的解决方案很简单, 用redis来存放即可, 网上也有解决方案
 - [odoo-redis](https://github.com/keerati/odoo-redis)
 - [simle_redis](https://github.com/Smile-SA/odoo_addons/tree/12.0/smile_redis_session_store)

都是通过模块的方式, 替换为RedisSession, 此问题解决(其实通过模块的方式替换session_store, 有可能会坑, 我遇到了)

## 附件(ir.attachment)
odoo提供了附件的功能, 用以存放图片, js, css等静态文件. odoo默认使用的包括`res.partner`模型的`image`字段, 各种编译之后的`scss.css`文件, 以及压缩之后的 `web.assets_common.js`, `web.assets_backend.js`等js文件. 存储方式都是在db的 `ir_attachment` 表的 `store_fname` 字段中存放文件路径. 
> 例:
> 如果`web.assets_backend.js`的`store_fname`是`8a/8a35aa0dd91cca1d44e647cc776184777664d162`, 那么可以该js文件实际存放在 `$(data_dir)/filestore/$(db_name)/$(store_fname)`

是的, 又出现了`data_dir`, 如果odoo应用程序部署在两台机器上, `data_dir`的读取问题是无法避免的.也就是说一个Odoo应用能找到对应的附件, 另一台服务器上的Odoo就只能返回file not exists. 要不用网络文件存储?这不失为一种方案, 但是我们是否可能让只有一台Odoo去接受会读取附件的请求呢? 可以哦...\^\_\^

# 解决方案
但是, 如果说, 我们扩展的一台应用服务机器,永远不读取`data_dir`, 是否可行? 哈哈, 一切都得感谢Odoo做的良好的api. 如果大家分析各自的系统日志, 会不会发现, 大量的请求都是 `search_read|name_get|default_get|name_search|read|load_views|fields_get`, 而且这类请求都不涉及到附件的读取. 如果说, 我们让这些请求流量到达新扩展的Odoo服务中, 是不是能减少主服务器的压力?
>例:
>大家可以对odoo服务的日志进行分析, 查看这类请求的占比, 我这边大概有 40%
>```bash
>grep werkzeug odoo-server.log | wc -l 	# 总的http请求量
>grep werkzeug odoo-server.log | egrep 'search_read|name_get|default_get|name_search|read|load_views|fields_get' | wc -l   # 满足这类请求的请求数量```

所以, Nginx Location 就来了, 我们下篇再说
