---
date: "2023-04-17"
title: Finding MD4 Collisions using Wang's attack
tags: ["go", "cryptopals", "cryptography"]
---

Cryptopals Set7 Problem 53 is about finding collisions in MD4 using wangs attack. In "Cryptanalysis of the Hash Functions MD4 and RIPEMD" Wang et al describe a method to find collisions in MD4. If you are interested in implementing this attack and are stuck I suggest looking for the paper "Improved Collision Attack on MD4 with Probability Almost 1" by Naito et al which describes the attack in more detail than the original paper. Here I summarise the attack and its implementation

### MD4 

MD4 is a hashing algorithm that hashes the input message by first padding it, dividing the input into blocks of size 64 bytes and then applying three rounds of transformation to each block. 

The pseudocode for md4 copied from [RFC 1320](https://www.ietf.org/rfc/rfc1320.html#section-3.4) 
```
First break down the padded message m of size N bytes 
into a uint32 array of words M

for i, j := 0, 0; i < N; i, j = i + 4, j+1 
    M[j] = uint32(m[i: i+4]) // In little endian format

We first define three auxiliary functions that each take as input
three 32-bit words and produce as output one 32-bit word.

    F(X,Y,Z) = XY v not(X) Z
    G(X,Y,Z) = XY v XZ v YZ
    H(X,Y,Z) = X xor Y xor Z


Do the following:

    /* Process each 16-word block. */
    For i = 0 to N/16-1 do

        /* Copy block i into X. */
        For j = 0 to 15 do
            Set X[j] to M[i*16+j].
        end /* of loop on j */

        /* Save A as AA, B as BB, C as CC, and D as DD. */
        AA = A
        BB = B
        CC = C
        DD = D           
        
        /* Round 1. */
        /* Let [abcd k s] denote the operation
                a = (a + F(b,c,d) + X[k]) <<< s. */
        /* Do the following 16 operations. */
        [ABCD  0  3]  [DABC  1  7]  [CDAB  2 11]  [BCDA  3 19]
        [ABCD  4  3]  [DABC  5  7]  [CDAB  6 11]  [BCDA  7 19]
        [ABCD  8  3]  [DABC  9  7]  [CDAB 10 11]  [BCDA 11 19]
        [ABCD 12  3]  [DABC 13  7]  [CDAB 14 11]  [BCDA 15 19           
        
        /* Round 2. */
        /* Let [abcd k s] denote the operation
                a = (a + G(b,c,d) + X[k] + 5A827999) <<< s. *           
        /* Do the following 16 operations. */
        [ABCD  0  3]  [DABC  4  5]  [CDAB  8  9]  [BCDA 12 13]
        [ABCD  1  3]  [DABC  5  5]  [CDAB  9  9]  [BCDA 13 13]
        [ABCD  2  3]  [DABC  6  5]  [CDAB 10  9]  [BCDA 14 13]
        [ABCD  3  3]  [DABC  7  5]  [CDAB 11  9]  [BCDA 15 13           
        
        /* Round 3. */
        /* Let [abcd k s] denote the operation
                a = (a + H(b,c,d) + X[k] + 6ED9EBA1) <<< s. */
        /* Do the following 16 operations. */
        [ABCD  0  3]  [DABC  8  9]  [CDAB  4 11]  [BCDA 12 15]
        [ABCD  2  3]  [DABC 10  9]  [CDAB  6 11]  [BCDA 14 15]
        [ABCD  1  3]  [DABC  9  9]  [CDAB  5 11]  [BCDA 13 15]
        [ABCD  3  3]  [DABC 11  9]  [CDAB  7 11]  [BCDA 15 15]

        /* Then perform the following additions. (That is, increment each
          of the four registers by the value it had before this block
          was started.) */
        A = A + AA
        B = B + BB
        C = C + CC
        D = D + DD

    end /* of loop on i */
```

### Wang's attack

Consider two blocks
```
M  = (m0, m1, ..., m15)
M' = (m'0, m'1, ..., m'15)
such that 
ΔM = M' - M = (Δm0, Δm1, Δm2, ... Δm15)

Where 
Δm0 = 2^31
Δm1 = 2^31 - 2^28
Δm12 = -2^16
Δmi = 0 for all other is
```
If the conditions on table 6 in paper are satisfied for block M, M and M' will have the same hash. Table 6 in the paper provides a sufficient set of conditions for M and M' to have a collision. It was later found that this set was not sufficient and Naito et al added two more conditions to the set.  

Our aim is to find an M such that it satisfies all these conditions. Just trying random Ms is infeasible. But as we will see a lot of these conditions can be fixed by taking a random M and applying simple transformations to M. 

The algorithm for finding a collision is 

```
    M = random bytes
    for i from 1..18
        if M does not satisfy the conditions of step i in table 6
            transform M to satisfy condition
```
There are two kinds of transforms that you can do on M to satisfy the conditions. The first 16 steps all operate on different blocks. As we will see this makes these steps easily invertible. The next two steps are more complicated. Changing `m[0]` for step 17 will change `a[1]` which will change `m[2]` which will change `d[1]` and so on

### First 16 rounds, Single Step Transformations

The first 16 conditions apply to outputs of the 16 steps of the first round. Consider the state variable a

let 
a0 be the initial value of a
a1 be the output of applying the transformation to a for the first time
a2 be the output of applying the transformation to a for the second time
a3 ...
a4 ...

Similarly define b0, b1 ... b4, c0, c1 ... c4, d0, d1 ... d4

The first round of transformation for a is

```
        a = (a + F(b,c,d) + X[k]) <<< s
```

here each step depends on a different word in the original block which allows us to find the message word given the output. Given a required a1 we can find `X[0]`

```
a1 = (a0 + F(b0, c0, d0) + X[0]) <<< 3 
a1 >>> 3 = a0 + F(b0, c0, d0) + X[0]
X[0] = (a1 >>> 3) - a0 - F(b0, c0, d0)
```

From table 6 first row we want a1 to satisfy the following condition
```
a1,7 = b0,7 // the 7th bit of a1 = 7th bit of b0
```
To satisfy this condition first find the a1 for original X[0] 
```
a1 = (a0 + F(b0, c0, d0) + X[0]) <<< 3
```
This is a1 from our original message
Now fix this a1 to satisfy the condition
```
a1 = a1 ^ (a1,7 ^ b1,7) // this ensures that we flip bit 7 if a1,7 != b1,7
```
Now we back calculate X[0]
```
X[0] = (a1 >>> 3) - a0 - F(b0, c0, d0)
```

If we implement just the first round transforms this gets the probability of collision down to 1 in a million. 


### Second Round, Multistep transformation

From second round onwards we cannot simply invert the operation to satisfy our condition because the new message word might not satisfy the conditions of the first 16 steps. These steps are complicated and I only implemented the first two to fix `a5` and `d5`

### Implementation

To implement the attack, I structured the code such that I could generate operations of table 6 using a parser. The code can be found here https://github.com/sukunrt/cryptopals/blob/main/crypto/wangmd/wangmd.go

The state is tracked using this struct
```go
type WangMD4 struct {
	m          [16]uint32
	a, b, c, d [16]uint32
	st         int
}
```
here m is the message broken into words. a, b, c, d are the md4 state variables and st is the step num currently being computed. 

Single step transformations are implemented as 
```go
	switch wmd.st {
	case 1:
		wmd.a[1] = wmd.a[1] ^ pb(wmd.a[1], 7) ^ pb(wmd.b[0], 7)
	case 2:
		wmd.d[1] = wmd.d[1] ^ pb(wmd.d[1], 7) ^ pb(wmd.d[1], 8) ^ pb(wmd.a[1], 8) ^ pb(wmd.d[1], 11) ^ pb(wmd.a[1], 11)
        ...
        }
```
The entire function is here
https://github.com/sukunrt/cryptopals/blob/main/crypto/wangmd/wangmd.go#L120  
These cases are generated using a parser, that takes in lines from the table in the form 
```
d1 d1,7 = 0, d1,8 = a1,8, d1,11 = a1,11
```
and generates 
```
	case 2:
		wmd.d[1] = wmd.d[1] ^ pb(wmd.d[1], 7) ^ pb(wmd.d[1], 8) ^ pb(wmd.a[1], 8) ^ pb(wmd.d[1], 11) ^ pb(wmd.a[1], 11)
```
The parser parses the rule, one condition at a time.   
`d1,7 = 0` becomes ` pb(wmd.d[1], 7) ` where pb(x, n) picks the nth byte of x setting everything else to 0.  
`d1,8 = a1,8` becomes `pb(wmd.d[1], 8) ^ pb(wmd.a[1].8)`.  
Finally, all the conditions are stitched together with the xor operator.   

The parser is available at 
https://github.com/sukunrt/cryptopals/blob/main/crypto/wangmd/codegen/main.go

### Testing

The test is available at https://github.com/sukunrt/cryptopals/blob/main/crypto/wangmd/wangmd_test.go  
It takes a couple of seconds to find a collision. It rougly takes 200k iterations per collision. I got
```
c2d093b8db8b6524118c66189dea3cfa8fb8477348b94285bd02785147f3b46a529f92de61dcc0812b594e7f753e9ff54c7b023a11865262d44939cb83c09e4e
c2d093b8db8b65a4118c66889dea3cfa8fb8477348b94285bd02785147f3b46a529f92de61dcc0812b594e7f753e9ff54c7b013a11865262d44939cb83c09e4e
```

The strings look very similar because their only difference is 
delta m0 = 2^31
delta m1 = 2^31 - 2^28
delta m12 = -2^16

To highlight the first difference take this prefix. The last hex digit is different
```
c2d093b8db8b652
c2d093b8db8b65a
```