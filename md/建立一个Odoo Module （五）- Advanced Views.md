---
title: 建立一个Odoo Module （五）- Advanced Views
date: 2016-09-22 18:02:13
tags:
---

# Advanced Views
## Tree views
Tree views 可以通过增加 attribute 来提供更深度的定制化

- decoration-{$name}
允许针对不同的record的特定值，来修改record 对应 **行** 的样式
值为普通的python 表达式，此表达式将会依次作用于每一个record，record 的各项属性，将会作为 context 传入此表达式，如果为表达式值为 True，对应行的样式就会修改。除了 record 的各项属性，Odoo也会自动传入，`uid` 和 `current_date`（as string yyyy-MM-dd 传入）
`{$name}` 可以是 `bf`( font-weight: bold ) `it` (font-style: italic)，也可以是Bootstrap 的一些样式（`danger`, `info`, `muted`, `primary`, `success`, `warning`）

```
<tree string="Idea Categories" decoration-info="state=='draft'"
    decoration-danger="state=='trashed'">
    <field name="name"/>
    <field name="state"/>
</tree>
```

- editable
值为 `"top"` 或者 `"bottom"`，可以让 Tree view 就地进入编辑状态，而不是点击进入 Form view 去修改，这两个就是 新的record 出现的位置。

***
练习 5-1
将 session tree 添加一定的色彩，当session duration < 5是，这一行 显示为 blue, duration >15 , 将其显示为 red
openacademy/views/openacademy.xml

```
            <field name="name">session.tree</field>
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <tree string="Session Tree" decoration-info="duration&lt;5" decoration-danger="duration&gt;15"> <!-- 修改 -->
                    <field name="name"/>
                    <field name="course_id"/>
                    <field name="duration" invisible="1"/> <!-- 修改 -->
                    <field name="taken_seats" widget="progressbar"/>
                </tree>
            </field>
```

***

## Calendars
将 record 以 calendar events 的方式展现，常用的 attributes 有：

- `color`
The name of the field used for color segmentation. Colors are automatically distributed to events, but events in the same color segment (records which have the same value for their @color field) will be given the same color.

- `date_start`
record's field holding the start date/time for the event

- `date_stop` （可选）
record's field holding the end date/time for the event

field (to define the label for each calendar event)

```
<calendar string="Ideas" date_start="invent_date" color="inventor_id">
    <field name="name"/>
</calendar>
```
***
练习 5-2
给session 添加一个 calendar view，

- 添加一个 computed field `end_date`，根据 `start_date`  和 `duration` 计算而来
- 添加 calendar view

openacademy/models.py

```python
# -*- coding: utf-8 -*-

from datetime import timedelta # new line
from openerp import models, fields, api, exceptions

class Course(models.Model):
```

```python
    attendee_ids = fields.Many2many('res.partner', string="Attendees")

    taken_seats = fields.Float(string="Taken seats", compute='_taken_seats')
    end_date = fields.Date(string="End Date", store=True,
        compute='_get_end_date', inverse='_set_end_date') # new field

    @api.depends('seats', 'attendee_ids')
    def _taken_seats(self):
```

```python
                },
            }
	# new add
    @api.depends('start_date', 'duration')
    def _get_end_date(self):
        for r in self:
            if not (r.start_date and r.duration):
                r.end_date = r.start_date
                continue

            # Add duration to start_date, but: Monday + 5 days = Saturday, so
            # subtract one second to get on Friday instead
            start = fields.Datetime.from_string(r.start_date)
            duration = timedelta(days=r.duration, seconds=-1)
            r.end_date = start + duration

    def _set_end_date(self):
        for r in self:
            if not (r.start_date and r.end_date):
                continue

            # Compute the difference between dates, but: Friday - Monday = 4 days,
            # so add one day to get 5 days instead
            start_date = fields.Datetime.from_string(r.start_date)
            end_date = fields.Datetime.from_string(r.end_date)
            r.duration = (end_date - start_date).days + 1
	# end
    @api.constrains('instructor_id', 'attendee_ids')
    def _check_instructor_not_in_attendees(self):
        for r in self:
```
openacademy/views/openacademy.xml

