---
title: Odoo ORM API （二）- Creating Models、Compatibility between new API and old API
date: 2016-09-26 17:16:21
tags:
---

# Creating Models
model fields 就像普通 python类属性一样定义：

```python
from openerp import fields, models, api

class AModel(models.Model):
	_name = 'a.model'

	field1 = fields.Char()
```
> 注意：
> 这意味着，在model中，两个field的 name 不能一样，否则将会出现意想不到的错误。

默认的，在用户界面中 field 的 label 是这个 field name 的 首字母大写，这个可以通过设置field 的 string 参数来修改

```python
field2 = fields.Integer(string='an other field')
```
也可以通过设置 default 参数来设置field 的默认值

```python
a_field = fields.Char(default='a value')
```
或者传递一个function给default

```python
def compute_default_value(self):
	return self.get_value()
a_field = fields.Char(default=compute_default_value)
```
## Computed fields
Fields 可以通过设置`compute` 参数来计算设置它的值（而不仅仅是通过读取数据库中的值），必须在方法中明确设置这个field的值，如果计算过程，可能会用到其它field，那么需要在depends()中明确指定。

```python
from openerp import api
total = fields.Float(compute='_compute_total')

@api.depends('value', 'tax')
def _compute_total(self):
    for record in self:
        record.total = record.value + record.value * record.tax
```

- 可以通过 '.' 来设置对子字段的依赖

```python
@api.depends('line_ids.value')
def _compute_total(self):
    for record in self:
        record.total = sum(line.value for line in record.line_ids)
```

- computed fields 默认情况下，是不存在database中的。它的值，仅仅在请求时才会计算返回。通过传入store=True，可以将这个字段存在db中，并且可以用来 search
- 针对computed field来搜索，也可以通过设置 search 参数来实现，值为一个方法的名字，这个方法将返回一个 search domain

```python
upper_name = field.Char(compute='_compute_upper', search='_search_upper')

def _search_upper(self, operator, value):
    if operator == 'like':
        operator = 'ilike'
    return [('name', operator, value)]
```
- 如果允许直接对computed field 赋值，可以使用 `inverse` 参数，传入值为 一个方法的名字，这个方法用以反向计算和设置相关字段的值

```python
document = fields.Char(compute='_get_document', inverse='_set_document')

def _get_document(self):
	for record in self:
		with open(record.get_document_path()) as f:
			record.document = f.read()

def _set_document(self):
	for record in self:
		if not record.document:
			continue
		with open(record.get_document_path()) as f:
			f.write(record.document)
```
- 多个fields 可以使用相同的 compute method 被同时设置

```python
discount_value = fields.Float(compute='_apply_discount')
total = fields.Float(compute='_apply_discount')

@depends('value', 'discount')
def _apply_discount(self):
    for record in self:
        # compute actual discount from discount percentage
        discount = record.value * record.discount
        record.discount_value = discount
        record.total = record.value - discount
```

### Related fields
一种computed field 的特殊情况就是， related fields。这将由当前record的 某个 关系型 fields 的某个字段的值来作为当前record的field的值，也可以被设置成为store=True

```python
nickname = fields.Char(related='user_id.partner_id.name', store=True)
```
## Onchange: updating UI on the fly
当某个用户在form view 界面修改了某个field的值，但是还没有点击Save时，可以自动的更新form view。
 
 - computed fields 会自动的检查和计算，他们不需要设置 onchange
 -  那些非 computed fields，onchange() 装饰器将会根据改变，自动变化field的值

```python
@api.onchange('field1', 'field2') # 如果这些field的值变化，将调用这个方法
def check_change(self):
	if self.field1 < self.field2:
		self.field3 = True
```
在方法执行时，变化。并且将这些变动传到客户端程序，然后显示出来。

- computed fields 和 new-api onchanges 都会被客户端自动的调用，而不需要在view中对这些field进行额外的设置
- 可以view中添加参数，用以关闭这个 trigger,

```
<field name="name" on_change="0"/>
```
当这个field在用户界面被用户修改时，即使这个字段被明确的设置在depends 或 onchange 参数中，也将不会调用任何计算方法。

> 注意
> 虽然在 onchange 中对 某个字段进行了赋值，但是这个修改将不会真正的修改database中的值，这个计算仅仅用于将这个值传给客户端而已。

## Low-level SQL

environments 的 `cr` 属性是当前数据库的事务 cursor，可以直接执行 SQL 语句，主要是为了那些，不好通过ORM直接描述的查询需求。

```
self.env.cr.execute('some_sql', param1, param2, param3）
```

由于models 使用同样的cursor，而且 Environment 中也会保存一定cache，在使用raw SQL对数据库执行修改之前，必须使这个 cache 无效，否则models的调用可能会出现未知错误。在执行 CREATE，UPDATE，DELETE 的 SQL语句之前，清空cache是很有必要的，如果只是执行 SELECT 语句，就没有这个必要了。
清空cache 可以使用 `self.env.invalidate_all()` 方法

# Compatibility between new API and old API

Odoo 最近才从老API 转移到新API，所以在两者之间互相转换是很有必要的。

- PRC 层（XML-RPC 和 JSON-RPC）都是根据 old API 执行的，而 methods 仅能通过新API 被执行。
- 复写来自 old code 中设置的方法，也有可能按照老式的方法重写

旧API 与新API 之间最大的不同就是：

-  old 中，Enviroment 的值（cursor， user ，context）是被显示的传入到method中
- record data（ids）也被显示的传入到某些方法中，也有可能不传入这个值
- method 都是作用于 id 组成的列表中，而不是recordset

默认的，methods 被假定只能用 new API ，而且不能从old API 中调用新式 API

> 提示:
> 从new API 中调用 old API
> 当在new API中调用 old API时，将会自动转化，不需要做额外的设置

```python
>>> # method in the old API style
>>> def old_method(self, cr, uid, ids, context=None):
...    print ids

>>> # method in the new API style
>>> def new_method(self):
...     # system automatically infers how to call the old-style
...     # method from the new-style method
...     self.old_method()

>>> env[model].browse([1, 2, 3, 4]).new_method()
[1, 2, 3, 4]
```

下面两个装饰器可以将使得old api 调用 new api 设置的方法

- `model()`
这个装饰器，将会不会传入ids，使得recordset的长度为0，old api 中就可以根据这些参数调用 cr, uid, *args, context；

```python
@api.model
def some_method(self, a_value):
	pass

# from old api
old_style_model.some_method(self, cr, uid, a_value, context=context)
```

- `multi()`
这个将会传入 list of ids，也可以是 空 list , old API 就是 cr, uid, ids, *args, context

```python
@api.multi
def some_method(self, a_value):
    pass
# can be called as
old_style_model.some_method(cr, uid, [id1, id2], a_value, context=context)
```

由于new-style 的 API 倾向返回 一个 recordset，而 old-api倾向返回一个 list of ids，这种情况也有一个装饰器来处理

`returns()`
这个函数用来返回一个recordset, 第一个参数应该是 recordset model 的name，或者 self
如果从新式API 中调用，将不会产生作用，但是当从old api 中调用时，将会将这些recordset 转换成 list of ids

```python
>>> @api.multi
... @api.returns('self')
... def some_method(self):
...     return self
>>> new_style_model = env['a.model'].browse(1, 2, 3)
>>> new_style_model.some_method()
a.model(1, 2, 3)
>>> old_style_model = pool['a.model']
>>> old_style_model.some_method(cr, uid, [1, 2, 3], context=context)
[1, 2, 3]
```

