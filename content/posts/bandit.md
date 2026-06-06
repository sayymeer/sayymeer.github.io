---
title: "Writeup Bandit | Overthewire"
data: 2026-06-03
tags: ["writeup"]
description: "Starting this blog to document my learning."
---

Giving [Bandit](https://overthewire.org/wargames/bandit) a try (for learning to read _man_ and _info_ pages)

## Level 0 -> 1

```sh
cat readme
```

## Level 1 -> 2

The filename is `-` (_dashed filenames_). After trying many things got to know that we can use `./-filename`.

> Also according to POSIX standard, `--` is used to signal the end of command options, anything typed after that is positional arguments. But for file named `-` it wont work with `cat` many utilities treat this as `stdin` or `stdout`.

```sh
cat ./-
```

## Level 2 -> 3

The filename is `--spaces in this filename--`.

```sh
cat ./--spaces\ in\ this\ filename--
```

## Level 3 -> 4

Password was in `inhere/...Hiding-From-You`

```sh
cat inhere/...Hiding-From-You
```

## Level 4 -> 5

Password is in only human readable file in `inhere` directory. I used `file` command to check which file has _ASCII text_. Also using `{0..9}` to expand cuz there were 9 files.

```bash
file ./-file0{0..9}
```

## Level 5 -> 6

The file is in `inhere` directory with following props:

- 1033bytes size
- human-readable
- not executable

There were multiple directories and files in the directories inside the folder. To search for this kind of file we have to use `find`. After going through man pages of `find`, it support _expressions_. Below are the required expressions

- `-size n` is for matching file size. It will be `1033c`, _c_ is for bytes. (I tried searching with size only and got the file).
- `-executable` matches files which are executable. `!` operator for matching with not executable.
- no direct support for human-readable

```sh
find inhere -size 1033c ! -executable
```

## Level 6 -> 7

File can be anywhere on server, so we will run `find` on `/` directory. And has following props:

- owned by user bandit7, `-user bandit7`
- owned by group bandit6, `-group bandit6`
- has 33 bytes size, `-size 33c`

Also I am redirecting `stderr` to `/dev/null` for the _Permission denied_ errors.

```sh
find / -size 33c -user bandit7 -group bandit6 2> /dev/null
```

## Level 7 -> 8

Password is in file `data.txt` next to the word **millionth**.

```sh
cat data.txt | grep millionth
```

## Level 8 -> 9

In file `data.txt` only one line is not repeated anywhere in the file, that line will be our password. The way I used first is to sort the file and then pass it to uniq with count option, and use grep to find line with 1 count. After going through `man` and `info` pages of `uniq` and `sort`, I found that `-u` flag only prints the unique line which is not repeated (it discard the last line that would be output of repeated input group).

```sh
// it would print the count (1) and the line.
sort data.txt | uniq -c | grep "1 "

// It would print only the line
sort data.txt | uniq -u
```

## Level 9 -> 10

In `data.txt` in one of the _human-readable strings_ preceeded with "=" characters. `strings` output only the _human-readable strings_ in the file.

```sh
strings data.txt | grep "="
```

## Level 10 -> 11

Password is base64 encoded string in `data.txt`

```bash
base64 -d data.txt
```

## Level 11 -> 12

Password is **ROT13** encoded string. `tr` is used for translating characters it takes 2 string as input and then translate according two the arrays. ROT13 shift every character 13 positions ahead in alphabets. So `A` becomes `N`. Below we are specifying input string to be `a-zA-Z` which expands to normal alphabetical sequence and then output string is `n-za-m` for translating lower case and `N-ZA-M` for translating upper case.

```sh
tr a-zA-Z n-za-mN-ZA-M < data.txt
```

## Level 12 -> 13

The password for the next level is stored in the file `data.txt`, which is a hexdump of a file that has been repeatedly compressed. `xxd -r` is used to reverse hexdump. I will cd into temp directory with `cd $(mktemp -d)`.

```sh
cd $(mktemp -d)
cp ~/data.txt .
xxd -r data.txt bin.data
```

After checking `file bin.data` it is gzip compressed. I have to rename the file bin.data to `bin.gz` and `gzip -d bin.gz` uncompressed it to file name `bin`. Running `file bin` I got to know it is `bzip` compressed file. `bzip -d bin` will decompress the data. Repeating this several times, checking file type and uncompressing. I got a tar archive, uncompressing it using `tar -xvf` and then repeating above steps. I finally got the file `data8`, which has password.

> gzip gives error when it does not find `.gz` extension, it tries to replace the file and renaming it when uncompressing. We can use `gzip -d -S ".data"` to know that _.data._ is suffix, then after decompression it will remove _.data_ from the file name.

## Level 13 -> 14

I first transfered the `sshkey.private` file from `bandit13@bandit` to my localhost with `scp`. And changed the file permissions to use it as _identity file_ with ssh.

- in `scp` capital `-P` flag is used for port unlike `ssh`.
- `600` file mode removes file's **rwx** access from other users and groups.
- `-i` flag is used for using _identity file_ with ssh.

```sh
scp -P 2220 bandit13@bandit:~/sshkey.private .
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@bandit -p 2220
```

## Level 14 -> 15

I used private key file to log in. Password for level 14 is in `/etc/bandit_pass/bandit14`. The password for the next level can be retrieved by submitting the password of the current level to port 30000 on localhost.

```sh
cat /etc/bandit_pass/bandit14 | nc localhost 30000
```

## Level 15 -> 16

We can get next level password by submitting this level password on `localhost:30001` with **SSL/TLS** encryption. `openssl` is a utility that deal with these encryption. `s_client` option helps to connect with above encryption.

```sh
// Open the connection and submit current password
openssl s_client -s localhost:30001
```

## Level 16 -> 17

We have to submit password to one of the open ports between `31000-32000` speaking SSL/TLS. After running `nmap localhost -p31000-32000 -sV`, I found the port. We will use the same openssl s_client command `-nocommands` flags because our current password starts with letter `k` in it which will invoke the interactive command for `KEYUPDATE`. After submitting the password we will get a RSA Private Key file.

## Level 17 -> 18

We will use `diff` command to see which line has changes.

```sh
diff password.{old,new}
```

## Level 18 -> 19

When i tried login to `bandit18` I got a _Bye Bye_ message. The password for the next level is stored in a file readme in the homedirectory. Unfortunately, someone has modified .bashrc to log you out when you log in with SSH. I copied the file to my local using `scp` and got the password. Or we can execute the `cat` command using `ssh`

```sh
ssh -p 2220 bandit18@bandit cat readme
```

## Level 19 -> 20

We have `setuid` binary, which gives us permissions for user `bandit20`.

```sh
./bandit20-do cat /etc/bandit_pass/bandit20
```
