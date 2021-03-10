# Let’s test all this

The CloudFormation template also launches a t3.small EC2 instance so that we can test. On this instance, a private key is generated during launch and its public key is used to update the sftpuser user.
The stack also mounts the EFS drive on it so we can check everything is working correctly.
As the VPC is completely private, we connect to the Instance via AWS Session Manager.
Select EC2 as service and go to the 2 EC2 Dashboard. You can use Filter field to filter the instance and select sfpclient one and click on Connect
<p>
Select Session Manager tab and click on Connect
<p>

Let’s sudo testuser & then check EFS is mounted
<p>
sudo su - testuser
df -k

Let’s create a test file & connect to the SFTP  & upload the file.
The template will have created a sftp_key file so let’s use it.
We can also check our home directory.<p>

[testuser@ip-10-1-0-142 ~]$ echo "this is a test file" > /tmp/testfile.txt <p>
[testuser@ip-10-1-0-142 ~]$ sftp -i sftp_key sftpuser@mysftp.myexample.com<p>
Connected to mysftp.myexample.com.<p>
sftp> pwd<p>
Remote working directory: /mysftp-accountideu-west-1/root<p>
sftp> ls<p>
sftp> mput /tmp/testfile.txt<p>
Uploading /tmp/testfile.txt to /mysftp-accountideu-west-1/root/testfile.txt
/tmp/testfile.txt                                                                                              100%   20     3.5KB/s   00:00<p>
sftp> ls<p>
testfile.txt<p>
sftp> quit<p>

then let’s disconnect

And  if we go on the EFS drive we can see the file has been copied

[testuser@ip-10-1-0-142 ~]$ cd /mnt/efs<p>
[testuser@ip-10-1-0-142 efs]$ ls<p>
root<p>
[testuser@ip-10-1-0-142 efs]$ cd root<p>
[testuser@ip-10-1-0-142 root]$ ls<p>
testfile.txt<p>
[testuser@ip-10-1-0-142 root]$ ls -l<p>

total 4<p>
-rw-rw-r-- 1 testuser testgroup 20 Dec  1 13:50 testfile.txt<p>
[testuser@ip-10-1-0-142 root]$ cat testfile.txt<p>
this is a test file<p>

And this concludes our test !
