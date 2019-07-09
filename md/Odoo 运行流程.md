---
title: Odoo 运行流程
date: 2017-03-31 18:11:00
tags:
---

起始文件
```python
__import__('pkg_resource').declare_namespace('odoo.addons')
import odoo

if __name__ == '__main__':
	odoo.cli.main()
```
默认会执行 odoo/cli/server.py 的 `Server` 类的 `run` 方法。

- 检查是否是 root 用户启动，如果是，将会 log Error 信息
- 处理 config 信息，
- 检查 config 中的 db_user 是否为 'postgres'， 如果是，程序退出，
- 如果不是，log 出 db_user and db_host
- 检查config中的 db_name，尝试创建 名字为 db_name的数据库，如果已经存在此数据库，pass
- 检查 translate_out and translate_in， 是否多进程，是否设置pid file

执行 odoo/service/server.py 的 start 函数

- load server wide modules，默认 ['web', 'web_kanban']
- 实例化 server, (gevent, prefork, multiThreading), 默认多线程的服务启动
- 检查dev_mode中是否有 ，`reload` 监听文件变动， `werkzeug` Debug App
- server 启动， loop   
- server终止后，检查配置中是否设置 'phoenix'， 自动重启

