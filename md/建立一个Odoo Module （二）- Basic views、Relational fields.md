---
title: 建立一个Odoo Module （二）- Basic views、Relational fields
date: 2016-09-21 18:35:08
tags:
---

# Basic Views
Views 定义了 model 中的 record 的展现方式，每种类型的 view 都代提供了 model 的一种数据可视化（list 展现， 图表的方式展现，等）， views 可以通过他们的 type （e.g. a list of partners）或者明确指定它的 external id 来被请求。对于一般的请求来说，the view with 正确的 type 和 最低优先级的 将被调用（所以， the lowest-priority view of each type 就是 默认view of each type）
view 的继承机制，使得我们可以在任意地方对该view进行添加或者删除其中的元素。
## Generic view declaration
A View 就被声明为一个record，不过这个record的 model 是 **ir.ui.view**。 view type 将在 record 的 **arch** field 中设置

```
<record model="ir.ui.view" id="view_id">
    <field name="name">view.name</field>
    <field name="model">object_name</field>
    <field name="priority" eval="16"/>
    <field name="arch" type="xml">
        <!-- view content: <form>, <tree>, <graph>, ... -->
    </field>
</record>
```
## Tree views
Tree views 通常也被称为 list views，以表格的方式展现所有 records
root element 是 `<tree>`，最普遍的就是将model的所有字段展现出来，每个 field 就是一个 column

```
<tree string="Idea list">
    <field name="name"/>
    <field name="inventor_id"/>
</tree>
```

## Form Views
Forms 是用来新建和修改 单一 record 
root element 是 `<form>` ，它集成了 高度结构化的 elements (groups, notebook) 还有交互式的 element（buttons and fields）：

```
<form string="Idea form">
    <group colspan="4">
        <group colspan="2" col="2">
            <separator string="General stuff" colspan="2"/>
            <field name="name"/>
            <field name="inventor_id"/>
        </group>

        <group colspan="2" col="2">
            <separator string="Dates" colspan="2"/>
            <field name="active"/>
            <field name="invent_date" readonly="1"/>
        </group>

        <notebook colspan="4">
            <page string="Description">
                <field name="description" nolabel="1"/>
            </page>
        </notebook>

        <field name="state"/>
    </group>
</form>
```
<hr/>
练习 2-1

通过XML 为Course object 设置 from view

openacademy/views/openacademy.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<openerp>
    <data>
	    <!-- 新增 -->
        <record model="ir.ui.view" id="course_form_view">
            <field name="name">course.form</field>
            <field name="model">openacademy.course</field>
            <field name="arch" type="xml">
                <form string="Course Form">
                    <sheet>
                        <group>
                            <field name="name"/>
                        </group>
                        <notebook>
                            <page string="Description">
                                <field name="description"/>
                            </page>
                            <page string="About">
                                This is an example of notebooks
                            </page>
                        </notebook>
                    </sheet>
                </form>
            </field>
        </record>
		<!-- 结束 -->
        <!-- window action -->
        <!--
            The following tag is an action definition for a "window action",
```
<hr/>
Form views 也可以使用普通的HTML

```
<form string="Idea Form">
    <header>
        <button string="Confirm" type="object" name="action_confirm"
                states="draft" class="oe_highlight" />
        <button string="Mark as done" type="object" name="action_done"
                states="confirmed" class="oe_highlight"/>
        <button string="Reset to draft" type="object" name="action_draft"
                states="confirmed,done" />
        <field name="state" widget="statusbar"/>
    </header>
    <sheet>
        <div class="oe_title">
            <label for="name" class="oe_edit_only" string="Idea Name" />
            <h1><field name="name" /></h1>
        </div>
        <separator string="General" colspan="2" />
        <group colspan="2" col="2">
            <field name="description" placeholder="Idea description..." />
        </group>
    </sheet>
</form>
```
## Search views
Search views 定制了 search field 用以在 list view 或者 其它aggregated views 中进行筛选。root element 就是 `<search>` ，在这其中就包含了，哪些 field 是可以被 search 的

```
<search>
    <field name="name"/>
    <field name="inventor_id"/>
