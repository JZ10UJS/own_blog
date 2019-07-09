---
title: 基于Nginx location下Odoo12(伪)读写分离 (二)
date: 2019-07-05 18:00:57
tags:
---

# 拓扑图
![请求关系](https://img-blog.csdnimg.cn/2019070516382479.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0pfejEw,size_16,color_FFFFFF,t_70)
从上图可以看到, 其实如果Nginx不代理某些静态文件的话, Nginx可以独立出主服务器,  但是专业的事情还是让专业的人来做, 各模块`static`目录下的`js, css, xml` 文件, `/web/content/.*\.(js|css)`, 就还是让Nginx来处理吧. 一些用户上传的附件, 还是由Odoo处理

# Nginx 配置
```nginx
upstream odoo-main {
    server 127.0.0.1:8469;  # 主库
}
upstream odoo-read {
    server 127.0.0.1:8569;  # 读从库
}
upstream odoo-longpolling {
    server 127.0.0.1:8472;
}

server {
    listen 80;
    server_name  odoo12.jz10.com;		# 随意配置一个域名, 自己在hosts文件里面添加
    access_log   /var/log/nginx/odoo12/access_odoo12.log;
    error_log    /var/log/nginx/odoo12/error_odoo12.log;

    # Add Headers for odoo proxy mode
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_connect_timeout 720s;
    proxy_read_timeout 720s;
    proxy_send_timeout 720s;
	
    location /longpolling/poll {
        proxy_pass  http://odoo-longpolling;
    }

    # 只读的请求
    location /web/dataset/search_read {
        proxy_pass  http://odoo-read;
    }
    location ~ /web/dataset/call_kw/[^/]+/(load_views|read|fields_get|name_search|name_get) {
        proxy_pass  http://odoo-read;
    }

    # 静态文件分离
    location ~ ^/web/content/.*\.(js|css)$ {
        expires 7d;
        root /data/Odoo-12;		# 指向odoo配置文件中的 $(data_dir)
    }
    location ~ ^/[^/]+/static/.*\.(less|scss)\.css$ {
        expires 7d;
        root /data/Odoo-12/less_css;
    }
    location ~ ^/[^/]+/static/ {
        expires 7d;
        root /data/Odoo-12/addons_static;
    }
	
	# 其他Odoo请求
    location / {
        proxy_redirect  off;
        proxy_pass  http://odoo-main;
    }
    
	# common gzip
 	gzip_types text/css text/scss text/plain text/xml application/xml application/json application/javascript;
 	gzip on;
}
```
简单粗暴, 分离了特定的静态文件请求和只读请求.

- `^/web/content/.*\.(js|css)$` 主要是odoo将各模块`js|css`文件压缩之后产生的`web.assets_bankend.js|css`等, 主要是进入`http://odoo12.jz10.com/web`之后会请求一次
- `^/[^/]+/static/.*\.(less|scss)\.css$` 这个是个模块`static`目录下的`scss|less`文件, 原本文件路径是`web/static/src/scss/banner.scss`, 但是访问的url会变成`/web/static/src/scss/banner.scss.css`, 这种主要是`http://odoo12.jz10.com/web?debug=assets`会请求这类url
- `~ ^/[^/]+/static/` 这个就是完全代理各模块的`static`目录下的各种文件了

这些静态文件都改为了Nginx代理, 也就是说不需要登录就能访问这些文件, 但是这类静态文件登不登陆又有什么关系捏.

# 静态文件分离[^1]
如果不想分离静态文件, 觉得这个东西可有可无, 这一块是可以不用参考的, 只需在Nginx配置中注释掉静态文件相关的location, 那样这些文件请求自然会被`odoo-main`处理.
静态文件魔改较多, 无法通过模块的方式进行扩展, 只能改源码了
```python
# odoo/addons/base/models/assetsbundle.py
from odoo.tools import config	# 添加


class AssetsBundle(object):
	# ....
	def clean_attachments(self, type):
		# ---
		# 一堆代码
		# ---
		# force bundle invalidation on other workers
        self.env['ir.qweb'].clear_caches()
        ats = ira.sudo().search(domain)	# 添加此行
		self.remove_my_att(ats)	# 添加此行
        return ats.unlink()		# 修改此行

	def remove_my_att(ats):
        for at in ats:
            path = os.path.join(config['data_dir'], at.url[1:])
            try:
                os.remove(path)
                if len(os.listdir(os.path.dirname(path))) == 0:
                    os.removedirs(os.path.dirname(path))
            except Exception as e:
                _logger.error(e)
            else:
                _logger.debug('rm file: %s', path)

    def save_attachment(self, type, content, inc=None):
        assert type in ('js', 'css')
        ira = self.env['ir.attachment']
        # -----
        # 就不复制了
        # -----
        if self.env.context.get('commit_assetsbundle') is True:
            self.env.cr.commit()

        self.clean_attachments(type)
        self.write_file(url, content.encode('utf8'))	# 添加此行
        return attachment
    
    # 添加此方法    
    def write_file(url, byte_content):
    	m_root = config['data_dir']
        if url.endswith('.less.css') or url.endswith('.scss.css'):
        	# 此类文件, 集中存放在 less_css 下
            full_path = os.path.join(m_root, 'less_css', url[1:])
        else:
        	# 其它, 其实就是 /web/content/.*\.(js|css)此类文件了
            full_path = os.path.join(m_root, url[1:])
        dn = os.path.dirname(full_path)
        if not os.path.isdir(dn):
            os.makedirs(dn)
        with open(full_path, 'wb') as fp:
            _logger.debug("create file %s on %s", url, full_path)
            fp.write(content)
	# ....

	# ....
    def preprocess_css(self, debug=False, old_attachments=None):
        #....
        if self.stylesheets:
            compiled = ""
            for atype in (SassStylesheetAsset, ScssStylesheetAsset, LessStylesheetAsset):
                assets = [asset for asset in self.stylesheets if isinstance(asset, atype)]
                #....
            while fragments:     
                # 一堆代码
                #....
                if debug:
                    try:
                   	    #....
		                # 一堆代码
		                #....
                        if self.env.context.get('commit_assetsbundle') is True:
                            self.env.cr.commit()
                        self.write_file(url, asset.content.encode('utf8'))	# 添加此行, 这里就是产生scss.css文件处, 这种要重启odoo之后, 才会创建, 普通安装模块不会触发preprocess_css
                    except psycopg2.Error:
                        pass
               
```

核心就是`write_file`, 我们在保存附件`(type=js|css)`的时候, 多保存了一份, 放在`$(data_dir)`目录下面, 预处理`scss`文件的时候, 也额外存了一份. 都是按照附件的url路径存放. 清理附件的时候, 也尝试去`$(data_dir)`下面去删除附件, 其实要是不在乎硬盘大小的话可以不去删除.

至此, 我们解决了`web/content/.*\.(js|css)` 和 `.scss.css`文件的Nginx代理问题, 用存2份的方式, 原生无规律的存放和按照url路径存放, 接下来就是各模块的`static`目录代理问题, 这个就是在导入模块时, 在`$(data_dir)/addons_static`目录下创建软链接即可.
```python
	# odoo/http.py line 1429
    def load_addons(self):
        """ Load all addons from addons path containing static files and
        controllers and configure them.  """
        # TODO should we move this to ir.http so that only configured modules are served ?
        statics = {}
        for addons_path in odoo.modules.module.ad_paths:
            for module in sorted(os.listdir(str(addons_path))):
                # --
                # 一堆代码
                # -- 
                        _logger.debug("Loading %s", module)
                        addons_manifest[module] = manifest
                        statics['/%s/static' % module] = path_static
                        # 在导入模块时, 创建各模块的软链接
                        m_dir = os.path.join(odoo.tools.config['data_dir'], 'addons_static', module)
                        if not os.path.isdir(m_dir):
                            os.makedirs(m_dir)
                            os.symlink(path_static, os.path.join(m_dir, 'static'))
```
重启Odoo服务后, 各模块`static`目录的软链接会自动创建, 如果此时访问`/web`, 界面应该不会渲染成功, 因为我们的各种静态文件并没有重新生成, 所以可以在数据库中执行如下命令.
```sql
delete from ir_attachment where url like '/web/content/%';
```
删除所有的Odoo产生的编译之后的静态文件, 然后重新访问`/web`, 此时就会自动创建新的静态文件, 在`$(data_dir)`下也会多存储一份.

至此, 如果你把Nginx upstream `odoo-read`改为`odoo-write`的端口, 那么应该可以成功的实现静态文件的请求的分离了.  甚至于你可以在直接启动新的`odoo-read`应用并且配置db为master的相关配置, 就能完整的跑起来. 但是, 我们的目标要指向从库, 如果你尝试把db配置为slave的信息, 那么`odoo-read`是启动不了的

我们下篇继续.

[^1]: [Odoo 的静态资源优化方案](http://www.oejia.net/blog/2016/12/16/odoo_static.html)
