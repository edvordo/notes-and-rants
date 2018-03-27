# Signign commits on github via GPG on Windows (10)

<sup>Valid as of: 2018-03-27</sup>

###### Sidenote

The [instructions github provides](https://help.github.com/articles/signing-commits-with-gpg/)
will get you most of the way there, but not everything (imho) is mentioned and
it's divided into separate articles

<sup>
(also explains using only console) anyway ..
</sup>

## The tools

- GPG
  + duh
  + you may have it already, just check for a program called `Kleopatra`
  (a GUI key manager) or just search for `GPG`
  + if you don't, [go to official `gnupg` site](https://www.gnupg.org/download/)
  to pick the download installer you want, or
  [download a realeas from gpg4win.org](https://gpg4win.org/download.html)
  + haven't found (nor did I bothered to look for any) portable versions
  + [[insert standard windows installation process with default options here]](https://i.imgur.com/pDI3Qqh.png)
- GIT
  + duh ..
  + chances are you already have git-related stuff setup if you are here :)
  
First thing that is not mentioned in the tutorial: you need a new `System Variable`

* Name: `GNUPGHOME`
* Value: `[drive]:\Users\[user]\AppData\Roaming\gnupg` (usualy) or 
a shortcut `%appdata%/gnupg` (haven't tested) 

## The key(s?)

1. Launch `Kleopatra`<sub>`v3.0.2-gpg4win-3.0.3`</sub>
2. `File -> New Key Pair` <sup>(or just `Ctrl+N`)</sup>
3. `Create a personal OpenPGP key pair`
4. Fill in `Name` - can be pretty much whatever, I used my github handle
5. Fill in `EMail` - now this needs to be a [***verified*** email associated to
your github account](https://github.com/settings/emails)
6. Click `Advanced Settings`
   1. Go with `RSA` and `+ RSA`
   2. Use the `4096 bits` versions <sup>(because why not)</sup>
   3. `[optional]` Check `Authentification`
   4. You can choose expiration, but you don't have to
7. `Next`
8. Confirm info & click `Create`
   1. You will be asked for a passphrase - use `KeePass` (or pwd manager of 
   choice) to generate and save one. **You need this for later!**
   2. <sup>use the pwd manager to generate your passphrase, it checks for 
   pwd strength</sup>
9. Backup the key pair, for your own sake

## github association

1. Double-click the created key in `Kleopatra`
2. Click `Export...`
3. [Copy the contents](https://i.imgur.com/hxIsnxz.png)
4. Go to your [SSH & GPG related settings page](https://github.com/settings/keys)
5. Add the key you just copied

## Signing commits

If using `phpStorm`, I didn't find how to do these without console. Of course,
you can always open the files within `.git` folder

In NetBeans you can go to `Teams -> Repository -> Show Configuration` which is
a text file.

If you have `TortoiseGIT`, there is GUI.

Anyway ..

### The console

#### Prerequisites

In `Kleopatra`, double click the desired key again and click on
`More details...`.
On the [window that opens](https://i.imgur.com/9ZOWMEX.png), you'll see the
key pair `ID`s, the first one (or rather the one has `Sign` listed in `Usage` 
column) is the one you need. You can click on the ID and
`ctrl+c` to copy it.

#### Console or (local/global) git config text file

Now you have a choice:

* In case you are a billion percent sure you'll never ever use any other
VCS service other than github, feel free to add the `--global` flag
(like this `git config --global`) to this command
* In case you use different VCS services (such as gitlab, bitbucket or your 
companys' own) you may want to do this per project

1. Open console
2. Type `git config user.signingkey 96B8A72857ADEC06` (supply your own key!) 
and hit enter
   1. From now on you can sign your commits with this key if you so choose
   2. you can do this automatically however, so onto step:
3. Type `git config commit.gpgsign true`
   1. This will make any future commit be auto signed by your GPG key

#### Gotchas

The other thing the tutorials (I found) out there won't tell you, is that in
case you encounter an error similar to 
`gpg: skipping "96B8A72857ADEC06": secret key not available` you need to do
one more console command, you might as well make this one global

`git config --global gpg.program path_to_gpg`

You can find the `path_to_gpg` using `where gpg` and just copy-paste it. This
happens only _sometimes_ apparently, but I wasted good hour on this.

In case the error persist, but instead of the key ID it stating your `user.name`
and/or `user.email`, you misconfigured something (a typo somwhere maybe?) :)

Commits may `warn` you about insecure memory usage, you can ignore these.

From GnuPG FAQ:

> GnuPG tries to lock memory so that no other process can see it and so that 
the memory will not be written to swap. If for some reason it’s not able to 
do this (for instance, certain platforms don’t support this kind of memory 
locking), GnuPG will warn you that it’s using insecure memory.
> 
> While it’s almost always better to use secure memory, it’s not necessarily 
a bad thing to use insecure memory. If you own the machine and you’re 
confident it’s not harboring malware, then this warning can probably be 
ignored.