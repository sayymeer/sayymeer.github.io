---
title: "Set 1 Cryptopals Writeup"
data: 2026-06-11
tags: ["writeup"]
description: "Writeup for Cryptopals Set 1 in python"
---

This is the markdown version of my python notebook. You can see my full notebook [here](https://github.com/sayymeer/cryptopals-writeups/blob/master/set1.ipynb).

> Always operate on raw bytes, never on encoded strings. Only use hex and base64 for pretty-printing.

I will create some basic function to encode/decode for later use.

```python
import base64
def encode_hex(a: bytes) -> str:
    return a.hex()

def decode_hex(a: str) -> bytes:
    return bytes().fromhex(a)

def encode_b64(a: bytes) -> str:
    return base64.b64encode(a).decode()

def decode_b64(a: str) -> bytes:
    return base64.b64decode(a)
```

## Challenge 1 (Convert hex to base64)

We are given `hex` string, we have to convert it to `b64`.

```python
hex_str = "49276d206b696c6c696e6720796f757220627261696e206c696b65206120706f69736f6e6f7573206d757368726f6f6d"
res_b64 = "SSdtIGtpbGxpbmcgeW91ciBicmFpbiBsaWtlIGEgcG9pc29ub3VzIG11c2hyb29t"

print(encode_b64(decode_hex(hex_str)) == res_b64)
```

    True

## Challenge 2 (Fixed XOR)

We need to write a function which take two equal len buffer and gives their xor.

```python
def xor(a: bytes, b:bytes) -> bytes:
    return bytes([x^y for x,y in zip(a,b)])

input1 = decode_hex("1c0111001f010100061a024b53535009181c")
input2 = decode_hex("686974207468652062756c6c277320657965")

output = "746865206b696420646f6e277420706c6179"

print(encode_hex(xor(input1, input2)) == output)
```

    True

## Challenge 3 (Single-byte XOR cipher)

We are given an encrypted hex encoded string. Encryption is done with `xor`ing the original text with repeated single xor. We will try every key from 0 to 127. And compare the decrypted string with english text distribution using **chi-squared statistic**, You can read about this [here](http://practicalcryptography.com/cryptanalysis/text-characterisation/chi-squared-statistic/). Below is the basic formula.

$$stat(C,E) = \Sigma_{i=A}^{i=Z} \frac{(C_i-E_i)^2}{E_i}$$

Here $C_i$ is the character count of ith letter and $E_i$ is character count of ith letter in distribution you are comparing against (multiply the _freq_ with total length of the text). Lower the result, similar the distributions are.

Also I will be taking _apparent length_ of the string (only including english alphabets) when multiplying with the english distribution frequencies and if the _apparent length_ is smaller than 10 characters, i am adding 100 (to increase the chi square statistics) because it wont be that accurate.

```python
# You can get this distribution from wikipedia
eng_distribution : dict[chr, float] = {
    'a': 0.082,
    'b': 0.015,
    'c': 0.028,
    'd': 0.043,
    'e':0.127,
    'f':0.022,
    'g':0.02,
    'h':0.061,
    'i':0.07,
    'j':0.0016,
    'k':0.0077,
    'l':0.04,
    'm':0.024,
    'n':0.067,
    'o':0.075,
    'p':0.019,
    'q':0.0012,
    'r':0.06,
    's':0.063,
    't':0.091,
    'u':0.028,
    'v':0.0098,
    'w':0.024,
    'x':0.0015,
    'y':0.02,
    'z':0.00074
}


def chi_square_statistic(a: str, distribution: dict[chr, float])->float :
    dist = {ch:0 for ch in eng_distribution.keys()}
    for i in a.lower():
        if i in dist.keys(): dist[i] = dist[i]+1
    apparent_len = sum([dist[i] for i in dist.keys()])
    summ = 0
    if apparent_len<10: summ = summ+100
    for k in distribution.keys():
        e_i = distribution[k]*apparent_len
        c_i = 0
        if k in dist.keys(): c_i = dist[k]
        summ = summ + ((c_i-e_i)**2)/e_i
    return summ
```

```python
cipher_text = decode_hex("1b37373331363f78151b7f2b783431333d78397828372d363c78373e783a393b3736")


# This will return top k candidates with least chi square statistic
def single_byte_xor_decrypt(cipher :bytes, k:int) -> list[tuple[float, str]]:
    decrypted_list :list[tuple[float, str]]= []

    # We will xor against every character in ascii
    for i in range(2**8):
        decoded = "".join([chr(x^i) for x in cipher])

        # \n is counted as non printable, so replacing it with $
        decoded = decoded.replace("\n","$")
        if not decoded.isascii() or not decoded.isprintable(): continue
        decrypted_list.append((chi_square_statistic(decoded, eng_distribution), decoded.replace("$","\n"), chr(i)))

    return sorted(decrypted_list)[:k]

print(single_byte_xor_decrypt(cipher_text,3))

```

    [(38.91679171937716, "Cooking MC's like a pound of bacon", 'X'), (52.342811475183375, "Dhhlni`'JD t'knlb'f'whric'ha'efdhi", '_'), (52.573890713512334, 'Eiimoha&KE!u&jomc&g&vishb&i`&dgeih', '^')]

We got the answer on 1st ranking `Cooking MC's like a pound of bacon`.

## Challenge 4 (Detect single-character XOR)

We are given a file with 60-character strings that has been encrypted by single-char XOR. The file is saved as `1.4.in`. For every encrypted string we will try to decode using previous method, take top 3 candidates and aggregate them in single list. After this, we will sort them and will look for any english sentence.

```python
with open("input/1.4.in", "r") as f:
    encrypted_bytes = list(map(decode_hex,f.read().strip().splitlines()))

deciphered_cands = {}
for cipher in encrypted_bytes:
    deciphered = single_byte_xor_decrypt(cipher,10)
    if len(deciphered) == 0: continue
    deciphered_cands[encode_hex(cipher)] = deciphered

for k in deciphered_cands.keys():
    print(k, deciphered_cands[k])
```

    7b5a4215415d544115415d5015455447414c155c46155f4058455c5b523f [(39.176282927465316, 'Now that the party is jumping\n', '5'), (193.39602699525628, '\n+3d0,%0d0,!d4%60=d-7d.1)4-*#N', 'q'), (410.84069108890606, 'dE]\n^BK^\n^BO\nZKX^S\nCY\n@_GZCDM ', '\x1f')]
    3f1b5a343f034832193b153c482f1705392f021f5f0953290c4c43312b36 [(43.94102361322612, 'Ok*DOs8BiKeL8_guI_ro/y#Y|<3A[F', 'p'), (61.30393812720791, ']y8V]a*P{Yw^*Mug[M`}=k1Kn.!SIT', 'b'), (89.11918512857585, 'D`!ODx3Ib@nG3Tl~BTyd\nr(Rw78JPM', '{'), (106.93670879855436, 'Mi(FMq:@kIgN:]ewK]pm-{![~>1CYD', 'r'), (108.38363954993727, "Tp1_Th#YrP~W#D|nRDit4b8Bg'(Z@]", 'k'), (130.49916399731492, 'Sw6XSo\n^uWyP\nC{iUCns3e?E` /]GZ', 'l'), (136.14728220846928, 'Fb#MFz1K`BlE1Vn|@V{f&p*Pu5:HRO', 'y'), (157.0054982908436, '@d%K@|7MfDjC7PhzFP}` v,Vs3<NTI', '\x7f'), (167.3084906436322, 'Hl-CHt?EnLbK?X`rNXuh(~\n^{;4F\\A', 'w'), (167.4102822424299, 'Vr3]Vj![pR|U!F~lPFkv6`:@e%*XB_', 'i')]
    37513b2d0a4e3e5211372a3a01334c5d51030c46463e3756290c0d0e1222 [(27.39835318990386, 'R4^Ho+[7tRO_dV)84fi##[R3LihkwG', 'e'), (32.053779602457716, '[=WAf"R>}[FVm_ 1=o`**R[:E`ab~N', 'l'), (66.3295735327604, 'H.DRu1A-nHUE~L3".|s99AH)Vsrqm]', '\x7f'), (95.22150241236443, "M+AWp4D(kMP@{I6'+yv<<DM,SvwthX", 'z'), (110.99001445261338, 'U3YOh,\\0sUHXcQ.?3an\n\n\\U4Knolp@', 'b'), (128.3558023785106, 'T2XNi-]1rTIYbP/>2`o%%]T5JonmqA', 'c'), (134.8847794112109, '];QG`\nT8{]@PkY&7;if,,T]<CfgdxH', 'j'), (158.5605198510146, 'N(BTs7G+hNSCxJ5\n(zu??GN/Putwk[', 'y'), (163.0288738959713, '_9SEb&V:y_BRi[\n59kd..V_>AdefzJ', 'h'), (168.20881664230237, 'O)CUr6F*iORByK4%){t>>FO.QtuvjZ', 'x')]
    1512371119050c0c1142245a004f033650481830230a1925085c1a172726 [(67.83428270685255, 'qvSu}ahhu&@>d+gR4,|TGn}Al8~sCB', 'd'), (69.77088267096578, 'dcF`ht}}`3U+q>rG!9iAR{hTy-kfVW', 'q'), (90.51400147552667, 'mjOia}tti:\\"x7{N(0`H[ra]p\nbo_^', 'x'), (129.08662881641368, 'klIog{rro<Z\n~1}H.6fN]tg[v"diYX', '~'), (134.37240783749144, 'wpUs{gnns F8b-aT2*zRAh{Gj>xuED', 'b'), (145.6517358446062, 'ebGaiu||a2T*p?sF 8h@SziUx,jgWV', 'p'), (148.5676677438338, 'y~[}ui``}.H6l#oZ<\nt\\OfuId0v{KJ', 'l'), (195.43778482961815, 'tsVpxdmmp#E;a.bW1)yQBkxDi={vFG', 'a'), (279.62361015448363, 'urWqyellq"D:`/cV0(xPCjyEh<zwGF', '`'), (286.7313496735869, 'afCemqxxe6P.t;wB\n<lDW~mQ|(ncSR', 't')]
    3649211f210456051e290f1b4c584d0749220c280b2a50531f262901503e [(44.90203782140112, '_ HvHm?lw@fr%1\nn KeAbC9:vO@h9W', 'i'), (56.59558861159029, '[\nLrLi;hsDbv!5 j\nOaEfG=>rKDl=S', 'm'), (68.58804966794108, '^!IwIl>mvAgs\n0%o!Jd@cB8;wNAi8V', 'h'), (92.82174461023921, 'T+C}Cf4g|Kmy.:/e+@nJiH21}DKc2\\', 'b'), (116.57368091645127, "\\#KuKn<otCeq&2'm#HfBa@:9uLCk:T", 'j'), (124.25940581021584, 'Z%MsMh:irEcw 4!k%N`DgF<?sJEm<R', 'l'), (145.2688690838172, 'D;SmSv\nwl[}i>*?u;P~ZyX"!mT[s"L', 'r'), (164.96740958173282, "A>VhVs!ri^xl;/:p>U{_|]'\nhQ^v'I", 'w'), (176.3716500974716, '@?WiWr sh_ym:.;q?Tz^}\\&%iP_w&H', 'v'), (180.26283553989776, "I6^`^{)zaVpd3'2x6]sWtU/,`YV~/A", '\x7f')]

We got the answer `Now that the party is jumping`.

## Challenge 5 (Implement repeating-key XOR)

We are given a stanza, we have to encrypt this with repeating key XOR with the key `ICE`

```python
KEY = "ICE"
PLAIN_TEXT = '''Burning 'em, if you ain't quick and nimble
I go crazy when I hear a cymbal'''

OUTPUT = "0b3637272a2b2e63622c2e69692a23693a2a3c6324202d623d63343c2a26226324272765272a282b2f20430a652e2c652a3124333a653e2b2027630c692b20283165286326302e27282f"

def repeating_key_xor(text: bytes, key:bytes) -> bytes:
    encrypt = []
    for i in range(len(text)):
        encrypt.append(text[i]^key[i%len(key)])
    return bytes(encrypt)


print(encode_hex(repeating_key_xor(bytes(PLAIN_TEXT, encoding="ascii"), bytes(KEY, encoding="ascii"))) == OUTPUT)

```

    True

## Challenge 6 (Breaking repeating-key XOR)

We are given a file `1.6.in` which is base64'd after encrypting with repeating-key xor. We have to decrypt it. I will follow the steps given in the challenge to decrypt it.
We will first implement the function to find _hamming distance_ between two strings. Hamming distance is the no. of different bits. If we xor two bytes, we can count the number of 1s and it will give the edit distance between them.

```python
def diff_bits(a: int, b:int):
    c = a^b
    count = 0
    for i in range(8):
        if (c>>i)&1: count = count+1
    return count

def hamming_distance(a: bytes, b: bytes):
    count = 0
    for i in zip(a,b):
        count = count + diff_bits(i[0],i[1])
    return count

print(hamming_distance(bytes("this is a test", encoding="ascii"), bytes("wokka wokka!!!", encoding="ascii")))
```

    37

We will guess the keysize, using the explained technique in the challenge. We will take the values from 2 to 40, and take 2 blocks of keysize, find the edit distance between them, normalize it by dividing with the guessed value and sort them.

```python
with open("input/1.6.in", "r") as f:
    encrypted_bytes = decode_b64("".join(f.read().strip().splitlines()))

guess_keysize = []
for keysize in range(2,40):
    # first two blocks
    first_block = encrypted_bytes[:keysize]
    second_block = encrypted_bytes[keysize:2*keysize]
    dist_first = hamming_distance(first_block, second_block)

    #second two blocks
    third_block = encrypted_bytes[2*keysize:3*keysize]
    fourth_block = encrypted_bytes[3*keysize:4*keysize]
    dist_sec = hamming_distance(third_block, fourth_block)

    avg_dist = (dist_first+dist_sec)/2
    normalized_dist = avg_dist/keysize
    guess_keysize.append((normalized_dist, keysize))

# We will take top 10 candidates
KEYSIZE = [x[1] for x in sorted(guess_keysize)[:10]]
print(KEYSIZE)
```

    [2, 5, 3, 13, 11, 29, 31, 18, 7, 20]

We will try every keysize from the list. For any keysize, e.g. "ICE", 1st letter of the plain text will be xored with 'I', 2nd with 'C', 3rd with 'E' and 4th again with 'I'. So if we take 1st, 4th, 7th... letter of the cipher, this is same as single key xor. We will do the same and solve single key xor like above.

```python
for size in KEYSIZE:
    blocks = []
    for i in range(size):
        col = []
        for j in range(i,len(encrypted_bytes),size):
            col.append(encrypted_bytes[j])
        blocks.append(bytes(col))
    candidates = []
    for col in blocks:
        t = single_byte_xor_decrypt(col,1)
        # If the list is empty continue
        if len(t) == 0: continue
        candidates.append(t)

    # To check whether we have enough candidates as the key size
    if len(candidates) != size: continue
    print("Size: ", size, end=", ")
    key = ""
    for i in candidates:
        key = key+i[0][2]
    print("Key = ",key)
    print()
    print("Decrypted Msg: ")
    print(str(repeating_key_xor(encrypted_bytes,bytes(key, encoding="ascii")), encoding="ascii"))
```

    Size:  29, Key =  Terminator X: Bring the noise

    Decrypted Msg:
    I'm back and I'm ringin' the bell
    A rockin' on the mike while the fly girls yell
    In ecstasy in the back of me
    Well that's my DJ Deshay cuttin' all them Z's
    Hittin' hard and the girlies goin' crazy
    Vanilla's on the mike, man I'm not lazy.

    I'm lettin' my drug kick in
    It controls my mouth and I begin
    To just let it flow, let my concepts go
    My posse's to the side yellin', Go Vanilla Go!

    Smooth 'cause that's the way I will be
    And if you don't give a damn, then
    Why you starin' at me
    So get off 'cause I control the stage
    There's no dissin' allowed
    I'm in my own phase
    The girlies sa y they love me and that is ok
    And I can dance better than any kid n' play

    Stage 2 -- Yea the one ya' wanna listen to
    It's off my head so let the beat play through
    So I can funk it up and make it sound good
    1-2-3 Yo -- Knock on some wood
    For good luck, I like my rhymes atrocious
    Supercalafragilisticexpialidocious
    I'm an effect and that you can bet
    I can take a fly girl and make her wet.

    I'm like Samson -- Samson to Delilah
    There's no denyin', You can try to hang
    But you'll keep tryin' to get my style
    Over and over, practice makes perfect
    But not if you're a loafer.

    You'll get nowhere, no place, no time, no girls
    Soon -- Oh my God, homebody, you probably eat
    Spaghetti with a spoon! Come on and say it!

    VIP. Vanilla Ice yep, yep, I'm comin' hard like a rhino
    Intoxicating so you stagger like a wino
    So punks stop trying and girl stop cryin'
    Vanilla Ice is sellin' and you people are buyin'
    'Cause why the freaks are jockin' like Crazy Glue
    Movin' and groovin' trying to sing along
    All through the ghetto groovin' this here song
    Now you're amazed by the VIP posse.

    Steppin' so hard like a German Nazi
    Startled by the bases hittin' ground
    There's no trippin' on mine, I'm just gettin' down
    Sparkamatic, I'm hangin' tight like a fanatic
    You trapped me once and I thought that
    You might have it
    So step down and lend me your ear
    '89 in my time! You, '90 is my year.

    You're weakenin' fast, YO! and I can tell it
    Your body's gettin' hot, so, so I can smell it
    So don't be mad and don't be sad
    'Cause the lyrics belong to ICE, You can call me Dad
    You're pitchin' a fit, so step back and endure
    Let the witch doctor, Ice, do the dance to cure
    So come up close and don't be square
    You wanna battle me -- Anytime, anywhere

    You thought that I was weak, Boy, you're dead wrong
    So come on, everybody and sing this song

    Say -- Play that funky music Say, go white boy, go white boy go
    play that funky music Go white boy, go white boy, go
    Lay down and boogie and play that funky music till you die.

    Play that funky music Come on, Come on, let me hear
    Play that funky music white boy you say it, say it
    Play that funky music A little louder now
    Play that funky music, white boy Come on, Come on, Come on
    Play that funky music

We got the key of size 29, `Terminator X: Bring the noise`. And the decrypted msg also.

## Challenge 7 (AES in ECB mode)

We are given a file `1.7.in` which is encrypted via AES-128 in ECB mode with key `YELLOW SUBMARINE` and then base64d. We have to decrypt it. I also tried decrypting using `openssl` command.

```sh
openssl enc -aes-128-ecb -d -a -in 1.7.in -K 59454c4c4f57205355424d4152494e45
```

It will print the output to `stdout`

- `-aes-128-ecb` is the cipher
- `-d` to decrypt
- `a` to process the file as `base64`
- `-in file` input file
- `-K` is for the raw key in hex

```python
# I will using pycryptodome
from Crypto.Cipher import AES

with open("input/1.7.in","r") as f:
    encrypted_bytes = decode_b64("".join(f.read().strip().splitlines()))


def aes_128_ecb_decrypt(enc_bytes: bytes,key: str) -> str:
    KEY = bytes(key, encoding="ascii")
    aess = AES.new(key=KEY, mode=AES.MODE_ECB)
    return str(aess.decrypt(enc_bytes), encoding="ascii")

print(aes_128_ecb_decrypt(encrypted_bytes, "YELLOW SUBMARINE"))
```

    I'm back and I'm ringin' the bell
    A rockin' on the mike while the fly girls yell
    In ecstasy in the back of me
    Well that's my DJ Deshay cuttin' all them Z's
    Hittin' hard and the girlies goin' crazy
    Vanilla's on the mike, man I'm not lazy.

    I'm lettin' my drug kick in
    It controls my mouth and I begin
    To just let it flow, let my concepts go
    My posse's to the side yellin', Go Vanilla Go!

    Smooth 'cause that's the way I will be
    And if you don't give a damn, then
    Why you starin' at me
    So get off 'cause I control the stage
    There's no dissin' allowed
    I'm in my own phase
    The girlies sa y they love me and that is ok
    And I can dance better than any kid n' play

    Stage 2 -- Yea the one ya' wanna listen to
    It's off my head so let the beat play through
    So I can funk it up and make it sound good
    1-2-3 Yo -- Knock on some wood
    For good luck, I like my rhymes atrocious
    Supercalafragilisticexpialidocious
    I'm an effect and that you can bet
    I can take a fly girl and make her wet.

    I'm like Samson -- Samson to Delilah
    There's no denyin', You can try to hang
    But you'll keep tryin' to get my style
    Over and over, practice makes perfect
    But not if you're a loafer.

    You'll get nowhere, no place, no time, no girls
    Soon -- Oh my God, homebody, you probably eat
    Spaghetti with a spoon! Come on and say it!

    VIP. Vanilla Ice yep, yep, I'm comin' hard like a rhino
    Intoxicating so you stagger like a wino
    So punks stop trying and girl stop cryin'
    Vanilla Ice is sellin' and you people are buyin'
    'Cause why the freaks are jockin' like Crazy Glue
    Movin' and groovin' trying to sing along
    All through the ghetto groovin' this here song
    Now you're amazed by the VIP posse.

    Steppin' so hard like a German Nazi
    Startled by the bases hittin' ground
    There's no trippin' on mine, I'm just gettin' down
    Sparkamatic, I'm hangin' tight like a fanatic
    You trapped me once and I thought that
    You might have it
    So step down and lend me your ear
    '89 in my time! You, '90 is my year.

    You're weakenin' fast, YO! and I can tell it
    Your body's gettin' hot, so, so I can smell it
    So don't be mad and don't be sad
    'Cause the lyrics belong to ICE, You can call me Dad
    You're pitchin' a fit, so step back and endure
    Let the witch doctor, Ice, do the dance to cure
    So come up close and don't be square
    You wanna battle me -- Anytime, anywhere

    You thought that I was weak, Boy, you're dead wrong
    So come on, everybody and sing this song

    Say -- Play that funky music Say, go white boy, go white boy go
    play that funky music Go white boy, go white boy, go
    Lay down and boogie and play that funky music till you die.

    Play that funky music Come on, Come on, let me hear
    Play that funky music white boy you say it, say it
    Play that funky music A little louder now
    Play that funky music, white boy Come on, Come on, Come on
    Play that funky music
    

## Challenge 8 (Detect AES in ECB mode)

In file `1.8.in` we have hex-encoded ciphertext encrypted with `aes-128-ecb`. In AES, same 16-bytes with same key will always encrypt into same 16 output bytes. We will hope that there are some 16-bytes which repeat in plain text, if yes then probably that ciphertext can be aes encrypted.

```python
with open("input/1.8.in", "r") as f:
    ciphers_list: [bytes] = list(map(decode_hex, f.read().strip().splitlines()))


for cipher in ciphers_list:
    length = len(cipher)
    mp = {}
    for i in range(int(length/16)):
        part = cipher[16*i: 16*(i+1)]
        if part in mp.keys(): mp[part] = mp[part]+1
        else: mp[part] = 1
    for key in mp.keys():
        if mp[key] > 1:
            print(encode_hex(cipher))
            break

```

    d880619740a8a19b7840a8a31c810a3d08649af70dc06f4fd5d2d69c744cd283e2dd052f6b641dbf9d11b0348542bb5708649af70dc06f4fd5d2d69c744cd2839475c9dfdbc1d46597949d9c7e82bf5a08649af70dc06f4fd5d2d69c744cd28397a93eab8d6aecd566489154789a6b0308649af70dc06f4fd5d2d69c744cd283d403180c98c8f6db1f2a3f9c4040deb0ab51b29933f2c123c58386b06fba186a
