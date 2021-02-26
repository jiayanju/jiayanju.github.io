---
title: "redisTemplate执行lua脚本抛出IllegalStateException异常" 

date: 2021-02-20T22:30:00+08:00 

categories: ["Redis"]

tags: ["Redis"] 

author: "Me" 

showToc: true

TocOpen: false 

draft: false 

hidemeta: false 

comments: false 

description: "" 

disableHLJS: true  

disableShare: false 

cover:
  image: "https://raw.githubusercontent.com/jiayanju/imgrepo/main/floor1.jpg"
  alt: ""
  caption: ""
  relative: false
  hidden: false

---

## 1. 问题描述

在项目中，实现延迟队列的过程中，需要实现一个分布式锁，由于项目已经使用了Redis，就打算基于Redis实现一个简单的分布式锁。对应分布式锁，在释放锁的时候需要使用lua脚本保证一下原子性，代码如下:

```java
  public boolean unlock(String uniqueId) {
    if (StringUtils.isEmpty(uniqueId)) {
      return false;
    }

    Long result = redisTemplate.execute(RedisScript.of(RELEASE_LOCK_LUA_SCRIPT), Collections.singletonList(getLockKey(lockKey)), uniqueId);
    if (Objects.equals(result, RELEASE_LOCK_SUCCESS_RESULT)) {
      return true;
    }
    return false;
  }
```

在运行的过程中抛出以下异常

```
Exception in thread "delay-queue-handler-task-2" org.springframework.data.redis.RedisSystemException: Redis exception; nested exception is io.lettuce.core.RedisException: java.lang.IllegalStateException
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:74)
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:41)
	at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:44)
	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:42)
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.convertLettuceAccessException(LettuceConnection.java:268)
	at org.springframework.data.redis.connection.lettuce.LettuceScriptingCommands.convertLettuceAccessException(LettuceScriptingCommands.java:236)
	at org.springframework.data.redis.connection.lettuce.LettuceScriptingCommands.eval(LettuceScriptingCommands.java:163)
	at org.springframework.data.redis.connection.DefaultedRedisConnection.eval(DefaultedRedisConnection.java:1311)
	at org.springframework.data.redis.connection.DefaultStringRedisConnection.eval(DefaultStringRedisConnection.java:1720)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at org.springframework.data.redis.core.CloseSuppressingInvocationHandler.invoke(CloseSuppressingInvocationHandler.java:61)
	at com.sun.proxy.$Proxy414.eval(Unknown Source)
	at org.springframework.data.redis.core.script.DefaultScriptExecutor.eval(DefaultScriptExecutor.java:84)
	at org.springframework.data.redis.core.script.DefaultScriptExecutor.lambda$execute$0(DefaultScriptExecutor.java:68)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:225)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:185)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:172)
	at org.springframework.data.redis.core.script.DefaultScriptExecutor.execute(DefaultScriptExecutor.java:58)
	at org.springframework.data.redis.core.script.DefaultScriptExecutor.execute(DefaultScriptExecutor.java:52)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:347)
	at cn.udesk.ecology.common.component.lock.RedisDistributeLock.unlock(RedisDistributeLock.java:118)
	at cn.udesk.ecology.common.component.delayqueue.DelayQueueHandler.run(DelayQueueHandler.java:57)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: io.lettuce.core.RedisException: java.lang.IllegalStateException
	at io.lettuce.core.LettuceFutures.awaitOrCancel(LettuceFutures.java:129)
	at io.lettuce.core.FutureSyncInvocationHandler.handleInvocation(FutureSyncInvocationHandler.java:69)
	at io.lettuce.core.internal.AbstractInvocationHandler.invoke(AbstractInvocationHandler.java:80)
	at com.sun.proxy.$Proxy413.eval(Unknown Source)
	at org.springframework.data.redis.connection.lettuce.LettuceScriptingCommands.eval(LettuceScriptingCommands.java:161)
	... 21 more
Caused by: java.lang.IllegalStateException
	at io.lettuce.core.output.CommandOutput.set(CommandOutput.java:85)
	at io.lettuce.core.protocol.RedisStateMachine.safeSet(RedisStateMachine.java:357)
	at io.lettuce.core.protocol.RedisStateMachine.decode(RedisStateMachine.java:138)
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:708)
	at io.lettuce.core.protocol.CommandHandler.decode0(CommandHandler.java:672)
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:658)
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:587)
	at io.lettuce.core.protocol.CommandHandler.channelRead(CommandHandler.java:556)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:357)
	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1410)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365)
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:919)
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:166)
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:714)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:650)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:576)
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:493)
	at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:989)
	at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74)
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
	... 1 more
```

