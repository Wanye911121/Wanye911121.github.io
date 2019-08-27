
# 关联对象(AssociatedObject)

## 使用场景

在日常开发中，我们使用关联对象的主要场景是**为分类添加“属性”**

属性：包括实例变量_xxx和get方法xxx、set方法setXXX。在分类中并不会自动生成实例变量和存取方法，**所以需要使用关联对象添加“属性”**

## 相关函数

```
用于给对象添加关联对象，传入 nil 则可以移除已有的关联对象；
void objc_setAssociatedObject ( id object, const void *key, id value, objc_AssociationPolicy policy );

用于获取关联对象；
id objc_getAssociatedObject ( id object, const void *key );

用于移除一个对象的所有关联对象。
void objc_removeAssociatedObjects ( id object );
```

## 实现原理

### 在了解具体实现前，我们需要知道几个类和数据结构

* AssociationsManager
  * AssociationsManager 通过持有一个自旋锁 **spinlock_t** 保证对 AssociationsHashMap 的操作是**线程安全**的，即每次只会有一个线程对 AssociationsHashMap 进行操作。
* AssociationsHashMap
  * **全局的唯一的**保存关联对象的哈希表（key：对象的指针，value对象的ObjcAssociationMap）
* ObjcAssociationMap
  * ObjectAssociationMap 则保存了从 **key** 到关联对象 **ObjcAssociation** 的映射，这个数据结构保存了当前对象对应的所有关联对象
* ObjcAssociation
  * ObjcAssociation 就是真正的关联对象的类，如何存储ObjcAssociation对象

### 实例图

objc_setAssociatedObject(obj, @selector(hello), @"Hello", OBJC_ASSOCIATION_RETAIN_NONATOMIC);

![910b3750b37f9d7059da617d6a9a0cc2.png](evernotecid://E7D2FF97-88A4-42B7-84F6-918A5CCFFB22/appyinxiangcom/25896710/ENResource/p129)

### objc_setAssociatedObject

```
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        ObjectAssociationMap *refs = i->second;
        ...
    }
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
```

* new_vaule!=nil(需要创建或更新关联对象的值)

    1. 使用 old_association(0, nil) 创建一个临时的 ObjcAssociation 对象
    2. 调用 acquireValue 对 new_value 进行 retain 或者 copy。

    3. 初始化一个 AssociationsManager，获取全局唯一的AssociationsHashMap
    4. 先使用 DISGUISE(object) 作为 key 寻找对应的 ObjectAssociationMap
    5. 如果没有找到，初始化一个 ObjectAssociationMap，再实例化 ObjcAssociation 对象添加到 Map 中，并调用 **setHasAssociatedObjects** 方法，表明当前对象含有关联对象
    6. 如果找到了对应的 ObjectAssociationMap，就要看 key 是否存在了，由此来决定是更新原有的关联对象，还是增加一个
    7. 释放创建的临时对象

* new_value == nil(删除一个关联对象)，删除对应key的关联对象。调用**erase**方法，擦除 ObjectAssociationMap 中 key 对应的节点。

### objc_getAssociatedObject

```
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        ((id(*)(id, SEL))objc_msgSend)(value, SEL_autorelease);
    }
    return value;
}
```

1. 获取AssociationsHashMap
2. 以 **DISGUISE(object)** 为 key 查找 AssociationsHashMap
3. 以 **void \*key** 为 key 查找 ObjcAssociation
4. 根据 **policy** 调用相应的方法
5. 返回关联对象 ObjcAssociation 的值

### objc_removeAssociatedObjects

为了加速移除对象的关联对象的速度，我们会通过标记位 **has_assoc** 来避免不必要的方法调用，在确认了对象和关联对象的存在之后，才会调用 _object_remove_assocations 方法移除对象上所有的关联对象

### 总结原理

* 关联对象其实就是 ObjcAssociation 对象
* 关联对象由 AssociationsManager 管理并在 AssociationsHashMap 存储
* 对象的指针以及其对应 ObjectAssociationMap 以键值对的形式存储在 AssociationsHashMap 中
* ObjectAssociationMap 则是用于存储关联对象的数据结构（key： void \* value: ObjcAssociation）
* 每一个对象都有一个标记位 has_assoc 指示对象是否含有关联对象(isa_t 结构体中的has_assoc)

## 具体使用

```
#import <objc/runtime.h>

@implementation DKObject (Category)

- (NSString *)categoryProperty {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)setCategoryProperty:(NSString *)categoryProperty {
    objc_setAssociatedObject(self, @selector(categoryProperty), categoryProperty, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```

>这里的 _cmd 代指当前方法的选择子，也就是 @selector(categoryProperty)

### objc_AssociationPolicy
	
| objc_AssociationPolicy | modifier |
| --- | --- |
| OBJC_ASSOCIATION_ASSIGN | assign |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC | nonatomic, strong |
| OBJC_ASSOCIATION_COPY_NONATOMIC | nonatomic, copy |
| OBJC_ASSOCIATION_RETAIN | atomic, strong  |
| OBJC_ASSOCIATION_COPY | atomic, copy |

## 为什么分类添加属性不会自动生成实例变量和getset方法

答：**因为类的内存布局是在编译期决定的，实例变量的布局已经固定**，使用 @property 已经无法向固定的布局中添加新的实例变量。所以我们需要使用关联对象以及两个方法来模拟构成属性的三个要素。


## 相关链接