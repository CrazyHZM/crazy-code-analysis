# 远程调用——thrift协议

> 目标：介绍thrift协议的设计和实现，介绍dubbo-rpc-thrift的源码。

## 前言

dubbo集成thrift协议，是基于Thrift来实现的，Thrift是一种轻量级，与语言无关的软件堆栈，具有用于点对点RPC的相关代码生成机制。Thrift为数据传输，数据序列化和应用程序级处理提供了清晰的抽象。代码生成系统采用简单的定义语言作为输入，并跨编程语言生成代码，使用抽象堆栈构建可互操作的RPC客户端和服务器。

## 源码分析

### （一）MultiServiceProcessor

该类对输入流进行操作并写入某些输出流。它实现了TProcessor接口，关键的方法是process。

```java
@Override
public boolean process(TProtocol in, TProtocol out) throws TException {

    // 获得十六进制的魔数
    short magic = in.readI16();

    // 如果不是规定的魔数，则打印错误日志，返回false
    if (magic != ThriftCodec.MAGIC) {
        logger.error("Unsupported magic " + magic);
        return false;
    }

    // 获得三十二进制魔数
    in.readI32();
    // 获得十六进制魔数
    in.readI16();
    // 获得版本
    byte version = in.readByte();
    // 获得服务名
    String serviceName = in.readString();
    // 获得id
    long id = in.readI64();

    ByteArrayOutputStream bos = new ByteArrayOutputStream(1024);

    // 创建基础运输TIOStreamTransport对象
    TIOStreamTransport transport = new TIOStreamTransport(bos);

    // 获得协议
    TProtocol protocol = protocolFactory.getProtocol(transport);

    // 从集合中取出处理器
    TProcessor processor = processorMap.get(serviceName);

    // 如果处理器为空，则打印错误，返回false
    if (processor == null) {
        logger.error("Could not find processor for service " + serviceName);
        return false;
    }

    // todo if exception
    // 获得结果
    boolean result = processor.process(in, protocol);

    ByteArrayOutputStream header = new ByteArrayOutputStream(512);

    // 协议头的传输器
    TIOStreamTransport headerTransport = new TIOStreamTransport(header);

    TProtocol headerProtocol = protocolFactory.getProtocol(headerTransport);

    // 写入16进制的魔数
    headerProtocol.writeI16(magic);
    // 写入32进制的Integer最大值
    headerProtocol.writeI32(Integer.MAX_VALUE);
    // 写入Short最大值的16进制
    headerProtocol.writeI16(Short.MAX_VALUE);
    // 写入版本号
    headerProtocol.writeByte(version);
    // 写入服务名
    headerProtocol.writeString(serviceName);
    // 写入id
    headerProtocol.writeI64(id);
    // 输出
    headerProtocol.getTransport().flush();

    out.writeI16(magic);
    out.writeI32(bos.size() + header.size());
    out.writeI16((short) (0xffff & header.size()));
    out.writeByte(version);
    out.writeString(serviceName);
    out.writeI64(id);

    out.getTransport().write(bos.toByteArray());
    out.getTransport().flush();

    return result;

}
```

### （二）RandomAccessByteArrayOutputStream

该类是随机访问数组的输出流，比较简单，我就不多叙述，有兴趣的可以直接看源码，不看影响也不大。

### （三）ClassNameGenerator

```java
@SPI(DubboClassNameGenerator.NAME)
public interface ClassNameGenerator {

    /**
     * 生成参数的类名
     */
    public String generateArgsClassName(String serviceName, String methodName);

    /**
     * 生成结果的类名
     * @param serviceName
     * @param methodName
     * @return
     */
    public String generateResultClassName(String serviceName, String methodName);

}
```

该接口是是可扩展接口，定义了两个方法。有两个实现类，下面讲述。

### （四）DubboClassNameGenerator

该类实现了ClassNameGenerator接口，是dubbo相关的类名生成实现。

