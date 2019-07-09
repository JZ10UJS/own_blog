---
title: Odoo Views （三） - Graphs Kanban
date: 2016-11-04 13:17:18
tags:
---

# Graphs

Graphs view 就是用来将一些 record 来数据可视化。root element is `<graph>`。可以对它设置如下属性：

- `type`   值可以是， `bar` (default), `pie`, `line`

- `statced` 仅仅在 `bar` 的情况下生效，如果设置为 `True`， 将会按照组分割，然后叠在一起。

`<grahp>` 下的孩子节点，只能是 `<field>` ，这个`<field>` 可以有下面几个属性：

- `name` （required） 指出需要在 group中使用哪些字段，通常是用来指明按照这个字段分组。

- `type` 指明这个字段是用来分组的，还是用来计算合计值的。有以下值：
 - `row` （default），按照这个字段来进行分组，
 - `col` 只在 pivot view 中生效，用来创建 column-wise groups
 - `measure` 用这个字段的值来进行统计。

- `interval` 作用在 date 和 datetime field上，按照指明的方式（`day`, `week`, `month`, `quater`, or `year`）来替换 按照默认的 按天分组，和按秒分组

> 注意： Graph view 的统计是根据数据库中的值来计算的，如果某个字段没有存在数据库中，那么不能在graph view中生效

## Pivots

用来显示数据透视表， 根节点是 `<pivot>` ，可以对它设置如下属性：

- `disable_linking` 设置为 `True`，可以取消默认的点击数据透视表单元格，跳转到对应的 list view 中

- `display_quantity` 设置为 `True` 后，将会默认显示数量

其他的都和 Graph View 一样

# Kanban

kanban view 是一中数据可视化的 kanban 面板：它将普通的 record 用小卡片的方式展现出来。一半像 list view，一半像不可编辑的 form view。这里面的 records 可能会是按照某种状态来分组显示，用以直观的查看 workflow的流程。也可以不分组，就普通的显示出来。
根节点是 `<kanban>` ，可以对其设置如下属性：

- `default_group_by` 指明当某个action或者search没有明确制定按照哪个字段分组时，就按照这里设定的值来分组显示，值应该为某个字段的`"name"`.

- `default_order` 指明 cards 的排序方式

- `class` 添加 HTML classes 到这个根节点中

- `quick_create` 用来指明，是否可以在kanban view中直接创建一个新的 record，而不用跳转到 form view去创建。默认情况下，如果当前 kanban是被分组显示的，那么就可以直接创建。没有分组的话，是不可以创建。设置为`true` ，两种情况都可以快捷创建。`false`，任何情况不可以快捷创建

可能的孩子节点有：

- `<field>`
有用来指明在 kanban logic中使用哪些字段来统计，如果这个 field 仅仅是在 kanban view中显示出来，那么就不需要 pre-declared，可设置以下属性：
 - `name` (required)
 - `sum`, `avg`, `min`, `max`, `count`。在 kanban column的顶部显示对应字段的统计值，the field's value is the label of the aggregation (a string). Only one aggregate operation per field is supported. （译者注：自己测试了一下，并没有效果。在自带模块中搜索，也没有看到kanban中的字段设置了这些属性）

- `<templates>`
定义一些 QWeb templates，为了更加清楚的查看 cards，可能定义多个 template。但是有一个是必需定义的，那就是`kanban-box`，每一个 record都会被 rendered once。kanban view 使用 javascript qweb，同时提供一些 context 变量：

 - `instance`  当前 Web Client instance
 - `widget` 当前 KanbanRecord()，可以用来获取 meta-information。这个 method 也可以直接在 template context 中调用，不需要通过 `widget.` 来调用。
 - `record` 一个包含所有显示请求的字段的 object。每个字段有两个属性 `value` 和 `raw_value`，前者更具当前用户提供的参数自动 formatted。后者则是通过`read()` 获取的原生值。（除了 date 和 datetime 两个字段，他们将会根据用户的 locale 来渲染）
 - `formats` the web.formats() module to manipulate and convert values
 - `read_only_mode` 就是字面意思。

## buttons and fields

虽然多数的 kanban templates 都是 standard QWeb，但是可以通过一定的方法，使 `<field>`, `<button>` , `<a>` 元素用不同的方式处理。

- 默认的，fields都显示它的 formatted value，除非 the match specific kanban view widget

- buttons and links 设置了 `type` 属性后，将会调用 Odoo 的相关方法，而不是其在 HTML 中的意义了. `type` 的值可以设置为：

 - `action`, `object`。 这个就和 List view , form view中的 button 类似了。
 - `open`  用 read only 的模式打开当前 record
 - `edit`  用 编辑的模式打开
 - `delete` 删除 record

## Javascript API

** `class KanbanRecord()`**
`Widget()` 用来将一个 record 渲染成一个 card。Available within its own rendering as widget in the template context.

- `kanban_color(raw_value)`
将当前 card 添加样式`oe_kanban_color_color_index`，默认定义好的 color_index 是 0到9。

- `kanban_getcolor(raw_value)`
将一个 color segmentation 转换成标准的 color_index(0 - 9)。color segmentation 可以是数字或这字符串

- `kanban_image(model, field, id[, cache][, options])`
生成一个 URL 用来将field 转成 image

- `kanban_text_ellipsis(string[, size=160])`
如果text超过指定的长度，就追加省略号来显示。
