​						

# 					Dubbo调用过程源码解读

消费者端发起调用。

![image-20200922110750962](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200922110750962.png)

```java
//MockClusterInvoker

@Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            //no mock
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            if (logger.isWarnEnabled()) {
                logger.info("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
            }
            //force:direct mock
            // 屏蔽：系统准备大型促销活动时，有目的的直接屏蔽某些不影响主业务逻辑的接口  ===> mock=force:return null
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
            try {
                result = this.invoker.invoke(invocation);
            } catch (RpcException e) {
                // 容错：当系统出现非业务异常(如：高并发导致超时、网络异常等)  ===> mock=fail:return null
                if (e.isBiz()) {
                    throw e;
                } else {
                    if (logger.isWarnEnabled()) {
                        logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                    }
                    result = doMockInvoke(invocation, e);
                }
            }
        }
        return result;
    }
```



消费者端请求服务过程。

![image-20200922150209144](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200922150209144.png)



服务端接收请求过程。netty监听处理器监听消息，然后将事件扔进线程池处理。

![image-20200922155108956](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200922155108956.png)

![image-20200922154448622](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200922154448622.png)

最终进入调用目标对象的过程线程栈。

![image-20200922155336710](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200922155336710.png)





<u>Dubbo是以Protocol为主线发布服务和引用远程服务的。</u>