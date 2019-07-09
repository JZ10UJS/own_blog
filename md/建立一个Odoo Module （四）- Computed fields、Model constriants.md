---
title: 建立一个Odoo Module （四）- Computed fields、Model constriants
date: 2016-09-22 14:54:07
tags:
---

# Computed fields and default values
目前为止，fields 都是直接将数据写到数据库或者从数据库读取数据。Fields 中同样可以通过调用方法获取动态计算值，而不是从数据库中取数据。
创建 computed field 的方式就是给一个 field 设置 `compute` 属性，并将其值 = method name。那么这个method 就会对 self 这个model的所有record自动生效。
> 注意
> self 不仅可以像普通 python class 定义中一样，调用方法，在 Odoo 中 self 同时代表 recordset，是一个有序的自己 model record 的集合，支持普通python 方法， len(self) and iter(self)， 同时也可以将两个recordset 相加。 recs1 + recs2
> 迭代 self， 将会把 records 中的record 一个一个返回，同时返回的每一个 record 也是一个 recordset，只不过 size == 1。也可以通过 点 来调用 单个record的属性。 record.name
> 

```python
import random
from openerp import models, fields, api

class ComputedModel(models.Model):
    _name = 'test.computed'

    name = fields.Char(compute='_compute_name')

    @api.multi
    def _compute_name(self):
        for record in self:
            record.name = str(random.randint(1, 1e6))
```

## Dependencies
computed field 的值通常是根据其他 field 的 value 计算出来的。Odoo 的 ORM 就希望开发者能够明确指出需要依赖的 field 是哪些，所以提供了一个装饰器 depend()。**无论在哪**，只要当被依赖的 fields 变动时，Odoo就会根据这个自动重新计算 computed field 的值。

```python
from openerp import models, fields, api

class ComputedModel(models.Model):
    _name = 'test.computed'

    name = fields.Char(compute='_compute_name')
    value = fields.Integer()

    @api.depends('value')
    def _compute_name(self):
        for record in self:
            record.name = "Record with value %s" % record.value
```
***
练习 4-1

 - 在 session model 中添加座位使用的 百分比
 - 将这个新增 field 添加到 tree form view 中
 - 将这个field 以 process bar的形式展现出来

openacademy/models.py

```
    course_id = fields.Many2one('openacademy.course',
        ondelete='cascade', string="Course", required=True)
    attendee_ids = fields.Many2many('res.partner', string="Attendees")
	# 新增
    taken_seats = fields.Float(string="Taken seats", compute='_taken_seats')

    @api.depends('seats', 'attendee_ids')
    def _taken_seats(self):
        for r in self:
            if not r.seats:
                r.taken_seats = 0.0
            else:
                r.taken_seats = 100.0 * len(r.attendee_ids) / r.seats
    # 结束
```
openacademy/views/openacademy.xml

```
                                <field name="start_date"/>
                                <field name="duration"/>
                                <field name="seats"/>
                                <!-- 新增 -->
                                <field name="taken_seats" widget="progressbar"/>
                                <!-- 结束 -->
                            </group>
                        </group>
                        <label for="attendee_ids"/>
```

```
                <tree string="Session Tree">
                    <field name="name"/>
                    <field name="course_id"/>
                    <!-- 新增 -->
                    <field name="taken_seats" widget="progressbar"/>
                    <!-- 结束 -->
                </tree>
            </field>
        </record>
```

***

## Default values
任何类型的 field 都可以设置 default value，只需要在定义时，添加 `default=x` 。这个 x 可以是python 中的 boolean, string, int, float 等。也可以接受一个函数，以recordset 作为参数，返回一个值

```python
name = fields.Char(default="Unknown")
user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
```

> Note:
>  `self.env` 提供了一系列的借口，用以处理一些东西
>  
>   - `self.env.cr` 或者 `self._cr` 是database的corsor， 可以通过这个，直接调用SQL
>   - `self.env.uid`  or `self._uid` 时当前登录用户的数据表 id
>   - `self.env.user` 当前登录用户的 record
>   - `self.env.context` or `self._context` 当前的 context directory
>   - `self.env.ref(xml_id)` 通过 xml_id 返回 对应的 record
>   - `self.env[model_name]` 返回 model_name 对应的 空 record

***
练习 4-2

- 给 start_date 设置默认值， 为 当前时间
- session 中添加 active 字段

openacademy/models.py

