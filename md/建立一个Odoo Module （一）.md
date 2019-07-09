---
title: 建立一个Odoo Module （一）
date: 2016-09-21 16:22:29
tags:
---

# 开启/关闭 Odoo 服务
Odoo 使用的是CS架构，这里的C指的是浏览器，客户端和服务器端之间通过RPC通信。
业务逻辑和扩展都是在服务器端执行，但是也支持在客户端执行一些操作（比如说：一些数据的展示，还有交互式的地图）。
开启服务，只需要在shell中简单的输入命令 odoo.py 。
```
$ python odoo.py
```
停止服务，连续按压 Ctrl-c 两次即可， 或者直接kill 这个进程。
# 创建一个 Odoo Module
服务端和客户端的扩展都是被打包成 modules，which 被有选择的存在数据库中。
Odoo modules 可以直接新增一个业务逻辑到Odoo System中，也可以修改或者扩展一个已有的业务逻辑：e.g. 一个 module 可以被创建，用来将你自己国家的会计准则添加到 Odoo 通用的会计模块（Account）中；或者为了数据可视化，直接新建一个 Odoo 通用模块中没有的东西。
Odoo 的所有东西都是从 module 开始和结束。Everything in Odoo thus starts and ends with modules.
## 一个 Module 的组成元素
一个 Odoo module 可能包含如下元素：
**Business objects**
像普通 Python class 一样的定义， 这些将被 Odoo 根据配置文件自动的保存
**Data files**
XML 或者 CSV 文件， 定义一些 metadata（views 或者 workflows），或者一些配置文件（modules parameterization）,  或者一些测试数据，等等。
**Web controllers**
处理来自浏览器的请求
**Static web data**
图像，CSS，js文件等

## Module 的构造
每一个 Odoo module 其实都是一个文件夹，他们都存放在一个 **Modules文件夹** 下面。 **Modules文件夹** 可以在命令行中通过参数 `--addons-path` 指定。
> Tip
> 
> 大多数的 命令行参数 都可以在配置文件中直接设置

Odoo module 都是通过一个 manifest 文件来指明一些东西的，这个文件就是这个module内的 **\__openerp\__.py** 文件
Odoo module 和一个普通 Python package 一样，都内含一个 \__init\__.py 文件， 该文件中包含一些基本的 python import 。
比如， 如果一个 Odoo module 只有一个 mymodule.py 文件，那么 \__init\__.py中可能就是这样：

```python
from . import mymodule
```
Odoo 也提供了一个机制用来帮助用户快速建立一个 empty module:

```
python odoo.py scaffold <module name> <where to put it>
```
这个命令将在你指定的目录下创建一系列的符合标准的目录和文件。这些文件中包含一些通用的 code 或者XML，将通过下面一系列教程解释这些文件的用途。
 
练习 1-1

Module 的创建
用下面命令创建一个 empty module Open Academy, and install it in Odoo.
```
$ python odoo.py scaffold openacademy addons
```
编辑 the manifest file to your module，其它文件不急。
openacademy/\__openerp\__.py
```python
# -*- coding: utf-8 -*-
{
    'name': "Open Academy",

    'summary': """Manage trainings""",

    'description': """
        Open Academy module for managing trainings:
            - training courses
            - training sessions
            - attendees registration
    """,

    'author': "My Company",
    'website': "http://www.yourcompany.com",

    # Categories can be used to filter modules in modules listing
    # Check https://github.com/odoo/odoo/blob/master/openerp/addons/base/module/module_data.xml
    # for the full list
    'category': 'Test',
    'version': '0.1',

    # 这个模块的依赖模块，如果没有，就填写base
    'depends': ['base'],

    # 总是会自动导入
    'data': [
        # 'security/ir.model.access.csv',
        'templates.xml',
    ],
    # 只有建立数据库时，勾选了导入测试数据，这些文件才会导入
    'demo': [
        'demo.xml',
    ],
}
```

openacademy/\__init\__.py
```
# -*- coding: utf-8 -*-
from . import controllers
from . import models
```

openacademy/controllers.py
```python
# -*- coding: utf-8 -*-
from openerp import http

# class Openacademy(http.Controller):
#     @http.route('/openacademy/openacademy/', auth='public')
#     def index(self, **kw):
#         return "Hello, world"

#     @http.route('/openacademy/openacademy/objects/', auth='public')
#     def list(self, **kw):
#         return http.request.render('openacademy.listing', {
#             'root': '/openacademy/openacademy',
#             'objects': http.request.env['openacademy.openacademy'].search([]),
#         })

#     @http.route('/openacademy/openacademy/objects/<model("openacademy.openacademy"):obj>/', auth='public')
#     def object(self, obj, **kw):
#         return http.request.render('openacademy.object', {
#             'object': obj
#         })
```