```java
public class DubboClassNameGenerator implements ClassNameGenerator {

    public static final String NAME = "dubbo";

    @Override
    public String generateArgsClassName(String serviceName, String methodName) {
        return ThriftUtils.generateMethodArgsClassName(serviceName, methodName);
    }

    @Override
    public String generateResultClassName(String serviceName, String methodName) {
        return ThriftUtils.generateMethodResultClassName(serviceName, methodName);
    }

}
```

### （五）ThriftClassNameGenerator

该类实现了ClassNameGenerator接口，是Thrift相关的类名生成实现。

```java
public class ThriftClassNameGenerator implements ClassNameGenerator {

    public static final String NAME = "thrift";

    @Override
    public String generateArgsClassName(String serviceName, String methodName) {
        return ThriftUtils.generateMethodArgsClassNameThrift(serviceName, methodName);
    }

    @Override
    public String generateResultClassName(String serviceName, String methodName) {
        return ThriftUtils.generateMethodResultClassNameThrift(serviceName, methodName);
    }

}
```

以上两个都调用了ThriftUtils中的方法。

### （六）ThriftUtils

该类中封装的方法比较简单，就一些字符串的拼接，有兴趣的可以直接查看我下面贴出来的注释连接。

### （七）ThriftCodec

该类是基于Thrift实现的编解码器。  这里需要大家看一下该类的注释，关于协议的数据：

```java
* |<-                                  message header                                  ->|<- message body ->|
* +----------------+----------------------+------------------+---------------------------+------------------+
* | magic (2 bytes)|message size (4 bytes)|head size(2 bytes)| version (1 byte) | header |   message body   |
* +----------------+----------------------+------------------+---------------------------+------------------+
* |<-  
```

#### 1.属性

```java
/**
 * 消息长度索引
 */
public static final int MESSAGE_LENGTH_INDEX = 2;
/**
 * 消息头长度索引
 */
public static final int MESSAGE_HEADER_LENGTH_INDEX = 6;
/**
 * 消息最短长度
 */
public static final int MESSAGE_SHORTEST_LENGTH = 10;
public static final String NAME = "thrift";
/**
 * 类名生成参数
 */
public static final String PARAMETER_CLASS_NAME_GENERATOR = "class.name.generator";
/**
 * 版本
 */
public static final byte VERSION = (byte) 1;
/**
 * 魔数
 */
public static final short MAGIC = (short) 0xdabc;
/**
 * 请求参数集合
 */
static final ConcurrentMap<Long, RequestData> cachedRequest =
        new ConcurrentHashMap<Long, RequestData>();
/**
 * thrift序列号
 */
private static final AtomicInteger THRIFT_SEQ_ID = new AtomicInteger(0);
/**
 * 类缓存
 */
private static final ConcurrentMap<String, Class<?>> cachedClass =
        new ConcurrentHashMap<String, Class<?>>();
```

#### 2.encode

```java
@Override
public void encode(Channel channel, ChannelBuffer buffer, Object message)
        throws IOException {

    // 如果消息是Request类型
    if (message instanceof Request) {
        // Request类型消息编码
        encodeRequest(channel, buffer, (Request) message);
    } else if (message instanceof Response) {
        // Response类型消息编码
        encodeResponse(channel, buffer, (Response) message);
    } else {
        throw new UnsupportedOperationException("Thrift codec only support encode " 
                + Request.class.getName() + " and " + Response.class.getName());
    }

}
```

该方法是编码的逻辑，具体的编码操作根据请求类型不同分别调用不同的方法。

#### 3.encodeRequest

