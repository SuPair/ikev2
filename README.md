# ikev2
anti-gfw for windows phone




## 一：安装strongswan

    apt-get install build-essential     #编译环境
    aptitude install libgmp10 libgmp3-dev libssl-dev pkg-config libpcsclite-dev libpam0g-dev     #编译所需要的软件

    wget http://download.strongswan.org/strongswan-5.2.0.tar.bz2
    tar -jxvf strongswan-5.2.0.tar.bz2
    cd strongswan-5.2.0
    ./configure --prefix=/usr --sysconfdir=/etc  --enable-openssl --enable-nat-transport --disable-mysql --disable-ldap  --disable-static --enable-shared --enable-md4 --enable-eap-mschapv2 --enable-eap-aka --enable-eap-aka-3gpp2  --enable-eap-gtc --enable-eap-identity --enable-eap-md5 --enable-eap-peap --enable-eap-radius --enable-eap-sim --enable-eap-sim-file --enable-eap-simaka-pseudonym --enable-eap-simaka-reauth --enable-eap-simaka-sql --enable-eap-tls --enable-eap-tnc --enable-eap-ttls
    
	make 
	
	make install

## 二：生成、安装证书
1：win7和Android、wp8.1等平台的VPN客户端走ikev2协议，需要制作相应的证书，先生成ca证书

    ipsec pki --gen --outform pem > caKey.pem
    ipsec pki --self --in caKey.pem --dn "C=CN, O=strongSwan, CN=strongSwan CA" --ca --outform pem > caCert.pem
2：然后是服务器端的证书

    ipsec pki --gen --outform pem > serverKey.pem
    ipsec pki --pub --in serverKey.pem | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "C=CN, O=strongSwan, CN=VPS的公网ip或域名" --san="VPS的公网ip或域名" --flag serverAuth --flag ikeIntermediate --outform pem > serverCert.pem
3：客户端的证书

    ipsec pki --gen --outform pem > clientKey.pem
    ipsec pki --pub --in clientKey.pem | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "C=CN, O=strongSwan, CN=client" --outform pem > clientCert.pem
生成的客户端证书 clientCert.pem 不能直接导入到win7或Anroid设备中，需先转换为.p12格式。执行后会提示要设置证书使用密码，可以设置一下密码也可以直接回车（密码为空）。

    openssl pkcs12 -export -inkey clientKey.pem -in clientCert.pem -name "client" -certfile caCert.pem -caname "strongSwan CA" -out clientCert.p12
4：安装证书

    cp caCert.pem /etc/ipsec.d/cacerts/
    cp serverCert.pem /etc/ipsec.d/certs/
    cp serverKey.pem /etc/ipsec.d/private/
客户端安装caCert.pem与clientCert.pem（clientCert.p12）

## 三：配置strongswan
1:  /etc/ipsec.conf

    config setup
        strictcrlpolicy=no
        uniqueids=no #允许多设备同时在线

    conn windowsphone
        keyexchange=ikev2
        ike=aes256-sha1-modp1024!
        esp=aes256-sha1!
        dpdaction=clear
        dpddelay=300s
        rekey=no
        left=%defaultroute
        leftsubnet=0.0.0.0/0
        leftauth=pubkey
        leftcert=serverCert.pem
        leftid="C=CN, O=strongSwan, CN=X.X.X.X" #C=国家，CN=自己vps的公网ip
        right=%any
        rightsourceip=10.11.1.0/24 #为客户端分配的虚拟地址池
        rightauth=eap-mschapv2
        rightsendcert=never
        eap_identity=%any
        auto=add
    
2: /etc/ipsec.secrets

    : RSA serverKey.pem
    用户名1 : EAP "密码1"
    wp设备名称\用户名2 : EAP "密码2"  #仅对windowsphone8.1设备
    #windowsphone8.1，在客户端输入的用户名发送到服务器显示为“设备名称\用户名”的形式，故认证需加上设备名称,设备名限制15字符

3: /etc/strongswan.conf

    #加入分配的dns
    charon {

            dns1 = 8.8.8.8
            dns2 = 208.67.222.222

    }

## 四：配置 Iptables 转发

    iptables -A INPUT -p udp --dport 500 -j ACCEPT
    iptables -A INPUT -p udp --dport 4500 -j ACCEPT
    echo 1 > /proc/sys/net/ipv4/ip_forward
    iptables -t nat -A POSTROUTING -s 10.11.1.0/24 -o eth0 -j MASQUERADE  #地址与上面地址池对应
    iptables -A FORWARD -s 10.11.1.0/24 -j ACCEPT     #同上
    #为避免VPS重启后NAT功能失效，可以把如上5行命令添加到 /etc/rc.local 文件中，添加在exit那一行之前即可。

## 五: 确保vpn持续运行

	crontab -e     #进入定时任务编辑状态
	在其中添加如下的一条定时任务，并保存退出。
	00 20 * * * /usr/sbin/ipsec restart
	修改完成后，需要重启一下定时任务的程序。
	/etc/init.d/cron restart

## 最后，启动strongswan:

    ipsec restart 
滚动日志:

    ipsec restart --nofork

## 参考链接：
* http://zh.opensuse.org/index.php?title=SDB:Setup_Ipsec_VPN_with_Strongswan&variant=zh
* http://si-you.com/?p=1167
* http://blog.ltns.info/linux/pure_ipsec_multi-platform_vpn_client_debian_vps/
* https://gist.github.com/losisli/11081793