openacademy/demo.xml

```
<openerp>
    <data>
        <!--  -->
        <!--   <record id="object0" model="openacademy.openacademy"> -->
        <!--     <field name="name">Object 0</field> -->
        <!--   </record> -->
        <!--  -->
        <!--   <record id="object1" model="openacademy.openacademy"> -->
        <!--     <field name="name">Object 1</field> -->
        <!--   </record> -->
        <!--  -->
        <!--   <record id="object2" model="openacademy.openacademy"> -->
        <!--     <field name="name">Object 2</field> -->
        <!--   </record> -->
        <!--  -->
        <!--   <record id="object3" model="openacademy.openacademy"> -->
        <!--     <field name="name">Object 3</field> -->
        <!--   </record> -->
        <!--  -->
        <!--   <record id="object4" model="openacademy.openacademy"> -->
        <!--     <field name="name">Object 4</field> -->
        <!--   </record> -->
        <!--  -->
    </data>
</openerp>
```

openacademy/models.py
```
# -*- coding: utf-8 -*-

from openerp import models, fields, api

# class openacademy(models.Model):
#     _name = 'openacademy.openacademy'

#     name = fields.Char()
```
openacademy/security/ir.model.access.csv

```
id,name,model_id/id,group_id/id,perm_read,perm_write,perm_create,perm_unlink
access_openacademy_openacademy,openacademy.openacademy,model_openacademy_openacademy,,1,0,0,0
```

openacademy/templates.xml

```
<openerp>
    <data>
        <!-- <template id="listing"> -->
        <!--   <ul> -->
        <!--     <li t-foreach="objects" t-as="object"> -->
        <!--       <a t-attf-href="{{ root }}/objects/{{ object.id }}"> -->
        <!--         <t t-esc="object.display_name"/> -->
        <!--       </a> -->
        <!--     </li> -->
        <!--   </ul> -->
        <!-- </template> -->
        <!-- <template id="object"> -->
        <!--   <h1><t t-esc="object.display_name"/></h1> -->
        <!--   <dl> -->
        <!--     <t t-foreach="object._fields" t-as="field"> -->
        <!--       <dt><t t-esc="field"/></dt> -->
        <!--       <dd><t t-esc="object[field]"/></dd> -->
        <!--     </t> -->
        <!--   </dl> -->
        <!-- </template> -->
    </data>
</openerp>
```

## Object-Relational Mapping
Odoo 的一个关键构件就是它的 ORM 了， 这个大量避免手写 SQL 语句，而且更加安全和易扩展。

业务对象 就像普通python class 一样的定义，但是需要继承 Model which integrates them into the automated persistence system.

Models 可以在定义时设置一系列属性，但是最重要的就是 \_name，这个属性是**必须**要设置的。通过此属性定义了 Odoo中的 model，这里是一个最简单的model 定义：

```
from openerp import models

class MinimalModel(models.Model):
	_name = 'test.model'    # 这个属性不能少
```
## Model fields
Fields 是用来指明这个 model 存放在哪，存什么东西。 Fields 的定义方式就像 python类属性一样：

```
from openerp import models, fields

class LessMinimalModel(models.Model):
	_name = 'test.model2'
	
	name = fields.Char()
```
###  Common Attributes 通用字段属性
和model 一样，field 也可以被配置不同的属性，这个属性通过传递参数来设置：

```
name = fields.Char(requried=True)
```
某些属性是通用的， 下面介绍一些常使用的：