```java
private void encodeRequest(Channel channel, ChannelBuffer buffer, Request request)
        throws IOException {

    // 获得会话域
    RpcInvocation inv = (RpcInvocation) request.getData();

    // 获得下一个id
    int seqId = nextSeqId();

    // 获得服务名
    String serviceName = inv.getAttachment(Constants.INTERFACE_KEY);

    // 如果是空的 则抛出异常
    if (StringUtils.isEmpty(serviceName)) {
        throw new IllegalArgumentException("Could not find service name in attachment with key " 
                + Constants.INTERFACE_KEY);
    }

    // 创建TMessage对象
    TMessage message = new TMessage(
            inv.getMethodName(),
            TMessageType.CALL,
            seqId);

    // 获得方法参数
    String methodArgs = ExtensionLoader.getExtensionLoader(ClassNameGenerator.class)
            .getExtension(channel.getUrl().getParameter(ThriftConstants.CLASS_NAME_GENERATOR_KEY, ThriftClassNameGenerator.NAME))
            .generateArgsClassName(serviceName, inv.getMethodName());

    // 如果是空，则抛出异常
    if (StringUtils.isEmpty(methodArgs)) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION,
                "Could not encode request, the specified interface may be incorrect.");
    }

    // 从缓存中取出类型
    Class<?> clazz = cachedClass.get(methodArgs);

    if (clazz == null) {

        try {

            // 重新获得类型
            clazz = ClassHelper.forNameWithThreadContextClassLoader(methodArgs);

            // 加入缓存
            cachedClass.putIfAbsent(methodArgs, clazz);

        } catch (ClassNotFoundException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        }

    }

    // 生成的Thrift对象的通用基接口
    TBase args;

    try {
        args = (TBase) clazz.newInstance();
    } catch (InstantiationException e) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
    } catch (IllegalAccessException e) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
    }

    // 遍历参数
    for (int i = 0; i < inv.getArguments().length; i++) {

        Object obj = inv.getArguments()[i];

        if (obj == null) {
            continue;
        }

        TFieldIdEnum field = args.fieldForId(i + 1);

        // 生成set方法名
        String setMethodName = ThriftUtils.generateSetMethodName(field.getFieldName());

        Method method;

        try {
            // 获得方法
            method = clazz.getMethod(setMethodName, inv.getParameterTypes()[i]);
        } catch (NoSuchMethodException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        }

        try {
            // 调用下一个调用链
            method.invoke(args, obj);
        } catch (IllegalAccessException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        } catch (InvocationTargetException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        }

    }

    // 创建一个随机访问数组输出流
    RandomAccessByteArrayOutputStream bos = new RandomAccessByteArrayOutputStream(1024);

    // 创建传输器
    TIOStreamTransport transport = new TIOStreamTransport(bos);

    // 创建协议
    TBinaryProtocol protocol = new TBinaryProtocol(transport);

    int headerLength, messageLength;

    byte[] bytes = new byte[4];
    try {
        // 开始编码
        // magic
        protocol.writeI16(MAGIC);
        // message length placeholder
        protocol.writeI32(Integer.MAX_VALUE);
        // message header length placeholder
        protocol.writeI16(Short.MAX_VALUE);
        // version
        protocol.writeByte(VERSION);
        // service name
        protocol.writeString(serviceName);
        // dubbo request id
        protocol.writeI64(request.getId());
        protocol.getTransport().flush();
        // header size
        headerLength = bos.size();

        // 对body内容进行编码
        // message body
        protocol.writeMessageBegin(message);
        args.write(protocol);
        protocol.writeMessageEnd();
        protocol.getTransport().flush();
        int oldIndex = messageLength = bos.size();

        // fill in message length and header length
        try {
            TFramedTransport.encodeFrameSize(messageLength, bytes);
            bos.setWriteIndex(MESSAGE_LENGTH_INDEX);
            protocol.writeI32(messageLength);
            bos.setWriteIndex(MESSAGE_HEADER_LENGTH_INDEX);
            protocol.writeI16((short) (0xffff & headerLength));
        } finally {
            bos.setWriteIndex(oldIndex);
        }

    } catch (TException e) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
    }

    buffer.writeBytes(bytes);
    buffer.writeBytes(bos.toByteArray());

}
```

该方法是对request类型的消息进行编码。

#### 4.encodeResponse

