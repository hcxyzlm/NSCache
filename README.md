# NSCache源码分析
NSCache的源码并不多，可以学习Foundation的编码风格。

### NSCache和NSDictionary的
共同点：
1. 都是基于key-value的存储
2. 都是容器类

不同点：
1. NSCache是线程安全的，NSDictionary不是
2. NSCache有管理缓存的策略，当系统资源将要耗尽时，它可以自动删减缓存

### 初始化
imageCache = [[NSCache alloc] init];

### 公开的属性
```obj

- (NSUInteger) countLimit; // 缓存的个数
- (NSUInteger) totalCostLimit; // 缓存的最大分配字节
- (id) delegate;               // 代理
- (BOOL) evictsObjectsWithDiscardedContent; // 存储nsobject类，类是否要实现NSDiscardableContent协议
- (NSString*) name;  // 可以设置特定的名字
```

### 私有属性
```objc
NSUInteger _costLimit;
  NSUInteger _totalCost;
  NSUInteger _countLimit;
  id _delegate;
  BOOL _evictsObjectsWithDiscardedContent;
  NSString *_name;
  NSMapTable *_objects;
  GS_GENERIC_CLASS(NSMutableArray, ValT) *_accesses; // LRU的管理内存策略
  int64_t _totalAccesses;
```

可以看出底层是基于NSMapTable实现key-value的方式,NSMapTable也是NSDictionary底层实现

查看m文件，里面有个_GSCachedObject的私有类
```objc
@interface _GSCachedObject : NSObject
{
  @public
  id object;
  NSString *key;
  int accessCount;
  NSUInteger cost;
  BOOL isEvictable;
}
```

可见 _GSCachedObject 就是对“键值对”的封装，因为缓存对象有重要性的分别，自然有 cost 作为表示

### 添加
```objc
- (void) setObject: (id)obj forKey: (id)key cost: (NSUInteger)num
{
// 先查出旧值
  _GSCachedObject *oldObject = [_objects objectForKey: key];
  _GSCachedObject *newObject;

  if (nil != oldObject)
    {
     // 删除旧值
      [self removeObjectForKey: oldObject->key];
    }
  [self _evictObjectsToMakeSpaceForObjectWithCost: num];
  
  // 新建_GSCachedObject
  newObject = [_GSCachedObject new];
  // Retained here, released when obj is dealloc'd
  
  // key, value赋值
  newObject->object = RETAIN(obj);
  newObject->key = RETAIN(key);
  newObject->cost = num;
  if ([obj conformsToProtocol: @protocol(NSDiscardableContent)])
    {
      newObject->isEvictable = YES;
      [_accesses addObject: newObject];
    }
    
  // 这个是存储obj真正的地方
  [_objects setObject: newObject forKey: key];
  RELEASE(newObject);
  _totalCost += num;
}
```

### 访问
```objc
- (id) objectForKey: (id)key
{
// 访问key
  _GSCachedObject *obj = [_objects objectForKey: key];

  if (nil == obj)
    {
      return nil;
    }
  if (obj->isEvictable)
    {
      // Move the object to the end of the access list.
      [_accesses removeObjectIdenticalTo: obj];
      [_accesses addObject: obj];
    }
  obj->accessCount++;
  _totalAccesses++;
  return obj->object;
}
```

### 删除
```objc
- (void) removeObjectForKey: (id)key
{
  _GSCachedObject *obj = [_objects objectForKey: key];

  if (nil != obj)
    {
      [_delegate cache: self willEvictObject: obj->object]; // delegate
      _totalAccesses -= obj->accessCount;
      [_objects removeObjectForKey: key];  //删除
      [_accesses removeObjectIdenticalTo: obj];
    }
}
```

### 删除所有
```objc
- (void) removeAllObjects
{
// 拿到迭代器
  NSEnumerator *e = [_objects objectEnumerator];
  _GSCachedObject *obj;

  while (nil != (obj = [e nextObject]))
    {
      [_delegate cache: self willEvictObject: obj->object];
    }
  [_objects removeAllObjects];
  [_accesses removeAllObjects];
  _totalAccesses = 0;
}
```

以上是个人的粗浅理解，如有错漏，欢迎指正！
