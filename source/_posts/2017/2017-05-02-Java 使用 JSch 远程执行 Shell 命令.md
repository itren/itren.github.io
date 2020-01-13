---
title: Java 使用 JSch 远程执行 Shell 命令
categories:
  - java
tags:
  - shell
abbrlink: 445dd1c3
date: 2017-05-01 23:29:53
updated: 2020-01-04 13:06:06
---

最近需要使用远程执行 Shell 命令，网上也有很多教程，但一般都是远程分发文件或者需要实现 UserInfo 接口，感觉都不够简介或者不满足我的需求。经过上网查询、翻阅官网终于发现可以实现我想要的功能了。

参考：http://wiki.jsch.org/index.php?Manual%2FExamples%2FJschExecExample

<!--more-->

## 添加Maven依赖

```
    <dependency>
                <groupId>com.jcraft</groupId>
                >artifactId>jsch</artifactId>
                <version>0.1.53</version>
            </dependency>
```

## 代码

```
    package quartz;

    import com.jcraft.jsch.Channel;
    import com.jcraft.jsch.ChannelExec;
    import com.jcraft.jsch.JSch;
    import com.jcraft.jsch.Session;

    import java.io.InputStream;

    /**
     * Created by chenxl on 2017/5/26.
     */
    public class Samplejsch {

        public static void main(String[] args) {

            try{
                JSch jsch=new JSch();

                String host=System.getProperty("user.name") + "@localhost";
                if(args.length>0){
                    host=args[0];
                }
                String user=host.substring(0, host.indexOf('@'));

                host=host.substring(host.indexOf('@')+1);

                user = "chenxl";
                host = "localhost";
                Session session=jsch.getSession(user, host, 22);

                //jsch.setKnownHosts("/home/anand/.ssh/known_hosts");
                //jsch.addIdentity("/home/anand/.ssh/id_rsa");

                // If two machines have SSH passwordless logins setup, the following line is not needed:
                session.setPassword("12345678");
                session.setConfig("StrictHostKeyChecking", "no");
                session.connect();

                String command="ps -a\n";
                // command=args[1];

                Channel channel=session.openChannel("exec");
                ((ChannelExec)channel).setCommand(command);

                //channel.setInputStream(System.in);
                channel.setInputStream(null);

                ((ChannelExec)channel).setErrStream(System.err);

                InputStream in=channel.getInputStream();

                channel.connect();
                byte[] tmp=new byte[1024];
                while(true){
                    while(in.available()>0){
                        int i=in.read(tmp, 0, 1024);
                        if(i<0)break;
                        System.out.print(new String(tmp, 0, i));
                    }
                    if(channel.isClosed()){
                        System.out.println("exit-status: "+channel.getExitStatus());
                        break;
                    }
                    try{Thread.sleep(1000);}catch(Exception ee) {ee.printStackTrace();}
                }
                channel.disconnect();
                session.disconnect();
            }
            catch(Exception e){
                System.out.println(e);
            }
        }       //end main

    }
```

## 结果

![](https://itgrocery.cn/2017/media/15781144297624.jpg)