```java
private void encodeResponse(Channel channel, ChannelBuffer buffer, Response response)
        throws IOException {

    // 获得结果
    RpcResult result = (RpcResult) response.getResult();

    // 获得请求
    RequestData rd = cachedRequest.get(response.getId());

    // 获得结果的类名
    String resultClassName = ExtensionLoader.getExtensionLoader(ClassNameGenerator.class).getExtension(
            channel.getUrl().getParameter(ThriftConstants.CLASS_NAME_GENERATOR_KEY, ThriftClassNameGenerator.NAME))
            .generateResultClassName(rd.serviceName, rd.methodName);

    // 如果为空，则序列化失败
    if (StringUtils.isEmpty(resultClassName)) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION,
                "Could not encode response, the specified interface may be incorrect.");
    }

    // 获得类型
    Class clazz = cachedClass.get(resultClassName);

    // 如果为空，则重新获取
    if (clazz == null) {

        try {
            clazz = ClassHelper.forNameWithThreadContextClassLoader(resultClassName);
            cachedClass.putIfAbsent(resultClassName, clazz);
        } catch (ClassNotFoundException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        }

    }

    TBase resultObj;

    try {
        // 加载该类
        resultObj = (TBase) clazz.newInstance();
    } catch (InstantiationException e) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
    } catch (IllegalAccessException e) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
    }

    TApplicationException applicationException = null;
    TMessage message;

    // 如果结果有异常抛出
    if (result.hasException()) {
        Throwable throwable = result.getException();
        int index = 1;
        boolean found = false;
        while (true) {
            TFieldIdEnum fieldIdEnum = resultObj.fieldForId(index++);
            if (fieldIdEnum == null) {
                break;
            }
            String fieldName = fieldIdEnum.getFieldName();
            String getMethodName = ThriftUtils.generateGetMethodName(fieldName);
            String setMethodName = ThriftUtils.generateSetMethodName(fieldName);
            Method getMethod;
            Method setMethod;
            try {
                // 获得get方法
                getMethod = clazz.getMethod(getMethodName);
                // 如果返回类型和异常类型一样，则创建set方法，并且调用下一个调用链
                if (getMethod.getReturnType().equals(throwable.getClass())) {
                    found = true;
                    setMethod = clazz.getMethod(setMethodName, throwable.getClass());
                    setMethod.invoke(resultObj, throwable);
                }
            } catch (NoSuchMethodException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            } catch (InvocationTargetException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            } catch (IllegalAccessException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            }
        }

        if (!found) {
            // 创建TApplicationException异常
            applicationException = new TApplicationException(throwable.getMessage());
        }

    } else {
        // 获得真实的结果
        Object realResult = result.getResult();
        // result field id is 0
        String fieldName = resultObj.fieldForId(0).getFieldName();
        String setMethodName = ThriftUtils.generateSetMethodName(fieldName);
        String getMethodName = ThriftUtils.generateGetMethodName(fieldName);
        Method getMethod;
        Method setMethod;
        try {
            // 创建get和set方法
            getMethod = clazz.getMethod(getMethodName);
            setMethod = clazz.getMethod(setMethodName, getMethod.getReturnType());
            setMethod.invoke(resultObj, realResult);
        } catch (NoSuchMethodException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        } catch (InvocationTargetException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        } catch (IllegalAccessException e) {
            throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
        }

    }

    if (applicationException != null) {
        message = new TMessage(rd.methodName, TMessageType.EXCEPTION, rd.id);
    } else {
        message = new TMessage(rd.methodName, TMessageType.REPLY, rd.id);
    }

    RandomAccessByteArrayOutputStream bos = new RandomAccessByteArrayOutputStream(1024);

    TIOStreamTransport transport = new TIOStreamTransport(bos);

    TBinaryProtocol protocol = new TBinaryProtocol(transport);

    int messageLength;
    int headerLength;

    //编码
    byte[] bytes = new byte[4];
    try {
        // magic
        protocol.writeI16(MAGIC);
        // message length
        protocol.writeI32(Integer.MAX_VALUE);
        // message header length
        protocol.writeI16(Short.MAX_VALUE);
        // version
        protocol.writeByte(VERSION);
        // service name
        protocol.writeString(rd.serviceName);
        // id
        protocol.writeI64(response.getId());
        protocol.getTransport().flush();
        headerLength = bos.size();

        // message
        protocol.writeMessageBegin(message);
        switch (message.type) {
            case TMessageType.EXCEPTION:
                applicationException.write(protocol);
                break;
            case TMessageType.REPLY:
                resultObj.write(protocol);
                break;
        }
        protocol.writeMessageEnd();
        protocol.getTransport().flush();
        int oldIndex = messageLength = bos.size();

        try {
            TFramedTransport.encodeFrameSize(messageLength, bytes);
            bos.setWriteIndex(MESSAGE_LENGTH_INDEX);
            protocol.writeI32(messageLength);
            bos.setWriteIndex(MESSAGE_HEADER_LENGTH_INDEX);
            protocol.writeI16((short) (0xffff & headerLength));
        } finally {
            bos.setWriteIndex(oldIndex);
        }

    } catch (TException e) {
        throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
    }

    buffer.writeBytes(bytes);
    buffer.writeBytes(bos.toByteArray());

}
```

