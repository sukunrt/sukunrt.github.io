---
date: "2023-04-06"
title: Finding MD4 Collisions using Wang's attack
tags: ["go", "cryptopals", "cryptography"]
---

Cryptopals Set7 Problem 53 is about finding collisions in MD4 using wangs attack. In ""Cryptanalysis of the Hash Functions MD4 and RIPEMD" Wang et al
describe a method to find collisions in MD4. If you are interested in implementing this attack and are stuck I suggest looking for the paper
"Improved Collision Attack on MD4 with Probability Almost 1" by Naito et al which describes the attack in more detail than the original paper. Here I try
to summarise the attack and its implementation

### MD4 

MD4 is a hashing algorithm that hashes the input message by first padding it, dividing the input in to blocks of size 512 bits and then repeatedly applying
three rounds of transformation to each block. 

The pseudocode for md4 copied from [RFC 1320](http://inserlink) 
```
We first define three auxiliary functions that each take as input
three 32-bit words and produce as output one 32-bit word.

        F(X,Y,Z) = XY v not(X) Z
        G(X,Y,Z) = XY v XZ v YZ
        H(X,Y,Z) = X xor Y xor Z

First break down the padded message m of size N bytes into a uint32 array of words M

for i, j := 0, 0; i < N; i, j = i + 4, j+1 
        M[j] = uint32(m[i: i+4]) // In little endian format

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
                [ABCD 12  3]  [DABC 13  7]  [CDAB 14 11]  [BCDA 15 19]

                /* Round 2. */
                /* Let [abcd k s] denote the operation
                        a = (a + G(b,c,d) + X[k] + 5A827999) <<< s. */

                /* Do the following 16 operations. */
                [ABCD  0  3]  [DABC  4  5]  [CDAB  8  9]  [BCDA 12 13]
                [ABCD  1  3]  [DABC  5  5]  [CDAB  9  9]  [BCDA 13 13]
                [ABCD  2  3]  [DABC  6  5]  [CDAB 10  9]  [BCDA 14 13]
                [ABCD  3  3]  [DABC  7  5]  [CDAB 11  9]  [BCDA 15 13]

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
