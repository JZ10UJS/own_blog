---
title: Odoo Views （二） - Form View
date: 2016-11-03 11:32:52
tags:
---

Form view 就是用来详细的展示单个 record 的各项值。跟节点就是 `<form>`，它由一些基本的 HTML 和一些特殊结构和语意来组成。

# Structural components 结构组成

用来设定 form 的基本页面结构

- **`notebook`**
定义了一些 `tabbed section`，每一个 tab 都是以 `<page>` 节点的方式定义出来，这个`<page>` 有以下属性：
 - `string` (required) tab 的 title
 - `accesskey` an HTML accesskey
 - `attrs` 标准的基于 record values 的动态属性
 
- **`group`**
用来设置forms 中的布局。默认的，groups定义了 两列，它的大多数子节点都只占用一列。如果 `<field>` 在 `<group>` 内部，那么将会默认的给这个field一个 label。那么这个 label 和 这个 field 各占 一列。
`<group>` 的默认列数可以通过设置 `col` 属性来被修改。而它里面的 element 可以设置`colspan` 属性来设置占用多少列。它的子节点是 horizontally 布局的。
`<group>` 可以设置 `string` 属性，来显示 title。

- **`newline`**
仅仅在`<group>` 的内部生效。强制换行

- **`separator`** 
一条小的水平分割线，可以设置 `string` 属性

- **`sheet`**
作为 `<form>` 的直接孩子节点。更加响应式的布局

- **`header`**
通常和`<sheet>` 一起使用，在 sheet 上面生成一个满宽度的 header，主要用于放置 workflow button 和 状态栏。

# Semantic components 语义组成

设置了一些与Odoo系统的交互。

- **`button`**  与 Odoo 系统交互，和 list view 中的 button 一样。

- **`field`**
渲染当前 record 的某个字段，有如下属性：
 - `name` 必选的，就是指明哪个字段将会被渲染出来。
 - `widget`  fields 通常有默认的渲染方式，通过这个可以指定其他渲染方式。
 - `options` JSON  object，用来给当前使用的 widget 提供一些特殊的配置信息
 - `class` HTML class，通常是下面几种值
  - `oe_inline`  阻止默认的换行
  - `oe_left`, `oe_right`  当前元素 `float` 到对应位置
  - `oe_read_only`, `oe_edit_only` 只在对应模式下，才显示这个field
  - `oe_no_button` 避免 `many2one` field 的导航 button
  - `oe_avatar` 图像 field 的 显示方式

- **groups** 特殊组的人，才显示这个字段

- **on_change** 8.0废弃，请使用 `openerp.api.onchange`

- **attrs** 根据不同规则，显示不同属性。

- **domain**  作用于关系型字段，通常用来筛选出合适的值，用来给用户选择。

- **context** 作用于关系型字段，去获取合适的值的时候，传入 context

- **readonly** 在编辑和读的模式下，都是只读的，不能修改

- **required**  设置之后，该字段必填，如果未填写，弹出提示。

- **nolabel** 只有当前 field 是 group 的直接孩子时，才生效。取消默认 label 显示
- **placeholder** 在字段没有值的时候，显示一些帮助信息。
- **mode** 作用于 `one2many` 字段，显示与这个字段有关联的record 。值有 `tree`, `form`, `kanban`, `graph`。默认是 `tree`
- **help** 当鼠标移动在这个字段上时，显示一些帮助信息
- **filename**
- **password** 这个字段的模式为 password， 填写后显示 星号。

# Business Views guidelines

Business views 主要是面向普通的用户，而非开发人员。比如在：Opportunites，Products，Partners，Tasks，Projects，etc.
一般来说，a business view 是由下面几个部分组成：

 1. 顶部的一个 status bar
 2. 中间的 sheet
 3. 底部是历史操作记录和各种留言
![这里写图片描述](http://img.blog.csdn.net/20161104104648765)
在XML中，大部分 form view 都是如下结构定义的

```xml
<form>
    <header> ... content of the status bar  ... </header>
    <sheet>  ... content of the sheet       ... </sheet>
    <div class="oe_chatter"> ... content of the bottom part ... </div>
</form>
```

## The Status Bar

这个bar 的作用就是显示当前 record 的 status 状态，以及一些 action buttons
![这里写图片描述](http://img.blog.csdn.net/20161104104711077)

### The Buttons

没什么好翻译的了，都是些设计界面时的建议，没什么技术相关，不翻了
----