```
            </field>
        </record>

        <!-- calendar view 新增-->
        <record model="ir.ui.view" id="session_calendar_view">
            <field name="name">session.calendar</field>
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <calendar string="Session Calendar" date_start="start_date"
                          date_stop="end_date"
                          color="instructor_id">
                    <field name="name"/>
                </calendar>
            </field>
        </record>

        <record model="ir.actions.act_window" id="session_list_action">
            <field name="name">Sessions</field>
            <field name="res_model">openacademy.session</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form,calendar</field> <!-- 修改 -->
        </record>

        <menuitem id="session_menu" name="Sessions"
```

***

## Search views
search view 中的 field 可以有一个 `filter_domain` 属性，可以覆盖掉 Odoo自动为这个field设置的 search domain。 在 filter_domain 中 self 代表的就是 用户在搜索框中输入的 数据。
search view 中也可以有 `<filter>` 元素， 用以预先定义一些快捷搜索，在其中必须有以下属性中的一个

- `domain` 就给 search 提供筛选条件
- `context` 给当前 search 提供 context  ，通常使用 `group_by` ，来分组搜素结果。

```
<search string="Ideas">
    <field name="name"/>
    <field name="description" string="Name and description"
           filter_domain="['|', ('name', 'ilike', self), ('description', 'ilike', self)]"/>
    <field name="inventor_id"/>
    <field name="country_id" widget="selection"/>

    <filter name="my_ideas" string="My Ideas"
            domain="[('inventor_id', '=', uid)]"/>
    <group string="Group By">
        <filter name="group_by_inventor" string="Inventor"
                context="{'group_by': 'inventor_id'}"/>
    </group>
</search>
```
如果想要在 action 中不使用 默认的 search view， 可以明确指定 `search_view_id` 的值
action 中同样可以设定 search view 的初始行为，只需要在 action 的 `context` 中传入一定的参数即可，如： `search_default_field_name`， 
***
练习 5-3

- 添加一个button，使其可以将 当前登录用户的 course 显示出来，再将其设置为默认。
- 添加一个button，使course 可以根据用户 分组显示

openacademy/views/openacademy.xml

```
                <search>
                    <field name="name"/>
                    <field name="description"/>
                    <!-- 添加 -->
                    <filter name="my_courses" string="My Courses"
                            domain="[('responsible_id', '=', uid)]"/>
                    <group string="Group By">
                        <filter name="by_responsible" string="Responsible"
                                context="{'group_by': 'responsible_id'}"/>
                    </group>
	                <!-- 结束 -->
                </search>
            </field>
        </record>
```

```
            <field name="res_model">openacademy.course</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form</field>
            <field name="context" eval="{'search_default_my_courses': 1}"/> <!-- 添加 -->
            <field name="help" type="html">
                <p class="oe_view_nocontent_create">Create the first course
                </p>
```

## Gantt
由于 Odoo community 版本不支持 gantt， 所以我就不翻译了。
https://github.com/odoo/odoo/issues/9640

## Graph views
Graph view 提供了一种全局的概览 以及 数据的分析，root element 是 `<graph>`
它有4种展现方式，修改默认的展现方式可以通过设置 `type` 属性

- Bar （默认）
在 bar chart 中，第一个是用来横向分组的，剩余的都是在 第一个分组的情况下，展现各自的数据。
默认的 bar 是 side-by-side, 但可以在 `<graph stacked='True'>` 来修改。

- Line 2D 线 
- Pie  2D 饼图

Graph view 里面的 `<field>` 都会被强制赋予 type 属性，两种选择：

- `row` （默认） the field should be aggregated by default
- `measure`  the field should be aggregated rather than grouped on

```
<graph string="Total idea score by Inventor">
    <field name="inventor_id"/>
    <field name="score" type="measure"/>
</graph>
```
> 注意： graph 不支持 computed field， 除非 此 computed field 设置了 store=True 属性

***
练习 5-4
在 给 session 添加一个graph view，在每个 course 下，显示 attendees number

- 添加 attendees_count which  computed field but stored
- add 相关view

openacademy/models.py

