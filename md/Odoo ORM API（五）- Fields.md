---
title: Odoo ORM API（五）- Fields
date: 2016-10-09 13:03:37
tags:
---

# Fields

## Basic fields  基础性字段

`class openerp.fields.Field(string=None, **kwargs)`
这些字段描述符也包含了对这个字段的定义，同时也对这条 record 的每个字段进行了一定的权限控制。下面的几个属性可以在定义字段时写入：
|-|Name|描述|
|--|--|--|
|Parameters:|string|在用户界面显示的label，如果没有设置，则用首字母大写的 field name|
||help|在用户界面，当鼠标移动在这个field上时，弹出一个提示框，内容为help中填写的内容|
||readonly(bool)|这个字段是不是**只读的**，默认为 False|
||required(bool)|这个字段是不是**必填的**，默认为 False|
||index(bool)|这个字段在数据表中是否设置**索引**，默认为 False|
||default|要么设置为静态的值，要么设置为一个函数，这个函数接受一个recordset作为参数，并返回一个值。 如： default=lambda self: self.env.user.name|
||states|一个字典，key为state字段中的各项值，value中可用的为"readonly", "required", "invisible"。<br/>**注意：** 使用这个字段，必需是这个model中设置了`state` 字段，这样才能被客户的正确的处理。这个的设置，通常是为了在某个阶段，才能让用户看到相应的信息<br/>states={'draft':[('readonly', False)], 'sent':[('readonly': False)]}|
||groups|以英文逗号分割的字符串，使这个字段只属于对应的groups。 <br/>如： groups="crm.group_sale_1,crm.group_sale_2"|
||copy(bool)|当在客户端复制某个record时，是否将这个字段的值复制过去。一般的field都是True, `One2many` 以及 computed field 和 related field 和 property field 都是 False|
||oldname(string)|这个field之前的名字，用以ORM能找到对应的字段进行更新数据|

### Computed fields

用以设置某个字段的值是直接根据计算而来，而不是仅仅的从数据库中读取。下面给出了这类字段所需要设置的属性，如果要定义一个computed field，只需要设置 compute 属性即可。
|--|Name|描述|
|--|--|--|
|Parameters:|compute|一个方法的名字，这个方法是用来计算这个field的值的|
||inverse|一个方法的名字，用以在设置这个字段后，针对对应的字段设置他们的值。可选的|
||search|一个方法的名字，定义针对这个字段的search行为。 可选的|
||store(bool)|这个字段是否存在数据库中。设置compute之后，默认为False|
||compute_sudo(bool)|这个字段是否需要以admin的权限的来计算，用以越过access rights， 默认False|


The methods given for compute, inverse and search are model methods. Their signature is shown in the following example:

```python
upper = fields.Char(compute='_compute_upper',
                    inverse='_inverse_upper',
                    search='_search_upper')

@api.depends('name')
def _compute_upper(self):
    for rec in self:
        rec.upper = rec.name.upper() if rec.name else False

def _inverse_upper(self):
    for rec in self:
        rec.name = rec.upper.lower() if rec.upper else False

def _search_upper(self, operator, value):
    if operator == 'like':
        operator = 'ilike'
    return [('name', operator, value)]
```
这个 compute method 将被作用到这个model中的每一个record中。`openerp.api.depends()` 这个装饰器必需作用与compute method上，用以指明这些依赖，以明确当那些依赖字段的值发生变化时，好重新计算这个field的值。重计算是自动的，并且确保了cache和database的一致性。注意：一个method可以作用于多个字段。只需要对这些计算字段的compute属性设置相同的compute method name 即可，这样这些方法只会针对这些字段被调用一次。
默认情况下，computed field不会存在数据库中，他们计算是 on-the-fly。可以通过添加一个属性 `store=True` 使得这个字段存在数据库中。带来的优势就是可以针对这个字段进行 search，并且是在数据库层就被search 完毕。劣势是当这个字段必需重新计算是，将会update database
inverse method，如他的名字一样，进行 compute method的逆运算。当你对 computed field设置某个值后，必需对这个computed field的 依赖字段进行某些改变，使得 compute method 对 依赖字段 计算之后，得到的值与你填入computed field的值相同。注意：如果没有设置inverse method，那么这个computed field是 readonly = True 的
search method 就是当有对这个field search时，hack掉基础的search 行为，并且返回一个新的domain，再进行search。必须是 field operator value

