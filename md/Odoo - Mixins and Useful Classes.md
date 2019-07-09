---
title: Odoo - Mixins and Useful Classes
date: 2017-03-31 15:59:12
tags:
---

Odoo 提供了一些 classed 和 mixins，用以帮助你轻松的建立一些常用的功能。

# **Messaging features**

## **Messaging integration**

### **Basic messaging system**
将 messaging feature 整合到你自己的 model 中将是非常的容易。仅需要在你的 model 的定义中添加继承 `mail.thread`，以及在你的 form view 中添加对应的字段和 widgets，就可以了。
```python
class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread']
    _description = 'Business Trip'

    name = fields.Char()
    partner_id = fields.Many2one('res.partner', 'Responsible')
    guest_ids = fields.Many2many('res.partner', 'Participants')
```
form view 中 添加
```
<record id="businness_trip_form" model="ir.ui.view">
    <field name="name">business.trip.form</field>
    <field name="model">business.trip</field>
    <field name="arch" type="xml">
        <form string="Business Trip">
            <!-- Your usual form view goes here
            ...
            Then comes chatter integration -->
            <div class="oe_chatter">
                <field name="message_follower_ids" widget="mail_followers"/>
                <field name="message_ids" widget="mail_thread"/>
            </div>
        </form>
    </field>
</record>
```
一旦你在 model 中添加了 chatter support，用户就可以非常方便的在这个 model 的任意 record 上，添加 message。这些修改或留言，也会自动的发给这个 record 的 followers ，当然前提是，系统的 email 配置都是正确的。


#### **Posting messages**

- `message_post(self,body='',subject=None,message_type='notification',subtype=None,parent_id=False,attachments=None,content_subtype='html',**kwargs)`
提交一个新的 message 到 existing thread，并返回这个 message 的 ID。
|-|-|
|-|-|
|Parameters:|**body(str)** -- message的内容，如果是html将会被转意<br/>**message_type(str)** -- see mail_message.type field<br/>**content_subtype(str)** -- 如果是普通文本数据，convert body into html<br/>**parent_id(int)** -- 用来回复之前的 message，主要是私人对话<br/>**attachments(list(tuple(str,str)))** -- list of tuple, tuple是 (`name`, `content`), content 不是 base64 encoded<br/>**\*\*kwargs** -- extra keyword arguments will be used as default column values for the new mail.message record|
|Returns:| id of newly created mail.message|
|Return Type:| int |

- `message_post_with_view(views_or_xmlid, **kwargs)`
用来 send email 或者 post a message，使用 ir.qweb 引擎来渲染这个 view_id。This method is stand alone, because there is nothing in template and composer that allows to handle views in batch. This method will probably disappear when templates handle ir ui views.
|-|-|
|-|-|
|Parameters:|**ir.ui.view record(str)** -- external id or record of the view that should be sent|

-- `message_post_with_template(template_id, **kwargs)`
通过指定 template 来发送 message。
|-|-|
|-|-|
|Parameters:|**template_id** -- 使用模板的 id<br/>**\*\*kwargs** -- 用来创建 mail.compose.message wizard的参数，继承自 mail.message|

#### **Receiving messages** 
下述方法都是当 mail gateway 收到新的邮件时，才会被调用。这些 email 可以被当前的 thread 自动回复，或者一个新的 thread。重写下面几个方法，将会使得你可以根据 email 中的一些特定值，再返回不同的邮件（i.e. 修改时间，以及根据 email-address，修改 CC address， etc.）。

- `message_new(msg_dict, custom_values=None)`

