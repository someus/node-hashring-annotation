一致性哈希的一个node库。

源码地址：https://github.com/3rd-Eden/node-hashring

虚拟节点：vnodes


设计一致性哈希的一般思路：

设key的哈希映射范围是0~2^32，机器k的权重为W(k)和虚拟节点数量为N(k)（所以这台机器中每个虚拟节点的权重是W(k)/N(k)喽）。每个虚拟节点的标识是`主机序号+虚拟节点序号`，做个哈希，排序，此时虚拟节点被打乱了。根据vnode的权重依次给其分配逐渐增大的序号（范围是0~2^32）。

对于某个key，哈希后判断该将其放到哪个虚拟节点，进而判断该放进哪个真实主机中。

若vnode有备份vnode，思路和上面类似，每个虚拟节点的标识是`主机序号+虚拟节点序号+是否是备份vnode+备份序号`。


`node-hashring`的思路更加简洁。

先看如下代码：
```js
  index = 0;
  self.ring = []
  // ....
  servers.forEach(function each(server) {
    var percentage = server.weight / total
      , vnodes = self.vnodes[server.string] || self.vnode
      , length = Math.floor(percentage * vnodes * servers.length)
      , key
      , x;

    // If you supply us with a custom vnode size, we will use that instead of
    // our computed distribution.
    if (vnodes !== self.vnode) length = vnodes;

    for (var i = 0; i < length; i++) {
      if (self.defaultport && server.port === self.defaultport) {
        x = self.digest(server.host +'-'+ i);
      } else {
        x = self.digest(server.string +'-'+ i);
      }

      for (var j = 0; j < self.replicas; j++) {
        key = hashValueHash(x[3 + j * 4], x[2 + j * 4], x[1 + j * 4], x[j * 4]);
        self.ring[index] = new Node(key, server.string);
        index++;
      }
    }
  });
```

首先根据权重、vnodes数量、主机数量（固定值）得到length，权重和length正比，vnodes数量和length正比。self.ring用来存放所有vnodes的信息，但从代码上看vnodes的数量和给定的vnodes的值并不一致。不过因为vnodes值和length值成正比，所以也可以看作比较合理。