---
title: Data Files
date: 2016-10-11 13:38:53
tags:
---

Odoo 是一个高度数据驱动的系统，他的 UI （menus and views），security (access right and aceess rule），reports 以及 plain data 都是通过 定义 record 来实现的

# Structure

在 Odoo 中设置一些数据的最主要的方式就是通过设置 XML data files，它的结构如下：

```
<!-- the root elements of the data file -->
<odoo>
  <operation/>
  ...
</odoo>
```
Data files 都是按照顺序读取执行的，所以当某个 operation 引用 另一个 operation 的时候，必需在 它的后面定义。

# Core operations

`record`
--------
`record` 就像它的名字一样，就是用来更新或者新建一条数据的，有下列属性：
	
- `model` （required )，创建或更新的数据是哪一个 Odoo Model 的
- `id` 这个 record 的 external identifier，强烈建议设置
	- 如果是创建， 可以帮助其它想要修改这条record，用来引用它
	- 如果是更新， 就直接通过查询 erxternal identifier 来找到它
- `context` context to use when creating the record
- `forcecreate` 如果没有这个对应的 record id，那么就创建一个新的record，且 external id 为提供的id，默认是 True

`field`
-------
每一个 record 都是有若干个 `field` tag 组成的，定义创建一个 record时，它的各项字段的值。 一个 `record` 如果没有 `field` 字段，那么就使用字段的默认值创建（create )，或者不做任何操作（update ）。
一个 `field` 有一个必需的设置的属性 `name`， 指明是为哪一个字段设置值

- Nothing
如果这个字段没有设置任何值 `<field name='xxx'></field>`，那么将会为这个field的值设置为False，可以用来清空某个字段的值，或者新建 record 时，避免使用这个field的默认值。

- `search`
中在 relational field 中使用，应该设置为一个 domain，通过执行domain，得到返回的值，并将值赋给这个 field。如果这个字段是 Many2one, 那么就使用查询后返回的 第一个值 作为这个field的值。

- `ref`
如果设置 `ref` 属性，那么 ref 的值就必须是一个 external identifier，那个对应 id 的record 就会被设置为本 record的此field的值，通常是在 Many2one field 中使用。
- `type`
是用来解释和转换的 field's content。可以设置为：
 - `xml`, `html`
将 field tag 的子节点，当成一个document，可以通过 `%(external_id)s` 在 form tag 中转换任意的 external id。`%%` 可以表示一个 百分号符号。
 - `file`
确保 field tag 的 content 是一个有效的 file path，这个字段的值为被设置成 `module,path`。
 - `char`
将 field tag 的content直接设为这个field的值，不经过任何转换
 - `base64`
base64-encodes the field's content, useful combined with the file attribute to load e.g. image data into attachments
 - `int` content 转成 int 后， 赋值
 - `float` content 转成 float 后， 赋值
 - `list`, `tuple` 
 should contain any number of value elements with the same properties as field, each element resolves to an item of a generated tuple or list, and the generated collection is set as the field's value
- `eval`
为如果上面的几种属性不符合你的特殊情况。可以使用这个，它基本和 python eval 一样，将任意表达式执行后的值赋值给 这个 field。
这个 eval 还带有一些context， （`time`, `datetime`, `timedelta`, `relativedelta`）。以及一个用来转换 external id的函数 `ref`，以及当前field 的model obj `obj`。
`delete`
--------
可以删除之前定义过的，任意数目的 records，有如下属性：

- `model` (required) 指定要删除的record 是哪一个 model 下的。
- `id` 删除指明 external id
- `search` 删除符合 search domain 的。 records

`id` 和 `search` 不能同时设置。

