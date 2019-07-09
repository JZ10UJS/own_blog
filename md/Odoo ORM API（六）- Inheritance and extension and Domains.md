---
title: Odoo ORM API（六）- Inheritance and extension and Domains
date: 2016-10-10 11:14:35
tags:
---

# Inheritance and extension

Odoo 提供了三种不同机制用来扩展models in a modular way:

- 继承一个已经存在的model，添加一些新的属性或者方法，但是，源model不会增加属性或方法，只是自己改变
- 在另外一个 Odoo module 中，直接扩展源model，给他添加新的属性或方法，而不用去修改源码
- delegating some of the model's fields to records it contains

![这里写图片描述](http://img.blog.csdn.net/20161010102211150)

## Classical inheritance

当同时使用`_inhert` 和 `_name` 属性时，Odoo 将根据 `_inherit` 创建一个名为 `_name` 的新的model，这个新的 model 将从`_inherit` 的 model 中获取所有的 fields，methods，meta-information。

```python
class Inheritance0(models.Model):
    _name = 'inheritance.0'

    name = fields.Char()

    def call(self):
        return self.check("model 0")

    def check(self, s):
        return "This is {} record {}".format(s, self.name)

class Inheritance1(models.Model):
    _name = 'inheritance.1'
    _inherit = 'inheritance.0'

    def call(self):
        return self.check("model 1")
```
你这样调用

```python
        a = env['inheritance.0'].create({'name': 'A'})
        a.call()
        b = env['inheritance.1'].create({'name': 'B'})
        b.call()
```
产生下述结果

```python
        This is model 0 record A
        This is model 1 record B
```
第二个 model 继承了第一个 model 的 name field 和 check method，但是覆盖了它的 call method，这种继承方式和普通的 Python 继承一样。

## Extension

当只设置了`_inherit`，没有设置`_name` 属性时，这个新的 model 就是直接对源 model 进行了扩展。这种机制通常是在其他 module 中对已有 module 中的model进行添加字段或方法，或者重新定制化他们（如：改变他们的默认 search order）

```python
class Extension0(models.Model):
    _name = 'extension.0'

    name = fields.Char(default="A")

class Extension1(models.Model):
    _inherit = 'extension.0'

    description = fields.Char(default="Extended")
```

```
        record = env['extension.0'].create({})
        record.read()[0]
```
将产生
```
        {'name': "A", 'description': "Extended"}
```
当然，也会返回一些 自动建立的字段值，'create_uid' 之类的

## Delegation

第三种机制就更加的灵活了，使用`_inherits` 属性，using the _inherits a model delegates the lookup of any field not found on the current model to "children" models. The delegation is performed via Reference fields automatically set up on the parent model:

```python
class Child0(models.Model):
    _name = 'delegation.child0'

    field_0 = fields.Integer()

class Child1(models.Model):
    _name = 'delegation.child1'

    field_1 = fields.Integer()

class Delegating(models.Model):
    _name = 'delegation.parent'

    _inherits = {
        'delegation.child0': 'child0_id',
        'delegation.child1': 'child1_id',
    }

    child0_id = fields.Many2one('delegation.child0', required=True, ondelete='cascade')
    child1_id = fields.Many2one('delegation.child1', required=True, ondelete='cascade')
```

```python
        record = env['delegation.parent'].create({
            'child0_id': env['delegation.child0'].create({'field_0': 0}).id,
            'child1_id': env['delegation.child1'].create({'field_1': 1}).id,
        })
        record.field_0
        record.field_1
```
f返回

```
        0
        1
```
可以直接对托管的字段进行修改：

```
        record.write({'field_1': 4})
```
使用这种机制，只有 field 会被继承，method 不会。

# Domains

一个 domain 就是一些条件组成的 list，每一个条件都是由3个元素组成，（`field_name`, `operator`, `value`）

- `field_name (str)`
当前model 的一个 field name，或者是 Many2one field 组成 dot 链。如：`'street'` ，`'partner_id.country'`
- `operator(str)` 用来比较 `field_name` 和 `value`，有：
 - `=` 
 - `!=`
 - `>`
 - `>=`
 - `<`
 - `<=`
 - `=?` 这个字段如果有值，就用`=`比较；如果这个字段值为 False 或者 None，返回 True
 - `=like` 这个字段是否满足 `value` pattern，在 pattern 中 `_` 代表 一个任意字符，`%`代表0或多个任意字符。
 - `like` pattern 直接就是 `%value%`
 - `not like`
 - `ilike` 不区分大小写的 `like`
 - `not ilike`
 - `=ilike` 不区分大小写的 `=like`
 - `in` `value` should be a list of items
 - `not in` 
 - `child_of`
- `value` variable type, must be comparable (through operator) to the named field


Domain 中的条件可以通过下面的逻辑运算符链接起来：

- `&` 逻辑 AND， 两个参数，将会默认的链接相邻的两个条件
- `|` 逻辑 OR， 两个参数
- `!`  逻辑 NOT，一个参数
	注意：表示否定的时候，尽量在条件中的 operator 表示，这样比在外面表示更加的清晰明了。

```python
[('name','=','ABC'),
 ('language.code','!=','en_US'),
 '|',('country_id.code','=','be'),
     ('country_id.code','=','de')]
```
这个翻译过来就是

```
    (name is 'ABC')
AND (language is NOT english)
AND (country is Belgium OR Germany)
```