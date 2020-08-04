### Tokenizer

#### 1. Python中的缩进

因为Python中的代码块是根据缩进来区分的，所以尽量减少不必要的缩进：

```python
if ch in self.dicts:
    return self.dicts[ch]
else:
    if ch == 'n':
        return self.readNull()
    elif ch == 't' or ch == 'f':
        return self.readBoolean()
    elif ch == '"':
        return self.readString()
    elif ch == '-' or self.isDigit(ch):
        return self.readNumber()
```

虽然这里自己是想把一些特殊符号和其它的分开来，但最好还是让其进行平行的判断，所以将其改为如下：

```python
if ch in self.dicts:
    return self.dicts[ch]
elif ch == 'n':
    return self.readNull()
elif ch == 't' or ch == 'f':
    return self.readBoolean()
elif ch == '"':
    return self.readString()
elif ch == '-' or self.isDigit(ch):
    return self.readNumber()
```

#### 2. 字符串拼接

和Java中一样，因为Python中的字符串也是不可变的，所以在拼接的时候使用`join`方法会比操作符`+`效率更高些（每次的+操作都会生成一个新的字符串）。

- 使用`+`运算符时的运行效率

```
times = 100
Running time: 7.8312558 Seconds
Average time: 0.078312558 Seconds

times = 500
Running time: 39.1252378 Seconds
Average time: 0.0782504756 Seconds
```

- 使用`join`方法时的运行效率

```
times = 100
Running time: 8.6673024 Seconds
Average time: 0.086673024 Seconds

times = 500
Running time: 41.5727631 Seconds
Average time: 0.0831455262 Seconds
```



#### 3. 多个or判断条件

当一个值可能会为多种i情况时，我们不要使用`or`运算符来连接多个`==`，而是直接使用`in`运算符，并将所有可能的值放到一个list中。例如下面的情况：

```python
return ch == '"' or ch == '\\' or ch == 'u' or ch == 'r' \
               or ch == 'n' or ch == 'b' or ch == 't' or ch == 'f'
```

比较影响可读性，而如果改用`in`运算符，可读性会好很多：

```python
return ch in ['"', '\\', 'u', 'r', 'n', 'b', 't', 'f']
```

#### 4. 函数的闭包

要注意设计一个函数的时候，应该让函数有描述清晰的返回值，而不能莫名奇妙的在函数内部进行死循环等。

```python
def readString(self):
    string = ''
    while True:  # while True应保证程序一定在某种情况下可以退出循环
        ch = self.charReader.nextChar()
        if ch == '\\':
            #...
        elif ch == '"':  # 遇到了字符串的终止位置
            return Token(TokenType.STRING, string)
        elif ch == '\r' or ch == '\n':  # 字符串中间不能出现这些
            raise Exception("Invalid character")
        else:  # 说明是个普通的字符
            string = string + ch
```

可以看到上面的代码中使用循环的条件是`while True`，假设charReader已经读到字符串末尾且没有发现要读字符串的另一个引号，那么会返回`'-1'`，但并没有对`'-1'`进行特定的判断，因而总是会返回`'-1'`并造成死循环。

所以首先我们的`while True`应该改为`while self.charReader.hasNext()`，保证最多只会读取到原字符串的末尾。 并在循环结束后抛出一个异常，因为循环结束说明到末尾也没有遇到要读字符串的另一个引号。

```python
def readString(self):
    string = ''
    while self.charReader.hasNext():
        ch = self.charReader.nextChar()
        if ch == '\\':
            #...
        elif ch == '"':  # 遇到了字符串的终止位置
            return Token(TokenType.STRING, string)
        elif ch == '\r' or ch == '\n':  # 字符串中间不能出现这些
            raise Exception("Invalid character")
        else:  # 说明是个普通的字符
            string = ''.join([string, ch])
    # 运行到这里说明直到结束也没有找到另一个引号
    raise Exception("String object lacks close quotation!")
```

同样的，在我们`if`条件没有触发的时候，那就会运行到函数的末尾，因而不需要额外的`else`子句，只需要直接抛出异常即可。

```python
def readBoolean(self):
    if self.charReader.peek() == 't' :
        if self.charReader.nextChar() == 'r' and self.charReader.nextChar() == 'u' and self.charReader.nextChar() == 'e':
            return Token(TokenType.BOOLEAN, True)
        else:
            raise Exception("Invalid json string")
    else:
        if self.charReader.nextChar() == 'a' and self.charReader.nextChar() == 'l'\
                and self.charReader.nextChar() == 's' and self.charReader.nextChar() == 'e':
            return Token(TokenType.BOOLEAN, False)
        else:
            raise Exception("Invalid json string")
```

其实这个函数就只有两种两种情况返回一个Token，为字符串`"true"`或者`"false"`，其余的直接抛出异常就可以了：

```python
def readBoolean(self):
    if self.charReader.peek() == 't' :
        if self.charReader.nextChar() == 'r' and self.charReader.nextChar() == 'u' and self.charReader.nextChar() == 'e':
            return Token(TokenType.BOOLEAN, True)
    else:
        if self.charReader.nextChar() == 'a' and self.charReader.nextChar() == 'l'\
                and self.charReader.nextChar() == 's' and self.charReader.nextChar() == 'e':
            return Token(TokenType.BOOLEAN, False)
    raise Exception("Invalid json string")
```

但能看到上面的函数有点长，如果要判断一个字符串是个很长的呢，那岂不是要很多个`and`进行连接。所以我们可以把这部分抽离出来。因为这里我们根据第一个字符就可以知道后面要判断的**长度len**和**值val**，那么可以直接取出后面len个字符组成的字符串，并和val进行比较即可。

```python
def readBoolean(self):
    if self.charReader.peek() == 't':
        if getBoolean("rue"):
            return Token(TokenType.BOOLEAN, True)
    else:
        if getBoolean("alse"):
            return Token(TokenType.BOOLEAN, False)
    raise Exception("Invalid json string")

# 判断剩余的字符串和charReader里面的是否一样
def getBoolean(self, remain_str):
    string = ''
    for i in range(len(remain_str)):
        string = ''.join([string, self.charReader.nextChar()])
    return string == remain_str
```

### Parser