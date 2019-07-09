---
title: Odoo ORM API（四）- Methond decorators
date: 2016-10-05 21:20:41
tags:
---

# Method decorators

这一小节介绍了对两种不同风格的API的管理，这两种API分别是 `traditional` 和 `record`，在`traditional` 风格中，一些参数（cr, uid, ids, context)是被显示的设置在函数定义中，而在`record` 风格的API中，定义函数是不需要主动设置这些参数的，这些参数都会被隐藏在 model instances 中传入，显得更加的 object-oriented。
比如：

```python
model = self.pool.get(MODEL)
ids = model.search(cr, uid, DOMAIN, context=context)
for rec in model.browse(cr, uid, ids, context=context):
    print rec.name
model.write(cr, uid, ids, VALUES, context=context)
```
也可以这样

```python
env = Environment(cr, uid, context) # cr, uid, context wrapped in env
model = env[MODEL]                  # retrieve an instance of MODEL
recs = model.search(DOMAIN)         # search returns a recordset
for rec in recs:                    # iterate over the records
    print rec.name
recs.write(VALUES)                  # update all records in recs
```

用`traditional` 方式的method会被自动的decorated，可以跟着下面的列子，看看效果

- `openerp.api.multi(method)`对一个 `self` 是 recordset的进行装饰，需要按照下述方法
 
```python
@api.multi
def method(self, args):
    ...
```
这样就可以通过两种方式对其调用

```python
# recs = model.browse(cr, uid, ids, context)
recs.method(args)

model.method(cr, uid, ids, args, context=context)
```

- `openerp.api.model(method)`
对一个 `self` 是一个recordset的method进行装饰，但是这个method与self中内容无太大关系，反而是与这个model有关联，就用这种方式进行装饰

```python
@api.model
def method(self, args):
    ...
```
可以通过下面两种方式调用
 

```python
# recs = model.browse(cr, uid, ids, context)
recs.method(args)

model.method(cr, uid, args, context=context)
```
注意：这种装饰之后，用traditional的方式调用，将不用传入ids。

- `openerp.api.depends(*args)`
返回一个装饰器，指定了某个compute field 依赖的fields（就相当于新API中的 function field)，每一个参数必须是字符串，字符串中可以用 dot将field name 链接起来

```python
pname = fields.Char(compute='_compute_pname')

@api.one
@api.depends('partner_id.name', 'partner_id.is_company')
def _compute_pname(self):
    if self.partner_id.is_company:
        self.pname = (self.partner_id.name or "").upper()
    else:
        self.pname = self.partner_id.name
```
也可以传一个function作为参数，在这种情况下，将直接用 model 调用 这个function 来设置他们之间的依赖

- `openerp.api.constrains(*args)`
用以设置限制条件，每个参数必须是字段的名字

```python
@api.one
@api.constrains('name', 'description')
def _check_description(self):
    if self.name == self.description:
        raise ValidationError("Fields name and description must be different")
```
在传入参数字段的值改变时，将会自动调用这个方法，进行检查。函数定义中，如果不满足条件，开发者应当raise `ValidationError`
> 警告:
> constrians 仅仅支持普通的field name，dotted name 字段中的字段，比如 partner_id.customer 。这种将会被忽略，不会检查。

- `openerp.api.onchange(*args)`
用以设置当某个字段变化时的效果，每一个参数必须是字段名字

```python
@api.onchange('partner_id')
def _onchange_partner(self):
    self.message = "Dear %s" % (self.partner_id.name or "")
```
在这个field出现的 **form view** 中，如果这个field的值发生改变，那么这个方法就会被调用。但是这个方法是作用在 form view 中的 值组成的一个 假的record 上面，方法中对这个record的改变也不是真正的改变，他仅仅是计算好值，并且将这个值发送给 client而已，用来改变form view 中对应的值得展现。
这个方法也可以返回一个字典，用以改变field domains 或者弹出一个警告窗口。

```
return {
    'domain': {'other_id': [('partner_id', '=', partner_id)]},
    'warning': {'title': "Warning", 'message': "What is this?"},
}
```

> 警告：
> onchange 和 constrains 一样，只能作用于普通的field name，关系型字段下的字段 （e.g: partner_id.tz），将会被忽略，并且不会生效。

- `openerp.api.returns(model, downgrade=None, upgrade=None)`
装饰返回 instances of mode 的方法
|Name|Description|
|-|-|
|Parameters:|**model** - 一个model的名字，或者`self`表示当前model<br/>**downgrade** - 一个function downgrade(self, value, *args, \*\*kwargs)用以将 record-style 的返回值 转换成 traditional-style的值<br/>**upgrade** - 一个function upgrade(self, value, *args, \*\*kwargs)用以将 traditional-style 的返回值 转换成 record-style的值|

这两个function中的 self，*args, \*\*kwargs 都是record-style中的参数。
这个装饰器使得函数的返回值根据调用者的style返回对应的值：
old-style：返回 id, ids, Flase
record-style：返回 recordset

```python
@model
@returns('res.partner')
def find_partner(self, arg):
    ...     # return some record

# output depends on call style: traditional vs record style
partner_id = model.find_partner(cr, uid, arg, context=context)

# recs = model.browse(cr, uid, ids, context)
partner_record = recs.find_partner(arg)
```
Note that the decorated method must satisfy that convention.
Those decorators are automatically inherited: a method that overrides a decorated existing method will be decorated with the same @returns(model).
 
- `openerp.api.one(method)`
对一个 self 是 recordset的，进行装饰。但是这个self应该是长度为1。这个被装饰后的method被调用时，将会自动一个loop 到调用它的每一个record上，并将每一个record的返回值组成一个列表返回。比如这个method同样被 returns() 装饰了，那么他将会把结果一个的组合起来，再返回。

```
@api.one
def method(self, args):
    return self.name
```
也可以这样调用 
```
# recs = model.browse(cr, uid, ids, context)
names = recs.method(args)

names = model.method(cr, uid, ids, args, context=context)
```
只能在9.0及以后使用，`one()` 虽然可以使代码简单明了，但是它的行为可能不会像开发者所预期的那样
所以，我们强烈推荐使用`multi()`，在其中对recordset 进行迭代。如果明确是只想作用于一个record上，可以在method中，使用 self.ensure_one()，然后就不需要使用迭代了。

- `openerp.api.v7(mothod_v7)`
对一个method装饰，使其只支持 old-api style。一个 new-api style可以通过定义一个名字相同的method，但是用 v8()装饰

```python
@api.v7
def foo(self, cr, uid, ids, context=None):
    ...

@api.v8
def foo(self):
    ...
```
Special care must be taken if one method calls the other one, because the method may be overridden! In that case, one should call the method from the current class (say MyClass), for instance:

```python
@api.v7
def foo(self, cr, uid, ids, context=None):
    # Beware: records.foo() may call an overriding of foo()
    records = self.browse(cr, uid, ids, context)
    return MyClass.foo(records)
```
Note that the wrapper method uses the docstring of the first method.

- `openerp.api.v8(method_v8)`
Decorate a method that supports the new-style api only. An old-style api may be provided by redefining a method with the same name and decorated with v7():

```
@api.v8
def foo(self):
    ...

@api.v7
def foo(self, cr, uid, ids, context=None):
    ...
```

Note that the wrapper method uses the docstring of the first method.

 
 
 
 

  
 
 
 
 
 

 
 
 

 
 
 

 
 
 
 
 
 
 
 

 

 
 
 
 
 
 