该方法是对response类型的请求消息进行编码。

#### 5.decode

```java
    @Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {

        int available = buffer.readableBytes();

        // 如果小于最小的长度，则还需要更多的输入
        if (available < MESSAGE_SHORTEST_LENGTH) {

            return DecodeResult.NEED_MORE_INPUT;

        } else {

            TIOStreamTransport transport = new TIOStreamTransport(new ChannelBufferInputStream(buffer));

            TBinaryProtocol protocol = new TBinaryProtocol(transport);

            short magic;
            int messageLength;

            // 对协议头中的魔数进行比对
            try {
//                protocol.readI32(); // skip the first message length
                byte[] bytes = new byte[4];
                transport.read(bytes, 0, 4);
                magic = protocol.readI16();
                messageLength = protocol.readI32();

            } catch (TException e) {
                throw new IOException(e.getMessage(), e);
            }

            if (MAGIC != magic) {
                throw new IOException("Unknown magic code " + magic);
            }

            if (available < messageLength) {
                return DecodeResult.NEED_MORE_INPUT;
            }

            return decode(protocol);

        }

    }

    /**
     * 解码
     * @param protocol
     * @return
     * @throws IOException
     */
    private Object decode(TProtocol protocol)
            throws IOException {

        // version
        String serviceName;
        long id;

        TMessage message;

        try {
            // 读取协议头中对内容
            protocol.readI16();
            protocol.readByte();
            serviceName = protocol.readString();
            id = protocol.readI64();
            message = protocol.readMessageBegin();
        } catch (TException e) {
            throw new IOException(e.getMessage(), e);
        }

        // 如果是回调
        if (message.type == TMessageType.CALL) {

            RpcInvocation result = new RpcInvocation();
            // 设置服务名和方法名
            result.setAttachment(Constants.INTERFACE_KEY, serviceName);
            result.setMethodName(message.name);

            String argsClassName = ExtensionLoader.getExtensionLoader(ClassNameGenerator.class)
                    .getExtension(ThriftClassNameGenerator.NAME).generateArgsClassName(serviceName, message.name);

            if (StringUtils.isEmpty(argsClassName)) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION,
                        "The specified interface name incorrect.");
            }

            // 从缓存中获得class类
            Class clazz = cachedClass.get(argsClassName);

            if (clazz == null) {
                try {

                    // 重新获得class类型
                    clazz = ClassHelper.forNameWithThreadContextClassLoader(argsClassName);

                    // 加入集合
                    cachedClass.putIfAbsent(argsClassName, clazz);

                } catch (ClassNotFoundException e) {
                    throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                }
            }

            TBase args;

            try {
                args = (TBase) clazz.newInstance();
            } catch (InstantiationException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            } catch (IllegalAccessException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            }

            try {
                args.read(protocol);
                protocol.readMessageEnd();
            } catch (TException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            }

            // 参数集合
            List<Object> parameters = new ArrayList<Object>();
            // 参数类型集合
            List<Class<?>> parameterTypes = new ArrayList<Class<?>>();
            int index = 1;

            while (true) {

                TFieldIdEnum fieldIdEnum = args.fieldForId(index++);

                if (fieldIdEnum == null) {
                    break;
                }

                String fieldName = fieldIdEnum.getFieldName();

                // 获得方法名
                String getMethodName = ThriftUtils.generateGetMethodName(fieldName);

                Method getMethod;

                try {
                    getMethod = clazz.getMethod(getMethodName);
                } catch (NoSuchMethodException e) {
                    throw new RpcException(
                            RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                }

                // 加入参数类型
                parameterTypes.add(getMethod.getReturnType());
                try {
                    parameters.add(getMethod.invoke(args));
                } catch (IllegalAccessException e) {
                    throw new RpcException(
                            RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                } catch (InvocationTargetException e) {
                    throw new RpcException(
                            RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                }

            }

            // 设置参数
            result.setArguments(parameters.toArray());
            // 设置参数类型
            result.setParameterTypes(parameterTypes.toArray(new Class[parameterTypes.size()]));

            // 创建一个新的请求
            Request request = new Request(id);
            // 把结果放入请求中
            request.setData(result);

            // 放入集合中
            cachedRequest.putIfAbsent(id,
                    RequestData.create(message.seqid, serviceName, message.name));

            return request;

            // 如果是抛出异常
        } else if (message.type == TMessageType.EXCEPTION) {

            TApplicationException exception;

            try {
                // 读取异常
                exception = TApplicationException.read(protocol);
                protocol.readMessageEnd();
            } catch (TException e) {
                throw new IOException(e.getMessage(), e);
            }

            // 创建结果
            RpcResult result = new RpcResult();

            // 设置异常
            result.setException(new RpcException(exception.getMessage()));

            // 创建Response响应
            Response response = new Response();

            // 把结果放入
            response.setResult(result);

            // 加入唯一id
            response.setId(id);

            return response;

            // 如果类型是回应
        } else if (message.type == TMessageType.REPLY) {

            // 获得结果的类名
            String resultClassName = ExtensionLoader.getExtensionLoader(ClassNameGenerator.class)
                    .getExtension(ThriftClassNameGenerator.NAME).generateResultClassName(serviceName, message.name);

            if (StringUtils.isEmpty(resultClassName)) {
                throw new IllegalArgumentException("Could not infer service result class name from service name " 
                        + serviceName + ", the service name you specified may not generated by thrift idl compiler");
            }

            // 获得class类型
            Class<?> clazz = cachedClass.get(resultClassName);

            // 如果为空，则重新获取
            if (clazz == null) {

                try {

                    clazz = ClassHelper.forNameWithThreadContextClassLoader(resultClassName);

                    cachedClass.putIfAbsent(resultClassName, clazz);

                } catch (ClassNotFoundException e) {
                    throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                }

            }

            TBase<?, ? extends TFieldIdEnum> result;
            try {
                result = (TBase<?, ?>) clazz.newInstance();
            } catch (InstantiationException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            } catch (IllegalAccessException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            }

            try {
                result.read(protocol);
                protocol.readMessageEnd();
            } catch (TException e) {
                throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
            }

            Object realResult = null;

            int index = 0;

            while (true) {

                TFieldIdEnum fieldIdEnum = result.fieldForId(index++);

                if (fieldIdEnum == null) {
                    break;
                }

                Field field;

                try {
                    field = clazz.getDeclaredField(fieldIdEnum.getFieldName());
                    field.setAccessible(true);
                } catch (NoSuchFieldException e) {
                    throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                }

                try {
                    // 获得真实的结果
                    realResult = field.get(result);
                } catch (IllegalAccessException e) {
                    throw new RpcException(RpcException.SERIALIZATION_EXCEPTION, e.getMessage(), e);
                }

                if (realResult != null) {
                    break;
                }

            }

            // 创建响应
            Response response = new Response();

            // 设置唯一id
            response.setId(id);

            // 创建结果
            RpcResult rpcResult = new RpcResult();

            // 用RpcResult包裹结果
            if (realResult instanceof Throwable) {
                rpcResult.setException((Throwable) realResult);
            } else {
                rpcResult.setValue(realResult);
            }

            // 设置结果
            response.setResult(rpcResult);

            return response;

        } else {
            // Impossible
            throw new IOException();
        }

    }
```

