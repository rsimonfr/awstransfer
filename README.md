Let’s test all this

The CloudFormation template also launches a t3.small EC2 instance so that we can test. On this instance, a private key is generated during launch and its public key is used to update the sftpuser user.
The stack also mounts the EFS drive on it so we can check everything is working correctly.
As the VPC is completely private, we connect to the Instance via AWS Session Manager.
Select EC2 as service and go to the 2 EC2 Dashboard. You can use Filter field to filter the instance and select sfpclient one and click on Connect

Select Session Manager tab and click on Connect


Let’s sudo testuser & then check EFS is mounted
sudo su - testuser
df -k

Let’s create a test file & connect to the SFTP  & upload the file.
The template will have created a sftp_key file so let’s use it.
We can also check our home directory.

[testuser@ip-10-1-0-142 ~]$ echo "this is a test file" > /tmp/testfile.txt
[testuser@ip-10-1-0-142 ~]$ sftp -i sftp_key sftpuser@mysftp.myexample.com
Connected to mysftp.myexample.com.
sftp> pwd
Remote working directory: /mysftp-854859737127eu-west-1/root
sftp> ls
sftp> mput /tmp/testfile.txt
Uploading /tmp/testfile.txt to /mysftp-854859737127eu-west-1/root/testfile.txt
/tmp/testfile.txt                                                                                              100%   20     3.5KB/s   00:00
sftp> ls
testfile.txt
sftp> quit


then let’s disconnect
And  if we go on the EFS drive we can see the file has been copied

[testuser@ip-10-1-0-142 ~]$ cd /mnt/efs
[testuser@ip-10-1-0-142 efs]$ ls
root
[testuser@ip-10-1-0-142 efs]$ cd root
[testuser@ip-10-1-0-142 root]$ ls
testfile.txt
[testuser@ip-10-1-0-142 root]$ ls -l
total 4
-rw-rw-r-- 1 testuser testgroup 20 Dec  1 13:50 testfile.txt
[testuser@ip-10-1-0-142 root]$ cat testfile.txt
this is a test file
[testuser@ip-10-1-0-142 root]$

And this concludes our test !
