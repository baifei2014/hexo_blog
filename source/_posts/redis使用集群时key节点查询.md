---
title: redis使用集群时key节点查询
tags: [redis, cluster, hashslot]
date: 2021-03-29 16:15:33
---

### 引言

上周，同事遇到一个问题，在使用redis时发现请求对应的key时有时无。因为是在lua脚本中使用的，一开始以为是语法问题，不管怎么调都不管用。最后问题终于解决了，原来是redis集群的问题，请求每次被随机分配到不同主机上，但是设置的键值只在其中一台机器上，所以请求时会出现有时查到，有时查不到的问题。如何解决网上也给出了方案，就是确保将key分配到同一个卡槽里。因为我平常也有阅读redis源码的兴趣，所以这次就深入了解一下这块的实现原理。

### redis集群简介

redis集群是在多个节点间进行数据共享的结构，使用数据分片来实现。redis将整个集群划分为`2^14`(16384)个槽，每个node节点负责一部分范围的哈希槽。在操作键时，通过公式公式`CRC16(key)%16384`来计算键key属于哪个槽，如果是所在节点的槽，直接处理，如果是其它节点的槽，返回重定向信息，由目标节点处理。这就是redis集群的基本实现原理。

### 集群节点查询

在集群中操作键时，首先要思考第一个问题，当前节点分配的哈希槽是不是请求中键所在的哈希槽。所以第一步，首先要根据请求中的参数判断，实现代码如下：

```c
/* 如果集群已启用，这里执行集群重定向.
 * 如果是以下两种情况不执行重定向:
 * 1) 这个命令是master节点发出的.
 * 2) 命令中没有key参数. */
if (server.cluster_enabled &&
    !(c->flags & CLIENT_MASTER) &&
    !(c->flags & CLIENT_LUA &&
      server.lua_caller->flags & CLIENT_MASTER) &&
    !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
      c->cmd->proc != execCommand))
{
    int hashslot;
    int error_code;
    clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                    &hashslot,&error_code);
    if (n == NULL || n != server.cluster->myself) {
        if (c->cmd->proc == execCommand) {
            discardTransaction(c);
        } else {
            flagTransaction(c);
        }
        clusterRedirectClient(c,n,hashslot,error_code);
        return C_OK;
    }
}
```

这里就是对注释中的条件判断，如果不是master节点请求，并且key正常，然后就调用`getNodeByQuery`函数执行node节点查询，这个函数代码比较多，这里截取主要逻辑查看：

```c
clusterNode *getNodeByQuery(client *c, struct redisCommand *cmd, robj **argv, int argc, int *hashslot, int *error_code) {
    clusterNode *n = NULL;
    robj *firstkey = NULL;
    multiState *ms;
    int i, slot = 0;
    /* Check that all the keys are in the same hash slot, and obtain this
     * slot and the node associated. */
    for (i = 0; i < ms->count; i++) {
        struct redisCommand *mcmd;
        robj **margv;
        int margc, *keyindex, numkeys, j;

        mcmd = ms->commands[i].cmd;
        margc = ms->commands[i].argc;
        margv = ms->commands[i].argv;

        keyindex = getKeysFromCommand(mcmd,margv,margc,&numkeys);
        for (j = 0; j < numkeys; j++) {
            robj *thiskey = margv[keyindex[j]];
            int thisslot = keyHashSlot((char*)thiskey->ptr,
                                       sdslen(thiskey->ptr));
        }
    }
    /* Base case: just return the right node. However if this node is not
     * myself, set error_code to MOVED since we need to issue a rediretion. */
    if (n != myself && error_code) *error_code = CLUSTER_REDIR_MOVED;
    return n;
}
```

从这个函数可以看出，redis首先解析命令里的key，然后遍历key，查出每个key所在的哈希槽，找到对应的节点。这样看貌似很简单，没贴出的代码里做了一些限制条件，想要保证这个函数调用成功，有需要满足下面两个条件任意一个：
- 1、参数中只有一个key
- 2、多个Key在同一个哈希槽，并且哈希槽是稳定的，没有执行中的重哈希。

如果哈希槽就在本节点，返回`NULL`，上层调用不进行重定向。如果不在本节点，返回`MOVED`或`ASK`指令，上层做重定向处理。如果是多个节点的哈希槽或节点宕机等其它情况，返回异常信息。

节点查询的逻辑到这就大概清楚了，但是键是怎么分配哈希槽的呢，在使用中又是怎么保证键分配在同一个哈希槽中呢，比如说对`id:10000`的用户来说，键`user_10000_collect`和`user_10000_like`怎么分配到一个哈希槽里？秘密就在`keyHashSlot`函数里。代码如下：

```c
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing between {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```

在`keyHashSlot`函数中，可以看到，会试图匹配`{...}`，如果能匹配到，则取`{}`包含的内容计算哈希槽返回，如果没有匹配到，那就按整个key计算哈希槽，由这里我们我们就能得出结论，如果我们想将键放入指定的节点，那就将关键值用`{}`包含，那样就能分配到相同哈希槽，落到相同节点处理。还有前面简介里说的集群16384个哈希槽我们也能在这得出结论，因为crc16算法的结果是和`0x3FFF`也就是16383取余的。


crc16算法redis官网有实现描述，这里不做赘述。

### 总结

经过上面的梳理，大概就能把redis集群使用时，键划分节点的过程搞明白了。分为以下几步：
1. 如果节点启用了集群，并且请求参数合法，调用节点查询函数获取目标节点；
2. 遍历请求参数中的键，尝试匹配键中`{}`符号，如果匹配到，根据`{}`中包含内容，获取哈希槽，否则根据整键获取哈希槽；
3. 如果是本机节点，继续处理，如果不是本节点，返回重定向信息，告诉客户端重定向。

以上，就是redis集群使用时的key节点查询逻辑以及相应源码实现的分析。有啥问题，欢迎讨论。



