该方法是对解码的逻辑。对于消息分为REPLY、EXCEPTION和CALL三种情况来分别进行解码。

#### 6.RequestData

```java
static class RequestData {
    /**
     * 请求id
     */
    int id;
    /**
     * 服务名
     */
    String serviceName;
    /**
     * 方法名
     */
    String methodName;

    static RequestData create(int id, String sn, String mn) {
        RequestData result = new RequestData();
        result.id = id;
        result.serviceName = sn;
        result.methodName = mn;
        return result;
    }

}
```

该内部类是请求参数实体。

### （八）ThriftInvoker

该类是thrift协议的Invoker实现。

#### 1.属性

```java
/**
 * 客户端集合
 */
private final ExchangeClient[] clients;

/**
 * 活跃的客户端索引
 */
private final AtomicPositiveInteger index = new AtomicPositiveInteger();

/**
 * 销毁锁
 */
private final ReentrantLock destroyLock = new ReentrantLock();

/**
 * invoker集合
 */
private final Set<Invoker<?>> invokers;
```

#### 2.doInvoke

```java
@Override
protected Result doInvoke(Invocation invocation) throws Throwable {

    RpcInvocation inv = (RpcInvocation) invocation;

    final String methodName;

    // 获得方法名
    methodName = invocation.getMethodName();

    // 设置附加值 path
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());

    // for thrift codec
    inv.setAttachment(ThriftCodec.PARAMETER_CLASS_NAME_GENERATOR, getUrl().getParameter(
            ThriftCodec.PARAMETER_CLASS_NAME_GENERATOR, DubboClassNameGenerator.NAME));

    ExchangeClient currentClient;

    // 如果只有一个连接的客户端，则直接返回
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        // 否则，取出下一个客户端，循环数组取
        currentClient = clients[index.getAndIncrement() % clients.length];
    }

    try {
        // 获得超时时间
        int timeout = getUrl().getMethodParameter(
                methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);

        RpcContext.getContext().setFuture(null);

        // 发起请求
        return (Result) currentClient.request(inv, timeout).get();

    } catch (TimeoutException e) {
        // 抛出超时异常
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, e.getMessage(), e);
    } catch (RemotingException e) {
        // 抛出网络异常
        throw new RpcException(RpcException.NETWORK_EXCEPTION, e.getMessage(), e);
    }

}
```

