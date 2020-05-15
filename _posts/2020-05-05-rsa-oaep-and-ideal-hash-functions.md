---
layout: post
title: "RSA-OAEP and Ideal Hash Functions"
author: "Tudor A. A. Soroceanu"
categories: math
tags: [math,public-key-cryptography,RSA,random-oracle]
image: roll-the-dice.jpg
---
{% assign fig_counter = 1 %}

I tutor the lecture about cryptanalysis at Freie Universität Berlin. This week I had a discussion about the RSA-OAEP padding and the used hash functions. The question was why the proof for the semantic security needs *ideal* hash functions. The short answer is the random oracle model. The long answer can be found in this post.

## Random Oracle Model

One of the core cryptographic primitives are [hash functions](https://en.wikipedia.org/wiki/Cryptographic_hash_function).
They map an (arbitrarily) large input to an output of fixed length. When used in cryptography hash functions are required to have additional properties. Some important ones amongst other things are: the mapping should be a one-way function (i.e. easy to compute, but the inversion should be infeasible to compute), the output should look kind of random,  and it should be hard to find two different inputs mapping to the same output.

When trying to find security proofs for systems using hash functions, cryptographers found it difficult to use real world hash functions, because their behavior is not truly random. Instead, they thought of an *ideal* hash function to use in the proofs. As it turns out, [random functions](https://en.wikipedia.org/wiki/Stochastic_process) posess all the desired properties of *ideal* hash functions.
A random function takes an arbitrarily large input and outputs a truly random value of a fixed size. When getting the same input query again, the function returns the same response as before. In this regard, random functions are consistent.
The problem with random functions is that they cannot be implemented because they require too much space and are too slow to compute[^slow_rand_func].

Nevertheless, random functions are used in a computational model called **random oracle modell** *(ROM)*. In the *ROM* every call to a hash function is substituted with a call to a *random oralce* using a completely secure channel. The random oracle then responds with a true random value and stores this input-output pair for future calls. Also, the random oracle does not need any time to lookup an already stored value or to generate a new random value.

Now cryptographers can perform their security analysis using the *ROM* and do not have to worry about possible flaws of instances of real world hash functions.
When implementing the proven scheme, the random function is replaced with a real implementation of a "good cryptographic hash function".

Unfortunately, [security proofs in the *ROM* have no security implications](https://arxiv.org/pdf/cs/0010019v1.pdf) for the standard model. However, they can indicate flaws in the investigated scheme that are not correlated to the use of hash functions.

In the remainder of the post, I'll outline the IND-CPA proofs for RSA-OAEP both in the *ROM* and the standard model. And I'll explain where the proof in *ROM* breaks when we don't use *ideal* hash functions.

## Semantic Security of RSA-OAEP

[Semantic security](https://en.wikipedia.org/wiki/Semantic_security) of cryptographic schemes indicates the ability of an adversary to obtain any kind of information about a plaintext, given the corresponding ciphertext.
In the case of asymmetric key encryption schemes, one of the semantic security properties is the indistinguishability under chosen-plaintext attack (IND-CPA).
The IND-CPA security is defined by the following game, in which the adversary is modeled as a Turing machine with a polynomial amount of computation time.
Here $\mathcal{E}_{pk}(m)$ represents the encryption of the message $m$ using the encryption scheme $\mathcal{E}$ with the public key $pk$. In the setting of the *ROM*, the challenger is modeled as a random oracle.

### The IND-CPA Game

1. The challenger generates a key pair $(pk,sk)$ and sends it to the adversary.
2. The adversary can perform a polynomial number of computations and chooses then two messages $m_0,m_1$ of same length, that are send to the challenger.
3. The challenger encrypts randomly one of the two messages and sends the ciphertext back to the adversary.
4. After another round of computations, the adversary must decide which message was encrypted.

Here is the IND-CPA game as table:

| Adversary  |                        | Random Oracle                 |
|:----------:|:----------------------:|:-----------------------------:|
|            | $\leftarrow (pk,sk)$   | generate key pair $(pk,sk)$   |
|choose messages $m_0,m_1$|$m_0,m_1\rightarrow$|choose random $$b\in \{ 0,1 \}$$|
|perform computations and choose $$b' \in \{0,1\}$$|$\leftarrow c$|$c=\mathcal{E}_{pk}(m_b)$|
|output $b'$|||

The adversary wins the game, if she chooses the right message, i.e. $b=b'$.
If she cannot decide the game with significant margin from 0.5%
(just picking randomly a message), the scheme is considered IND-CPA secure.
In the *ROM* setting the adversary can perform random oracle queries during her computation phases. This is used to mimic calls to a hash function.

### RSA-OAEP

The [RSA](http://people.csail.mit.edu/rivest/Rsapaper.pdf) public-key cryptosystem is still one of the most used schemes and is taught in basically every course about public-key cryptography due to its mathematical simplicity.
Since the actual math is not important for understanding the IND-CPA proofs, we will assume that RSA consists of a key generation function $\mathcal{K}$ that produces public-secret-key-pairs $(pk,sk)$, an encryption function $\mathcal{E}$, and a decryption function $\mathcal{D}$, such that

$$\mathcal{D_{sk}}(\mathcal{E_{pk}}(m))=m$$

for all valid messages $m$. Additionally, we just assume that the decryption without the key is computationally infeasible. This way, RSA is a [trapdoor function](https://en.wikipedia.org/wiki/Trapdoor_function).

Using the [IND-CPA game above](#the-ind-cpa-game) one can show that RSA is not semantically secure without any randomized padding:
If an adversary gets a chiphertext $c$ from the IND-CPA game, she can simply compute $c_0=\mathcal{E}_{pk}(m_0)$ and $$c_1=\mathcal{E}_{pk}(m_1)$$ and check if $$c=c_0$$ or $$c=c_1$$.

Therefore, the [*Optimal Asymmetric Encryption Padding* (OAEP)](https://link.springer.com/content/pdf/10.1007%2FBFb0053428.pdf) was introduced by Bellare and Rogaway.
In OAEP two hash functions $G$ and $H$ are used to obfuscate the message additionally with a random value $r$. This random value is used to compute $X = m \oplus G(r)$ and $Y=r \oplus H(X)$.
The concatenation $X||Y$ is then encrypted using RSA.
Figure 1 shows the schematic construction of RSA-OAEP.

{% include image.html url="https://upload.wikimedia.org/wikipedia/commons/1/18/Oaep-diagram-20080305.png" description="Ozga at English Wikipedia / CC BY-SA (https://creativecommons.org/licenses/by-sa/3.0)" caption="Schematic representation of OAEP." counter=fig_counter %}
{% assign fig_counter = fig_counter | plus: 1 %}

### IND-CPA Security of RSA-OAEP in the *ROM*

This is a high-level outline of the IND-CPA proof, for more details [see the original paper by Bellare and Rogaway](https://link.springer.com/content/pdf/10.1007%2FBFb0053428.pdf).

To show that RSA-OAEP is IND-CPA secure in the *ROM*, we initialize an IND-CPA game.
We can use random oracles, so all the calls of the hash functions $G$ and $H$ are replaced with queries to the random oracle, respectively the *ideal* hash function.
Since the hash functions now output perfect random values in our model, the encryption that the adversary gets is the encryption of a perfectly random text $X||Y$.
Assuming that RSA is a trapdoor function and has no vulnerabilities, the adversary can only try to guess the outputs of the *ideal* hash functions.
For each guessed output she can perform an encryption with the public key and check if the output matches the encrypted message $c$ from the challenger.
Intuitively, one can see that the adversary only wins the IND-CPA game if she can guess the outputs of the hash functions correctly.
Now, if these outputs are truly random, she has a negligible chance in winning this game.

We have a problem if the hash functions are not *ideal* and the output is not a real random value.
Then the adversary could have a better grasp at guessing some values.
Or the hash function could interfere with the encryption/trapdoor function.
The [Bleichenbacher attack on RSA](https://link.springer.com/content/pdf/10.1007%2FBFb0055716.pdf) for example exploits regularities in the plaintext introduced by the padding.

So, like always with the *ROM*, we cannot draw any conclusions for real world applications of RSA-OAEP regarding hash functions.
But luckily, Kiltz, O'Neill and Smith showed that the random oracle is not necessary to prove the IND-CPA security of RSA-OAEP.

### IND-CPA Security of RSA-OAEP in the Standard Model

In their [paper](https://link.springer.com/content/pdf/10.1007%2F978-3-642-14623-7_16.pdf), Kiltz, O'Neill and Smith prove the IND-CPA security without a random oracle.
But this complicates the proof by a lot. We will just outline the main ideas of the proof.

First, they show that *lossy* trapdoor functions (LTDFs), if correctly used with a padding, don't need the random oracle in order to prove IND-CPA security.
Lossy means that given a ciphertext $c$ and a public key $pk$ for RSA, it is not possible to check if $c$ was encrypted using the $pk$ or if another key $pk'$ was used.
Like the [IND-CPA game](#the-ind-cpa-game) the lossy property can also be modeled as a game.

Second, they show that OAEP is a suitable padding scheme, such that it fools distinguishers, without assuming any restrictions for the hash functions.

In the last step RSA-OAEP is proven to be a LTDF. And an adversary for the IND-CPA game cannot be more successful than an adversary for the LTDF game.
Since the adversary for the LTDF game in the standard model has only a negligible advantage, RSA-OAEP is also IND-CPA secure in the standard model.

## Conclusion

Returning to the initial question, we saw where and why *ideal* hash functions are used.
They are a method to overcome the limitations of real world hash functions and allow therefore for simpler security proof in the so called *random oracle model*.
Sadly, security proofs in the *ROM* do not have any direct security implications for the real world, but allow for a first check of a scheme.
As outlined in the last section, security proofs in the standard model are much harder to achieve.
The future upcoming of quantum computers complicates this matter further, since now also [adversaries that can query the random oracle with quantum computers](https://eprint.iacr.org/2010/428.pdf) must be considered. This makes security proofs already in the (quantum-)random oracle model more difficult.

---

[^slow_rand_func]: The only way to implement random functions is to store a huge lookup table. In order to find the right entry a Turing machine would need more time that we have left to live.