### Related fields

related field的值就是一个 由关系型字段的field names 用 dot 链接的string，当然最后一个field name可能不是关心型的。如果 cp_name = fields.Char(related="parent_id.company_id.name")。这个related field的一些属性将会自动的从 源 field 中直接复制: string, help, readonly, required(这个必须是所有的field name都是required，这个related field才会被设置成required=True), groups, digits, size, translate, sanitize, selection, comodel_name, domain, context。
默认的，这个字段也不会存在数据库中。但是可以通过添加store=True，使其存在数据库中。就像computed field 一样， related field 在他依赖的字段值发生变化时，他也会自动的重新计算。

### Company-dependent fields

就是以前的 'property' fields，这类字段的值是依赖于 company，也就是说，归属于不同公司的user对某一个record的field，得到的值是不同的。
**company_dependent** 这个field是否是company dependent （boolean）

### Incremental definition

A field is defined as class attribute on a model class. 如果这个Modle是继承的，也可以通过重新定义一个名字相同的field用以扩展源model的field的定义。在这中情况下，这个field的属性将从源model的field的属性复制，并且用你自己写的新的属性用来update。
例如：第二个class就是给仅仅给`state` 字段添加了一个 help

```python
class First(models.Model):
    _name = 'foo'
    state = fields.Selection([...], required=True)

class Second(models.Model):
    _inherit = 'foo'
    state = fields.Selection(help="Blah blah blah")
```

`class openerp.fields.Char(string=None, **kwargs)`
	Bases :  `openerp.fields._String`
	基础的string field，可以设定长度限制，通常在客户端用一行表示
|--|Name|Description|
|--|--|--|
|Parameters:|size(int)|这个字段最大的size|
||translate|使得这个field的值可以被翻译，使用`translate=True` 来翻译这个字段值，也可以给他设置成一个callable, transalte(callback, value)，翻译values使用 callback(terms)依赖获取这个value的翻译|

`class openerp.fields.Boolean(string=None, **kwargs)`
Bases: `openerp.fields.Field`

`class openerp.fields.Integer(string=None, **kwargs)`
Bases: `openerp.fields.Field`

`class openerp.fields.Float(string=None, digits=None, **kwargs)`
Bases: `openerp.fields.Field`
可以通过设置digits属性来实现Float的精度
**digits**  一个tuple（total, decimal)，或者一个function 接收一个 database cursor作为参数，并且返回一个 （total, decimal)

`class openerp.fields.Text(string=None, **kwargs)`
Bases :  `openerp.fields._String`
和Char非常相似，但是没有size属性，通常是作为多行显示的 Text box.
**translate** 这个和 Char 是一样的

`class openerp.fields.Selection(selection=None, string=None, **kwargs)`
Bases: `openerp.fields.Field`
|--|Name|Description|
|--|--|--|
|Parameters:|selection|制定这个字段的可能的值，它要么设置成 多个（value, string)组成的一个列表， 要么是一个 model method, 或者 model name。|
||selection_add|如果这个字段是 extended的话，可以通过设置这个属性，给源字段添加一些选项。必需是list of pairs(values, string)|
selection 属性是必需设置的，除非是这个字段是related field 或者是 field extensions。

`class openerp.fields.Html(string=None, **kwargs)`
Bases :  `openerp.fields._String`

`class openerp.fields.Date(string=None, **kwargs)`
Bases :  `openerp.fields.Field`

 - `static context_today(record, timestap=None)`