## 2. 问题分析

查看异常堆栈信息，执行路径代码

org.springframework.data.redis.core.script.DefaultScriptExecutor#execute(org.springframework.data.redis.core.script.RedisScript<T>, org.springframework.data.redis.serializer.RedisSerializer<?>, org.springframework.data.redis.serializer.RedisSerializer<T>, java.util.List<K>, java.lang.Object...)

```java
	public <T> T execute(final RedisScript<T> script, final RedisSerializer<?> argsSerializer,
			final RedisSerializer<T> resultSerializer, final List<K> keys, final Object... args) {
		return template.execute((RedisCallback<T>) connection -> {
			final ReturnType returnType = ReturnType.fromJavaType(script.getResultType());
			final byte[][] keysAndArgs = keysAndArgs(argsSerializer, keys, args);
			final int keySize = keys != null ? keys.size() : 0;
			if (connection.isPipelined() || connection.isQueueing()) {
				// We could script load first and then do evalsha to ensure sha is present,
				// but this adds a sha1 to exec/closePipeline results. Instead, just eval
				connection.eval(scriptBytes(script), returnType, keySize, keysAndArgs);
				return null;
			}
			return eval(connection, script, returnType, keySize, keysAndArgs, resultSerializer);
		});
	}
```

org.springframework.data.redis.connection.ReturnType#fromJavaType

```java
/**
	 * @param javaType can be {@literal null} which translates to {@link ReturnType#STATUS}.
	 * @return never {@literal null}.
	 */
	public static ReturnType fromJavaType(@Nullable Class<?> javaType) {

		if (javaType == null) {
			return ReturnType.STATUS;
		}
		if (javaType.isAssignableFrom(List.class)) {
			return ReturnType.MULTI;
		}
		if (javaType.isAssignableFrom(Boolean.class)) {
			return ReturnType.BOOLEAN;
		}
		if (javaType.isAssignableFrom(Long.class)) {
			return ReturnType.INTEGER;
		}
		return ReturnType.VALUE;
	}
```

有上面这个方法可以看出，如果传入的javaType为null，则ReturnType为STAUTS，如果是Long，则返回INTEGER

org.springframework.data.redis.connection.lettuce.LettuceScriptingCommands#eval

```java
	public <T> T eval(byte[] script, ReturnType returnType, int numKeys, byte[]... keysAndArgs) {

		Assert.notNull(script, "Script must not be null!");

		try {
			byte[][] keys = extractScriptKeys(numKeys, keysAndArgs);
			byte[][] args = extractScriptArgs(numKeys, keysAndArgs);
			String convertedScript = LettuceConverters.toString(script);
			if (isPipelined()) {
				pipeline(connection.newLettuceResult(
						getAsyncConnection().eval(convertedScript, LettuceConverters.toScriptOutputType(returnType), keys, args),
						new LettuceEvalResultsConverter<T>(returnType)));
				return null;
			}
			if (isQueueing()) {
				transaction(connection.newLettuceResult(
						getAsyncConnection().eval(convertedScript, LettuceConverters.toScriptOutputType(returnType), keys, args),
						new LettuceEvalResultsConverter<T>(returnType)));
				return null;
			}
			return new LettuceEvalResultsConverter<T>(returnType)
					.convert(getConnection().eval(convertedScript, LettuceConverters.toScriptOutputType(returnType), keys, args));
		} catch (Exception ex) {
			throw convertLettuceAccessException(ex);
		}
	}
```

