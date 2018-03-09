---
layout: post
title: "Why you shouldn't use one IV for Cipher Block Chaining (CBC)?"
permalink: /blog/2018/3/9/not-use-one-iv-for-cbc
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2018-3-9-do-not-use-cbc-with-same-iv.md"
excerpt: "One can use different algorithm types and modes for encryption. One of the famous cryptographic modes is Cipher Block Chaining or CBC..."
---
Suppose we're gonna use *AES* with *CBC* mode for our encryption needs. This is how our first attempt looks like in Kotlin:
{% highlight kotlin %}
val iv = // Generate a static IV
val key = // The AES key
val cipher = Cipher.getInstance("AES/CBC/PKCS5Padding")
cipher.init(Cipher.ENCRYPT_MODE, key, iv)

val plainText = "Don't do this".toByteArray()
val cipherText = cipher.doFinal(plainText)
{% endhighlight %}
Something is seriously wrong with this code and over the course of this post, we're going to find out what!

## Cipher Types
---
There are two basic types of symmetric algorithms: **Block Ciphers** and **Stream Ciphers**. A Block cipher operates on blocks of plaintext for encryption and ciphertext for decryption. Stream ciphers operate on streams of plaintext and ciphertext one bit or byte at a time. 

>Usually, with a block cipher, the same plaintext block will always encrypt to the same ciphertext block, using the same key. With a stream cipher, the same plaintext bit or byte will encrypt to a different bit or byte every time it is encrypted.

## Cipher Block Chaining (CBC)
---
CBC is a very common cryptographic mode which uses a technique called **Chaining** in order to make sure that same plaintext block will generate a
different ciphertext block. Chaining will add a *feedback* to the block cipher: The result of the encryption of previous block will be passed to the encryption of current block. To be more specific, before we encrypt the current plaintext block, we first XOR the current plaintext block with previous ciphertext block and then encrypt the result. With chaining, each block will modify the encryption of the next block, hence the encryption of each block depends on all the previous blocks. Mathematically speaking, encryption can be represented as following:
<center>$$ \mathbf{C}_{i} = \mathbf{E}_{k}(\mathbf{P}_{i} \oplus \mathbf{C}_{i-1}) $$</center>
$$ \mathbf{C}_{i} $$ is the ciphertext for $$ i $$th block, $$ \mathbf{E}_{k} $$ encapsulates the encryption function with $$ k $$ as the key and $$ \mathbf{P}_{i} $$ is the $$ i $$th block of plaintext.

The decryption process is as simple as the encryption:
<p>$$ \mathbf{C}_{i} = \mathbf{E}_{k}(\mathbf{P}_{i} \oplus \mathbf{C}_{i-1}) \implies  \mathbf{D}_{k}(\mathbf{C}_{i}) = \mathbf{D}_{k}(\mathbf{E}_{k}(\mathbf{P}_{i} \oplus \mathbf{C}_{i-1})) = \mathbf{P}_{i} \oplus \mathbf{C}_{i-1} $$ </p>
Hence, decryption can be represented as:
<center>$$ \mathbf{P}_{i} = \mathbf{C}_{i-1} \oplus \mathbf{D}_{k}(\mathbf{C}_{i}) $$</center>

### Trouble in Paradise
---
In its simples form, CBC would generate different ciphertexts for the same plaintext if and only if the previous blocks of those plaintexts are different. Consequently, two completely identical plaintexts would encrypt to the same ciphertext. And to make matters even worse, two message with identical beginnings would have identical beginnings in their ciphertexts, too.

In other words, patterns in the plaintext wouldn't disappear in the ciphertext and this information can be very useful for a seasoned cryptanalyst. 

### Initialization Vector (IV)
---
We can prevent this by encrypting a random data as the first block (and before the actual first block). This block of random data is called **Initialization Vector** or IV for short. 

With the addition of IVs, same plaintext messages encrypt to different ciphertext messages, because the IVs used for those scenarios are (hopefully) different.

### One IV to rule them all
---
By now, you probably can spot the problem in that code. We were using a **static IV for all encryptions/decryptions**. Since we were using an IV, we probably wanted to same plaintexts get encrypted to different ciphertexts. And on the other hand, by using the same IV over and over again, we've defeated this goal spectacularly! As a matter of fact, using a static IV is not any different with not using an IV at all.

### How to maintain multiple IVs
---
Don't worry, The IV need not be secret; it can be transmitted in the clear with the ciphertext. Quoting from *[Applied Cryptography](https://www.amazon.com/Applied-Cryptography-Protocols-Algorithms-Source/dp/0471117099)* by *Bruce Schneier*:

Assume that we have a message of several blocks: $$ \mathbf{B}_{1}, \mathbf{B}_{2}, . . ., \mathbf{B}_{i} $$. $$ \mathbf{B}_{1} $$ is encrypted with the $$ IV $$. $$ \mathbf{B}_{2} $$ is encrypted using the ciphertext of $$ \mathbf{B}_{1} $$ as the $$ IV $$. $$ \mathbf{B}_{3} $$ is encrypted using the ciphertext of $$ \mathbf{B}_{2} $$ as the $$ IV $$, and so on. So, if there are *n* blocks, there are *n−1* exposed $$ IVs $$, even if the original $$ IV $$ is kept secret. So there's no reason to keep the $$ IV $$ secret; the $$ IV $$ is just a dummy ciphertext block—you can think of it as $$ \mathbf{B}_{0} $$ to start the chaining.

