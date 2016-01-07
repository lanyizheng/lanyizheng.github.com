##1.说明
reduce:将一个可以迭代的对象应用到两个带有参数的方法上，我们称这个方法为fun,遍历这个可迭代的对象，将其中元素依次作为fun的参数，但是这个函数有两个参数，那些作为参数呢？

```python
reduce(fun,sequence[,initial_val])
```
reduce函数有三个参数，第一个参数就是作用函数，第二个函数就是可迭代的对象，第三个是迭代初始值。
如果存在第三个参数，也就是有初始迭代对象，那么 initial_val作为fun函数的第一个参数， sequence 的第一个元素作为fun的第二个参数，得到返回结果的作为下一次函数的第一个参数，sequence的第二个参数作为下一次迭代过程中的第二个参数，以此类推。
如果不存在第三个参数，那么sequence的第一个参数作为fun函数的第一个参数，sequence的第二个参数作为fun函数第二个参数，以此类推。
##2.例子
下面有几个例子：

```python
reduce(lambda x,y:x+y,[1,2,3,4,5])
#计算1到5的和
```
下面是一个统计词频的例子：

```python
str="an apple a banana three apple a desk"
list=str.split(' ')
def fun(x,y):
	if y in x:
		x[y]=x[y]+1
	else:
		x[y]=1
	return x
result=reduce(fun,list,{})
#输出结果是
>>>{'a': 2, 'apple': 2, 'three': 1, 'an': 1, 'desk': 1, 'banana': 1}
```