底层使用异步的命令方式

io.lettuce.core.AbstractRedisAsyncCommands#eval(java.lang.String, io.lettuce.core.ScriptOutputType, K[], V...)

```java
public <T> RedisFuture<T> eval(String script, ScriptOutputType type, K[] keys, V... values) {
        return (RedisFuture<T>) dispatch(commandBuilder.eval(script, type, keys, values));
    }
```

io.lettuce.core.RedisCommandBuilder#eval(java.lang.String, io.lettuce.core.ScriptOutputType, K...)

```java
<T> Command<K, V, T> eval(String script, ScriptOutputType type, K... keys) {
        LettuceAssert.notNull(script, "Script " + MUST_NOT_BE_NULL);
        LettuceAssert.notEmpty(script, "Script " + MUST_NOT_BE_EMPTY);
        LettuceAssert.notNull(type, "ScriptOutputType " + MUST_NOT_BE_NULL);
        LettuceAssert.notNull(keys, "Keys " + MUST_NOT_BE_NULL);

        CommandArgs<K, V> args = new CommandArgs<>(codec);
        args.add(script.getBytes()).add(keys.length).addKeys(keys);
        CommandOutput<K, V, T> output = newScriptOutput(codec, type);
        return createCommand(EVAL, output, args);
    }
```

从上面可以看到CommandOutput的构造 

CommandOutput<K, V, T> output = newScriptOutput(codec, type);

```java
protected <T> CommandOutput<K, V, T> newScriptOutput(RedisCodec<K, V> codec, ScriptOutputType type) {
        switch (type) {
            case BOOLEAN:
                return (CommandOutput<K, V, T>) new BooleanOutput<K, V>(codec);
            case INTEGER:
                return (CommandOutput<K, V, T>) new IntegerOutput<K, V>(codec);
            case STATUS:
                return (CommandOutput<K, V, T>) new StatusOutput<K, V>(codec);
            case MULTI:
                return (CommandOutput<K, V, T>) new NestedMultiOutput<K, V>(codec);
            case VALUE:
                return (CommandOutput<K, V, T>) new ValueOutput<K, V>(codec);
            default:
                throw new RedisException("Unsupported script output type");
        }
    }
```

STATUS时，返回***StatusOutput对象***

看一下下面的异常信息

```
Caused by: java.lang.IllegalStateException
	at io.lettuce.core.output.CommandOutput.set(CommandOutput.java:85)
	at io.lettuce.core.protocol.RedisStateMachine.safeSet(RedisStateMachine.java:357)
	at io.lettuce.core.protocol.RedisStateMachine.decode(RedisStateMachine.java:138)
```

CommandOutput是个抽象类, set方法抛出异常

```java
public void set(long integer) {
        throw new IllegalStateException();
}
```

而StatusOutput对象是CommandOutput的实现类单并没有实现set方法，因此抛出了异常。

## 3. 问题解决

从上面的分析得出，应该使用IntegerOutput实现类，这个实现类对应的returnType是Long类型。所以应该对RedisScript设置resultType为Long.class.

```java
  public boolean unlock(String uniqueId) {
    if (StringUtils.isEmpty(uniqueId)) {
      return false;
    }

    DefaultRedisScript<Long> script = new DefaultRedisScript<>(RELEASE_LOCK_LUA_SCRIPT, Long.class);
    Long result = redisTemplate.execute(script, Collections.singletonList(getLockKey(lockKey)), uniqueId);
    if (Objects.equals(result, RELEASE_LOCK_SUCCESS_RESULT)) {
      return true;
    }
    return false;
  }
```

至此，问题解决~