# print

python字符串中，单引号和双引号是等价的

```python
# 转义
print("abc") # abc
print("i'm ok") # i'm ok
print("i\'m really \"OK\"") #i'm really "OK"

# 被 r'' 包裹的字符不转义fam\"'"'''''"'" 
print(r' fam\"'"'''''"'" ')

# 多行内容
print('''0
a
b
c
d''')


print('abc'*3)
# abcabcabc


# 字符串格式化
a=3
b="ur"
s = '{}==={}'.format(a,b)
# s: 3===ur
s = '{1}==={0}.format(a,b)
# s: ur===3
print(f'{a}==={b}')
# s: 3===ur
s = '{color} + {n} + {x}'.format(n=10, x=3.14, color="blue")
# blue + 10 + 3.14

```

# string

```python
# 字符串拼接
print("hello"+"45")
s = "you"+"me"
print(s[0])    #y
print(s[-1])    #e
print(s[0:3])   #you
n = len(s) #5

line = "1 2 3 4 5"
numbers = line.split()
# numbers ['1','2','3','4','5']

line = "1,2,3,4,5"
numbers = line.split(',')
# numbers ['1','2','3','4','5']

s = ','
p=s.join(numbers)
# p='1,2,3,4,5'

# 替换
s = "hello bob"
t = s.replace('bob','alice')

# 大小写
t = s.upper()
t = s.lower()

# 去除空格
s = "   a   "
t = s.lstrip()
t = s.rstrip()
t = s.strip()

# 在多行中写一行数据
a = ("are"
	 "you"
	 "ok?")
b = "thank" \
    "you"

# a:  are you ok?
# b:  thank you
```

# list
python使用[]生成列表
```python
a = [1.2.4,'hi',8+2.3]
# [1,2.4,'hi',10.3]

b = a+a
# b:[1,2.4,'hi',10.3,1,2.4,'hi',10.3]

b.append('www')
# b:[1,2.4,'hi',10.3,1,2.4,'hi',10.3,'www']
```
list中的值可以通过其分片修改
```python
a = [10, 11, 12, 13, 14]
a[0] = 100
print a
# [100, 11, 12, 13, 14]

a = [10, 11, 12, 13, 14]
a[1:3] = [1, 2, 3, 4]
print a
# [10, 1, 2, 3, 4, 13, 14]

a = [10, 1, 2, 11, 12]
a[1:3] = []
print a
# [10, 11, 12]

# 删除元素
a = [1002, 'a', 'b', 'c']
del a[0]
del a[1:]
del a[::2]

# 判断元素是否存在
print(1002 in a)
# True

# 统计元素数量
a = [1,3,4,5,6,7,8,9,0,2,2,2,12]
cnt = a.count(2)

# 搜索
i = a.index(12)   # 不存在时报错

# 插入元素
a.insert(1,"112")

# 移除某个元素，移除第一个出现的某个元素，不包含时报错
a.remove(4)

# 排序
a.sort()
b = sorted(a)

# 反向
a.reverse()
b = a[::-1]



```

# Set
python 使用{}生成 set
```python
s={1,3,2,4,4,4,4}
# s = {1,2,3,4}

s.add(1)
# s = {1,2,3,4}

b={4,5,6}
c=a&b
# c={4}
c=a|b
# c={1,2,3,4,5,6}

c=a-b
# c={1,2,3}

c=b-a
# c={5,6}

c=a^b   # a|b-a&b
# c={1,2,3,5,6}
```

# Dictionary

python使用{key:value}生成 Dictionary
key 必须是不可变类型

```python
a = {}
b = dict()

d={'a':1,'b':2}

d['a']=6
d['c']=56

e=d.keys()
# e=['a','b','c']

e=d.values()
# e=[1,2,6,56]

# 遍历 dict
for k,v in d.items():  
    print(f"Key:{k}, Value:{v}")

# get方法
a = {}
a["hi"] = "ih"
print(a.get("hihi", "oo"))

# 删除元素
a["oi"] = "io"
a.pop("oi")
del a["hi"]


```

# 整数进制转换

```python
# 十六进制
print(hex(255))
# 0xff

print(oct(255))
# 0377

print(bin(255))
# 0b11111111

print(int('1123'))
# 1113

# 将 16 进制的 FF 转换为整数
print(int('FF'), 16)

# 将 9 进制的 1874 转换为整数
print(int('1874'), 9)

```

# 分片

```python
# 左闭右开
s = "absfoiashnfioa"
p = s[3:6]
# foi
p = s[-6]
# h
p = s[::3]
# 每隔三个取一个，afano
p = s[::-5]
# ahf

```

# 元组

与列表相似，元组`Tuple`也是个有序序列，但是元组是不可变的，用`()`生成。
