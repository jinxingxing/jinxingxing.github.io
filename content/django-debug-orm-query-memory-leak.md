Title: Django Debug 模式下大量查询时的内存占用问题排查
Category: Python
Tags: Python, Django, Debug, SQL, ORM, Models, Query, 查询, Memory, leak, 内存，泄漏 
Authors: jinxing
Date: 
Modified:
Slug: 
Summary: 

Django Debug 模式下大量查询时的内存占用问题排查
===============================================

### 现象
1. 在本地做测试，启动 100个 agent，执行 uptime 10次，每次执行输出都立即返回输出给 server
2. top 查看，发现 `task_queue.py` 进程占用内存达到 1G+

### 排查
1. 使用 `meliae` 库 dump 出对象的引用关系数据， 然后加载 dump 出的关系数据，查看统计信息，启动 embed IPython	进行手动分析	
```Python
from meliae import loader, scanner
scanner.dump_all_objects('./objects_dump.txt')
om = loader.load('./objects_dump.txt')
print om.summarize()
from IPython import embed
embed()

```

2. 发现 unicode 对象占用90%以上的内存
```Python
Total 154829 objects, 392 types, Total size = 1157.5MiB (1213776200 bytes)
 Index   Count   %      Size   % Cum     Max Kind
     0   45339  291193593360  98  98 1611120 unicode
     1   16725  10   7236600   0  98  196888 dict
     2   28517  18   2633114   0  99   89468 str
     3    1599   1   1445496   0  99     904 type
     4     558   0   1442400   0  99   12624 module

```

3. 通过配合 IPython, ctypes 一步步查到引用 unicode 对象的元凶

```Python
In [2]: au = om.get_all('unicode')[0]	# 获取所有 unicode 对象，取第一个返回

In [3]: au
Out[3]: unicode(140189670108784 1608304B 1par 'INSERT INTO `echelon_changelogentry` (`content_type`, `object_id`, `object_str`, `action`, `timestam"')		# 发现是保存的数据库查询语句

In [4]: au.p	# 所有引用该 unicode 对象的列表
Out[4]: [dict(140189669965904 280B 4refs 1par)]

In [5]: au.p[0].p
Out[5]: [list(140189709938200 98632B 12116refs 1par)]

In [6]: au.p[0].p[0].p
Out[6]: [DatabaseWrapper(140189709836048 3416B 51refs 2par)]	# 通过一层层查看，这个变量比较可疑

In [7]: au.p[0].p[0].p[0].p
Out[7]: 
[dict(140189721487352 280B 2refs 1par),
 DatabaseErrorWrapper(140189676039824 344B 3refs 1par)]

In [8]: au.p[0].p[0].p[0].p[0].p
Out[8]: [dict(140189721487072 280B 2refs 1par)]

In [9]: au.p[0].p[0].p[0].p[0].p[0].p
Out[9]: [thread._local(140189724187920 96B 2refs 1par)]		# 已经到线程顶层

# 这里定义了一个通过 id 查找到对象的函数
In [10]: def byid(id_):	
   ....:     import ctypes
   ....:     return ctypes.cast(id_, ctypes.py_object).value
   ....: 

In [13]: dw = byid(140189709836048)
# 查找对象的属性，看是那一个变量指向了 list(140189709938200 98632B 12116refs 1par)
In [15]: [attr for attr in dir(dw) if id(getattr(dw, attr)) == 140189709938200]
Out[15]: ['queries']  # 就是这个 queries

In [17]: len(dw.queries)
Out[17]: 12116
```

4. 查看源代码 `django/db/backends/__init__.py`
```Python
class BaseDatabaseWrapper(object):
    def __init__(self, settings_dict, alias=DEFAULT_DB_ALIAS,
                 allow_thread_sharing=False):
        # ...
        self.queries = []  # 正是因为这个 queries 保存了所有的查询语句导致内存不释放
        # ...
```

5. 搜索对 queries 的调用，发现并没有对该属性的操作，于是使用 property 拦劫对 queries 的访问
```Python
# 已将 self.queries 重命名为 self._queries
def get_queries(self):
    return self._queries

def set_queries(self, value):
    self._queries = value

queries = property(get_queries, set_queries)
```

6. 在 `get_queries()` 和 `set_queries()` 处下断点，然后通过查看调用栈发现因为开启了 Debug 所以  `BaseDatabaseWrapper.cursor()` 函数返回的 cursor 对象会将所有查询 append 到 queries 。。。
```Python
class CursorDebugWrapper(CursorWrapper):
    def execute(self, sql, params=None):
        start = time()
        try:
            return super(CursorDebugWrapper, self).execute(sql, params)
        finally:
            stop = time()
            duration = stop - start
            sql = self.db.ops.last_executed_query(self.cursor, sql, params)
            self.db.queries.append({
                'sql': sql,
                'time': "%.3f" % duration,
            })
            logger.debug('(%.3f) %s; args=%s' % (duration, sql, params),
                extra={'duration': duration, 'sql': sql, 'params': params}
            )
```

7. 将 Debug 设为 False 后内存占用恢复正常

### 教训
**不要在 Debug 模式下进行性能测试！！！** 