`function`
------
将会调用一个model的 method，有两个属性必需设置，`name` 和 `model`，分别指明 method的name 和 这个 method 是哪一个 model 下的。method 的参数可以通过 设置 eval 属性来传递，但是eval的结果必需是一个 sequence，或者 通过 `value` tag 来实现。
`workflow`
------
`workflow` tag 用以传递一个信号到已经存在的 workflow 中
The workflow tag sends a signal to an existing workflow. The workflow can be specified via a ref attribute (the external id of an existing workflow) or a value tag returning the id of a workflow.

The tag also has two mandatory attributes model (the model linked to the workflow) and action (the name of the signal to send to the workflow).


# Shortcuts

由于Odoo的一些model是非常复杂的，所以提供了另一种相比通过record更加简单的方式来定义他们。

`minuteman`
---
通过一系列的默认和预设的操作，来定一个`ir.ui.menu` record。

- Parent menu
 - 如果设置了 `parent` 属性，那么它的值应该是另一个 menu item 的 external id
 - 如果没有设置 `parent` ，将会尝试从 `name` 属性中 按照 `/` 来分隔，找到对应的层级来创建，如果找不到，就依次创建对应的 menu item。
 - 如果`name` 中没有 `/` ，那么就把这个menu 作为一个 `top-level` 的menu 。
- Menu name
如果没有设置 `name` 属性，那么就从这个 menu 对应的 `action` 的 name，如果没有 action，就使用 `id` 。
- Groups
`groups` 的值是res.groups model 下的group 对应的 external id 由 逗号分割的 string。如果某个 external id 前面有`-` ，那么就是把它从 menu groups 中删除。
`<menuitem groups='sales.group_manager,-sales.group_see_own_leads'`。
- `action`
就是点击这个menuitem 时触发 action 的对应的 external id
- `id`
这个 menuitem 的 external id。

`template`
---
Creates a QWeb view requiring only the arch section of the view, and allowing a few optional attributes:

- `id`  view 的 external id
- `name`, `inherit_id`, `priority`
和普通的 `ir.ui.view` 是一样的效果
- `primary`， 如果设置为 True， 并且有 `inherit_id`, 使得这个view 为 primary
- `groups`， 逗号分割的 external id
- `page`，如果设置为 True，使得这个 template 是一个 website page，（可链接，可删除）
- `optional`,
值为 `enabled` or `disabled`，使得这个view 是否可以在 website 界面被禁用。如果没有设置，那么它的值就是 `enabled`

`report`
---

用一些默认值来创建一个`ir.actions.report.xml`
Mostly just proxies attributes to the corresponding fields on ir.actions.report.xml, but also automatically creates the item in the More menu of the report's model.


# CSV data files

XML data files 是灵活，但是当你想要为某个 model 建立一大堆数据的时候，就显得特别的繁琐冗长。
为了解决这种情况，data files 也可以用 CSV 的方式定义，它通常用来设置 access right.

- 文件的名字是 `model_name.csv`
- 第一行是要设置数据的field name，但是第一列必需是 `id`， 用来确定是 update 还是 create
- 之后的每一行都代表一个 record

如果要创建 state 可以这样创建一个文件 名字为 `res.country.state.csv`
```
"id","country_id:id","name","code"
state_au_1,au,"Australian Capital Territory","ACT"
state_au_2,au,"New South Wales","NSW"
state_au_3,au,"Northern Territory","NT"
state_au_4,au,"Queensland","QLD"
state_au_5,au,"South Australia","SA"
state_au_6,au,"Tasmania","TAS"
state_au_7,au,"Victoria","VIC"
state_au_8,au,"Western Australia","WA"
state_us_1,us,"Alabama","AL"
state_us_2,us,"Alaska","AK"
state_us_3,us,"Arizona","AZ"
state_us_4,us,"Arkansas","AR"
state_us_5,us,"California","CA"
state_us_6,us,"Colorado","CO"
```
每一行（record ) 

- 第一列，external id, 用以明确是 create 还是 update
- 第二列，country object 的 external id。 注意 country 必需先定义
- 第三列，`name` field for `res.country.state`
- 第四列，`code` field for `res.country.state`


