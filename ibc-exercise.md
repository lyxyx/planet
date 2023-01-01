**1.**   用ignite生成一条新的区块链名字叫planet。

```
ignite scaffold chain planet --no-module
```

**2.**  使用ignite生成一个Blog的模块，并且集成IBC。

```
ignite scaffold module blog --ibc

modify app/app.go
modify proto/planet/blog/genesis.proto
create proto/planet/blog/packet.proto
modify testutil/keeper/blog.go
modify x/blog/genesis.go
create x/blog/module_ibc.go
create x/blog/types/events_ibc.go
modify x/blog/types/genesis.go
modify x/blog/types/keys.go

🎉 Module created blog.

```

**3.** 给blog模块添加针对博文（post）的增删改查。

```
ignite scaffold list post title content creator --no-message --module blog

modify proto/planet/blog/genesis.proto
create proto/planet/blog/post.proto
modify proto/planet/blog/query.proto
modify x/blog/client/cli/query.go
create x/blog/client/cli/query_post.go
create x/blog/client/cli/query_post_test.go
modify x/blog/genesis.go
modify x/blog/genesis_test.go
create x/blog/keeper/grpc_query_post.go
create x/blog/keeper/grpc_query_post_test.go
create x/blog/keeper/post.go
create x/blog/keeper/post_test.go
modify x/blog/types/genesis.go
modify x/blog/types/genesis_test.go
modify x/blog/types/keys.go

🎉 post added. 

```

**4.** 添加已发生成功博文（sentPost）的增删改查。

```
ignite scaffold list sentPost postID title chain creator --no-message --module blog

modify proto/planet/blog/genesis.proto
modify proto/planet/blog/query.proto
create proto/planet/blog/sent_post.proto
modify x/blog/client/cli/query.go
create x/blog/client/cli/query_sent_post.go
create x/blog/client/cli/query_sent_post_test.go
modify x/blog/genesis.go
modify x/blog/genesis_test.go
create x/blog/keeper/grpc_query_sent_post.go
create x/blog/keeper/grpc_query_sent_post_test.go
create x/blog/keeper/sent_post.go
create x/blog/keeper/sent_post_test.go
modify x/blog/types/genesis.go
modify x/blog/types/genesis_test.go
modify x/blog/types/keys.go

🎉 sentPost added. 

```

**5.** 添加发送超时博文（timeoutPost）的增删改查。

```
ignite scaffold list timedoutPost title chain creator --no-message --module blog

modify proto/planet/blog/genesis.proto
modify proto/planet/blog/query.proto
create proto/planet/blog/timedout_post.proto
modify x/blog/client/cli/query.go
create x/blog/client/cli/query_timedout_post.go
create x/blog/client/cli/query_timedout_post_test.go
modify x/blog/genesis.go
modify x/blog/genesis_test.go
create x/blog/keeper/grpc_query_timedout_post.go
create x/blog/keeper/grpc_query_timedout_post_test.go
create x/blog/keeper/timedout_post.go
create x/blog/keeper/timedout_post_test.go
modify x/blog/types/genesis.go
modify x/blog/types/genesis_test.go
modify x/blog/types/keys.go

🎉 timedoutPost added. 

```

**6.** 添加IBC发送数据包和确认数据包的结构。

```
ignite scaffold packet ibcPost title content --ack postID --module blog

```

**7.** 创建一个新的ibc数据包，名称为updatePost, 可更新Post（通过postID）的标题和内容

```
ignite scaffold packet updatePost postID title content --ack postID --module blog

```
  
**8.** 在proto/blog/packet.proto目录下修改`IbcPostPacketData` 和 `UpdatePostPacketData` ，添加创建人`Creator`， 并重新编译proto文件。在x/blog/keeper/msg_server_ibc_post.go。编译完成后在x/blog/keeper/msg_server_ibc_post.go中发送数据包前更新`Creator`。

```
ignite chain build
```

**9.** 修改keeper方法中的`OnRecvIbcPostPacket` & `OnRecvUpdatePostPacket`。

```
ibc_post.go 

id := k.AppendPost(
        ctx,
        types.Post{
            Creator: packet.SourcePort + "-" + packet.SourceChannel + "-" + data.Creator,
            Title:   data.Title,
            Content: data.Content,
        },
    )

    packetAck.PostID = strconv.FormatUint(id, 10)

```

