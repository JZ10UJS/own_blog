---
title: Odoo Views （四） - Search
date: 2016-11-16 16:51:56
tags:
---

# Search

Search view 与之前的几个都不同，它不是用来显示 record 的。作用与制定的 model，并且用来筛选其他 view的内容。root element 是 `<search>` 不接受属性。子节点有：

- `<field>`
fields 可以定义一些 domain 和 context，当用户输入搜索字符串时，将会把搜索行为和定义的 domain  **AND** 之后，再去查询。`<field>` 可以有下列属性：
 - `name`  指明根据哪个 field 来筛选

 - `string` field 的 label

 - `operator` 
默认情况下，将根据生成 `[(name, operator, provided_value)]`，`name` 就是字段名字，`provided_value` 就是用户在搜索框中填入数据。`operator` 属性设置之后就可以修改默认的 operator（`char field` 是 `ilike`， `float field` 是 `=`）
 
 - `filter_domain`
 一个完整的 domain，用来指明field 的 search domain，可以用 `self` 来表示用户输入的数据。通常是用来一次搜索作用于多个字段上（e.g. `<field name='name' filter_domain="['|', ('name', 'ilike', self), ('customer_id', 'ilike', self)]"`，返回 name 或者 customer_id.name 符合用户输入的 record）
 
 - `context`
 - `groups` 仅对特殊组生效
 - `widget` 使用特定的 widget，odoo8.0中唯一使用的就是 作用于`many2one` field的 `selection` widget

 - `domain`
if the field can provide an auto-completion (e.g. Many2one), filters the possible completion results.

- `<filter>`
这个就是预先定义好的筛选条件，在UI 界面中它只有生效和无效两种状态。可有下列属性
 - `string`  the label of the filter
 - `domain`  an Odoo domain，将会追加到当前 action 的 domain 的后面。
 - `context` python  字典，合并到 action domian中，用来生成 search domain
 - `name` 指明 这个filter的 logic name，设置之后，就可以在 enable it by default。也方便后续继承这个 search view，用以定位。
 - `help`  帮助信息
 - `groups` 对特殊组生效
 
- `<separator>`
这样两个条件就是 **or**
```
<filter domain="[('state', '=', 'draft')]"/>
<filter domain="[('state', '=', 'done')]"/>
```
两个条件是 **and**
```
<filter domain="[('state', '=', 'draft')]"/>
<separator/>
<filter domain="[('delay', '<', 15)]"/>
```

- `<group>` 根据组来显示，更加的清晰。


## Search defaults

可以通过在 action的 context中设置`search_default_name` 来实现自动搜索，for fields，就指明字段的值，如果是 filter，就传入 boolean value。比如：假设 `foo` 是一个 field, `bar` 是个 filter。那么 action 中的 context 就应该为：
```
{
  'search_default_foo': 'acro',
  'search_default_bar': 1
}
```
中当点击这个action是，就会自动的 enable `bar` filter，并且搜索 `foo` field的值 ilike acro 的。