</search>
```
如果没有设置 search view，那么Odoo将自动生成一个 search view，但是只包含model中 **name** 字段。这也是为什么上一章讲到 **name** 是 特殊字段。
<hr/>
练习 2-2
为course 添加 search view，可以通过 title 或者 description 搜索到

openacademy/views/openacademy.xml
```
            </field>
        </record>
		<!-- 新增 -->
        <record model="ir.ui.view" id="course_search_view">
            <field name="name">course.search</field>
            <field name="model">openacademy.course</field>
            <field name="arch" type="xml">
                <search>
                    <field name="name"/>
                    <field name="description"/>
                </search>
            </field>
        </record>
        <!-- 结束 -->

        <!-- window action -->
        <!--
            The following tag is an action definition for a "window action",
```
<hr/>
# Relations between models
一个 model 的 record 可能会关联到另一个 model 的 record。比如，一个销售订单 record 会包含客户 record，在客户record中含有客户的相关数据；而且，销售订单record 里面通常还有 sale order line records.
<hr/>
练习 2-3
创建一个 会议 model , session model
在 module Open Acadmy中，我们希望的 session model 是 在 course 的基础上，在某时间段，有一定的参与人数的会议。
对 session 创建一个model， 包括 name, start date, duration and a number of seats。再添加 action 和 menu 用以展现 sessions，

 1. 创建 class Session in openacademy/models.py
 2. 添加 access to the session object in openacademy/view/openacademy.xml

openacademy/models.py

```python

    name = fields.Char(string="Title", required=True)
    description = fields.Text()


class Session(models.Model):
    _name = 'openacademy.session'

    name = fields.Char(required=True)
    start_date = fields.Date()
    # 6代表浮点数的total number， 2代表有两位小数
    duration = fields.Float(digits=(6, 2), help="Duration in days")
    seats = fields.Integer(string="Number of seats")
```

openacademy/views/openacademy.xml

```
        <!-- Full id location:
             action="openacademy.course_list_action"
             It is not required when it is the same module -->
             
		<!-- 新增 -->
        <!-- session form view -->
        <record model="ir.ui.view" id="session_form_view">
            <field name="name">session.form</field>
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <form string="Session Form">
                    <sheet>
                        <group>
                            <field name="name"/>
                            <field name="start_date"/>
                            <field name="duration"/>
                            <field name="seats"/>
                        </group>
                    </sheet>
                </form>
            </field>
        </record>

        <record model="ir.actions.act_window" id="session_list_action">
            <field name="name">Sessions</field>
            <field name="res_model">openacademy.session</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form</field>
        </record>

        <menuitem id="session_menu" name="Sessions"
                  parent="openacademy_menu"
                  action="session_list_action"/>
        <!-- 结束 -->
    </data>
</openerp>
```
<hr/>
## Relational fields
Relational fields 将同一 model 的records 连接起来（通常表示层级的关系），或者不同model的 records连接起来。
field types 有：

 - Many2one（other_model, ondelete='set null'）

使用方式为

```python
print foo.other_id.name
```

 - One2many（other_model, related_field）
 
一种虚拟的关系，和Many2one相反，一个 One2many 通常包含多个records，所以需要使用loop语句
```python
for other in foo.other_ids:
	print other.name
```
**由于 One2many 是虚拟关系，所以必须有一个 other_model，而且 other_model 中必须有一个 Many2one field，且 field name 必须为 related_field **
 
- Many2many（other_model）

双向多对多的关系，任意方的 record 都可以link 到 对面的 多个 records

```
for other in foo.other_ids:
	print other.name
```
<hr/>
练习 2-4
使用 many2one，使 model Course and Session 相互关联起来

 - A course 应该有一个负责人 responsible user， 这个field 应该关联到一个 Odoo 的内建 model : res.users
 - A session 有一个教员 instructor， 关联到內建model : res.partner
 - A session 关联到 a course ，就是 model: openacademy.course， 并且设置 required=True
 - 修改 Views

openacademy/models.py

```python
    name = fields.Char(string="Title", required=True)
    description = fields.Text()
	
	# 新增
    responsible_id = fields.Many2one('res.users',
        ondelete='set null', string="Responsible", index=True)
    # 结束

