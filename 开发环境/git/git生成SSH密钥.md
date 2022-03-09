# git生成SSH密钥
git不再支持密码push,为自己的电脑创建SSH密钥,可实现无需密码push  

1. 用命令行生成密钥  
```
C:\Users\Administrator>
ssh-keygen -t rsa -C "18817366719@163.com"
```
连续按下3次回车  

2. 此时在 C:\Users\Administrator\.ssh 路径下会产生两个文件id-rsa和id_rsa.pub  
用记事本打开id_rsa.pub,复制里面的内容,在[github密钥界面](https://github.com/settings/keys)粘贴,创建密钥  

3. 设置URL  
```
git remote set-url origin git@github.com:solairewrite/Notes.git
```

4. git push  
第一次需要建立连接  
```
The authenticity of host 'github.com (20.205.243.166)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Please type 'yes', 'no' or the fingerprint: yes
Warning: Permanently added 'github.com,20.205.243.166' (RSA) to the list of known hosts.

```
输入yes回车,在路径下会生成known_hosts  

以后这台电脑就可以直接提交了  

5. 连接验证  
```
ssh -T git@github.com

// 连接到腾讯工蜂
ssh -T git@git.code.tencent.com
```

6. [SourceTree配置SSH](https://jingyan.baidu.com/article/9faa7231cdec65473d28cb11.html)  

7. 腾讯工蜂测试连接失败  
进入`C:\Users\Administrator\.ssh`文件夹,创建config文件  
```
Host *
HostkeyAlgorithms +ssh-rsa
PubkeyAcceptedKeyTypes +ssh-rsa
```
