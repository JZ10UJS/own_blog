---
title: Odoo Web前端界面详解 - 1
date: 2018-02-03 15:55:20
tags:
---

近期准备对odoo web模块进行一些整理，大致的说说odoo的前端界面是如何生成的，又是如何与后端交互，主要着重点就是odoo的js这块的东西，由于odoo11的js代码已经有了较大的改变，所以就以**odoo11**来分析吧。

为了简单叙述，暂时不考虑多个db的情况（主要是懒得说没有db或者多个db实例的情况）当odoo指定数据库开启服务时（也就是`odoo-bin -d <some_db_name>` ），我们使用chrome的隐身模式访问`http://127.0.0.1:8069` ，此时打开chrome的开发者模式，查看Network可以看到，请求和相应如下

![这里写图片描述](http://img.blog.csdn.net/20180203140906031?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSl96MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 1. 输入http://127.0.0.1:8069/之后发生了什么

`192.168.56.102`， 这个是我的虚机ip地址，返回响应是200，可以通过源代码中， 我们看到
```python
# web/controllers/main.py:435

@http.route('/', type='http', auth='none')
def index(self, s_action=None, db=None, **kwargs):
		return http.local_redirect('/web', query=request.params, keep_hash=True)
```
按照代码命名来看，应该返回一个`30x`的redirect响应，为什么实际返回的确是200的响应头呢？
继续查看代码，可以发现就是`keep_hash=True` 这个参数所引起的，在`local_redirect` 函数中由于keep_hash，
```python
# odoo/http.py:156
def redirect_with_hash(url, code=303):
    if request.httprequest.user_agent.browser in ('firefox',):
        return werkzeug.utils.redirect(url, code)
    url = pycompat.to_text(url)
    if urls.url_parse(url, scheme='http').scheme not in ('http', 'https'):
        url = u'http://' + url
    url = url.replace("'", "%27").replace("<", "%3C")
    return "<html><head><script>window.location = '%s' + location.hash;</script></head></html>" % url
```
调用了`redirect_with_hash` 函数，而多数浏览器是不支持带着hash redirect的，就直接返回了html，并且带有`script`的js代码，使得从js的角度实现带有hash地址的redirect。
通过curl命令验证，实际上确实是如此返回
![这里写图片描述](http://img.blog.csdn.net/20180203142533188?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSl96MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 2. 跳转至/web之后
继续看代码，
```python
# web/controllers/main.py:442
@http.route('/web', type='http', auth="none")
def web_client(self, s_action=None, **kw):
    ensure_db()
    if not request.session.uid:
        return werkzeug.utils.redirect('/web/login', 303)
    if kw.get('redirect'):
        return werkzeug.utils.redirect(kw.get('redirect'), 303)

    request.uid = request.session.uid
    try:
        context = request.env['ir.http'].webclient_rendering_context()
        response = request.render('web.webclient_bootstrap', qcontext=context)
        response.headers['X-Frame-Options'] = 'DENY'
        return response
    except AccessError:
        return werkzeug.utils.redirect('/web/login?error=access')
```
`ensure_db()`，就是一些确定当前用户是否选择db，没有就跳转到/web/database/selector, 或者/web/database/create，由于上面我说了只使用一个db，因为其他情况不想多讲，所以在ensure_db中，默认替我们选择好了唯一一个db，并且存在request.session中了。接下来，由于我们尚未登陆，request.session.uid为空，就返回了 `/web/login` 303的响应。
![这里写图片描述](http://img.blog.csdn.net/20180203144007642?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSl96MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 3. 跳转至/web/login
代码如下
```python
# web/controllers/main.py:467
@http.route('/web/login', type='http', auth="none", sitemap=False)
def web_login(self, redirect=None, **kw):
	ensure_db()
	request.params['login_success'] = False
	
	# 如果是带有redirect的get请求，并且session中有uid（也就是用户已经登陆的情况下),返回redirect
	# 我们目前不是这个情况
	if request.httprequest.method == 'GET' and redirect and request.session.uid:
	    return http.redirect_with_hash(redirect)
	
	# 设置request.uid
	if not request.uid:
	    request.uid = odoo.SUPERUSER_ID
	
	values = request.params.copy()
	try:
	    values['databases'] = http.db_list()
	except odoo.exceptions.AccessDenied:
	    values['databases'] = None
	    
	# 登陆的相关逻辑
	if request.httprequest.method == 'POST':
	    old_uid = request.uid
	    uid = request.session.authenticate(request.session.db, request.params['login'], request.params['password'])
	    if uid is not False:
	        request.params['login_success'] = True
	        return http.redirect_with_hash(self._login_redirect(uid, redirect=redirect))
	    request.uid = old_uid
	    values['error'] = _("Wrong login/password")
	else:
	    if 'error' in request.params and request.params.get('error') == 'access':
	        values['error'] = _('Only employee can access this database. Please contact the administrator.')
	
	if 'login' not in values and request.session.get('auth_login'):
	    values['login'] = request.session.get('auth_login')
	
	if not odoo.tools.config['list_db']:
		# 这个值决定了是否在登陆界面出现Manage Databases超链接
	    values['disable_database_manager'] = True
	
	# GET请求，返回的响应就是这个了，
	response = request.render('web.login', values)
	response.headers['X-Frame-Options'] = 'DENY'
	return response
```
到这，就不得不提一下odoo自己的Qweb模板引擎了，这个`'web.login'`，就是qweb view的id，具体可以通过全局查找定位到`'web.login'` 的位置，其实就是在`web/views/webclient_templates.xml`中301行的位置，qweb有它自己独特的语法，这个要了解就去odoo官网查看吧，然后通过values对这个模板进行渲染，然后返回200的响应，就出现了基本的登录界面了
![这里写图片描述](http://img.blog.csdn.net/20180203151023250?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSl96MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

综上，这就是基本的输入界面到返回界面的一些过程。

# 例子 - 去除登陆页面的Powered by Odoo链接

从上面的第3步，我们可以看到，最后登录界面是由`'web.login'` 模板来显示的，通过odoo的继承方式，我们很容易的就可以去除这个链接，通过查找，这个链接实际是出现在`'web.login_layout'` qweb视图中，

```
<template id='remove_login_odoo_link' inherit_id='web.login_layout'>
    <xpath expr='//div[@t-if="not disable_footer"]' position='replace'>
        <div class="text-right" t-if="not disable_footer">
            <t t-if="not disable_database_manager">
                <a class="" href="/web/database/manager">Manage Databases</a>
            </t>
        </div>
    </xpath>
</template>
```

全部代码，查看 https://github.com/JZ10UJS/extra-addons 的 `remove_login_odoo_link` 模块，

完成后，效果如下
![这里写图片描述](http://img.blog.csdn.net/20180203155336528?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSl96MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)