```python
    hours = fields.Float(string="Duration in hours",
                         compute='_get_hours', inverse='_set_hours')
	# 新增
    attendees_count = fields.Integer(
        string="Attendees count", compute='_get_attendees_count', store=True)

    @api.depends('seats', 'attendee_ids')
    def _taken_seats(self):
        for r in self:
```

```python
        for r in self:
            r.duration = r.hours / 24
	# 新增
    @api.depends('attendee_ids')
    def _get_attendees_count(self):
        for r in self:
            r.attendees_count = len(r.attendee_ids)

    @api.constrains('instructor_id', 'attendee_ids')
    def _check_instructor_not_in_attendees(self):
        for r in self:
```
openacademy/views/openacademy.xml

```
            </field>
        </record>
		# 新增
        <record model="ir.ui.view" id="openacademy_session_graph_view">
            <field name="name">openacademy.session.graph</field>
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <graph string="Participations by Courses">
                    <field name="course_id"/>
                    <field name="attendees_count" type="measure"/>
                </graph>
            </field>
        </record>

        <record model="ir.actions.act_window" id="session_list_action">
            <field name="name">Sessions</field>
            <field name="res_model">openacademy.session</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form,calendar,gantt,graph</field> # 修改
        </record>

        <menuitem id="session_menu" name="Sessions"
```
***

## Kanban
通常用来显示，任务进度，生产流程等，root element是 `<kanban>`
kanban view 将一对的 cards 已 coloumn 的形式展现，每一 card 代表一个 record ，每一个 column 都代表 按照 某个 field 的分类。
比如， project 可能按照 stage 状态被分类（column 就是 stage），或者被 负责人responsible （column 就是 user）， 等。
kanban view 为每一个 card 定义了类似 form view的 效果，当然也支持原生HTML 和 QWeb
***
练习 5-5

- 添加一个 integer `color` 字段到 Session model
- 添加kanban view, update action

openacademy/models.py

```python
    duration = fields.Float(digits=(6, 2), help="Duration in days")
    seats = fields.Integer(string="Number of seats")
    active = fields.Boolean(default=True)
    color = fields.Integer() # new line

    instructor_id = fields.Many2one('res.partner', string="Instructor",
        domain=['|', ('instructor', '=', True),
```
openacademy/views/openacademy.xml

```
        </record>
		<!-- 新增 -->
        <record model="ir.ui.view" id="view_openacad_session_kanban">
            <field name="name">openacad.session.kanban</field>
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <kanban default_group_by="course_id">
                    <field name="color"/>
                    <templates>
                        <t t-name="kanban-box">
                            <div
                                    t-attf-class="oe_kanban_color_{{kanban_getcolor(record.color.raw_value)}}
                                                  oe_kanban_global_click_edit oe_semantic_html_override
                                                  oe_kanban_card {{record.group_fancy==1 ? 'oe_kanban_card_fancy' : ''}}">
                                <div class="oe_dropdown_kanban">
                                    <!-- dropdown menu -->
                                    <div class="oe_dropdown_toggle">
                                        <i class="fa fa-bars fa-lg"/>
                                        <ul class="oe_dropdown_menu">
                                            <li>
                                                <a type="delete">Delete</a>
                                            </li>
                                            <li>
                                                <ul class="oe_kanban_colorpicker"
                                                    data-field="color"/>
                                            </li>
                                        </ul>
                                    </div>
                                    <div class="oe_clear"></div>
                                </div>
                                <div t-attf-class="oe_kanban_content">
                                    <!-- title -->
                                    Session name:
                                    <field name="name"/>
                                    <br/>
                                    Start date:
                                    <field name="start_date"/>
                                    <br/>
                                    duration:
                                    <field name="duration"/>
                                </div>
                            </div>
                        </t>
                    </templates>
                </kanban>
            </field>
        </record>

        <record model="ir.actions.act_window" id="session_list_action">
            <field name="name">Sessions</field>
            <field name="res_model">openacademy.session</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form,calendar,gantt,graph,kanban</field> <!-- 修改 -->
        </record>

        <menuitem id="session_menu" name="Sessions"
                  parent="openacademy_menu"
```

***