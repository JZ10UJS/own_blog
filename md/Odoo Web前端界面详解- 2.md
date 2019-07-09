---
title: Odoo Web前端界面详解- 2
date: 2018-02-04 16:28:32
tags:
---

上篇讲到，出现登陆界面，在我们输入用户名密码之后，odoo又做了什么，chrome中开发人员模式，看到请求如下
![这里写图片描述](http://img.blog.csdn.net/20180204151025167?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSl96MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
代码如下
```python
# web/controllers/main.py:483
		# 登陆逻辑，
        if request.httprequest.method == 'POST':
            old_uid = request.uid
            uid = request.session.authenticate(request.session.db, request.params['login'], request.params['password'])
            if uid is not False:
                request.params['login_success'] = True
                # 密码验证通过之后，返回一个带hash的响应，实际上是200响应头，js实现跳转到/web
                return http.redirect_with_hash(self._login_redirect(uid, redirect=redirect))
            request.uid = old_uid
            values['error'] = _("Wrong login/password")
        else:
            if 'error' in request.params and request.params.get('error') == 'access':
                values['error'] = _('Only employee can access this database. Please contact the administrator.')

# 在/web route下
# web/controllers/main.py:443
@http.route('/web', type='http', auth="none")
    def web_client(self, s_action=None, **kw):
        ensure_db()
        if not request.session.uid:
            return werkzeug.utils.redirect('/web/login', 303)
        if kw.get('redirect'):
            return werkzeug.utils.redirect(kw.get('redirect'), 303)
		
		# /web/login成功之后的跳转，session已经有uid, 返回html
        request.uid = request.session.uid
        try:
	        # 获取当前用户的一些context,包括菜单之类的
            context = request.env['ir.http'].webclient_rendering_context()
            # 用模板渲染html
            response = request.render('web.webclient_bootstrap', qcontext=context)
            response.headers['X-Frame-Options'] = 'DENY'
            return response
        except AccessError:
            return werkzeug.utils.redirect('/web/login?error=access')
```
简短就是，验证登陆用户名密码成功之后，将用户uid写入session之中，然后跳转至/web页面；/web请求时，获取用户的一些context包括能看到的菜单等，然后通过模板渲染，最后返回响应。

`'web.webclient_bootstrap'`模板可以在web/views/webclient_templates.xml:597行找到，通过模板我们可以看到只生成了一些基本的静态文件，以及当前登录用户的所能访问到的菜单。所有具体的页面内容都是后来由js处理和生成的。

所有的静态文件，如果没有开启debug=assets的话，都是由ir_qweb.py进行压缩成单独的一个js文件，当然其实有两个，一个是web.assets_common，一个是web.assets_backend（我们主要就是研究这个咯）
在webclient_templates.xml中找到id='web.assets_backend'的template，和'web.assets_common'的template。 common里面导入一些通用的js文件，backend的js文件主要是登录成功后渲染/web里面内容所单独需要的。

下一篇，我们主要讲讲common中的boot.js 和 class.js。