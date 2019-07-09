---
title: Odoo ORM API（三）- Model Reference
date: 2016-09-29 17:39:30
tags:
---

# Model Reference

`class openerp.models.Model(pool, cr)`
OPENERP 的 Models 都是继承自这个 class

```python
class user(Model):
	...
```
这个在每个数据库中值会被系统实例化一次。

## Structual attributes

- `_name`
business object name， 通常是 '.'，用于存放在 module的命名空间中

- `_rec_name`
可选的，用于将某个字段设置成`name`， 将被 osv 的 name_get()调用（默认为 `name`）

- `_inherit`

 - 如果 `_name` 设置了，那么它的值就是 他想继承的的父类的 `_name`，如果只是继承一个父类，可以直接设置成str，将会把数据存在自己的表上
 - 如果 `_name` 没有设置，将会直接在父类的表上进行拓展修改。

- `_order`
默认的order选项（default: 'id'）, 如果search（）中设置了 order 参数，将以后者为准

- `_auto`
是否创建一个数据库表（default：True），如果设置为False，需要复写 init（）用以创建数据表

- `_table`
当创建数据表时，使用的名字，通常会根据默认参数自动生成

- `_inherits`
传入一个字典，key 是继承的model的`_name`，值为field的名字。

```
_inherits = {
    'a.model': 'a_field_id',
    'b.model': 'b_field_id'
}
```
这个新的model，与父类相关的值都不会存在自己的表中，而是存储在父类的表中。

- `_constrains`
list of (constraint_function, message, field) ，8.0之后不再赞成使用，建议使用 `api.constrains()`

- `_sql_constraints`
list of (name, sql_definition, message) triples , 当建立数据表时，用以限制。