该方法是thrift协议的调用链处理逻辑。

### （九）ThriftProtocol

该类是thrift协议的主要实现逻辑，分别实现了服务引用和服务调用的逻辑。

#### 1.属性

```java
/**
 * 默认端口号
 */
public static final int DEFAULT_PORT = 40880;

/**
 * 扩展名
 */
public static final String NAME = "thrift";

// ip:port -> ExchangeServer
/**
 * 服务集合，key为ip:port
 */
private final ConcurrentMap<String, ExchangeServer> serverMap =
        new ConcurrentHashMap<String, ExchangeServer>();

private ExchangeHandler handler = new ExchangeHandlerAdapter() {

    @Override
    public Object reply(ExchangeChannel channel, Object msg) throws RemotingException {

        // 如果消息是Invocation类型的
        if (msg instanceof Invocation) {
            Invocation inv = (Invocation) msg;
            // 获得服务名
            String serviceName = inv.getAttachments().get(Constants.INTERFACE_KEY);
            // 获得服务的key
            String serviceKey = serviceKey(channel.getLocalAddress().getPort(),
                    serviceName, null, null);
            // 从集合中获得暴露者
            DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);
            // 如果暴露者为空，则抛出异常
            if (exporter == null) {
                throw new RemotingException(channel,
                        "Not found exported service: "
                                + serviceKey
                                + " in "
                                + exporterMap.keySet()
                                + ", may be version or group mismatch "
                                + ", channel: consumer: "
                                + channel.getRemoteAddress()
                                + " --> provider: "
                                + channel.getLocalAddress()
                                + ", message:" + msg);
            }

            // 设置远程地址
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            return exporter.getInvoker().invoke(inv);

        }

        // 否则抛出异常，不支持的请求消息
        throw new RemotingException(channel,
                "Unsupported request: "
                        + (msg.getClass().getName() + ": " + msg)
                        + ", channel: consumer: "
                        + channel.getRemoteAddress()
                        + " --> provider: "
                        + channel.getLocalAddress());
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        // 如果消息是Invocation类型，则调用reply，否则接收消息
        if (message instanceof Invocation) {
            reply((ExchangeChannel) channel, message);
        } else {
            super.received(channel, message);
        }
    }

};
```

