---
title: 建立一个Odoo Module （三）- Inheritance
date: 2016-09-22 10:20:42
tags:
---

# inheritance 继承
## Model inheritance 
Odoo 提供了两种继承机制（in a module way），用以继承已有的model。
第一种机制允许一个 module 去修改定义在另一个 module 的 model：

 - 添加一个新的 fields 到该 model
 - 复写 fields 的定义
 - 给这个 model 添加限制条件
 - 添加一个新的 method 
 - 复写 model 已有的 method
 
第二种继承机制（delegation）The second inheritance mechanism (delegation) allows to link every record of a model to a record in a parent model, and provides transparent access to the fields of the parent record.
![这里写图片描述](http://img.blog.csdn.net/20160921190503103)

## View inheritance
为了避免直接修改views（修改源文件），Odoo 提供了 view 的继承机制， where children "extension" views are applied on top of root views, and can add or remove content from their parent.
一个扩展的view，需要使用`inherit_id` 来指明其 parent 的id，并且，在它的 `arch` field 中，使用 `xpath` 元素来选择和修改parent view。

```
<!-- improved idea categories list -->
<record id="idea_category_list2" model="ir.ui.view">
    <field name="name">id.category.list2</field>
    <field name="model">idea.category</field>
    <field name="inherit_id" ref="id_category_list"/>
    <field name="arch" type="xml">
        <!-- find field description and add the field
             idea_ids after it -->
        <xpath expr="//field[@name='description']" position="after">
          <field name="idea_ids" string="Number of ideas"/>
        </xpath>
    </field>
</record>
```

 - `expr` 
一个 XPath 选择表达式， 如果在parent view中**找不到符合**表达式要求的元素，或者**多个符合**要求的元素， 将直接报错！
 - `position`
 如果找到符合要求的 element，那么将在 element 的什么位置，对其进行修改
   - `inside` 将 `<xpath>` body 的内容，append到符合 expr 的 element 的里面
   - `replace` 用 `<xpath>` body 的内容 replace 符合 expr 的 element 
   - `before`  用 `<xpath>` body 的内容 insert到 符合 expr 的 element 的前面， 作为 element的 sibling
   - `after` 用 `<xpath>` body 的内容 insert到 符合 expr 的 element 的后面， 作为 element的 sibling
   - `attributes` 用 `<attribute>` 来 修改 element 的 属性
```
<!-- 这两种方法是同样的效果， 当且仅当 parent view 只有一个满足要求的field -->
<xpath expr="//field[@name='description']" position="after">
    <field name="idea_ids" />
</xpath>

<field name="description" position="after">
    <field name="idea_ids" />
</field>
```

***
练习 3-1

- 通过使用 Model inheritance， 修改已经存在的 Partner model，给其添加一个 boolean field `instructor`  和一个 many2many 的字段，用以表示 Partner 与 Session 之间的关系
- 使用 View inheritance，将这两个字段展现在 Partner 的 Form view 中

> 提示：
> 这个需要开启 debug 模式，用以找寻 partner form view 的 external id。

openacademy/\__init\__.py

```python
# -*- coding: utf-8 -*-
from . import controllers
from . import models
# 新增
from . import partner
# 结束
```
openacademy/\__openerp\__.py

```python
        # 'security/ir.model.access.csv',
        'templates.xml',
        'views/openacademy.xml',
        # 新增
        'views/partner.xml',
        # 结束
    ],
    # only loaded in demonstration mode
    'demo': [
```
openacademy/partner.py

```python
# -*- coding: utf-8 -*-
from openerp import fields, models

class Partner(models.Model):
    _inherit = 'res.partner'

    # Add a new column to the res.partner model, by default partners are not
    # instructors
    instructor = fields.Boolean("Instructor", default=False)

    session_ids = fields.Many2many('openacademy.session',
        string="Attended Sessions", readonly=True)
```
openacademy/views/partner.xml

```
<?xml version="1.0" encoding="UTF-8"?>
 <openerp>
    <data>
        <!-- Add instructor field to existing view -->
        <record model="ir.ui.view" id="partner_instructor_form_view">
            <field name="name">partner.instructor</field>
            <field name="model">res.partner</field>
            <field name="inherit_id" ref="base.view_partner_form"/>
            <field name="arch" type="xml">
                <notebook position="inside">
                    <page string="Sessions">
                        <group>
                            <field name="instructor"/>
                            <field name="session_ids"/>
                        </group>
                    </page>
                </notebook>
            </field>
        </record>

        <record model="ir.actions.act_window" id="contact_list_action">
            <field name="name">Contacts</field>
            <field name="res_model">res.partner</field>
            <field name="view_mode">tree,form</field>
        </record>
        <menuitem id="configuration_menu" name="Configuration"
                  parent="main_openacademy_menu"/>
        <menuitem id="contact_menu" name="Contacts"
                  parent="configuration_menu"
                  action="contact_list_action"/>
    </data>
</openerp>
```

***
### Domains

在 Odoo 中，Domains 就是给 record 添加各种限制条件的。 一个 domain 就是由一些针对 model 的所有record 进行筛选的 各种条件的 集合。每一个条件都是由 3个元素组成，（field_name, operator, value）
比如：我们要针对 Product model ，筛选出 类型为 服务（service）, 且 unit price  大于 1000
```python
[('product_type', '=', 'service'), ('unit_price', '>', 1000)]
```
上面条件被默认的添加了一个 **隐式的 AND**。通过 `&`（AND），`|`（OR），`!`（NOT）可以将复杂的条件组合起来，但是这些逻辑运算符 是在各种条件的**前面**，而不是在中间。比如：我们寻找 要么类型为 service 的product， 要么 unit_price **不在** 1000 到 2000之间的 product。

```python
['|',
	('product_type', '=', 'service'）,
	'!', '&',
			('unit_price', '>=', 1000),
			('unit_price', '<=', 2000)]
```
domain 参数 可以被添加到 relational fields 中，用来在选择时，只提供给你符合要求的record
***
练习 3-2
当为session 选择 instructor 时，只有 instructor == True 的partner 才被显示出来
openacademy/models.py

```python
    duration = fields.Float(digits=(6, 2), help="Duration in days")
    seats = fields.Integer(string="Number of seats")
	# 新增
    instructor_id = fields.Many2one('res.partner', string="Instructor",
        domain=[('instructor', '=', True)])
    # 结束
    course_id = fields.Many2one('openacademy.course',
        ondelete='cascade', string="Course", required=True)
    attendee_ids = fields.Many2many('res.partner', string="Attendees")
```
> 注意：
> 如果 domain 被定义成 list 的样式（明确的python list 类型），将在服务器端被解释运行， 那么 条件中的 right value 必须是定死的，不能是动态的value。
> 如果 domain 被定义成 str(list)，将在客户端解释运行，那么条件中的 right value 可以是 field 的 names 
> 其实就是 .py 文件中的domain ， 和 xml file 中的domain 有细微差别

练习 3-3
创建两个partner的category，分别为：Teacher/Level 1 、Teacher/Level 2。修改session中的instrutor，使其可以为 partner 中 instructor ==True的，或者 category 为 Teacher的
openacademy/models.py

```python
    seats = fields.Integer(string="Number of seats")

    instructor_id = fields.Many2one('res.partner', string="Instructor",
	    # 修改
        domain=['|', ('instructor', '=', True),
                     ('category_id.name', 'ilike', "Teacher")])
        # 结束
    course_id = fields.Many2one('openacademy.course',
        ondelete='cascade', string="Course", required=True)
    attendee_ids = fields.Many2many('res.partner', string="Attendees")
```
openacademy/views/partner.xml

```
        <menuitem id="contact_menu" name="Contacts"
                  parent="configuration_menu"
                  action="contact_list_action"/>
                  
		<!-- 新增 -->
        <record model="ir.actions.act_window" id="contact_cat_list_action">
            <field name="name">Contact Tags</field>
            <field name="res_model">res.partner.category</field>
            <field name="view_mode">tree,form</field>
        </record>
        <menuitem id="contact_cat_menu" name="Contact Tags"
                  parent="configuration_menu"
                  action="contact_cat_list_action"/>

        <record model="res.partner.category" id="teacher1">
            <field name="name">Teacher / Level 1</field>
        </record>
        <record model="res.partner.category" id="teacher2">
            <field name="name">Teacher / Level 2</field>
        </record>
        <!-- 结束 -->
    </data>
</openerp>
```