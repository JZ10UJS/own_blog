---
title: 建立一个Odoo Module （六）- Workflows
date: 2016-09-23 15:12:42
tags:
---

# Workflows
Workflows 是通过model 来描述业务逻辑之间的变化过程，Workflows are also used to track processes that evolve over time.
***
练习 6-1
给session 添加一个 `state` field ，用来帮助弄 workflow
一个 session 有3个状态，分别是 Draft（默认），Confirmed， Done
在session form中，添加一个 read-only field 用来显示 state，添加一个button 用来调整 state。

- Draft -> Confirmed
- Confirmed -> Draft
- Confirmed -> Done
- Done -> Draft

openacademy/models.py

```python
    attendees_count = fields.Integer(
        string="Attendees count", compute='_get_attendees_count', store=True)
	# 新增
    state = fields.Selection([
        ('draft', "Draft"),
        ('confirmed', "Confirmed"),
        ('done', "Done"),
    ], default='draft')

    @api.multi
    def action_draft(self):
        self.state = 'draft'

    @api.multi
    def action_confirm(self):
        self.state = 'confirmed'

    @api.multi
    def action_done(self):
        self.state = 'done'
	# 结束
    @api.depends('seats', 'attendee_ids')
    def _taken_seats(self):
        for r in self:
```
openacademy/views/openacademy.xml

```
            <field name="model">openacademy.session</field>
            <field name="arch" type="xml">
                <form string="Session Form">
	                <!-- 新增 -->
                    <header>
                        <button name="action_draft" type="object"
                                string="Reset to draft"
                                states="confirmed,done"/>
                        <button name="action_confirm" type="object"
                                string="Confirm" states="draft"
                                class="oe_highlight"/>
                        <button name="action_done" type="object"
                                string="Mark as done" states="confirmed"
                                class="oe_highlight"/>
                        <field name="state" widget="statusbar"/>
                    </header>
                    <!-- 结束 -->

                    <sheet>
                        <group>
                            <group string="General">
```
***

workflows 可以与任何 Odoo object 相关联起来，而且完全的可定制化。Workflows 是用来的管理将商业闭环，还可以通过可视化的方式，修改业务之间的转化关系和各种触发条件的钩子。Workflows, activities (nodes or actions) and transitions (conditions) are declared as XML records, as usual. The tokens that navigate in workflows are called workitems.
> 注意：
> workflow 与 session 关联之后，只有新建的 session 才有 workflow instance，之前的 session 都不会有 workflow instance

***
练习 6-2
替换上面的 伪 workflow， 用真正的 workflow 来重写，所以 button 点击后，调用的就不是 object的method，而是 workflow的 trigger。
openacademy/\__openerp\__.py

```python
        'templates.xml',
        'views/openacademy.xml',
        'views/partner.xml',
        'views/session_workflow.xml', # new line
    ],
    # only loaded in demonstration mode
    'demo': [
```
openacademy/models.py

```python
        ('draft', "Draft"),
        ('confirmed', "Confirmed"),
        ('done', "Done"),
    ])  # 去掉了 default

    @api.multi
    def action_draft(self):
```
openacademy/views/openacademy.xml

```
            <field name="arch" type="xml">
                <form string="Session Form">
                    <header>
	                    <!-- 3个 button 的 type 由 object 变为 workflow -->
                        <button name="draft" type="workflow" 
                                string="Reset to draft"
                                states="confirmed,done"/>
                        <button name="confirm" type="workflow" 
                                string="Confirm" states="draft"
                                class="oe_highlight"/>
                        <button name="done" type="workflow" 
                                string="Mark as done" states="confirmed"
                                class="oe_highlight"/>
                        <field name="state" widget="statusbar"/>
```
openacademy/views/session_workflow.xml

