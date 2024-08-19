# Cryptography land mines
## Symmetric algorithm keys

The rule of thumb in cryptography says never to invent the wheel. In practical terms that means two things. First, one should never invent their own cryptoalgorithm. Anyway, it will be (almost certainly) worse than existing ones, and even worse, you will not even know that. Second, one should never implement already known algorithms but use crypto libraries that are widely available and proven to be secure. So, we should use existing wheels.

But can we use them properly?
```javascript
import { randomBytes, createHash, createCipheriv } from 'crypto';

const secret = Buffer.from("tzJ3EK4gtTDM9TeZX35Z3ZWzGsDVWxmn");

function encrypt(text, secret) {
    const iv = randomBytes(16);
    const key = createHash('sha256').update(secret).digest();
    
    const cipher = createCipheriv('aes-256-cbc', key, iv);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return {
        iv: iv.toString('hex'),
        content: encrypted
    };
}

console.log(encrypt("The quick brown fox jumps over the lazy dog", secret));
```

Above is the piece of source code (stripped and somewhat changed) that I stumbled upon not so long ago. Let's have a closer look at how it is structured and what it does. The first thing to mention is that this code was meant to be a part of the automation procedure, and a secret was going to be pulled from a secrets manager (here the secret is hardcoded for the sake of simplicity, otherwise a hardcoded secret is a good way to lose your bonus). An encryption function accepts the secret and a text. It generates an input vector, hashes the secret to produce an encryption key, does the encryption using the standard crypto library and splits out the result. 
![](image%201.png)

All neat. If you run the code, it beautifully returns the correct values. Let's check. To verify that the result is correct (and consequently, the code is correct), I will use the [CyberChef](https://gchq.github.io/CyberChef/) tool. Here is the link for the [recipe](https://gchq.github.io/CyberChef/#recipe=Register%28'%5Eiv:%28.*%29$',true,true,false%29Register%28'%5Emessage:%28.*%29$',true,true,false%29Regular_expression%28'User%20defined','%5Ekey:%28.*%29$',true,true,false,false,false,false,'List%20capture%20groups'%29SHA2%28'256',64,160%29Register%28'%28%5B%5C%5Cs%5C%5CS%5D*%29',true,false,false%29Find_/_Replace%28%7B'option':'Regex','string':'.*'%7D,'$R1',false,false,false,false%29AES_Decrypt%28%7B'option':'Hex','string':'$R2'%7D,%7B'option':'Hex','string':'$R0'%7D,'CBC','Hex','Raw',%7B'option':'Hex','string':''%7D,%7B'option':'Hex','string':''%7D%29&input=a2V5OnR6SjNFSzRndFRETTlUZVpYMzVaM1pXekdzRFZXeG1uCml2OjA4NDVmNGY0YWRmMDRmNDI1OGQ5YTU4N2QyNzY4M2ViCm1lc3NhZ2U6MmQwNDQwNDUwYWEwMzQxYTIxZmM3Njg4MjRhYzYwYWIyMTM3YTNjMDkwYzI5Y2I0ZTE0OWUxOTlhNDUzNjFjN2EzNTA2ODgyY2E2Nzc4NDE2MWU3MmVmZGM5MDRjMGJl&oeol=CR)
![](image%202.png)

But... there would not be this article if something wasn't wrong. First, let's revise some theory. 
> The Advanced Encryption Standard (AES), ... is a specification for the encryption of electronic data established by the U.S. National Institute of Standards and Technology (NIST) in 2001. For AES, NIST selected three members of the Rijndael family, each with a block size of 128 bits, but three different key lengths: 128, 192 and 256 bits. [[Advanced Encryption Standard - Wikipedia](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)]

