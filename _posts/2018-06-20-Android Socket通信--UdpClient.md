---
layout:     post                    # 使用的布局（不需要改）
title:     Android Socket通信    # 标题 
subtitle:  UdpClient        #副标题
date:       2018-06-20               # 时间
author:     Jack                      # 作者
header-img: img/post-bg-os-metro.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - Socket
    - Udp
    
---


# Android Socket通信--UdpClient

 UdpClientConnecter

 

```java
package com.hadisi.socket;

import android.os.Bundle;
import android.os.Handler;
import android.os.Message;

import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;

public class UdpClientConnector {
    private static UdpClientConnector mUdpClientConnector;
    private ConnectLinstener mListener;
    private Thread mSendThread;

    private byte receiveData[] = new byte[1024];
    private String mSendHexString;

    private boolean isSend = false;

    public interface ConnectLinstener {
        void onReceiveData(String data);
    }

    public void setOnConnectLinstener(ConnectLinstener linstener) {
        this.mListener = linstener;
    }

    public static UdpClientConnector getInstance() {
        if (mUdpClientConnector == null) {
            mUdpClientConnector = new UdpClientConnector();
        }
        return mUdpClientConnector;
    }

    Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case 1000:
                    if (mListener != null) {
                        mListener.onReceiveData(msg.getData().getString("data"));
                    }
                    break;
            }
        }
    };

    /**
     * 创建udp发送连接（服务端ip地址、端口号、超时时间）
     *
     * @param ip
     * @param port
     * @param timeOut
     */
    public void createConnector(final String ip, final int port, final int timeOut) {
        if (mSendThread == null) {
            mSendThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {
                        if (!isSend)
                            continue;
                        DatagramSocket socket = null;
                        try {
                            socket = new DatagramSocket();
                            socket.setSoTimeout(timeOut);
                            InetAddress serverAddress = InetAddress.getByName(ip);
                            byte data[] = mSendHexString.getBytes("utf-8");
                            DatagramPacket sendPacket = new DatagramPacket(data, data.length, serverAddress, port);
                            DatagramPacket receivePacket = new DatagramPacket(receiveData, receiveData.length);
                            socket.send(sendPacket);
                            socket.receive(receivePacket);
                            Message msg = new Message();
                            msg.what = 1000;
                            Bundle bundle = new Bundle();
                            bundle.putString("data", new String(receivePacket.getData(),receivePacket.getOffset(),receivePacket.getLength()));
                            msg.setData(bundle);
                            mHandler.sendMessage(msg);
                            socket.close();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                        isSend = false;
                    }
                }
            });
            mSendThread.start();
        }
    }

    /**
     * 发送数据
     *
     * @param str
     */
    public void sendStr(final String str) {
        mSendHexString = str;
        isSend = true;
    }
}

```

### 怎么用它？

- 获取UdpClientConnector单利对象

  ```java
  udpClientConnector = UdpClientConnector.getInstance();
  ```

  

- 创建udp发送连接（服务端ip地址、端口号、超时时间）；**用udp广播通信ip为"255.255.255.255"**

  ```java
  udpClientConnector.createConnector("服务端ip地址",服务器端端口号,超时时间);
  ```

- 实现接收数据接口OnConnectLinstener,onReceiveData(String data)中的data就是服务端返回的数据；

 

```java
udpClientConnector.setOnConnectLinstener(new UdpClientConnector.ConnectLinstener() {
    @Override
    public void onReceiveData(String data) {
        //do somethings
    }
});
```

