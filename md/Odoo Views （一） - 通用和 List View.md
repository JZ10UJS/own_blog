---
title: Odoo Views （一） - 通用和 List View
date: 2016-11-02 15:48:07
tags:
---

# Common Structure

View objects 提供了一系列的字段接口，除非特殊指明，否则这些字段都是可选的。
`name` （必选）
用以描述这个View，通常是为了在一堆list中找到它，主要是为了可读性。

`model`  它指明这个view相关的model

`priority` 
客户端可以通过指明 `id` 或 `(model, type)` 的形式来调用 view。 如果是后面这种情况，所以对应 model 的 type 正确的 view 都会被搜索出来，并返回这些 view 中 priority 最低的哪一个 （it is the 'default view'）。`priority` 也在 view inheritance 时，设置了 order

`arch`  the description of the view's layout

`groups_id` 指明允许哪些组来访问或使用当前 view

`inherit_id` 指明当前 view 的 parent view

`mode` 
如果这个 view 没有设置 inherit_id 字段，那么 `mode` 的值就只能是 `primary`，如果设置了 inherit_id 字段，那么默认的，`mode` 的值被设置为 `extension`，但是，也可以强制设置其值为 `primary`。

`application`
website feature defining togglable views. By default, views are always applied

# Inheritance

## View matching

- 如果某个 view 被 `(model, type)` 的方式被请求调用，将会返回对应 model 的对应 type 的 view, 这个 view 必须是`mode=primary` 并且 lowest priority 。
- 当某个 view 被 `id` 指明调用，如果这个 view 的 mode 不是 primary，将返回它最近的 parent view 且 parent view 的 `mode=primary`

## View resolution

通过对 matched `primary` view 的解析，生成最后的 `arch`

1. 首先，如果这个 view 有 parnet，parent view 将会被完整的解析，并且当前 view 对 parent view 修改的部分也将被处理。
2. 其次，如果这个 view 没有 parent， 那么这个 view 中设置的 arch 就是 arch
3. 最后，如果这个 view 有 children，那么所有的 children 都将被找出来，并按照 depth-firth 的顺序依次处理他们对 parent view 继承部分的修改。

通过这三个步骤，生成最终的arch 

## Inheritance specs

三种继承写法：

- 设置一个包含 `expr` 属性的 `xpath` tag。`expr` 就是普通的 `XPath` 表达式，作用于`arch`，返回第一个符合要求的节点。
- 设置一个包含 `name` 属性的 `field` tag。返回 `name` 相同的 `field`
- 其他的 tag， 但是第一个属性是 `name`，且其他属性完全一样（除了 `position` 属性）

可以通过添加 `postition` 属性，来实现对 matched node的修改：

`inside` （default） 作为 matched node 的最后一个孩子节点插入

`replace` 替换掉整个 matched node

`after` 在 matched node 后面添加，作为其兄弟节点

`before` 插在 matched node 之前

`attributes` 
是如果是这种情况，那么这个content 应该为一系列 `<attribute>` 且有 `name` 属性，以及这个 `<attribute>` 可选的 body。

- 如果 `<attribute>` 有 body，那么 matched node 就会被设置一个属性为 attribute 中的 name的值，这个属性为 attribute的 body
- 如果 `<attribute>` 没有body， 那么 matched node 对应名字为 `name的值` 的属性，将会被删除。如果 matched node 没有 `name 的值` 的属性，就将会抛出异常

view spec 将会被顺序的执行。

# Lists

list views 的跟节点就是 `<tree>`，list views 的跟节点可以设置以下属性：

`editable`
没默认情况下，当你点击 list view 中的某一行时，将会打开这一行对应 record 的 form view 中。而通过设置 `editable` 属性，将使得 list view 可以被就地更改，而不用去对应的form view去修改。
这个属性的值可以被设置为 `top` 或者 `bottom`，这将是新增的 record 出现的位置。
inline form view 是从 list view 派生出来的，所以 form view 中的 fields 和 button的大部分属性都可以被 list view所接收，但是如果list view不是 editable的话，这些属性也通常不会起作用。

`default_order`
复盖这个 view 对应model 的 order。 值为 逗号分割的字段名的组合，如果在后面添加 `desc` 表示 reverse

```xml
<tree default_order="sequence,name,id desc">
```

`colors` - 9.0 时，被废弃，改为 `decoration-{$name}`

