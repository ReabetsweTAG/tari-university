# Introduction to Schnorr Signatures

- [Overview](#overview)
- [Let's get Started](#lets-get-started)
- [Basics of Schnorr Signatures](#basics-of-schnorr-signatures)
  - [Public and Private Keys](#public-and-private-keys)
  - [Creating a Signature](#creating-a-signature)
    - [Approach Taken](#approach-taken)
    - [Why do we Need the Nonce?](#why-do-we-need-the-nonce)
    - [ECDH](#ecdh)
- [Schnorr Signatures](#schnorr-signatures)
  - [So Why All the Fuss?](l#so-why-all-the-fuss)
  - [Naïve Signature Aggregation](#naïve-signature-aggregation)
  - [Key Cancellation Attack](#key-cancellation-attack)
  - [Better Approaches to Aggregation](#better-approaches-to-aggregation)
- [MuSig](#musig)
  - [MuSig Demonstration](#musig-demonstration)
  - [Security Demonstration](#security-demonstration)
- [References](#references)
- [Contributors](#contributors)

## Overview

Private-public key pairs are the cornerstone of much of the
cryptographic security underlying everything from secure web browsing to banking to cryptocurrencies. Private-public key pairs
are _asymmetric_. This means that given one of the numbers (the private key), it's possible to derive the other one 
(the public key). However, doing the reverse is not feasible. 
It's this asymmetry that allows one to share the public key, uh, publicly and be confident that no-one can
figure out our private key (which we keep very secret and secure).

Asymmetric key pairs are employed in two main applications: In _authentication_, where you prove that you have knowledge of the private
key; and _encryption_ where messages can be encoded and only the person possessing the private key can decrypt and read the message.

In this introduction on digital signatures, we'll be talking about a particular class of keys: those derived from 
elliptic curves. There are other asymmetric schemes, not least of which those based on products of prime numbers, 
including RSA keys. [[1]]

We're going to assume you know the basics of elliptic curve cryptography (ECC). If not, don't stress, there's a
[gentle introduction](../crypto-1/sources/PITCHME.link.md) in a previous chapter.

## Let's get Started

This is an interactive introduction to digital signatures. It uses Rust code to demonstrate some of 
the idea presented here, so you can see them at work. The code for this introduction makes use 
of the [libsecp256k-rs](https://github.com/tari-labs/libsecp256k1) library. 

That's a mouthful, but secp256k1 is the name of the elliptic curve that secures a *lot* of things in many
cryptocurrencies' transactions, including Bitcoin. 

This particular library has some nice features. We've overridden the `+` (addition) and `*` (multiplication)
operators so that the Rust code looks a lot more like mathematical formulae. This makes it much easier
to play with the ideas we'll be exploring.

**WARNING!** _Don't use this library in production code_. It hasn't been battle-hardened, so [use this one in
production instead](https://github.com/rust-bitcoin/rust-secp256k1).

## Basics of Schnorr Signatures

### Public and Private Keys

The first thing we'll do is create a public and private key from an elliptic curve.

On secp256k1, a private key is simply a scalar integer value between 0 and ~2<sup>256</sup>. That's roughly how many
atoms there are in the universe, so we have a big sandbox to play in.

We have a special point on the secp256k1 curve called _G_, that acts as the 'origin'. A public key is calculated by 
adding _G_ on the curve to itself, \\( k_a \\) times. This is the definition of multiplication by a scalar and is
 written as such:
$$
  P_a = k_a G
$$
Let's take an example from [this post](https://chuckbatson.wordpress.com/2014/11/26/secp256k1-test-vectors/), where
it is known that the public key for `1`, when written in uncompressed format is `0479BE667...C47D08FFB10D4B8`.
The following code snippet demonstrates this:

{{#playpen src/pubkey.rs}}

### Creating a Signature

#### Approach Taken

Reversing ECC math multiplication (i.e. division) is pretty much infeasible when using properly chosen random values for your scalars ([[5]], [[6]]).
 This property is called the _Discrete Log Problem_ and is used as the principle behind a lot of cryptography and digital signatures. 
 A valid digital signature is evidence that the person providing the signature knows the private key corresponding to the public key the message
is associated with, or that they have solved the Discrete Log Problem. 

The approach to creating signatures always follows this recipe:

1. Generate a secret once-off number (called a _nonce_), _r_
2. Create a public key, _R_ from _r_ (where _R = r.G_)
3. Send the following to Bob, your recipient: your message (_m_), _R_, and your public key (_P = k.G_)

The actual signature is created by hashing the combination of all the public information above to create a _challenge_, e:
$$
    e = H(R || P || m)
$$
The hashing function is chosen so that _e_ has the same range as your private keys. In our case, we want something that
returns a 256 bit number, so SHA256 is a good choice.

Now the signature is constructed using your private information:
$$
    s = r + ke 
$$
Now Bob can also calculate _e_, since he already knows _m, R, P_. But he doesn't know your private key, or nonce.

**Note:** When you construct the signature like this, it's known as a _Schnorr signature_, which we'll discuss in more 
detail in the next section. There are other ways of constructing _s_, such as ECDSA [[2]], which is used in Bitcoin.

But see this:

$$ sG = (r + ke)G $$

Multiply out the RHS:

$$ sG = rG + (kG)e $$

Substitute \\(R = rG \\) and \\(P = kG \\) and we have:
$$ sG = R + Pe $$

So Bob must just calculate the public key corresponding to the signature (_s.G_) and check that it equals the RHS of the last
equation above (_R + P.e_), all of which Bob already knows.

#### Why do we Need the Nonce?

Why do we need a nonce in the standard signature?

Let's say we naïvely sign a message m with
$$
    e = H(R || m)
$$
and then the signature would be \\(s = ek \\). 

Now as before, we can check that the signature is valid:
$$
\begin{align}
  sG &= ekG \\\\
     &= e(kG) = eP
\end{align}
$$
So far so good. But anyone can read your private key now because _s_ is a scalar, so \\(k = \frac{s}{e} \\)
 is not hard to do.
With the nonce you have to solve \\( k = (s - r)/e \\), but r is unknown, so this is not a feasible calculation as long
 as _r_ has been chosen randomly.

We can show that leaving off the nonce is indeed highly insecure:

{{#playpen src/no-nonce.rs}}

#### ECDH

How do parties that want to communicate securely generate a shared secret for encrypting messages? One way is called
the Elliptic Curve Diffie-Hellmam exchange (ECDH) which is a simple method for doing just this.

ECDH is used in many places, including the Lightning Network during channel negotiation [[3]].

Here's how it works. Alice and Bob want to communicate securely. A simple way to do this is to use each other's public keys and
calculate
$$
\begin{align}
  S_a &= k_a P_b \tag{Alice} \\\\
  S_b &= k_b P_a \tag{Bob} \\\\
  \implies S_a = k_a k_b G &\equiv S_b = k_b k_a G
\end{align}
$$
{{#playpen src/ecdh.rs}}

For security reasons, the private keys are usually chosen at random for each session (you'll see the term
_ephemeral_ keys being used), but then we have the problem of not being sure the other party is who they say they
are (perhaps due to a man-in-the-middle attack [[4]]).

Various additional authentication steps can be employed to resolve this problem, which we won't get into here. 

## Schnorr Signatures

If you follow the crypto news, you'll know that that new hotness in Bitcoin is Schnorr Signatures.

<p align="center"><img src="./img/schnorr-meme.jpg" width="600" /></p>

But in actual fact, they're old news! The Schnorr signature is considered the simplest digital signature scheme 
to be provably secure in a random oracle model. It is efficient and generates short signatures. 
It was covered by U.S. Patent 4,995,082 which expired in February 2008 [[7]].

### So Why All the Fuss?

What makes Schnorr signatures so interesting, and [potentially dangerous](#key-cancellation-attack), is their simplicity. 
Schnorr signatures are _linear_, so you have some nice properties.

Elliptic curves have the multiplicative property. So if you have two scalars _x, y_ with corresponding points, _X, Y_, 
the following holds:
$$
  (x + y)G = xG + yG = X + Y 
$$
Schnorr signatures are of the form \\( s = r + e.k \\). This construction is linear too, so it fits nicely with
the linearity of elliptic curve math.

You saw this property in the previous section, when we were verifying the signature. Schnorr signatures' linearity 
makes it very attractive for things like;

- signature aggregation;
- [atomic swaps](../../protocols/atomic-swaps/AtomicSwaps.md);
- ['scriptless' scripts](../scriptless-scripts/introduction-to-scriptless-scripts.md).

### (Naïve) Signature Aggregation

Let's see how the linearity property of Schnorr signatures can be used to construct a 2-of-2 multi-signature.

Alice and Bob want to co-sign something (a Tari transaction, say) without having to trust each other; 
i.e. they need to be able to prove ownership of their respective keys, and the aggregate signature is
only valid if _both_ Alice and Bob provide their part of the signature. 

Assuming private keys are denoted \\( k_i \\) and public keys \\( P_i \\). If we ask Alice and Bob to each 
supply a nonce, we can try:
$$
\begin{align}
  P_{agg} &= P_a + P_b \\\\
  e &= H(R_a || R_b || P_a || P_b || m) \\\\
  s_{agg} &= r_a + r_b + (k_a + k_b)e \\\\
        &= (r_a + k_ae) + (r_b + k_ae) \\\\
        &= s_a + s_b
\end{align}
$$
So it looks like Alice and Bob can supply their own _R_, and anyone can construct the 2-of-2 signature 
from the sum of the _Rs_ and public keys. This does work:

{{#playpen src/aggregation_1.rs}}

But this scheme is not secure!

### Key Cancellation Attack

Let's take the previous scenario again, but this time, Bob knows Alice's public key and nonce ahead of time, by
waiting until she reveals them. 

Now Bob lies, and says that his public key is \\( P_b' = P_b - P_a \\) and public nonce is \\( R_b' = R_b - R_a \\).

Note that Bob doesn't know the private keys for these faked values, but that doesn't matter.

Everyone assumes that \\(s_{agg} = R_a + R_b' + e(P_a + P_b') \\) as per the aggregation scheme.

But Bob can create this signature himself: 
$$
\begin{align}
  s_{agg}G &= R_a + R_b' + e(P_a + P_b') \\\\
    &= R_a + (R_b - R_a) + e(P_a + P_b - P_a) \\\\
    &= R_b + eP_b \\\\
    &= r_bG + ek_bG \\\\
  \therefore s_{agg} &= r_b + ek_b = s_b
\end{align}
$$
{{#playpen src/cancellation.rs}}

### Better Approaches to Aggregation

In the key attack above, Bob didn't know the private keys for his published _R_ and _P_ values. We could defeat Bob
by asking him to sign a message proving that he _does_ know the private keys.

This works, but it requires another round of messaging between parties, which is not conducive to a great user experience.

A better approach would be one that incorporates one or more of the following features:

- Must be provably secure in the plain public-key model, without having to prove knowledge of secret keys, as we might have asked Bob to do in the naïve approach above;
- should satisfy the normal Schnorr equation, i.e. the resulting signature can be verified with an expression of the form \\( R + e X \\).
- allows for Interactive Aggregate Signatures (IAS) where the signers are required to cooperate;
- allows for Non-interactive Aggregate Signatures (NAS) where the aggregation can be done by anyone;
- allow each signer to sign the same message, _m_;
- allow each signer to sign their own message, \\( m_i \\).

## MuSig

MuSig is a recently proposed ([[8]],[[9]]) simple signature aggregation scheme that satisfies all of the properties above.

### MuSig Demonstration

We'll demonstrate the interactive MuSig scheme here, where each signatory signs the same message. 
The scheme works as follows:

1. Each signer has a public-private key pair as before.
2. Each signer publishes the public key of their nonce, \\( R_i \\).
3. Everyone calculates the same "shared public key", _X_ as follows: 

$$
    \begin{align}
        \ell &= H(X_1 || \dots || X_n) \\\\
        a_i &= H(\ell || X_i) \\\\
        X &= \sum a_i X_i \\\\
    \end{align}
$$

Note that in the ordering of public keys above, some deterministic convention should be used, such as the lexicographical
order of the serialised keys.

1. Everyone also calculates the shared nonce, \\( R = \sum R_i \\).
2. The challenge, _e_ is \\( H(R || X || m) \\).
3. Each signer provides their contribution to the signature as:

$$
    s_i = r_i + k_i a_i e
$$

Notice that the only departure here from a standard Schnorr signature is the inclusion of the factor \\( a_i \\).

1. The aggregate signature is the usual summation, \\( s = \sum s_i \\).

Verification is done by confirming that 
$$
  sG \equiv R + e X \\
$$
as usual. Proof:
$$
\begin{align}
  sG &= \sum s_i G \\\\
     &= \sum (r_i + k_i a_i e)G \\\\
     &= \sum r_iG + k_iG a_i e \\\\
     &= \sum R_i + X_i a_i e \\\\
     &= \sum R_i + e \sum a_i X_i \\\\
     &= R + e X \\\\
     \blacksquare
\end{align}
$$
Let's demonstrate this using a three-of-three multisig:

{{#playpen src/musig.rs}}

### Security Demonstration

As a final demonstration, let's show how MuSig defeats the cancellation attack from the naïve signature scheme described
above. Using the same idea as in [the previous section](#key-cancellation-attack), Bob has provided fake values for his
nonce and public keys:
$$
\begin{align}
  R_f &= R_b - R_a \\\\
  X_f &= X_b - X_a \\\\
\end{align}
$$
This leads to both Alice and Bob calculating the following "shared" values:
$$
\begin{align}
  \ell &= H(X_a || X_f) \\\\
  a_a &= H(\ell || X_a) \\\\
  a_f &= H(\ell || X_f) \\\\
  X &= a_a X_a + a_f X_f \\\\
  R &= R_a + R_f (= R_b) \\\\
  e &= H(R || X || m)
\end{align}
$$
He then tries to construct a unilateral signature following MuSig:
$$
  s_b = r_b + k_s e
$$
Let's assume for now that \\( k_s \\) doesn't need to be Bob's private key, but that he can derive it using information
he knows. For this to be a valid signature, it must verify to \\( R + eX \\). So therefore
$$
\begin{align}
  s_b G          &= R + eX \\\\
  (r_b + k_s e)G &= R_b + e(a_a X_a + a_f X_f) & \text{The first term looks good so far}\\\\
                 &= R_b + e(a_a X_a + a_f X_b - a_f X_a) \\\\
                 &= (r_b + e a_a k_a + e a_f k_b - e a_f k_a)G & \text{The r terms cancel as before} \\\\
  k_s e &=  e a_a k_a + e a_f k_b - e a_f k_a & \text{But nothing else is going away}\\\\
  k_s &= a_a k_a + a_f k_b - a_f k_a \\\\              
\end{align}
$$
In the previous attack, Bob had all the information he needed on the right-hand side of the analogous calculation. In MuSig,
Bob must somehow know Alice's private key and the faked private key (the terms don't cancel anymore) in order to create a unilateral signature
and so his cancellation attack is defeated.

## References

[[1]] "RSA (Cryptosystem)" [online]. Available: https://en.wikipedia.org/wiki/RSA_(cryptosystem). Date accessed: 2018-10-11.

[1]: https://en.wikipedia.org/wiki/RSA_(cryptosystem)
"Wikipedia RSA Cryptography"


[[2]] "Elliptic Curve Digital Signature Algorithm", Wikipedia [online]. Available: https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm.
Date accessed: 2018-10-11.

[2]: https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm
"Wikipedia: ECDSA"

[[3]] "BOLT #8: Encrypted and Authenticated Transport, Lightning RFC", Github [online].
Available: https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md. Date accessed: 2018-10-11.

[3]: https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md
"BOLT #8: Encrypted and Authenticated Transport"

[[4]] "Man in the Middle Attack", Wikipedia [online].
Available: https://en.wikipedia.org/wiki/Man-in-the-middle_attack. Date accessed: 2018-10-11.

[4]: https://en.wikipedia.org/wiki/Man-in-the-middle_attack
"Wikipedia: Man in the Middle Attack"

[[5]] "How does a Cryptographically Secure Random Number Generator Work?" StackOverflow" [online].
Available:
https://stackoverflow.com/questions/2449594/how-does-a-cryptographically-secure-random-number-generator-work. Date accessed: 2018-10-11.

[5]: https://stackoverflow.com/questions/2449594/how-does-a-cryptographically-secure-random-number-generator-work
"StackOverflow: How does a Cryptographically Secure Random Number Generator Work?"

[[6]] "Cryptographically Secure Pseudorandom Number Generator", Wikipedia [online].
Available: https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator. Date accessed: 2018-10-11.

[6]: https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator
"Cryptographically secure pseudorandom number generator"

[[7]] "Schnorr Signature", Wikipedia [online]. Available: _https://en.wikipedia.org/wiki/Schnorr_signature_.
Date accessed: 2018-09-19.

[7]: https://en.wikipedia.org/wiki/Schnorr_signature
"Wikipedia: Schnorr Signature"

[[8]] "Key Aggregation for Schnorr Signatures", Blockstream [online]. Available: _https://blockstream.com/2018/01/23/musig-key-aggregation-schnorr-signatures.html_. Date accessed: 2018-09-19.

[8]: https://blockstream.com/2018/01/23/musig-key-aggregation-schnorr-signatures.html
"Blockstream: Key Aggregation for Schnorr Signatures"

[[9]] Gregory Maxwell, Andrew Poelstra, Yannick Seurin and Pieter Wuille, "Simple Schnorr Multi-signatures with Applications to Bitcoin" [online]. Available: https://eprint.iacr.org/2018/068.pdf. Date accessed: 2018-09-19.?


[9]: https://eprint.iacr.org/2018/068.pdf

"Simple Schnorr Multi-signatures with Applications to Bitcoin".

## Contributors

- [CjS77](https://github.com/CjS77)
- [SWvHeerden](https://github.com/SWvHeerden)
- [hansieodendaal](https://github.com/hansieodendaal)
- [neonknight64](https://github.com/neonknight64)