```python
    _name = 'openacademy.session'

    name = fields.Char(required=True)
    start_date = fields.Date(default=fields.Date.today)   # 修改
    duration = fields.Float(digits=(6, 2), help="Duration in days")
    seats = fields.Integer(string="Number of seats")
    active = fields.Boolean(default=True)  # 新增

    instructor_id = fields.Many2one('res.partner', string="Instructor",
        domain=['|', ('instructor', '=', True),
```
openacademy/views/openacademy.xml

```
                                <field name="course_id"/>
                                <field name="name"/>
                                <field name="instructor_id"/>
                                <field name="active"/> <!-- 新增 -->
                            </group>
                            <group string="Schedule">
                                <field name="start_date"/>
```
> 注意：
> Odoo 有一个內建机制， 将会**自动的隐藏**掉 active 字段值为 False 的record

***
# Onchange
Odoo onchange 机制主要用于在 client 中的 form view。当某个字段改变时，自动 update form view，不会主动存在database 中。注意，这个主要作用于 **客户端**
比如：某个model 拥有三个字段 amout, unit_price and price。我们希望在form中修改其它两个字段时， price 的值会自动的改变。为此，提供了一个装饰器 onchange() 来实现这个功能，

```
<!-- content of form view -->
<field name="amount"/>
<field name="unit_price"/>
<field name="price" readonly="1"/>
```
```python
# onchange handler
@api.onchange('amount', 'unit_price')
def _onchange_price(self):
    # set auto-changing field
    self.price = self.amount * self.unit_price
    # Can optionally return a warning and domains
    return {
        'warning': {
            'title': "Something bad happened",
            'message': "It was very bad indeed",
        }
    }
```
***
练习  4-3
添加一个针对 字段变化的 onchange, 当form中字段值填写不合法的时候，弹出警告

openacademy/models.py
```python
                r.taken_seats = 0.0
            else:
                r.taken_seats = 100.0 * len(r.attendee_ids) / r.seats
	# 新增
    @api.onchange('seats', 'attendee_ids')
    def _verify_valid_seats(self):
        if self.seats < 0:
            return {
                'warning': {
                    'title': "Incorrect 'seats' value",
                    'message': "The number of available seats may not be negative",
                },
            }
        if self.seats < len(self.attendee_ids):
            return {
                'warning': {
                    'title': "Too many attendees",
                    'message': "Increase seats or remove excess attendees",
                },
            }
    # 结束
```

***
# Model constraints

Odoo提供了两种方式来设置自动验证 invariants: Python constraints and SQL constraints
Python constraints 就像普通method一样，只不过需要使用装饰器 constrains()， 不过也是需要指明 record中的那些字段需要使用 constraint，如果不满足要求，method应该 raise an exception

```python
from openerp.exceptions import ValidationError

@api.constrains('age')
def _check_something(self):
	for record in self:
		if record.age > 20:
			raise ValidationError("your record is too old: %s" % record.age)
		
```
***
练习 4-4
添加一个限制条件，用以明确，instructor not in the attendees in his/her own session
openacademy/models.py

```
# -*- coding: utf-8 -*-

from openerp import models, fields, api, exceptions # 修改

class Course(models.Model):
    _name = 'openacademy.course'
```

```
                    'message': "Increase seats or remove excess attendees",
                },
            }
	# 新增
    @api.constrains('instructor_id', 'attendee_ids')
    def _check_instructor_not_in_attendees(self):
        for r in self:
            if r.instructor_id and r.instructor_id in r.attendee_ids:
                raise exceptions.ValidationError("A session's instructor can't be an attendee")
```

***
SQL constraints 是通过给 model 设置属性 `_sql_constraints`， 这个也和domain一样是由一堆 三个元素组成的 tuple 的 list集合。（name, sql_definition, message），name 就是这个 SQL constraint 的名字，sql_definition 就是 [table_constraint](https://www.postgresql.org/docs/9.3/static/ddl-constraints.html)， message 就是 error message。
***
练习 4-5

- CHECK course 的title 和 description 是否一样
- 明确 course name 是 unique

openacademy/models.py

```
    session_ids = fields.One2many(
        'openacademy.session', 'course_id', string="Sessions")
	# 新增
    _sql_constraints = [
        ('name_description_check',
         'CHECK(name != description)',
         "The title of the course should not be the description"),

        ('name_unique',
         'UNIQUE(name)',
         "The course title must be unique"),
    ]
    # 结束


class Session(models.Model):
    _name = 'openacademy.session'
```

***