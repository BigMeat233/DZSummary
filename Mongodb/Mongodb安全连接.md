# Mongodb安全连接

## <font color="red">TLS使用的所有证书必须是v3证书</font>

## <font color="red">证书签发需要使用openssl，详情参见[使用openssl构造证书链](../Common/使用openssl构造证书链.md)</font>

## 关于Membership Authenticate（集群内身份校验）

1. 启动TLS模式必须配置```net.tls.mode```和```net.tls.certificateKeyFile```
2. Replica/Cluster进行身份校验时，本成员需要向其他成员出示```net.tls.clusterFile```，
   并使用```net.tls.CAFile```验证其他成员出示的```net.tls.clusterFile```
3. 即```net.tls.clusterFile```为```net.tls.CAFile```中的末端证书（一般为中间证书）签发时才能通过身份验证
4. 因为Membership Authenticate是相互的，
   所以Replica/Cluster每个成员上都需要配置```mode```、```certificateKeyFile```、```clusterFile```和```CAFile```

## 关于Server与Client的TLS

（Client端使用的field在不同的Driver上有差异，下述的field为mongo使用）

1. 启动TLS模式需要Server端配置```net.tls.mode```和```net.tls.certificateKeyFile```，Client端配置```tlsCAFile```
   >
   > 若Server端配置了```net.tls.CAFile```，
   > 则Client端进行连接时必须带上```tlsCertificateKeyFile```

2. 进行TLS连接时，在Client端使用```tlsCAFile```验证Server端出示的```net.tls.certificateKeyFile```是否有效
   >
   > 若Server端配置了```net.tls.CAFile```，
   > 则Server端还会使用```net.tls.CAFile```验证Client端出示的```tlsCertificateKeyFile```是否有效

3. 即Server端的```net.tls.certificateKeyFile```为Client端的```tlsCAFile```中的末端证书（一般为中间证书）签发时才能成功建立TLS连接
   >
   > 若Server端配置了```net.tls.CAFile```，
   > 即Client端的```tlsCertificateKeyFile```为Server端的```net.tls.CAFile```中的末端证书（一般为中间证书）签发时才能成功建立TLS连接

## 关于net.tls.certificateKeyFile

（Client端使用的field在不同的Driver上有差异，下述的field为mongo使用）  

1. 用途：用于Server与Client建立TLS连接
   （此文件与下述的```net.tls.clusterFile```都是由业务证书和私钥聚合而成，但用途不同，其中包含的证书和私钥也可能不同，需注意区分）
2. 由证书（仅业务证书，不包含证书链）、私钥聚合而成
3. 在Server与Client进行TLS连接时，Server将出示此证书，Client端将会使用```tlsCAFile```对此证书的有效性进行检验
4. 即Server端的```net.tls.certificateKeyFile```为Client端的```tlsCAFile```中的末端证书（一般为中间证书）签发时才能成功建立TLS连接

## 关于net.tls.clusterFile

（Client端使用的field在不同的Driver上有差异，下述的field为mongo使用）  

1. 用途：Replica/Cluster进行Membership Authenticate时对成员机器的身份进行验证
   （此文件与与上述的```net.tls.certificateKeyFile```都是由业务证书和私钥聚合而成，但用途不同，其中包含的证书和私钥也可能不同，需注意区分）
2. 由证书（仅业务证书，不包含证书链）、私钥聚合而成
3. 在Replica/Cluster成员机器通讯时，
   需要进行Membership Authenticate，此时本成员需要向其他成员出示```net.tls.clusterFile```，
   并使用```net.tls.CAFile```验证其他成员出示的```net.tls.clusterFile```
4. 即```net.tls.clusterFile```为```net.tls.CAFile```中的末端证书（一般为中间证书）签发时才能通过Authenticate

## 关于net.tls.CAFile

1. 由根证书、中间证书聚合而成（聚合时可以乱序）
2. 对于server端来讲，所有收到的证书，都将使用CAFile来验证证书的有效性
   > server收到的证书可能为：server端**replica/cluster进行membership校验**或**与client建立连接**时，其他成员或client出示的证书
3. 即所有server收到的证书，为```CAFile```中的末端证书（一般为中间证书）签发时才可通过校验