class Session(models.Model):
    _name = 'openacademy.session'
```
```python
	start_date = fields.Date()
    duration = fields.Float(digits=(6, 2), help="Duration in days")
    seats = fields.Integer(string="Number of seats")
    
	# 新增
    instructor_id = fields.Many2one('res.partner', string="Instructor")
    course_id = fields.Many2one('openacademy.course',
        ondelete='cascade', string="Course", required=True)
    # 结束
```
openacademy/views/openacademy.xml

```
                    <sheet>
                        <group>
                            <field name="name"/>
                            <!-- 新增 -->
                            <field name="responsible_id"/>
                            <!-- 结束 -->
                        </group>
                        <notebook>
                            <page string="Description">
```

```
            </field>
        </record>
		<!-- 新增 -->
        <!-- override the automatically generated list view for courses -->
        <record model="ir.ui.view" id="course_tree_view">
            <field name="name">course.tree</field>
            <field name="model">openacademy.course</field>
            <field name="arch" type="xml">
                <tree string="Course Tree">
                    <field name="name"/>
                    <field name="responsible_id"/>
                </tree>
            </field>
        </record>
        <!-- 结束 -->

        <!-- window action -->
        <!--
            The following tag is an action definition for a "window action",
```

```
                <form string="Session Form">
                    <sheet>
                        <group>
	                        <!-- 新增 -->
                            <group string="General">
                                <field name="course_id"/>
                                <field name="name"/>
                                <field name="instructor_id"/>
                            </group>
                            <group string="Schedule">
                                <field name="start_date"/>
                                <field name="duration"/>
                                <field name="seats"/>
                            </group>
                            <!-- 结束 -->
                        </group>
                    </sheet>
                </form>
            </field>
        </record>
		<!-- 新增 -->
        <!-- session tree/list view -->
        <record model="ir.ui.view" id="session_tree_view">
            <field name="name">session.tree</field>
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <tree string="Session Tree">
                    <field name="name"/>
                    <field name="course_id"/>
                </tree>
            </field>
        </record>
        <!-- 结束 -->

        <record model="ir.actions.act_window" id="session_list_action">
            <field name="name">Sessions</field>
            <field name="res_model">openacademy.session</field>
```

练习 2-5
通过使用 one2many field, 修改模型以反映course和session之间的关系。

openacademy/models.py

```python
    responsible_id = fields.Many2one('res.users',
	    # 新增
        ondelete='set null', string="Responsible", index=True)
    session_ids = fields.One2many(
        'openacademy.session', 'course_id', string="Sessions")
        # 结束


class Session(models.Model):
```
openacademy/views/openacademy.xml

```
                            <page string="Description">
                                <field name="description"/>
                            </page>
                            <!-- 新增 -->
                            <page string="Sessions">
                                <field name="session_ids">
                                    <tree string="Registered sessions">
                                        <field name="name"/>
                                        <field name="instructor_id"/>
                                    </tree>
                                </field>
                            </page>
                            <!-- 结束 -->
                        </notebook>
                    </sheet>
```

练习 2-6
通过使用many2many field， 修改 session model 使其 每个session 都与一定数目的 attendees，attendees 来自 內建model  res.parter

openacademy/models.py
```python
	instructor_id = fields.Many2one('res.partner', string="Instructor")
    course_id = fields.Many2one('openacademy.course',
        ondelete='cascade', string="Course", required=True)
    # 新增    
    attendee_ids = fields.Many2many('res.partner', string="Attendees")
    # 结束
```
openacademy/views/openacademy.xml
```
                                <field name="seats"/>
                            </group>
                        </group>
                        <!-- 新增 -->
                        <label for="attendee_ids"/>
                        <field name="attendee_ids"/>
                        <!-- 结束 -->
                    </sheet>
                </form>
            </field>
```