#### 2.export

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {

    // can use thrift codec only
    // 只能使用thrift编解码器
    URL url = invoker.getUrl().addParameter(Constants.CODEC_KEY, ThriftCodec.NAME);
    // find server.
    // 获得服务地址
    String key = url.getAddress();
    // client can expose a service for server to invoke only.
    // 客户端可以为服务器暴露服务以仅调用
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer && !serverMap.containsKey(key)) {
        // 加入到集合
        serverMap.put(key, getServer(url));
    }
    // export service.
    // 得到服务key
    key = serviceKey(url);
    // 创建暴露者
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // 加入集合
    exporterMap.put(key, exporter);

    return exporter;
}
```

该方法是服务暴露的逻辑实现。

#### 3.refer

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {

    // 创建ThriftInvoker
    ThriftInvoker<T> invoker = new ThriftInvoker<T>(type, url, getClients(url), invokers);

    // 加入到集合
    invokers.add(invoker);

    return invoker;

}
```

该方法是服务引用的逻辑实现。

#### 4.getClients

```java
private ExchangeClient[] getClients(URL url) {

    // 获得连接数
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 1);

    // 创建客户端集合
    ExchangeClient[] clients = new ExchangeClient[connections];

    // 创建客户端
    for (int i = 0; i < clients.length; i++) {
        clients[i] = initClient(url);
    }
    return clients;
}
```

该方法是获得客户端集合。

#### 5.initClient

```java
private ExchangeClient initClient(URL url) {

    ExchangeClient client;

    // 加上编解码器
    url = url.addParameter(Constants.CODEC_KEY, ThriftCodec.NAME);

    try {
        // 创建客户端
        client = Exchangers.connect(url);
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url
                + "): " + e.getMessage(), e);
    }

    return client;

}
```

该方法是创建客户端的逻辑。

#### 6.getServer

```java
private ExchangeServer getServer(URL url) {
    // enable sending readonly event when server closes by default
    // 加入只读事件
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    // 获得服务的实现方式
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    // 如果该实现方式不是dubbo支持的方式，则抛出异常
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    ExchangeServer server;
    try {
        // 获得服务器
        server = Exchangers.bind(url, handler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    // 获得实现方式
    str = url.getParameter(Constants.CLIENT_KEY);
    // 如果客户端实现方式不是dubbo支持的方式，则抛出异常。
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```

该方法是获得server的逻辑实现。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-rpc/dubbo-rpc-thrift/src/main/java/com/alibaba/dubbo/rpc/protocol/thrift

该文章讲解了远程调用中关于thrift协议实现的部分，要对Thrift。接下来我将开始对rpc模块关于webservice协议部分进行讲解。

