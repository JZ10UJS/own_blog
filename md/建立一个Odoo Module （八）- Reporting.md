---
title: 建立一个Odoo Module （八）- Reporting
date: 2016-09-24 14:02:17
tags:
---

# Reporting
## Printed reports
在Odoo 8.0，Odoo 提供了一个全新的，基于 QWeb、Bootstrap、Wkhtmltopdf 的report engine。
一个 report 是由下面两部分组成的：

- `ir.actions.report.xml`，可以用快捷方式`report` 代替，设置一个report所需的基本参数

```
<report
    id="account_invoices"
    model="account.invoice"
    string="Invoices"
    report_type="qweb-pdf"
    name="account.report_invoice"
    file="account.report_invoice"
    attachment_use="True"
    attachment="(object.state in ('open','paid')) and
        ('INV'+(object.number or '').replace('/','')+'.pdf')"
/>
```

- 一个标准的 QWeb view，用以设置report的实际格式

```
<t t-call="report.html_container">
    <t t-foreach="docs" t-as="o">
        <t t-call="report.external_layout">
            <div class="page">
                <h2>Report title</h2>
            </div>
        </t>
    </t>
</t>
the standard rendering context provides a number of elements, the most
important being:

"docs"
    the records for which the report is printed
"user"
    the user printing the report
```


其实reports都是标准的 web pages, 所以，他们可以通过URL来输出结果，比如，Invoice 的report 就可已通过下面两个URL访问

- http://localhost:8069/report/html/account.report_invoice/1
- http://localhost:8069/report/pdf/account.report_invoice/1

分别访问 html 版本 和 pdf 版本的 report

> 注意：
> 如果说你的 pdf 格式的 report 出现没有 css 样式的情况（html 格式的pdf 有样式，但是pdf格式的没有样式，只有数据），多数情况是因为你的wkhtmltopdf 与你的服务器连接出现的错误。
> 如果你通过查询日志，发现当你请求 pdf 格式report时，css 等文件并未GET 成功，那么问题就明确了
> wkhtmltopdf 程序是通过系统参数`web.base.url`作为root path 来连接各文件的，但是这个参数是根据 每次 administrator 登录后，自动改变的。如果说，你的服务器使用的某种反向代理，可能就为导致获取不到css文件的问题，你可以通过设置下面两个参数来解决这个问题
>  - `report.url`, pointing to an URL reachable from your server (probably http://localhost:8069 or something similar). It will be used for this particular purpose only.
>  `web.base.url.freeze`, when set to True, will stop the automatic updates to web.base.url.

*** 
练习 8-1
为session model 创建一个report，report中应该显示每一个session的 名字，开始，结束日期，以及列出参加人
openacademy/\__openerp\__.py

```python
        'views/openacademy.xml',
        'views/partner.xml',
        'views/session_workflow.xml',
        'reports.xml', # new line
    ],
    # only loaded in demonstration mode
    'demo': [
```
openacademy/reports.xml

```
<openerp>
<data>
    <report
        id="report_session"
        model="openacademy.session"
        string="Session Report"
        name="openacademy.report_session_view"
        file="openacademy.report_session"
        report_type="qweb-pdf" />

    <template id="report_session_view">
        <t t-call="report.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="report.external_layout">
                    <div class="page">
                        <h2 t-field="doc.name"/>
                        <p>From <span t-field="doc.start_date"/> to <span t-field="doc.end_date"/></p>
                        <h3>Attendees:</h3>
                        <ul>
                            <t t-foreach="doc.attendee_ids" t-as="attendee">
                                <li><span t-field="attendee.name"/></li>
                            </t>
                        </ul>
                    </div>
                </t>
            </t>
        </t>
    </template>
</data>
</openerp>
```
## Dashboards
***
练习 8-2
创建一个 dashboard，内含一个 graph view ，一个 sessions 的 calendar view 以及一个 courses 的 list view。并为这个 dashboard 创建一个menuitem，并且当用户点击Openacademy时，作为默认展示。

- 创建一个 openacademy/views/session_board.xml 。内含一个 board view，一个action 用以展现这个view，
> 注意：dashboard view 可以分为 1，1-1,1-2,2-1，以及1-1-1

- 修改openacademy/\__openerp\__.py

openacademy/\__openerp\__.py
```python
    'version': '0.1',

    # any module necessary for this one to work correctly
    'depends': ['base', 'board'], # 修改

    # always loaded
    'data': [
```

```python
        'views/openacademy.xml',
        'views/partner.xml',
        'views/session_workflow.xml',
        'views/session_board.xml', # new line
        'reports.xml', 
    ],
    # only loaded in demonstration mode
```
openacademy/views/session_board.xml

```
<?xml version="1.0"?>
<openerp>
    <data>
        <record model="ir.actions.act_window" id="act_session_graph">
            <field name="name">Attendees by course</field>
            <field name="res_model">openacademy.session</field>
            <field name="view_type">form</field>
            <field name="view_mode">graph</field>
            <field name="view_id"
                   ref="openacademy.openacademy_session_graph_view"/>
        </record>
        <record model="ir.actions.act_window" id="act_session_calendar">
            <field name="name">Sessions</field>
            <field name="res_model">openacademy.session</field>
            <field name="view_type">form</field>
            <field name="view_mode">calendar</field>
            <field name="view_id" ref="openacademy.session_calendar_view"/>
        </record>
        <record model="ir.actions.act_window" id="act_course_list">
            <field name="name">Courses</field>
            <field name="res_model">openacademy.course</field>
            <field name="view_type">form</field>
            <field name="view_mode">tree,form</field>
        </record>
        <record model="ir.ui.view" id="board_session_form">
            <field name="name">Session Dashboard Form</field>
            <field name="model">board.board</field>
            <field name="type">form</field>
            <field name="arch" type="xml">
                <form string="Session Dashboard">
                    <board style="2-1">
                        <column>
                            <action
                                string="Attendees by course"
                                name="%(act_session_graph)d"
                                height="150"
                                width="510"/>
                            <action
                                string="Sessions"
                                name="%(act_session_calendar)d"/>
                        </column>
                        <column>
                            <action
                                string="Courses"
                                name="%(act_course_list)d"/>
                        </column>
                    </board>
                </form>
            </field>
        </record>
        <record model="ir.actions.act_window" id="open_board_session">
          <field name="name">Session Dashboard</field>
          <field name="res_model">board.board</field>
          <field name="view_type">form</field>
          <field name="view_mode">form</field>
          <field name="usage">menu</field>
          <field name="view_id" ref="board_session_form"/>
        </record>

        <menuitem
            name="Session Dashboard" parent="base.menu_reporting_dashboard"
            action="open_board_session"
            sequence="1"
            id="menu_board_session" icon="terp-graph"/>
    </data>
</openerp>
```

 
 
 
 
 
  
 
 
 
 
 
 
 


 
 
 