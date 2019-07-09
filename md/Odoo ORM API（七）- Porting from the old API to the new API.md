---
title: Odoo ORM API（七）- Porting from the old API to the new API
date: 2016-10-11 09:14:11
tags:
---

# Porting from the old API to the new API

- 在 new API 中避免了直接使用 ids 组成的 list，而使用 recordsets
- 使用 old API 方式写的 method 将会被 ORM 自动的转换，没有必要切换到老的 pool 来调用 old API，可以直接把它当成 new API ，用 recordset 来调用
- `search()` 直接返回一个 recordsets
- `fields.related` and `fields.function` 都被替换成普通的field，只不过给它设置了 `related=` 或者 `compute=` 参数
- `computed=` 设置的 method 必需设置`depends()`，必需将所有依赖的字段全部列出来，多列出来比少列出来要好，以免该 重新计算的时候没有计算
- 移除了所有`computed` fields 的 `onchange`  method，`computed` fields 在他依赖的字段发生变化时，将会自动的重新计算，也会被clinet 自动的 `onchange`
- 装饰器 `model()` 和 `multi()` 是为了从 old API 调用它时，自动转换。如果你的项目中全是 new API 的话，那么这两个装饰器不会起作用
- 移除 `_default`，该而在相关字段定义时，设置 `default`  参数。
- 如果某个字段的 `string=` 参数就是 field name的首字母大写，那么你应该避免设置这个参数

	```python
	name = fields.Char(string="Name")
	```


- `multi=` 参数也不会在 new API 定义中生效，可以通过给相关 field 设置 `compute=` 同一个方法
- `compute=` `inverse=` `search=` 的值都是作为字符串传递的，这也方便复写
- 必需确保所有的 field name 和 method name 的名字没有冲突，如果重名了，Odoo 将不会产生任何的警告，因为在 Odoo 系统生效之前，Python 已经先行处理了。
- 通常 new-api 中 import 就是 `from openerp import fields, models`，如果有必要使用装饰器，使用`from openerp import api, fields, models`
- 尽量不要使用 `one()` 装饰器，它很有可能不会按照你预期的结果返回
- 移除了显示的定义`create_uid`, `create_date`, `write_uid`, `write_date` fields，他们现在像普通的fields一样，可以被读写 out-of-the-box
- when straight conversion is impossible (semantics can not be bridged) 或者 odl API 版本不太符合新的要求，在 new API 中改进了。你可以使用装饰器`v7()` `v8()` 来装饰同一名字的method，那么当你从 old-style 的方式调用这个method，那么将使用 v7装饰的method, 如果你是以 recordset来调用这个 method，那么将使用 v8 装饰的method。One implementation can call (and frequently does) call the other by switching context.

	> 使用这两个装饰器，将会使得methods非常难的复写，并且不宜被理解

- `_columns` or `_all_columns` 应该用 `_fields` 替换，which provides access to instances of new-style openerp.fields.Field instances (rather than old-style openerp.osv.fields._column).
Non-stored computed fields created using the new API style are not available in _columns and can only be inspected through _fields

- 在 method 中重新定义 self 将是没有必要的， may break translation introspection
- `Enviroment` 对象通常依赖于一些 threadlocal state，通常在使用之前就被初始化定义好了。如果你在 env 没有初始化的情况下，想通过new API context 做一些事情，你必需要使用`openerp.api.Envrionment.manage()`。比如是一个新的线程，或者python的交互模式中。

	```python
>>> from openerp import api, modules
>>> r = modules.registry.RegistryManager.get('test')
>>> cr = r.cursor()
>>> env = api.Environment(cr, 1, {})
Traceback (most recent call last):
  ...
AttributeError: environments
>>> with api.Environment.manage():
...     env = api.Environment(cr, 1, {})
...     print env['res.partner'].browse(1)
...
res.partner(1,)
```

## Automatic bridging of old API methods

当初始化一个 model 的时候，这个 method 的方法将会被自动扫描，如果这个 model 是按照 old-api的方式被定义，那么这些方法也会被自动的 bridged。这使得它们可以在 new-api style 中被调用。
如果 methods 的第二个位置参数（after self）是 `cr` 或者 `cursor`，那么这个method 就 matched as "old-API style"，system 也能识别到第三个位置参数 being called `uid` or `user` ，第四个位置参数 `id ` or `ids`，以及关键字参数`context`。
当从 new API context 下面调用这些 符合上述要求的 methods，Odoo 将会自动的从当前Environment 带入（cr, user, context)，从 self 中 带入 （id or ids）。
复非常少见，但是还是有必要说一下的，通过装饰器，可以对 old-style method 自定义转换：
 
 - 可以通过装饰器 `noguess()` 来停止对这个 method 的自动 bridge。这样你就必需按照定义的方式从 按照 新旧 API 方式分别调用
 - 自定义 bridge，通常是因为有些 method 定义的不太准确（因为 参数的命名不是按照期望的那样）：
- `cr()`
将会自动的在已设置的参数之前，添加一个 current cr
- `cr_uid()`
将会自动的在已设置的参数之前，添加一个 current cr, uid
- `cr_uid_ids()`
将会自动的在已设置的参数之前，添加一个 current cr, uid, ids
- `cr_uid_id()`
loop over the current recordset and call the method for each record,将会自动的在已设置的参数之前，添加一个 current cr, uid, id。这个装饰器之后，即使从 new-api 调用也会返回 list，而不是recordset

	这几个方法都可以在后面添加 `_context()`，用以传入 关键字参数context。 如： cr_context(), cr_uid_context() 等



 - dual implementations using v7() and v8() will be ignored as they provide their own "bridging" 	