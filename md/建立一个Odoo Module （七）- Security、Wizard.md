---
title: 建立一个Odoo Module （七）- Security、Wizard
date: 2016-09-23 17:35:40
tags:
---

# Security
ERP中，必有一个访问控制机制，来实现安全上的把控
## Group-based access control mechanisms
Groups 就像普通的 record 一样创建，只不过他的 model 是 res.groups。在定义menu时，可以通过设置group实现对menu的访问权限。但是，对象仍然可以不经过menu得到访问，所以，真正的 object 级别的访问控制权限（read，write，create, unlink）必须在 groups 中定义好。这一过程，通常是通过在module中的 CSV 文件来设置的。但是，我们同样可以针对 view 或者 object 中的特定 field设置控制权限，只需要对他们设置 groups 属性。
## Access rights
Access rights 是 model ir.model.access 的 record，每一条 access right 都是关联到特定的 Model和 Group（或者不设定Group，即全局生效），还有4个permission：read，write，create，unlink。access rights通常是通过一个名为 ir.model.access.csv 的 文件建立。

```csv
id,name,model_id/id,group_id/id,perm_read,perm_write,perm_create,perm_unlink
access_idea_idea,idea.idea,model_idea_idea,base.group_user,1,1,1,0
access_idea_vote,idea.vote,model_idea_vote,base.group_user,1,1,1,0
```
***
练习 7-1
通过用户界面，手动设置 access rights
创建一个名字为 “John Smith”User，一个只有 read Model session 权限名为 "OpenAcademy / Session Read"的group。 

- 通过 Settings ‣ Users ‣ Users 创建 John Smith
- 通过 Settings ‣ Users ‣ Groups 创建 session_read，仅给予这个group read session 的权限
- 将 John Smith 加入到 session_read 这个
- 用 John Smith 登录然后验证access right 是否正确

***
练习 7-2
通过编辑data files，添加 access control

- 创建openacademy/security/security.xml 用来创建 OpenAcademy Manager group
- 编辑 openacademy/security/ir.model.access.csv 添加 access right
- 在 openacademy/\__openerp\__.py中添加上面的文件

openacdemy/\__openerp\__.py

```python

    # always loaded
    'data': [
        'security/security.xml', # 添加此行
        'security/ir.model.access.csv',
        'templates.xml',
        'views/openacademy.xml',
        'views/partner.xml',
```
openacademy/security/ir.model.access.csv

```
id,name,model_id/id,group_id/id,perm_read,perm_write,perm_create,perm_unlink
course_manager,course manager,model_openacademy_course,group_manager,1,1,1,1
session_manager,session manager,model_openacademy_session,group_manager,1,1,1,1
course_read_all,course all,model_openacademy_course,,1,0,0,0
session_read_all,session all,model_openacademy_session,,1,0,0,0
```
openacademy/security/security.xml

```
<openerp>
    <data>
        <record id="group_manager" model="res.groups">
            <field name="name">OpenAcademy / Manager</field>
        </record>
    </data>
</openerp>
```

## Record rules

a record rule 可以针对model 下的某些特定 records 设置访问权限。同时，record rule 也是 model `ir.rule` 的 record，他可以关联到特定groups，特定models，设置 read, write, create, unlink 权限，还有设置 domain， 用以设置权限应用到这个 model 下面的哪些 records
例：除了record 的状态在 cancel 的情况下，record是不能被删除的。

```
<record id="delete_cancelled_only" model="ir.rule">
    <field name="name">Only cancelled leads may be deleted</field>
    <field name="model_id" ref="crm.model_crm_lead"/>
    <field name="groups" eval="[(4, ref('base.group_sale_manager'))]"/>
    <field name="perm_read" eval="0"/>
    <field name="perm_write" eval="0"/>
    <field name="perm_create" eval="0"/>
    <field name="perm_unlink" eval="1" />
    <field name="domain_force">[('state','=','cancel')]</field>
</record>
```
***
练习 7-3
给Group “OpenAcademy / Manager”和 “Course”model 设置 record rule，只给予 responsible of a course 的写 和删除的权限，如果一个 course 没有设置 responsible 的话，那么所有人都可以对它进行 修改。
openacademy/security/security.xml

```
        <record id="group_manager" model="res.groups">
            <field name="name">OpenAcademy / Manager</field>
        </record>
	    <!-- 新增 -->
        <record id="only_responsible_can_modify" model="ir.rule">
            <field name="name">Only Responsible can modify Course</field>
            <field name="model_id" ref="model_openacademy_course"/>
            <field name="groups" eval="[(4, ref('openacademy.group_manager'))]"/>
            <field name="perm_read" eval="0"/>
            <field name="perm_write" eval="1"/>
            <field name="perm_create" eval="0"/>
            <field name="perm_unlink" eval="1"/>
            <field name="domain_force">
                ['|', ('responsible_id','=',False),
                      ('responsible_id','=',user.id)]
            </field>
        </record>
    </data>
</openerp>
```

# Wizards

Wizard 可以通过 dynamic form 给 user 提供 交互式的sessions，a warzard 就是一个简单的model，继承自`TransientModel` ，而不是普通的`Model`， class  `TransientModel` 也是继承自 `Model`, 拥有Model的所有特性，但是多出如下几点：

