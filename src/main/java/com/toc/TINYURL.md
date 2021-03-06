<a name="index">**Index**</a>

<a href="#0">短连接设计</a>  
&emsp;<a href="#1">1. 确定问题的范围</a>  
&emsp;<a href="#2">2. 短链接设计</a>  
&emsp;&emsp;<a href="#3">2.1. 使是否使用数据库</a>  
&emsp;&emsp;<a href="#4">2.2. 确定短连接长度</a>  
&emsp;&emsp;<a href="#5">2.3. 方案一: 摘要算法</a>  
&emsp;&emsp;&emsp;<a href="#6">2.3.1. 单使用md5加密</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#7">2.3.1.1. hash冲突</a>  
&emsp;&emsp;&emsp;<a href="#8">2.3.2. 混合使用MD5 与 base 62</a>  
&emsp;&emsp;<a href="#9">2.4. 方案二:使用UUID的发号器方案</a>  
&emsp;&emsp;<a href="#10">2.5. 数据库映射</a>  
&emsp;<a href="#11">3. 短链接跳转，301还是302重定向</a>  
&emsp;<a href="#12">4. 数据库设计</a>  
&emsp;&emsp;<a href="#13">4.1. 分库分表</a>  
&emsp;&emsp;<a href="#14">4.2. 读写分离</a>  
&emsp;<a href="#15">5. 预防攻击</a>  
# <a name="0">短连接设计</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

相关文章：
- https://github.com/soulmachine/system-design/blob/master/cn/tinyurl.md
- https://github.com/xitu/system-design-primer/blob/translation/solutions/system_design/pastebin/README-zh-Hans.md
- https://www.cnblogs.com/rickiyang/p/12178644.html
- https://www.zhihu.com/question/20103344/answer/573638467
- https://segmentfault.com/a/1190000006140476

## <a name="1">确定问题的范围</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
1. 数据量，增长数据量。
2. 链接过期时间设置。
3. 使用数据库或不使用数据库保存。


## <a name="2">短链接设计</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
### <a name="3">使是否使用数据库</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
使用数据库，可以收集数据，进而再进行数据分析。而不适用数据库纯粹只是生成短连接服务。

### <a name="4">确定短连接长度</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
1. 使用大小写字母+数字的62进制方案，取6~8位便可以支持上亿的不同短连接

62^6 = 568,00235584
62^7 = 35216,14606208

2. 使用32进制小写字母+数字 36位
36^8 = 3w亿


### <a name="5">方案一: 摘要算法</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

#### <a name="6">单使用md5加密</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
使用MD5 摘要算法进行长URL加密(分为16位和32位)，获取前8个字符，当成短连接地址。
##### <a name="7">hash冲突</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
1. 加密后，必定会出现hash冲突的情况，可以对原有的url进行加减字符重新加密。或者对于加密后的md5再次md5加密。
2. 使用url+时间戳生成，生成失败后获取新的时间戳生成。

#### <a name="8">混合使用MD5 与 base 62</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
1. 使用 MD5 来哈希用户的 IP 地址 + 时间戳
    - MD5 是一个普遍用来生成一个 128-bit 长度的哈希值的一种哈希方法
2. 用 Base 62 编码 MD5 哈希值
    - 对于 urls，使用 Base 62 编码 [a-zA-Z0-9] 是比较合适的, 对于每一个原始输入只会有一个 hash 结果，Base 62 是确定的（不涉及随机性）
> Base 64 是另外一个流行的编码方案，但是对于 urls，会因为额外的 + 和 - 字符串而产生一些问题


### <a name="9">方案二:使用UUID的发号器方案</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

使用如雪花主键的UUID当成长链接对应的键，


### <a name="10">数据库映射</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
加密后，存储数据库根据数据纬度讨论长链接与短链接的映射关系。
1. 若长链接需要区分分享的用户，地点，时间等信息，长链接与短链接以1对多的方式存储。可用于后序的数据分析。
2. 若长连接无需区分用户的类型，可直接公用相同的短链接。如何建立映射关系可见下方：
- 可选的方案为，使用LRU或者Redis带过期时间的Hash，来进行数据映射。


## <a name="11">短链接跳转，301还是302重定向</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
这个问题主要是考察你对301和302的理解，以及**浏览器缓存**机制的理解。

301是永久重定向，302是临时重定向。短地址一经生成就不会变化，所以用301是符合http语义的。但是如果用了301， Google，百度等搜索引擎，搜索的时候会直接展示**真实地址**(即短链接对应的长连接地址，浏览器缓存)，那我们就无法统计到短地址被点击的次数了，也无法收集用户的Cookie, User Agent 等信息，这些信息可以用来做很多有意思的大数据分析，也是短网址服务商的主要盈利来源。

所以，正确答案是302重定向。


## <a name="12">数据库设计</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

使用shortlink 当成组件
```
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

### <a name="13">分库分表</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
由于短链接可直接做进制转换，并具有唯一性

使用短连接作为sharding key，进行水平分库拆分。

另外可以基于一季度做数据归档。

### <a name="14">读写分离</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>


## <a name="15">预防攻击</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>
1. 频繁生成短链接的 使用ip黑名单屏蔽。
2. 使用redis建立长连接->ID 的缓存。