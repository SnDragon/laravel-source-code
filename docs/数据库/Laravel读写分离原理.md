无论是使用原生SQL查询,查询构建器还是Eloquent模型,执行数据库操作可以大致分为四步:
1. 生成连接对象(5.5之后的版本在这一步不会真正创建数据库连接)
2. 构造SQL
3. 选择读连接还是写连接，建立与数据库的连接
4. 执行SQL

## 1. 生成连接对象

无论是哪种方式,最后都会运行到`Illuminate\Database\Connectors\ConnectionFactory::make`方法
```php
    public function make(array $config, $name = null)
    {
        // 解析和准备DB配置
        $config = $this->parseConfig($config, $name);
        // 如果有配置读写分离则调用createReadWriteConnection方法建立两个连接
        // 这里用到了懒加载到思想，不会真正建立连接，见下文
        if (isset($config['read'])) {
            return $this->createReadWriteConnection($config);
        }
        // 没有配置读写分离则调用createSingleConnection方法建立单个连接
        return $this->createSingleConnection($config);
    }
```


createReadWriteConnection方法:
```php
    protected function createReadWriteConnection(array $config)
    {
        // 创建\Illuminate\Database\Connection对象
        $connection = $this->createSingleConnection($this->getWriteConfig($config));
        // 创建获取读PDO的闭包并赋值给$connection的readPdo属性
        return $connection->setReadPdo($this->createReadPdo($config));
    }
    
    protected function createSingleConnection(array $config)
    {
        // 这里的pdo是一个闭包，执行闭包才会真正建立连接获取到写PDO
        $pdo = $this->createPdoResolver($config);
        /**
         * 根据不同的数据库驱动(mysql,pgsql,sqlite等)建立不同的连接
         * 把$pdo闭包对象作为对应的构造函数参数传进去
        **/
        return $this->createConnection(
            $config['driver'], $pdo, $config['database'], $config['prefix'], $config
        );
    }
    
    // 实际实现还会根据database是否配置了host选项去做不同解析,这里简化了
    protected function createPdoResolver(array $config)
    {
        return function () use ($config) {
            return $this->createConnector($config)->connect($config);
        };
    }
```
值得注意的一点是,这里的pdo是一个闭包对象实际上是做了优化了，这里创建Connection对象用到的$pdo/$readPdo对象是通过createPdoResolver获取到的闭包，也就是不会真正去建立PDO对象。

对比5.1:
```php
    protected function createSingleConnection(array $config)
    {
        // 真正的建立连接
        $pdo = $this->createConnector($config)->connect($config);

        return $this->createConnection($config['driver'], $pdo, $config['database'], $config['prefix'], $config);
    }
```

> 5.1在createReadWriteConnection这一步就会同时建立读连接和写连接，假如一个请求是只读的，这意味着写连接是不必要的，而且建立连接后需要等到请求结束才会释放，对于某些耗时的请求意味着写连接被白白占用，甚至导致数据库连接过多的错误，而新版本通过引入闭包实现了懒加载，解决了这个问题。

## 2. 构造SQL

主要是根据链式调用拼接SQL语句，绑定参数等,不是本文重点,这里不展开

## 3. 选择读连接还是写连接
最新版本会在这一步选择读连接还是写连接并真正与数据库建立连接
### select语句:

`Illuminate\Database\Query\Builder.php`
```php
    protected function runSelect()
    {
        // 调用第一步获取到的Connection对象去执行查询
        // 这里会传useWritePdo属性指定是使用读连接还是写连接(默认是false,即使用读连接)
        // 可以在查询之前调用useWritePdo方法,手动指定用写连接查询
        // select ... for update也会设置useWritePdo=true,参考lockForUpdate方法
        return $this->connection->select($this->toSql(), $this->getBindings(), ! $this->useWritePdo);
    }
```

`Illuminate\Database\Connection.php`

```php
    public function select($query, $bindings = [], $useReadPdo = true)
    {
        return $this->run($query, $bindings, function ($me, $query, $bindings) use ($useReadPdo) {
            if ($me->pretending()) {
                return [];
            }

            // For select statements, we'll simply execute the query and return an array
            // of the database result set. Each element in the array will be a single
            // row from the database table, and will either be an array or objects.
            $statement = $this->getPdoForSelect($useReadPdo)->prepare($query);

            $statement->execute($me->prepareBindings($bindings));

            return $statement->fetchAll($me->getFetchMode());
        });
    }
```

可以看到决定使用读连接还是写连接的关键代码是这一句:
```php
$this->getPdoForSelect($useReadPdo)
```
具体实现:
```php
    protected function getPdoForSelect($useReadPdo = true)
    {
        // 根据$useReadPdo去决定调用getReadPdo()还是getPdo()
        return $useReadPdo ? $this->getReadPdo() : $this->getPdo();
    }

    // getPdo很简单,如果$this->pdo是闭包(第一次执行的时候),则执行闭包建立与数据库的连接并重新赋值，否则直接返回
    public function getPdo()
    {
        if ($this->pdo instanceof Closure) {
            return $this->pdo = call_user_func($this->pdo);
        }

        return $this->pdo;
    }

    // 获取读Pdo,不一定就使用读连接
    public function getReadPdo()
    {
        if ($this->transactions >= 1) {
            return $this->getPdo();
        }

        if ($this->getConfig('sticky') && $this->recordsModified) {
            return $this->getPdo();
        }

        // 跟getPdo()类似,用闭包实现懒加载
        if ($this->readPdo instanceof Closure) {
            return $this->readPdo = call_user_func($this->readPdo);
        }

        return $this->readPdo ?: $this->getPdo();
    }
    
```
可以看到就算调用了getReadPdo()方法，最终也不一定是使用读连接,以下几种情况会使用写连接:
* 当前活跃事务数>0
* 配置了sticky属性为true(Laravel5.5引入)且$recordsModified为true(曾经执行过修改语句)
* readPdo为空时(例如没有配置读写分离时不会初始化readPdo)

### 修改语句
```php
    // delete类似
    public function update($query, $bindings = [])
    {
        return $this->affectingStatement($query, $bindings);
    }
    
    
    public function affectingStatement($query, $bindings = [])
    {
        return $this->run($query, $bindings, function ($me, $query, $bindings) {
            if ($me->pretending()) {
                return 0;
            }

            // 通过getPdo方法使用写Pdo
            $statement = $me->getPdo()->prepare($query);

            $statement->execute($me->prepareBindings($bindings));

            // 记录修改过
            $this->recordsHaveBeenModified(
                ($count = $statement->rowCount()) > 0
            );

            return $count;
        });
    }
    
    // insert或其他语句会执行到statement这里
    public function statement($query, $bindings = [])
    {
        return $this->run($query, $bindings, function ($me, $query, $bindings) {
            if ($me->pretending()) {
                return true;
            }

            $bindings = $me->prepareBindings($bindings);

            $this->recordsHaveBeenModified();

            return $me->getPdo()->prepare($query)->execute($bindings);
        });
    }
```

## 4. 执行SQL
这一步就是去调PHP原生的PDO相关方法去执行SQL语句，然后封装结果,这里不再赘述。

