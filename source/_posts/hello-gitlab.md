title: Hello GitLab
---

In this post, I am going to markdown how I build up this blog with ci and custom my own domain by using [GitLab](https://www.gitlab.com) .

## 1. Setup blog environment

### Install the package

``` bash
# apt update -y && apt install -y nodejs npm git
```

### Confirm the version of tools

``` bash
# nodejs -v
v8.10.0
# npm -v
3.5.2
# git --version
git version 2.17.1
```

### Install the hexo

``` bash
# npm install hexo -g
```

### Confirm the version of hexo

``` bash
# hexo -v
hexo-cli: 2.0.0
os: Linux 4.15.0-20-generic linux x64
http_parser: 2.7.1
node: 8.10.0
v8: 6.2.414.50
uv: 1.18.0
zlib: 1.2.11
ares: 1.14.0
modules: 57
nghttp2: 1.30.0
openssl: 1.0.2n
icu: 60.2
unicode: 10.0
cldr: 32.0.1
tz: 2017c
```

### Initialize the directory

``` bash
# mkdir m4d3bug.gitlab.io
# cd m4d3bug.gitlab.io/
~/m4d3bug.gitlab.io# hexo init
npm WARN optional Skipping failed optional dependency /chokidar/fsevents:
npm WARN notsup Not compatible with your operating system or architecture: fsevents@1.2.9
INFO  Start blogging with Hexo!
```

## 2. Setup git repository

### New the remote repository

![](https://i.loli.net/2019/08/10/priZIByc4Tfks7S.png)

![](https://i.loli.net/2019/08/10/2Qau7YVb5IGfjNH.png)

### New the local repository

``` bash
~/m4d3bug.gitlab.io# git init
Initialized empty Git repository in /root/m4d3bug.gitlab.io/.git/
~/m4d3bug.gitlab.io# git remote add origin git@gitlab.com:m4d3bug/m4d3bug.gitlab.io.git
~/m4d3bug.gitlab.io# git remote -v
origin  git@gitlab.com:m4d3bug/m4d3bug.gitlab.io.git (fetch)
origin  git@gitlab.com:m4d3bug/m4d3bug.gitlab.io.git (push)
```

### 3. Setup the access right in the GitLab

#### Generate local SSH keys and public keys

``` bash
~/m4d3bug.gitlab.io# ssh-keygen -t rsa -b 4096 -C "m4d3bug@ubuntu" -N ""
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:dGucfhOAOXbHYm@@@@@@@@@@@@@@@/L@@@@@@@@@@@@ m4d3bug@ubuntu
The key's randomart image is:
+---[RSA 4096]----+
|          .++.   |
|         + .  .  |
|        B * oo . |
|       =E@ *= O  |
|       .S.=..* * |
|       o.oo o.=  |
|         ..+oB   |
|          o.=.o  |
|        .o o..   |
+----[SHA256]-----+
~/m4d3bug.gitlab.io# cat /root/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1y.../iiF@@@@@@@@@@@@@@@@@@@@@== m4d3bug@ubuntu
```

#### Upload the public key

![](https://i.loli.net/2019/08/10/grnETp16mayWYlw.png)

#### Update the policy of yourself

![](https://i.loli.net/2019/08/10/UGSMXCbTFYlA1yh.png)

## 4. Start up the first commit

``` bash
~/m4d3bug.gitlab.io# git checkout -b beta
Switched to a new branch 'beta'
~/m4d3bug.gitlab.io# git config --global user.email "m4d3bug@gmail.com"
~/m4d3bug.gitlab.io# git config --global user.name "m4d3bug"
~/m4d3bug.gitlab.io# git add .
~/m4d3bug.gitlab.io# git commit -m "Init Commit"
~/m4d3bug.gitlab.io# git push --set-upstream origin beta
```

##### Now you can see that the remote repository has created a new branch called beta and includes the above files.

![](https://i.loli.net/2019/08/10/OvHGo1j3MutW7rR.png)

## 5. Setup the pipeline

### Create the .gitlab-ci.yml

``` yaml
image: node:6.10.2

cache:
    paths:
        - node_modules/

variables:
    GIT_SUBMODULE_STRATEGY: recursive

before_script:
    - npm install hexo-cli -g
    - npm install

pages:
    script: 
        - hexo generate
    artifacts:
        paths:
            - public
    only:
#       - master
        - beta
```





```
git checkout master
git merge beta --allow-unrelated-histories

```

 



