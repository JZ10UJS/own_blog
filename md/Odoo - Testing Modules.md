---
title: Odoo - Testing Modules
date: 2016-10-18 14:43:25
tags:
---

 Odoo 通过对 Python 的 unittest 封装提供了对模块的测试功能
 如果要自定义一些测试，只需要简单的在你的模块目录下创建一个 tests 文件夹，它将会自动的检测你的模块。
 这写module的名字应该是以`test_` 开头，并且应该在`tests/__init__.py` 文件中明确的 import 

```bash
your_module
|-- ...
`-- tests
    |-- __init__.py
    |-- test_bar.py
    `-- test_foo.py
```
中在这个 `__init__.py` 中应该包含

```python
from . import test_foo, test_bar
```
> 注意
> 在这个文件中没有import的，将不会被调用测试

在8.0之前，只会测试在`tests/__init__.py` 文件中的 在`fast_suite` 或 `checks` 两个列表中的指明的 test module,
8.0及以后，会测试所有在 `tests/__init__.py` 文件中 import 的 test module.

test runner 像 unittest 中定义的普通的一样，跑所有符合 unittest 定义要求的测试函数，Odoo同时也提供了一系列 utilities 来帮助测试 Odoo 相关的数据:

`class odoo.tests.common.TransactionCase(methodName='runTest')`
---
这中情况下，每一个 test method 都是自己单独的一个 transaction，自己单独的一个 db cursor。在每一个test method 结束之后，这个 transaction 会roll back， cursor 关闭。

- `browse_ref(xid)`
	返回对应的 external id 的 record object
	|-|-|
	|--:|--|
	|Parameters: | xid - 完整的 external id。包含 '.' |
	|Raise: | ValueError if not found |
	|Returns: | `BaseModel` |
	

- `ref(xid)`
	返回对应的 external id 的 database id。
	|-|-|
	|--:|--|
	|Parameters: | xid - 完整的 external id。包含 '.' |
	|Raise: | ValueError if not found |
	|Returns: | registered id |

`class odoo.tests.common.SingleTransactionCase(methodName='runTest')`
---

这情况下，所有的 test method 都是同一个 transaction，同一个 db cursor。在最后一个 test method 结束之后，这个 transaction 会roll back， cursor 关闭。

-  `browse_ref(xid)`  同上
-  `ref(xid)`  同上


`odoo.tests.common.at_install(flag)`
--
设置一个test的 at-install state，flag是一个 boolean，用来指定在 odoo module 安装的时候，是否应该测试。
通常，tests 是在这个Odoo模块安装完成之后，下一个模块安装之前，进行测试。

`odoo.tests.common.post_install(flag)`
--
设置一个test的 post-install state，flag是一个 boolean，用来指定在 odoo 一系列 module 安装之后，是否应该测试。
By default, tests are not run after installation of all modules in the current installation set.


-----
最常用的是 `TransactionCase`，在每一个method中只测试一个property

```python
class TestModelA(common.TransactionCase):
    def test_some_action(self):
        record = self.env['model.a'].create({'field': 'value'})
        record.some_action()
        self.assertEqual(
            record.field,
            expected_field_value)

    # other tests...
```
Running tests
--

当你设置了 `--test-enable` 参数，启动 Odoo server 并且要安装或者更新 modules 时，就会自动的进行测试。
As of Odoo 8, running tests outside of the install/update cycle is not supported.