`fonts`  - 9.0 被废弃， 改为 `decoration-{$name}`

`decoration-{$name}`
允许根据 record 的某些属性，来修改这行 record 的 text style
这个属性的值为一个 python 表达式，他将依次作用与当前显示的每一个 record，用每一个 record的属性来作为这个表达式的context，来运算。如果这个表达式为 true，那么对应的样式就将会作用与这一行上。同时，这个 context 除了 record的属性，还由Odoo 提供`uid` （当前登录用户的 id）和 `current_date` （当前日期被 strftime `yyyy-MM-dd` 的字符串）
`{$name}` 可以是 `bf` (`font-weight: bold`), `it`( `font-style: italic`), 或者任意 bootstrap contextual color (`danger`,`info`,`muted`,`primary`,`success`,`warning`).

`create`, `edit`, `delete`
通过设置这些属性为 `false`，可以使的对应的操作被隐藏。

`on_write`
仅仅在 editable list 中生效。值应该为这个 list 对应的 model 的某个 method的 name。在创建或者修改某个 record之后，这个method 将会被调用，同时这个record 的 `id` 也将会被传入这个 method中。
这个 method 应该返回 a list of ids of other records to load or update。

`string`  可选，用来翻译。8.0 废弃

**这个 root tag 的 children 通常有：**

**`button`** 在 list cell 中显示一个 button

- `icon`  用来显示这个 button

- `string`
 - 如果没有设置 `icon`, 那么这个就是 button 的 text
 - 如果设置了 `icon`，那么这个就是 button 的 `alt` text

- `type`  button 的类型，用以表明当点击这个button时，odoo采取何种行为
 - `workflow`  （default）
传递一个型号给 workflow，button 的 `name` 就是 workflow的 signal。同时这一行对应的 record 将会被作为一个参数传递给 workflow
 - `object`
调用这个list对应 model 的某个方法，button 的 `name` 就是方法的名字，传入当前 record id 和 context
 - `action`
调用 `ir.actions` 的某个action。button 的 `name` 就是这个 action 的 database id。同时 context 中将会包含这个 list 对应的 model （`active_model`)，当前行对应的 record（`active_id`），已经当前页面中所有 loaded 的 record（`active_ids`， 可能是经过筛选之后的 sub recordset）。
- `name`  see `type`

- `args`  see `type`

- `attrs` 基于当前 record value 的动态属性
一系列 attribute 和 domain 组成的 dict。domain 在当前 context 环境下作用于对应 record，如果 true,  cell 就被设置这个 attribute。
通常 attribute 是 `invisible` 和 `readonly`。

- `states`
就是 `attrs` 中对 `invisible` 的快捷方式。用逗号分割的state 选项 组成的字符串。这个list 对应的 model 中必需设置了 `state` 字段。而且 `state` 字段在 这个 view 中设置了。如果当前 state 不在列出的里面，就隐藏这个 button

- `context` 
点击button，将会 merge 到这个 view 的 context 中。然后传入后方

- `confirm` 
调用某个方法之前，弹出一个对话框，用以用户确认。

**`field`**
指明哪些 field 将会被 column 的形式被展现出来。属性有：

- `name` 这个model 的字段的名字。在同一个field name 在同一个 view 中只能出现一次。

- `string` 这个 field 的标题， 没设置的话，就用 model 对应字段的 string

- `invisible`  获取这个字段的值，但是不显示出来。通常是因为在某些其他字段中需要使用到这个字段，比如说在 `<tree>` 的 `decoration-{$name}` 的表达式 中。

- `groups` 设置哪些组可以看到这个字段

- `widget`  改变某些字段的显示方式
 - `progressbar`  将 `float` field 用 progressbar 的型式展现
 - `many2onebutton`  将一个 `many2one` field 的值替换成某个符号，如果这个field 有值就是某个icon，
 - `handle`, 就是作用于 `sequence` 这个字段，将这个字段的值从数字变成一个表示可以拖曳的 icon

- `sum`, `avg` 
在底部显示相关字段的和或者平均值。通常要通过 `group_operator` 来显示。
 ```xml
<field name="planned_revenue" sum="Expected Revenues"/>
<field name="probability" avg="Avg. of Probability"/>
```

> Note:
> 如果 list view 是 `editable` 的，那么 form view 中的任意 field attribute 都可以作用于 inline form view 上。