返回客户端的timezone对应的时间，通常用来计算默认值 。**timestap(datetime)** 可选的，用以替代当前的时间，必需是datetime，date不行，这个方法返回 一个 string

 - `static from_string(value)`
将一个 ORM value 返回成 date value

 - `static to_string(value)`
将一个date value 转换成一个 符合ORM要求的format 之后的string

 - `statci today(*args)`
返回符合ORM 要求的format的当前的日期，通常用来计算默认值


`class openerp.fields.Datetime(string=None, **kwargs)`
Bases :  `openerp.fields.Field`

- `static context_timestamp(record, timestamp)`
将给出的timestamp转换成客户端的对应timezone的时间，这个方法不是用来设置 _defaults initializer，因为datetime field是在客户端显示时自动修改的。如果要设置默认值，请使用fields.datetime.now()。
|--|Name|
|--|--|
|Parameters:|timestamp(datetime)，普通的UTC datetime, 用以将其转换成客户端timezone的datetime|
|Return type|datetime|
|Returns|一个被客户端timezone转换之后的datetime|


- `static from_string(value)`
将一个ORM value转换成 datetime

- `static now(*args)`
返回符合ORM 要求的format的当前的时间，通常用来计算默认值

- `static to_string(value)`
将一个datetime转换成符合ORM要求的 format之后的 string value。

## Relational fields 关系型字段

`class openerp.fields.Many2one(comodel_name=None, string=None, **kwargs)`
Bases: `openerp.fields._Relational`
这个字段的值是一个 size 0 或者 1的 recordset
|--|Name|Description|
|--|--|--|
|Parameters:|comodel_name (string)|目标model的name|
||domain(domain or string)|可选，用以在客户端供用户选择时，先进行一定的筛选|
||context(dict)|可选，用以在客户端处理这个字段时，设置他的context|
||ondelete|设置当引用的record被删除是，如果对本record进行的行为，可填：`set null`, `restrict`, `cascade`|
||auto_join|whether JOINs are generated upon search through that field (boolean, by default False)|
||delegate|set it to True to make fields of the target model accessible from the current model (corresponds to _inherits)|
comodel_name 是必需设置的，除了这个field 是 related field or field extendsions

`class openerp.fields.One2many(comodel_name=None, inverse_name=None, string=None, **kwargs)`
Bases: `openerp.fields._RelationalMulti`
One2many field; 这个字段的值是一个recordset，all the records in comodel_name such that the field inverse_name is equal to the current record.
|--|Name|Description|
|--|--|--|
|Parameters:|comodel_name (string)|目标model的name|
||inverse_name|对应target model(B model)的 Many2one的field的名字，这个field的 comodel_name必需是A model|
||domain|同Many2one|
||context|同上|
||auto_join|同上|
||limit|optional limit to use upon read (integer)|
comodel_name 和 inverse_name 是必需设置的，除了这个field 是 related field or field extendsions

`class openerp.fields.Many2many(comodel_name=None, relation=None, column1=None, column2=None, string=None, **kwargs)`
Bases: `openerp.fields._RelationalMulti`
Many2many field; 这个field的值是一个recordset
|--|Name|Description|
|--|--|--|
|Parameters:|comodel_name (string)|目标model的name|
||relation(string)|可选，储存两者关系的表的名字|
||column1(string)|optional name of the column referring to "these" records in the table relation|
||column2(string)|optional name of the column referring to "those" records in the table relation|
||domain(domain or string)|和其他两种relation field一样|
||context(dict)|同上|
||limit(int)|同上|
comodel_name 是必需设置的，除了这个field 是 related field or field extendsions 
relations column1 column2 是可选的，如果没提供，那么Odoo将会自动的根据对应的两个model自动生成。

`class openerp.fields.Referece(selection=None, string=None, **kwargs)`
 Bases: `openerp.fields.Selection`