```
<openerp>
    <data>
        <record model="workflow" id="wkf_session">
            <field name="name">OpenAcademy sessions workflow</field>
            <field name="osv">openacademy.session</field>
            <field name="on_create">True</field>
        </record>

        <record model="workflow.activity" id="draft">
            <field name="name">Draft</field>
            <field name="wkf_id" ref="wkf_session"/>
            <field name="flow_start" eval="True"/>
            <field name="kind">function</field>
            <field name="action">action_draft()</field>
        </record>
        <record model="workflow.activity" id="confirmed">
            <field name="name">Confirmed</field>
            <field name="wkf_id" ref="wkf_session"/>
            <field name="kind">function</field>
            <field name="action">action_confirm()</field>
        </record>
        <record model="workflow.activity" id="done">
            <field name="name">Done</field>
            <field name="wkf_id" ref="wkf_session"/>
            <field name="kind">function</field>
            <field name="action">action_done()</field>
        </record>

        <record model="workflow.transition" id="session_draft_to_confirmed">
            <field name="act_from" ref="draft"/>
            <field name="act_to" ref="confirmed"/>
            <field name="signal">confirm</field>
        </record>
        <record model="workflow.transition" id="session_confirmed_to_draft">
            <field name="act_from" ref="confirmed"/>
            <field name="act_to" ref="draft"/>
            <field name="signal">draft</field>
        </record>
        <record model="workflow.transition" id="session_done_to_draft">
            <field name="act_from" ref="done"/>
            <field name="act_to" ref="draft"/>
            <field name="signal">draft</field>
        </record>
        <record model="workflow.transition" id="session_confirmed_to_done">
            <field name="act_from" ref="confirmed"/>
            <field name="act_to" ref="done"/>
            <field name="signal">done</field>
        </record>
    </data>
</openerp>
```
> 为了检测，是否建立了 workflow ，可以到 **Settings ‣ Technical ‣ Workflows ‣ Instances** 查看

***
练习 6-3
添加一个根据条件自动触发的 workflow

openacademy/views/session_workflow.xml
```
            <field name="act_to" ref="done"/>
            <field name="signal">done</field>
        </record>
		<!-- 添加 -->
        <record model="workflow.transition" id="session_auto_confirm_half_filled">
            <field name="act_from" ref="draft"/>
            <field name="act_to" ref="confirmed"/>
            <field name="condition">taken_seats &gt; 50</field>
        </record>
    </data>
</openerp>
```
***
练习 6-4
用 server action 代替 python methods
workflow 和 server action 都可以从用户界面，手动建立。

openacademy/views/session_workflow.xml
```
            <field name="on_create">True</field>
        </record>
		<!-- 新增 -->
        <record model="ir.actions.server" id="set_session_to_draft">
            <field name="name">Set session to Draft</field>
            <field name="model_id" ref="model_openacademy_session"/>
            <field name="code">
model.search([('id', 'in', context['active_ids'])]).action_draft()
            </field>
        </record>
        <!-- 结束 -->
        <record model="workflow.activity" id="draft">
            <field name="name">Draft</field>
            <field name="wkf_id" ref="wkf_session"/>
            <field name="flow_start" eval="True"/>
            <!-- 修改 -->
            <field name="kind">dummy</field>
            <field name="action"></field>
            <field name="action_id" ref="set_session_to_draft"/>
            <!-- 结束 -->
        </record>
		<!-- 新增 -->
        <record model="ir.actions.server" id="set_session_to_confirmed">
            <field name="name">Set session to Confirmed</field>
            <field name="model_id" ref="model_openacademy_session"/>
            <field name="code">
model.search([('id', 'in', context['active_ids'])]).action_confirm()
            </field>
        </record>
        <!-- 结束 -->
        <record model="workflow.activity" id="confirmed">
            <field name="name">Confirmed</field>
            <field name="wkf_id" ref="wkf_session"/>
            <!-- 修改 -->
            <field name="kind">dummy</field>
            <field name="action"></field>
            <field name="action_id" ref="set_session_to_confirmed"/>
            <!-- 结束 -->
        </record>
		<!-- 新增 -->
        <record model="ir.actions.server" id="set_session_to_done">
            <field name="name">Set session to Done</field>
            <field name="model_id" ref="model_openacademy_session"/>
            <field name="code">
model.search([('id', 'in', context['active_ids'])]).action_done()
            </field>
        </record>
         <!-- 结束 -->
        <record model="workflow.activity" id="done">
            <field name="name">Done</field>
            <field name="wkf_id" ref="wkf_session"/>
            <!-- 修改 -->
            <field name="kind">dummy</field>
            <field name="action"></field>
            <field name="action_id" ref="set_session_to_done"/>
            <!-- 结束 -->
        </record>

        <record model="workflow.transition" id="session_draft_to_confirmed">
```