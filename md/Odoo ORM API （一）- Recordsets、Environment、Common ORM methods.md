---
title: Odoo ORM API （一）- Recordsets、Environment、Common ORM methods
date: 2016-09-26 15:55:40
tags:
---

# Recordsets
从8.0开始提供一种新式API，接下来也会长期支持这个新式的API。在本片中，也是介绍怎么在新旧API之间切换，但是旧API介绍的不多，如果有需要，请查看老版文档。
与 models 和 records 的交互都是通过一个特有的东西 recordsets 来执行的，它是一个根据id 已经排好序的 同 models 的 record 的集合
> 注意：
> 虽然名字是 recordsets，是一个集合，但是仍然有较小的概率导致里面包含重复的 record，这个将在未来被修复。

在models中定义的 方法， 都是作用于 recordset中，`self` 就代表 recordset

```python
class AModel(models.Model):
	_name = 'a.model'
	
	def a_method(self):
		# self 可以是 数据库中 0 或者所有当前model record 的集合
		self.do_operation()
```
通过迭代 self，将 yield 一个 新的长度为1的 recordset，这个和普通的python string一样，迭代一个string，yield 一个长度为1的新的 string

```python
def do_opertion(self):
	print self  # a.model(1, 2, 3, 4, 5)
	for record in self:
		print record   # => a.model(1,) then a.model(2,) .... a.model(5,)
	
```
## Field access
recordsets 提供一个接口：models的各个field 可以直接被 record 通过 ‘.’来读和写，但是这个recordset的长度必须是 1，通过这种方式设置 field的值时，将会调用的一个database的update

```python
>>> record.name
Examle name
>>> record.company_id.name
Company name
>>> record.name="Bob"
```
如果在 recordset长度不等于 1的情况下，访问或者修改field的值，将会抛出一个异常
通过这种方式，获取关系型field的值（Many2one, One2many, Many2many），将会返回一个 recordsets，如果没有设置这些的字段的值得话，读取这些字段，将会返回这个字段对应model的空recordset。

> 重要：
> 每一次对field进行赋值，都将会涉及一次database的update操作，所以，当你在某个时间，需要对多个字段的值进行修改，请使用`write()`

```
# 3 * len(records) 的数据库操作
for record in records:
	record.a = 1
	record.b = 2
	record.c = 3

# len(records)次 数据库操作
for record in records:
	record.write({'a':1, 'b':2, 'c':3})

# 1 次数据库操作
records.write({'a':1,'b':2','c':3})
```

## Set operations
Recordsets 是不可变的，但是同一model下面的不同recordset可以通过一定的操作符进行运算，并且返回一个新的recordsets，

- `record in set` 将返回 这个record（必须是长度为1的recordsets）是否在`set` 中。`not in` 就是相反的意思
- `set1 <= set2` 以及 `set1 < set2` 将返回 set1 是否是 set2 的一个子集。
- `set1 >= set2` 以及 `set1 > set2` 将返回 set1 是否完全包含 set2。
- `set1 | set2` 是将两个 set 合并起来，返回一个 不包含重复record的新的recordsets
- `set1 & set2` 返回他们的交集
- `set1 - set2` 返回 Set1 中所有不在 set2 中的 recordsets

## Other recordset operations
Recordsets 是可迭代的，所以大部分Python的方法都可以作用于它（比如：map，filter，sorted...），但是这些方法返回的是一个 `list` 而不是一个 iterator，这将导致返回的数据，不能使用方法，以及set的各种运算。
因此，我们提供一个下面的几个方法，使得返回的数据任然是recordsets

- `filtered()`
返回一个recordsets，这个recordsets中的所有record都满足filter提供的函数，或者在filter中传入一个string，里面是record的某个字段

```python
# only keep records whose company is the current user'
records.filtered(lambda r: r.company_id == user.company_id)

# only keep records whose partner is a company
records.filtered('partner_id.is_company')
```

- `sorted()`
返回一个根据提供参数作为排序依据的 recordsets

```python
# returns a list of summing two fields for each record in the set
records.sorted(lambda r: r.field1 + r.field2)
```