- `_parent_store`
Alongside `parent_left` and `parent_right`，设置一个 [nested set](https://en.wikipedia.org/wiki/Nested_set_model) 用以快速查询当前model的record。（default: False）

## CRUD （create，retrieve，update，delete）

- `create(vals) -> record`
为这个 model 创建一个新的 record。
这个新的 record 使用提供的值来初始化，如果必要的话，从`default_get()` 取值
| Name | 描述 |
|---:|------------|
|参数|vals(dict),{'field_name': field_value, ...}|
|返回值|一个创建的新的record|
|报错|AcessError: 如果当前user没有创建这个object的权限，或者用户试图绕过访问规则创建请求对象.<br>ValidateError: 给selection字段传入一个无效的值。<br>UserError: 如果创建了一个循环调用，如：给一个res.partner设置它自己为自己的parent_id|


- `browse([ids]) -> records`
根据提供的ids，返回对应id的record组成的recordset
可以不提供Id，或者只有一个Id，或者sequence of ids

- `unlink()`
删除当前的recordsets
可能会抛出 AcessError（和write一样） 或者 UserError（如果这个record是其它records的默认属性）

- `write(vals)`
把当前recordset中所有record根据提供的参数，修改值
其余基本和 create 一样
 
 - 如果是数字类的 fields（Integer，Float），传入的value也应该是对应的 type
 - Boolean field 的话，就应该传入 boolean
 - Selection field，传入对应的选项，多数是str，极少为 integer
 - Many2one，应该传入对应record的 database id
 - 其它的非关系型的字段，就传入 str
> 注意：
> 由于历史的兼容性的原因， Date 和 Datetime fields 读写中得到的都是 str 类型，而非对应的 date 和 datetime 类型。date string 都是 UTC-only 的，format 格式为 `"%Y-%m-%d"`  和 `"%Y-%m-%d %H:%M:%S"`

 - One2many 和 Many2many 使用一个特殊的命令，还将这些record与这个字段关联起来
这个命令就是 由多个元祖（3个元素）组成的列表，每一个元祖都是一条命令，它们将按顺序执行，通常是这些：

 1. `(0, _, values)` 给这个field 添加一个新的 record，这个新的record 是根据 values(dict)的值创建出来的。
 2. `(1, id, values)` 用values（dict）来更新这个 field 已经关联到的 id 为 传入 id的 record， 不能用在 `create()`中
 3. `(2, id, _)` 将 id 为传入 id的record 从 field的关联中删除，并且将其从数据库中删除。不能用在create()中
 4. `(3, id, _)` 将 id 为传入 id的record 从 field的关联中删除，但是不从数据库中删除。不能作用在One2many字段上，也不能用在create()中
 5. `(4, id, _)` 将一个已经存在的id为传入id的record添加到field的关联中，不能作用在One2many字段上，
 6. `(5, _, _)` 将这个field的关联集清空，等同于对这个集合中的每一个record执行 `(3, id, _)` 命令。不能用在One2many，不用再create（）
 7. `(6, _, ids)` 用ids这些records来替代原有field对应的关联集，等同于先对这个record执行`(5, _, _)`，再对ids中的每一个record，执行`(4, id, _)`。
> 提示：
> 上面 的 `_` 都代表不重要的意思，通常填入 False 或者 0

 - `read([fields])`
读取对应field的值，并将他们以字典的形式返回。每一个record一个字典，最后返回一个列表。

 - `read_group(*args, **kwargs)`
Get the list of records in list view grouped by the given groupby fields
| Name | 描述 |
|----:|------------|
|参数|**cr** - database cursor<br>**uid** - 当前用户id<br>**domain** - 普通的domain，表明在哪些符合要求的record进行read_group<br>**fields**(list) - 列出需要展现的字段<br>**groupby**(list) - 列出根据哪些字段来分组，这中间要么直接写出字段名称，或者`field:groupby_function`，groupby_function目前只支持`day`,`week`,`month`,`quarter`,`year`，并且只能作用于 Date 和 Datetime 类型的字段<br>**offset**(int) - 可选的，用以跳过几条record<br>**limit**(int) - 可选，只返回一定数目的record<br>**context**(dict) - 基本的context参数，比如lang、time zone<br>**orderby**(list)，就和search()中的orderby一样，目前仅支持many2one<br>**lazy**(bool)	- 如果为True，那么就只groupby第一个groupby，其它的groupby传入到__context key中。如果是False,一次性将所有的groupby全部执行|
|返回值|多个dict(每个record一个dict)组成的list,每个dict包含：<br>the values of fields grouped by the fields in groupby argument<br>__domain: list of tuples specifying the search criteria<br>__context: dictionary with argument like groupby<br>|
|返回类型|[{'field_name_1': value, ...]|
|异常|AcessError: 如果当前user没有创建这个object的权限，或者用户试图绕过访问规则创建请求对象.|

## Searching

- `search(args[, offset=0][, limit=None][, order=None][, count=False])`
最主要的就是args中的domain
| Name | 描述 |
|---:|------------|
|参数|**args** - 一个 search domain， 如果没有domain，传入一个空列表，表示所有<br>**offset** (int) 忽略多少个<br>**limit**（int） - 最大返回多少个<br> **order**（str），一个由field name组成的str，其中用逗号分隔<br>**count**（bool） - 如果设置为True，仅仅用来计数符合要求的数目，并且返回这个数。默认是False|
|返回值|多数情况下，返回一个符合要求的recordset|
|报错|AcessError: 如果当前user没有创建这个object的权限，或者用户试图绕过访问规则创建请求对象.|


- `search_count(args) -> int`
返回符合domain的record的数目

- `name_search(name='', args=None, operator='ilike', limit=100) -> records`
 将`name`符合pattern的record筛选出来，包含 domian（args）
 相当于进行search（）后，再name_get()
  | Name | 描述 |
|---:|------------|
|参数|**name** - 搜索的pattern <br>**args** - 一个 search domain， 如果没有domain，传入一个空列表，表示所有<br>**operator**（str），`ilike`, `=`，等<br>**limit**（int） - 最大返回多少个<br> |
|返回类型|list|
|返回值|符合要求的record的 list of pairs （Id，text_repr）|

##Recordset operations

- `ids` 返回这个recordset所有record的数据表id组成的list

- `ensure_one()` 验证这个recordset的长度是否为1，否则抛出异常

- `exists()` -> records ，返回这个recordset中所有实际存在的recordset。

- `filtered(func)` 
返回这个func(rec)为True的所有record组成的recordset。这个func可以是一个函数，也可以是一个以点分隔的field names。

- `sorted(key=None, reverse=False)`
返回根据key排序的recordset，key和普通python sorted 的 key 差不多

- `mapped(func)`
将func作用于每一个record上，如果func返回recordset，那么总体就返回recordset，否则总体返回list
func可以是 函数，也可以是以点分隔的field names。

## Enviroment swapping

- `sudo([user=SUPERUSER])`
返回一个适用于传入user的recordset
默认情况下，返回一个superuser权限的recordset，这将会直接越过 access control 和 record rule。
> 注意:
> 由于使用 sudo 将会越过各种权限设置，所以查询出来的数据可能会包含本来设置为区分的record
> 比如（多公司环境下的record）
>
> 而且，这个新的recordset 将不会被cache。所以，查询数据将会产生一定的延迟。

- `with_context([context][, **overrides])` -> records
返回一个新的符合给定context的recordset

```python
# current context is {'key1': True}
r2 = records.with_context({}, key2=True)
# -> r2._context is {'key2': True}
r2 = records.with_context(key2=True)
# -> r2._context is {'key1': True, 'key2': True}
```

- `with_env(env)`
返回一个新的符合给定env的recordset
> 注意：
> 这个新的environment将不会采用当期environment的cache，所以后续的数据查询将会有延迟，因为要直接从数据库中读取。

## Fields and views querying

- `fields_get([fields][, attributes])`
返回每个field的definition
返回值是一个dict，传入的field作为key。同样可以传入_inherit的model的field，每一个field对应一个字典，其中string，help和selection都会被自动翻译。
Parameters	
allfields -- list of fields to document, all if empty or not provided
attributes -- list of description attributes to return for each field, all if empty or not provided

- `fields_view_get([view_id | view_type='form'])`
Get the detailed composition of the requested view like fields, model, view architecture
Parameters	
view_id -- id of the view or None
view_type -- type of the view to return if view_id is None ('form', 'tree', ...)
toolbar -- true to include contextual actions
submenu -- deprecated
Returns	dictionary describing the composition of the requested view (including inherited views and extensions)
Raises	
AttributeError --
if the inherited view has unknown position to work with other than 'before', 'after', 'inside', 'replace'
if some tag other than 'position' is found in parent view
Invalid ArchitectureError -- if there is view type other than form, tree, calendar, search etc defined on the structure

## ???

- `default_get(fields_list)` -> default_values
返回这个fields_list中所有field对应default value，这些default value由context，user default，和 models自己定义的决定。
Parameters	`fields_list` -- a list of field names
Returns	返回一个字典，fields_name : default value。如果这个field，有defualt value

- `copy(default=None)`
复制当前record，并用default更新
Parameters	default (dict) -- dictionary of field values to override in the original values of the copied record, e.g: {'field_name': overridden_value, ...}
Returns	new record

- `name_get()` -> [(id, name), ...]
返回这个recordset中所有record对应的名字，默认是返回record的 `display_name` 字段。
	|name|descrition|
	|--|--|
	|Returns	|list of pairs (id, text_repr) for each records|
	|Return type	|list(tuple)|

- `name_create(name)` -> record
通过 `create()` 创建一个新的record，但是只传入一个参数（display_name）。
这个新的record将采用一切合适的default Value，包括context中包含的。
	|name|descrition|
	|--|--|
	|Parameters|name -- display name of the record to create|
	|Returns	|	the name_get() pair value of the created record|
	|Return type	|record|


## Automatic fields

- `id` 

- `_log_access` 默认为True

- `create_date` 创建record的时间  type： datetime

- `create_uid` 创建record的用户  type： res.users

- `write_date` 最近一次修改的时间 type：datetime

- `write_uid` 最近一次对其修改的用户 type： res.users

## Reserved field names

有些字段名字，是作为保留字段，被预先定义好了。如果你要在某个model中实现这种相关的功能，可以通过定义这些名字的field来实现。

- `name`   type Char
`_rec_name` 字段的默认值，通常用来显示一个record的可读的名字

- `active`  type Boolean
用以全局的改变这个record的可见，如果`active` 被设置为False，那么绝大多数的search和listing中都不会显示  

- `sequence` type Integer
修改order的标准，同时允许在tree view界面，进行拖曳，用以调整顺序

- `state` type Selection
一个object的周期，可以被用在field的states属性中

- `parent_id` type Many2one
设置各种层级，并且可以在domain中使用`child_of`  operator

- `parent_left`
和 `_parent_store` 一起使用，通常是为了高效率的 tree structure
- `parent_right`
同上