```
update_post.go
	Id, err := strconv.ParseUint(data.PostID, 10, 64)
	if err != nil {
		return packetAck, errors.New("PostID is Error")
	}

	_, found := k.GetPost(ctx, Id)
	if !found {
		return packetAck, errors.New("Blog is not exit!")
	}

	// update
	k.SetPost(
		ctx,
		types.Post{
			Id:      Id,
			Creator: packet.SourcePort + "-" + packet.SourceChannel + "-" + data.Creator,
			Title:   data.Title,
			Content: data.Content,
		},
	)

```

**10.** 修改keeper方法中的`OnAcknowledgementIbcPostPacket ` & `OnAcknowledgementUpdatePostPacket`。

```
k.AppendSentPost(
            ctx,
            types.SentPost{
                Creator: data.Creator,
                PostID:  packetAck.PostID,
                Title:   data.Title,
                Chain:   packet.DestinationPort + "-" + packet.DestinationChannel,
            },
        )
```

**11.** 修改keeper方法中的`OnTimeoutIbcPostPacket ` & `OnTimeoutUpdatePostPacket`。

```
k.AppendTimedoutPost(
        ctx,
        types.TimedoutPost{
            Creator: data.Creator,
            Title:   data.Title,
            Chain:   packet.DestinationPort + "-" + packet.DestinationChannel,
        },
    )
```

**12.** 添加链启动的配置文件。

```
# earth.yml
accounts:
  - name: alice
    coins: ["1000token", "100000000stake"]
  - name: bob
    coins: ["500token", "100000000stake"]
validator:
  name: alice
  staked: "100000000stake"
faucet:
  name: bob
  coins: ["5token", "100000stake"]
genesis:
  chain_id: "earth"
init:
  home: "$HOME/.earth"
  
# mars.yml
accounts:
  - name: alice
    coins: ["1000token", "1000000000stake"]
  - name: bob
    coins: ["500token", "100000000stake"]
validator:
  name: alice
  staked: "100000000stake"
faucet:
  host: ":4501"
  name: bob
  coins: ["5token", "100000stake"]
host:
  rpc: ":26659"
  p2p: ":26658"
  prof: ":6061"
  grpc: ":9092"
  grpc-web: ":9093"
  api: ":1318"
genesis:
  chain_id: "mars"
init:
  home: "$HOME/.mars"
```


**13.** 分别启动两条链

```
ignite chain serve -c earth.yml

ignite chain serve -c mars.yml
```

**14.** 启动中继器

```
rm -rf ~/.ignite/relayer

ignite relayer configure -a \
  --source-rpc "http://0.0.0.0:26657" \
  --source-faucet "http://0.0.0.0:4500" \
  --source-port "blog" \
  --source-version "blog-1" \
  --source-gasprice "0.0000025stake" \
  --source-prefix "cosmos" \
  --source-gaslimit 300000 \
  --target-rpc "http://0.0.0.0:26659" \
  --target-faucet "http://0.0.0.0:4501" \
  --target-port "blog" \
  --target-version "blog-1" \
  --target-gasprice "0.0000025stake" \
  --target-prefix "cosmos" \
  --target-gaslimit 300000

ignite relayer connect
```

**15.** 从earth链向mars链发送博文数据包（注意修改channel id）

```
planetd tx blog send-ibc-post blog channel-4 "Hello" "Hello Mars, I'm Alice from Earth" --from alice --chain-id earth --home ~/.earth
```

**16.** 通过rpc查询验证结果。

```
planetd q blog list-post --node tcp://localhost:26659

planetd q blog list-sent-post
```

**17.** 修改blog，和内容Blog id = 0 记录的标题和内容

```
planetd tx blog send-update-post blog channel-0 "0" "HelloWorld" "Hello Mars, I'm Alice from Earth..." --from alice --chain-id earth --home ~/.earth
```

**18.** 通过rpc查询验证结果。

```
planetd q blog list-post --node tcp://localhost:26659

planetd q blog list-sent-post
```

**19.** 截图参考

[nodes](https://raw.githubusercontent.com/lyxyx/planet/master/nodes.png)

[add post](https://raw.githubusercontent.com/lyxyx/planet/master/add_post.png)

[update post](https://raw.githubusercontent.com/lyxyx/planet/master/update-post.png)