- `string` (unicode，default：field's name)
The label of the field in UI 

- `required` （bool，default：False）
如果设置为 True，那么这个字段不能为空！在新建一个record时，要么每次都传入值，要么给该字段设置一个 default value

- `help` （unicode，default：""）
给 User 提供一个help，在 UI 界面

- `index`（bool， default：False）
让 Odoo 在数据库中，为此字段建立 database index
### Simple fields 
这里对 fields 进行了分类，大致两种：
“simple”fields 就是直接存储数据
 “relational”fields 就是用来连接不同的model 或者 连接相同的model。

simple fields 大概就是 Boolean， Date，Char

### Reserved fields 保留字段
Odoo 会对每一个model自动的建立一些字段，这些字段不要手动的传入Value，系统会自动write，但是如果有必要的话，可以 read , **一定不要 write**。
- `id`（Id）
		the unique identifier for a record in its model
		
- `create_date`（Datetime） record 的建立时间
		
- `create_uid` （Many2one）建立 record 的用户

- `write_date` （Datetime） record 的最近修改时间

- `write_uid` （Many2one） 最后一个修改 record 的用户	

### Special fields
一般情况下，我们需要为每一个 model 设置一个 `name` 字段，这个字段将用在各种display 和 搜索行为上。当然这个 `name` 也可以通过设置 \_rec\_name 来修改为其它的字段。
 
练习 1-2

Define a model
在 openacademy module 中定义一个名为 Course 的 data model。包含 title 和 description 字段，其中title 为必填。

openacademy/models.py

```
from openerp import models, fields

class Course(models.Model):
	_name = 'openacademy.course'

	name = fields.Char(string="Title", required=True)
	description = fields.Text()
```
## Data files
尽管Odoo的各种行为是通过Python code来实现，但是这些都是启动时，通过加载数据文件，然后通过python code 来定制Odoo的各种行为，所以，Odoo 是一个高数据驱动型系统。甚至，有些module存在的意义就是为了给Odoo添加数据。

Module data 是通过 data files 来设置的，包含`<record>` 的XML 文件。每一个 `<record>` 都在数据库中建立一个新的 database record 或者 更新 database record。

```
<openerp>
    <data>
        <record model="{model name}" id="{record identifier}">
            <field name="{a field name}">{a value}</field>
        </record>
    </data>
</openerp>
```
- `model` 是这个这个record 想要修改的 Odoo model 的名字， 就是 `\_name` 字段设置的值 

- `id` 就是 external identifier ，有了这个就可以直接refering to 这个record ，不需要知道这个record在数据表中的明确 id

- `<field>` 元素有个 `name` 属性，这个就是 model 中 field 的 名字（e.g. `description`）， 中间就是 field's value

Data files 必须在 manifest 文件中指明，以便在启动Odoo服务时，自动导入该文件。这个可以放在 `'data'` 列表中（always loaded）或者 `'demo'` 列表中（only loaded in demonstration mode）。

练习 1-3

设置测试数据

openacademy/demo.xml

```
<openerp>
    <data>
        <record model="openacademy.course" id="course0">
            <field name="name">Course 0</field>
            <field name="description">Course 0's description

Can have multiple lines
            </field>
        </record>
        <record model="openacademy.course" id="course1">
            <field name="name">Course 1</field>
            <!-- no description for this one -->
        </record>
        <record model="openacademy.course" id="course2">
            <field name="name">Course 2</field>
            <field name="description">Course 2's description</field>
        </record>
    </data>
</openerp>
```

## Actions and Menus
Actions and menus 是数据库中最常见的 records，通常是通过data files 来设置。Actions 有三种方式可以被触发：

 1. 通过点击 menu items （这些 menu 被 linked 到特定 action）
 2. 通过点击 button （如果这个 button 被 连接到 actions）
 3. 作为一个普通的 object 被传入到 context 中

由于通过 ir.ui.menus 定义 menus 非常的繁琐，所以提供了一种快捷方式 `<menuitem>`。

```
<record model="ir.actions.act_window" id="action_list_ideas">
    <field name="name">Ideas</field>
    <field name="res_model">idea.idea</field>
    <field name="view_mode">tree,form</field>
</record>
<menuitem id="menu_ideas" parent="menu_root" name="Ideas" sequence="10"
          action="action_list_ideas"/>
```
**注意：**
**action 必须定义在menuitem 之前**
**data files 是从上到下，有序读取执行，所以action 的id 必须在menuitem 使用之前，先定义好。**

练习 1-4

建立菜单入口，是用户通过点击可以

 - 将所有courses 用list展示
 - 可以新建/修改 courses 信息

新建 openacademy/views/openacademy.xml， 内含 actions and menus
将其添加到 manifest 的 data list 中

openacademy/\__openerp\__.py

```python
'data': [
        # 'security/ir.model.access.csv',
        'templates.xml',
        'views/openacademy.xml',   # 新增此行
],
# only loaded in demonstration mode
'demo': [
```
openacademy/views/openacademy.xml

```
<openerp>
    <data>
        <!-- window action -->
        <!--
            The following tag is an action definition for a "window action",
            that is an action opening a view or a set of views
        -->
        <record model="ir.actions.act_window" id="course_list_action">
            <field name="name">Courses</field>
            <field name="res_model">openacademy.course</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form</field>
            <field name="help" type="html">
                <p class="oe_view_nocontent_create">Create the first course
                </p>
            </field>
        </record>

        <!-- top level menu: no parent -->
        <menuitem id="main_openacademy_menu" name="Open Academy"/>
        <!-- A first level in the left side menu is needed
             before using action= attribute -->
        <menuitem id="openacademy_menu" name="Open Academy"
                  parent="main_openacademy_menu"/>
        <!-- the following menuitem should appear *after*
             its parent openacademy_menu and *after* its
             action course_list_action -->
        <menuitem id="courses_menu" name="Courses" parent="openacademy_menu"
                  action="course_list_action"/>
        <!-- Full id location:
             action="openacademy.course_list_action"
             It is not required when it is the same module -->
    </data>
</openerp>

```