- `mapped()`
将作为参数提供的function作用于每一个record上，如果返回的值是一个recordsets就返回一个recordsets，否则返回一个list

```python
# 返回一个由各个record的两个字段之和组成的list
records.mapped(lambda r: r.field1 + r.field2)
```
也可以传入string用以获取字段值：

```python
# returns a list of names
records.mapped('name')

# returns a recordset of partners
record.mapped('partner_id')

# returns the union of all partner banks, with duplicates removed
record.mapped('partner_id.bank_ids')
```

# Environment
`Environment` 储存了各种各样的 contextual data： 数据库连接的 cursor（用以操作数据库），当前登录的user（用以检测各项权限），当前的context（storing arbitrary metadata），同样也储存 caches。
所有的recordsets都有一个 不可变的environment，可以通过 `env` 来获取 `user` ，`cr`，`context` 等。

```python
>>> records.env
<Environment object ...>
>>> records.env.user
res.user(3)
>>> records.env.cr
<Cursor object ...>
```
当从一个recordsets创建另一个recordset时，environment会被继承过去。recordset通过environment可以创建一个新的**空的**recordset，并且通过新的recordsets来获取对应model的数据：

```python
>>> self.env['res.partner']
res.partner
>>> self.env['res.partner'].search([['is_company', '=', True], ['customer', '=', True]])
res.partner(7, 18, 12, 14, 17, 19, 8, 31, 26, 16, 13, 20, 30, 22, 29, 15, 23, 28, 74)
```
## Altering the environment
通过recordset可以提供一个新的environment，并且返回一个根据新的environment下的recordset

- `sudo()`
通过提供的 user recordsets 创建一个新的 environment，如果不提供参数的话，将默认使用administrator作为参数，返回一个根据新的environment作用下的recordset

```python
# 以administrator的权利，创建一个新的 res.partner
env['res.partner'].sudo().create({'name': "A Partner"})

# 显示所有 public 用户能看到的 res.partner
public = env.ref('base.public_user')
env['res.partner'].sudo(public).search([])
```
- `with_context()`
 
 1. 可以传入一个位置参数，用以替换当前environment下的context
 2. 可以传入一堆位置参数，更新当前的environment的context

```python
# look for partner, or create one with specified timezone if none is found
env['res.partner'].with_context(tz=a_tz).find_or_create(email_address)
```

- `with_env()`
替换当前的environment

# Common ORM methods

- `search()`
接受一个 search domain，返回一个满足条件的recordset，也可以通过设置 `offset` and `limi` 参数来控制返回子集，通过设置`order` 来设置排序条件

```python
>>> # searches the current model
>>> self.search([('is_company', '=', True), ('customer', '=', True)])
res.partner(7, 18, 12, 14, 17, 19, 8, 31, 26, 16, 13, 20, 30, 22, 29, 15, 23, 28, 74)
>>> self.search([('is_company', '=', True)], limit=1).name
'Agrolait'
```

> 提示:
> 如果说，只想要查询满足条件的record有多少，请使用 `search_count()`


- `create()`
接受一些键值对，返回一个根据上面参数创建的新的recordset

```python
>>> self.create({'name': "New Name"})
res.partner(78)
```

- `write()`
接受一些键值对，并修改recordset中所有record对应key的值，不返回值

- `browse()`
接受一个database id 或者 一个由 id组成的 list，返回对应model的 id等于提供参数的 recordset

```python
>>> self.browse([7, 18, 12])
res.partner(7, 18, 12)
```

- `exists()`
返回一个在数据库中存在的recordsets，通常用于检验某个record是否还在数据库中

```python
if not record.exists():
	raise Exception('The record has been deleted')
```
或者在执行某种删除操作后，再处理一下

```python
records.may_remove_some()
# 保留那些没有被删除的record
records = records.exists()
```

- `ref()`
Environment method，用以根据 external id 返回一个record

```python
>>> env.ref('base.group_public')
res.groups(2)
```

- `ensure_one()`
检查这个recordset是否长度为1，如果不是，将会抛出异常

```python
records.ensure_one()
assert len(records) == 1, "Excepted singleton"
```