So, in simple words, AES-256 is an encryption algorithm, that can process blocks of 128 bits using a 256-bit key. And another piece of theory.
> Kerckhoffs's principle holds that a cryptosystem should be secure, even if everything about the system, except the key, is public knowledge. Combining these two pieces, we can state that for our cryptosystem to be secure, we must use strong keys (secrets). [[Kerckhoffs's principle - Wikipedia](https://en.wikipedia.org/wiki/Kerckhoffs%27s_principle)]

Everyone knows that, right?

Now let's have a look at the secret that we use. We can see that it is random, but unfortunately not random enough. In our case, the secret is a set of 32 alphanumeric symbols, making it exactly 256 bit length. Nevertheless, given that those are alphanumeric symbols, they occupy only a small chunk of the ASCII table. Using this strategy, we restrict the possible value of each byte from 256 options (byte can have 2^8 values) to only 62 (26 uppercase letters + 26 lowercase letters + 10 numbers) cutting the strength of each byte four times, and the entire password - in 10^22 times. Experienced readers will notice, that the secret is hashed, effectively randomizing the key. So what is the problem? The problem is that the key remains a derivative of the initial secret not changing the strength of the secret material, since the potential attacker may know that the hashing is there (remember Kerckhoffs's principle - we must not rely on the system design, but only on the secret strength).

What to do? The solution is simple. The secret should be a random byte array, that is stored in hexadecimal or base64 form (since the majority of secrets management solutions expect strings as secret values) in a safe place (remember about bonuses).

To generate the secret you can use the following terminal commands:
on Linux
```bash
echo -n $(dd if=/dev/urandom bs=32 count=1 status=none) | sha256sum
```
on Mac
```bash
echo -n $(dd if=/dev/urandom bs=32 count=1 status=none) | shasum -a 256
```
or if you are on Windows (believe me, PowerShell generator is a bit wild), or if you are not a CLI type of person (which you need to be IMO) you can use this nodeJS snippet instead
```javascript
import { createHash, randomBytes } from 'crypto'

console.log(createHash('sha256').update(randomBytes(32)).digest("hex"));
```

Here is the example using the nodeJS snippet
![](image%205.png)

Eventually, the code will look like.
```javascript
import { randomBytes, createHash, createCipheriv } from 'crypto';

const secret = Buffer.from("6999e160ca53a3ccd18f0e3e7f4290dc44cdb2aeb9b0e30c07df90189d598a31", "hex")

function encrypt(text, secret) {
    const iv = randomBytes(16);
    const key = createHash('sha256').update(secret).digest()
    
    const cipher = createCipheriv('aes-256-cbc', key, iv);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return {
        iv: iv.toString('hex'),
        message: encrypted
    };
}

console.log(encrypt("The quick brown fox jumps over the lazy dog", secret))
```

But stop, why hashing is still in there? Practically it is redundant, but in theory, hashing improves the probability distribution rendering the final key more (better?) random. Effectively, from an attacker's standpoint of view, it does not matter what brute force the secret or the key value, since both are random. But hashing the secret makes your appsec engineer's heart feel better. However, you can remove the hashing if you have already hashed the secret before, for example in a terminal while generating one.
![](image%203.png)

Let’s do the [decryption](https://gchq.github.io/CyberChef/#recipe=Register%28'%5Eiv:%28.*%29$',true,true,false%29Register%28'%5Emessage:%28.*%29$',true,true,false%29Regular_expression%28'User%20defined','%5Ekey:%28.*%29$',true,true,false,false,false,false,'List%20capture%20groups'%29From_Hex%28'None'%29SHA2%28'256',64,160%29Register%28'%28%5B%5C%5Cs%5C%5CS%5D*%29',true,false,false%29Find_/_Replace%28%7B'option':'Regex','string':'.*'%7D,'$R1',false,false,false,false%29AES_Decrypt%28%7B'option':'Hex','string':'$R2'%7D,%7B'option':'Hex','string':'$R0'%7D,'CBC','Hex','Raw',%7B'option':'Hex','string':''%7D,%7B'option':'Hex','string':''%7D%29&input=a2V5OjY5OTllMTYwY2E1M2EzY2NkMThmMGUzZTdmNDI5MGRjNDRjZGIyYWViOWIwZTMwYzA3ZGY5MDE4OWQ1OThhMzEKaXY6YTU0ZmI2OTY3N2ZkMjY1M2U2NjhhNzdkNDg5YWYyOWYKbWVzc2FnZTo1OTBmZDVmNjhhOWNjNGRhNjgwYWY1MzExM2ZlNTJmZTNlYTA3YWE5NzU5NzIxMjk3MjRjZjRlNGYxZDY0ODg3NWRjMWI0ZjhlMTI4MDRjMDE0YzQ2NDc1YTA5ZWJmYTI&oeol=NEL). 
![](image%204.png)

Everything worked out as expected.

As for the conclusion, use the wheels correctly, and never be afraid to ask for help. Especially, when it comes to security matters.
