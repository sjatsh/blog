---
layout: post
title: "使用Spring事务时遇到的Session问题"
tag: "SpringBoot"
date: "2016-08-05 00:00:00 +0800"
categories: "技术"
---

#### 问题  

当开启事务时，我们需要更新一个表对象数据时，可能我们并没有所有的表对象的数据，这时我们需要先查看他的原始数据，然后更新数据到我们重新封装的实例对象中去。这时如果我们直接去save()或者update()这个对象就回报这样的错。

<!--more--> 

```
org.springframework.dao.DuplicateKeyException: A different object with the same identifier value was already associated with the session
```


#### 原因 

因为在查询对象数据的时候session中就已经保了这个对象，现在又要再次去更新或者保存这个对象，hibernate就会不知道到底哪一个才是我们需要去操作的对象。因为这时session中有了两个相同id但是不同的实体，这是我们需要先删除session中存在的对象。  


#### 删除session中缓存对象的方式：

##### 1.clear()(不推荐使用，会直接清空session，操作影响太大)

##### 2.evict()(定点删除session中的某个对象)

这个也是我自己的做法，这样删除影响最小。  

附上代码：
```
	//通过物料编号获取物料id
	String                 hql            = "from CGeneralMaterial cgm where cgm.materialNo='" + materialNo + "'";
	List<CGeneralMaterial> materialIdList = dao.executeQuery(hql);
	Integer                materialId     = materialIdList.get(0).getMaterialId();

	saveMaterial.setMaterialId(materialId);
	/**
	  * 这里是删除session中相同的对象
	  */
	session.evict(materialIdList.get(0));//少了这句就会报上面的错误
	dao.update(saveMaterial);
```