- Wizard record 不是固定的，他们将会在一定时间后，就被从database中删除掉，所以叫做 Transient
- Wizard models 是不需要给它设置权限的，因为任意用户都拥有它的所有权限
- Wizard records 可能会用到 many2one 连接到 普通的 record，但是普通的 record 是**不能**通过 many2one 连接到  wizard records

我们将要创建一个wizard，使得我们可以对 一个或多个 session ，一次性选择好他们的 attendees。
***
练习 7-4
创建一个wizard model，内含 一个关联到 session的many2one field，一个关联到 Partner 的 many2many field
openacademy/\__init\__.py

```python
from . import controllers
from . import models
from . import partner
from . import wizard # new line
```
openacademy/wizard.py

```python
# -*- coding: utf-8 -*-

from openerp import models, fields, api

class Wizard(models.TransientModel):
    _name = 'openacademy.wizard'

    session_id = fields.Many2one('openacademy.session',
        string="Session", required=True)
    attendee_ids = fields.Many2many('res.partner', string="Attendees")
```

## Launching wizards

Wizards 是通过 `ir.actions.act_window` 被调用，在这个action中设置 field `target` 为 new，这样设置之后，通过menu点击触发action，将会打开一个新的窗口。
还有另一种方式开启wizard，在 `ir.actions.act_window` 中添加 field  `src_model`， 指定特定的model才能调用action，The wizard will appear in the contextual actions of the model, above the main view. Because of some internal hooks in the ORM, such an action is declared in XML with the tag act_window.

```
<act_window id="launch_the_wizard"
            name="Launch the Wizard"
            src_model="context.model.name"
            res_model="wizard.model.name"
            view_mode="form"
            target="new"
            key2="client_action_multi"/>
```
Wizard 也拥有普通 views的特性，但是 它的button可以设置 `special="cancel"`，用以关闭窗口，且wizard中执行的其它操作，将不会生效。
***
练习 7-5

- 给 wizard 定义一个 form view
- 添加一个 action 用以调用 wizard，同时将action 的 src_model 设置为 Session model
- 在 wizard 中给 session field 一个默认值，这个可以通过 `self._context` 拿到当前的 

openacademy/wizard.py

```python
class Wizard(models.TransientModel):
	_name = 'openacademy.wizard'

	def _default_session(self):
		return self.env['openacademy.session'].browse(self._context.get('active_id'))

	session_id = fields.Many2one('openacademy.session', string='Session',
		required=True, default=_default_session)
	attendee_ids = fields.Many2many('res.partner', string='Attendees')
```
openacademy/views/openacademy.xml

```
                  parent="openacademy_menu"
                  action="session_list_action"/>
		<!-- 新增 -->
		<record model='ir.ui.view' id='wizard_form_view'>
			<field name='name'>wizard.form</field>
			<field name='model'>openacademy.wizard</field>
			<field name='arch' type='xml'>
				<form string='Add Attendees'>
					<group>
						<field name='session_id'/>
						<field name='attendee_ids'/>
					</group>
				</form>
			</field>
		</record>

		<act_window id='launch_session_wizard'
					name='Add Attendees'
					src_model='openacademy.session'
					res_model='openacademy.wizard'
					view_mode='form'
					target='new'
					key2='client_action_multi'/>
		<!-- 结束 -->
	</data>
</openerp>
```
***
练习 7-6
在wizard 添加一个button， 实现将 session 和 partern 关联起来的功能
openacademy/views/openacademy.xml

```
                        <field name="attendee_ids"/>
                    </group>
                    <!-- 新增 footer -->
                    <footer>
                        <button name="subscribe" type="object"
                                string="Subscribe" class="oe_highlight"/>
                        or
                        <button special="cancel" string="Cancel"/>
                    </footer>
                </form>
            </field>
        </record>
```
openacademy/wizard.py
```python
    session_id = fields.Many2one('openacademy.session',
	    string="Session", required=True, default=_default_session)
    attendee_ids = fields.Many2many('res.partner', string="Attendees")
	
	# 新增
	@api.multi
	def subscribe(self):
		self.session_id.attendee_ids |= self.attendee_ids
		return {}
```
***
练习 7-7
将 attendee_ids 一次赋给 多个session
openacademy/views/openacademy.xml

```
                <form string="Add Attendees">
                    <group>
                        <field name="session_ids"/> <!-- 修改 -->
                        <field name="attendee_ids"/> 
                    </group>
                    <footer>
                        <button name="subscribe" type="object"
```
openacademy/wizard.py

```python
class Wizard(models.TransientModel):
	_name = 'openacademy.wizard'

	def _default_sessions(self):
		return self.env['openacademy.session'].browse(self._context.get('active_ids')) # ids
	# ids
	session_ids = fields.Many2many('openacademy.session', 
	    string="Sessions", required=True, default=_default_sessions)
    attendee_ids = fields.Many2many('res.partner', string="Attendees")

	@api.multi
	def subscribe(self):
		# modified
		for session in self.session_ids:
			session.attendee_ids |= self.attendee_ids
		return {}
```