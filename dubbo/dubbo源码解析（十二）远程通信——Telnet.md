# 远程通讯——Telnet

> 目标：介绍telnet的相关实现逻辑、介绍dubbo-remoting-api中的telnet包内的源码解析。

## 前言

从dubbo 2.0.5开始，dubbo开始支持通过 telnet 命令来进行服务治理。本文就是讲解一些公用的telnet命令的实现。下面来看一下telnet实现的类图：

![telnet类图](https://github.com/CrazyHZM/crazy-code-analysis/blob/master/dubbo/image/%E5%8D%81%E4%BA%8C/telnet%E7%B1%BB%E5%9B%BE.png?raw=true)

可以看到，实现了TelnetHandler接口的有六个类，除了TelnetHandlerAdapter是以外，其他五个分别对应了clear、exit、help、log、status命令的实现，具体用来干嘛，请看官方文档的介绍。

## 源码分析

### （一）TelnetHandler

```java
@SPI
public interface TelnetHandler {

    /**
     * telnet.
     * 处理对应的telnet命令
     * @param channel
     * @param message telnet命令
     */
    String telnet(Channel channel, String message) throws RemotingException;

}
```

该接口上telnet命令处理器接口，是一个可扩展接口。它定义了一个方法，就是处理相关的telnet命令。

### （二）TelnetHandlerAdapter

该类继承了ChannelHandlerAdapter，实现了TelnetHandler接口，是TelnetHandler的适配器类，负责在接收到HeaderExchangeHandler发来的telnet命令后分发给对应的TelnetHandler实现类去实现，并且返回命令结果。

```java
public class TelnetHandlerAdapter extends ChannelHandlerAdapter implements TelnetHandler {

    /**
     * 扩展加载器
     */
    private final ExtensionLoader<TelnetHandler> extensionLoader = ExtensionLoader.getExtensionLoader(TelnetHandler.class);

    @Override
    public String telnet(Channel channel, String message) throws RemotingException {
        // 获得提示键配置，用于nc获取信息时不显示提示符
        String prompt = channel.getUrl().getParameterAndDecoded(Constants.PROMPT_KEY, Constants.DEFAULT_PROMPT);
        boolean noprompt = message.contains("--no-prompt");
        message = message.replace("--no-prompt", "");
        StringBuilder buf = new StringBuilder();
        // 删除头尾空白符的字符串
        message = message.trim();
        String command;
        // 获得命令
        if (message.length() > 0) {
            int i = message.indexOf(' ');
            if (i > 0) {
                // 获得命令
                command = message.substring(0, i).trim();
                // 获得参数
                message = message.substring(i + 1).trim();
            } else {
                command = message;
                message = "";
            }
        } else {
            command = "";
        }
        if (command.length() > 0) {
            // 如果有该命令的扩展实现类
            if (extensionLoader.hasExtension(command)) {
                try {
                    // 执行相应命令的实现类的telnet
                    String result = extensionLoader.getExtension(command).telnet(channel, message);
                    if (result == null) {
                        return null;
                    }
                    // 返回结果
                    buf.append(result);
                } catch (Throwable t) {
                    buf.append(t.getMessage());
                }
            } else {
                buf.append("Unsupported command: ");
                buf.append(command);
            }
        }
        if (buf.length() > 0) {
            buf.append("\r\n");
        }
        // 添加 telnet 提示语
        if (prompt != null && prompt.length() > 0 && !noprompt) {
            buf.append(prompt);
        }
        return buf.toString();
    }

}
```

该类只实现了telnet方法，其中的逻辑还是比较清晰，就是根据对应的命令去让对应的实现类产生命令结果。

### （三）ClearTelnetHandler

该类实现了TelnetHandler接口，封装了clear命令的实现。

```java
@Activate
@Help(parameter = "[lines]", summary = "Clear screen.", detail = "Clear screen.")
public class ClearTelnetHandler implements TelnetHandler {

    @Override
    public String telnet(Channel channel, String message) {
        // 清除屏幕上的内容行数
        int lines = 100;
        if (message.length() > 0) {
            // 如果不是一个数字
            if (!StringUtils.isInteger(message)) {
                return "Illegal lines " + message + ", must be integer.";
            }
            lines = Integer.parseInt(message);
        }
        StringBuilder buf = new StringBuilder();
        // 一行一行清除
        for (int i = 0; i < lines; i++) {
            buf.append("\r\n");
        }
        return buf.toString();
    }

}
```

### （四）ExitTelnetHandler

该类实现了TelnetHandler接口，封装了exit命令的实现。

```java
@Activate
@Help(parameter = "", summary = "Exit the telnet.", detail = "Exit the telnet.")
public class ExitTelnetHandler implements TelnetHandler {

    @Override
    public String telnet(Channel channel, String message) {
        // 关闭通道
        channel.close();
        return null;
    }

}
```

### （五）HelpTelnetHandler

该类实现了TelnetHandler接口，封装了help命令的实现。

```java
@Activate
@Help(parameter = "[command]", summary = "Show help.", detail = "Show help.")
public class HelpTelnetHandler implements TelnetHandler {

    /**
     * 扩展加载器
     */
    private final ExtensionLoader<TelnetHandler> extensionLoader = ExtensionLoader.getExtensionLoader(TelnetHandler.class);

    @Override
    public String telnet(Channel channel, String message) {
        // 如果需要查看某一个命令的帮助
        if (message.length() > 0) {
            if (!extensionLoader.hasExtension(message)) {
                return "No such command " + message;
            }
            // 获得对应的扩展实现类
            TelnetHandler handler = extensionLoader.getExtension(message);
            Help help = handler.getClass().getAnnotation(Help.class);
            StringBuilder buf = new StringBuilder();
            // 生成命令和帮助信息
            buf.append("Command:\r\n    ");
            buf.append(message + " " + help.parameter().replace("\r\n", " ").replace("\n", " "));
            buf.append("\r\nSummary:\r\n    ");
            buf.append(help.summary().replace("\r\n", " ").replace("\n", " "));
            buf.append("\r\nDetail:\r\n    ");
            buf.append(help.detail().replace("\r\n", "    \r\n").replace("\n", "    \n"));
            return buf.toString();
            // 如果查看所有命令的帮助
        } else {
            List<List<String>> table = new ArrayList<List<String>>();
            // 获得所有命令的提示信息
            List<TelnetHandler> handlers = extensionLoader.getActivateExtension(channel.getUrl(), "telnet");
            if (handlers != null && !handlers.isEmpty()) {
                for (TelnetHandler handler : handlers) {
                    Help help = handler.getClass().getAnnotation(Help.class);
                    List<String> row = new ArrayList<String>();
                    String parameter = " " + extensionLoader.getExtensionName(handler) + " " + (help != null ? help.parameter().replace("\r\n", " ").replace("\n", " ") : "");
                    row.add(parameter.length() > 50 ? parameter.substring(0, 50) + "..." : parameter);
                    String summary = help != null ? help.summary().replace("\r\n", " ").replace("\n", " ") : "";
                    row.add(summary.length() > 50 ? summary.substring(0, 50) + "..." : summary);
                    table.add(row);
                }
            }
            return "Please input \"help [command]\" show detail.\r\n" + TelnetUtils.toList(table);
        }
    }

}
```

help分为了需要查看某一个命令的帮助还是查看全部命令的帮助。

### （六）LogTelnetHandler

该类实现了TelnetHandler接口，封装了log命令的实现。

```java
@Activate
@Help(parameter = "level", summary = "Change log level or show log ", detail = "Change log level or show log")
public class LogTelnetHandler implements TelnetHandler {

    public static final String SERVICE_KEY = "telnet.log";

    @Override
    public String telnet(Channel channel, String message) {
        long size = 0;
        File file = LoggerFactory.getFile();
        StringBuffer buf = new StringBuffer();
        if (message == null || message.trim().length() == 0) {
            buf.append("EXAMPLE: log error / log 100");
        } else {
            String str[] = message.split(" ");
            if (!StringUtils.isInteger(str[0])) {
                // 设置日志级别
                LoggerFactory.setLevel(Level.valueOf(message.toUpperCase()));
            } else {
                // 获得日志长度
                int SHOW_LOG_LENGTH = Integer.parseInt(str[0]);

                if (file != null && file.exists()) {
                    try {
                        FileInputStream fis = new FileInputStream(file);
                        try {
                            FileChannel filechannel = fis.getChannel();
                            try {
                                size = filechannel.size();
                                ByteBuffer bb;
                                if (size <= SHOW_LOG_LENGTH) {
                                    // 分配缓冲区
                                    bb = ByteBuffer.allocate((int) size);
                                    // 读日志数据
                                    filechannel.read(bb, 0);
                                } else {
                                    int pos = (int) (size - SHOW_LOG_LENGTH);
                                    // 分配缓冲区
                                    bb = ByteBuffer.allocate(SHOW_LOG_LENGTH);
                                    // 读取日志数据
                                    filechannel.read(bb, pos);
                                }
                                bb.flip();
                                String content = new String(bb.array()).replace("<", "&lt;")
                                        .replace(">", "&gt;").replace("\n", "<br/><br/>");
                                buf.append("\r\ncontent:" + content);

                                buf.append("\r\nmodified:" + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
                                        .format(new Date(file.lastModified()))));
                                buf.append("\r\nsize:" + size + "\r\n");
                            } finally {
                                filechannel.close();
                            }
                        } finally {
                            fis.close();
                        }
                    } catch (Exception e) {
                        buf.append(e.getMessage());
                    }
                } else {
                    size = 0;
                    buf.append("\r\nMESSAGE: log file not exists or log appender is console .");
                }
            }
        }
        buf.append("\r\nCURRENT LOG LEVEL:" + LoggerFactory.getLevel())
                .append("\r\nCURRENT LOG APPENDER:" + (file == null ? "console" : file.getAbsolutePath()));
        return buf.toString();
    }

}
```

log命令实现原理就是从日志文件中把日志信息读取出来。

### （七）StatusTelnetHandler

该类实现了TelnetHandler接口，封装了status命令的实现。

```java
@Activate
@Help(parameter = "[-l]", summary = "Show status.", detail = "Show status.")
public class StatusTelnetHandler implements TelnetHandler {

    private final ExtensionLoader<StatusChecker> extensionLoader = ExtensionLoader.getExtensionLoader(StatusChecker.class);

    @Override
    public String telnet(Channel channel, String message) {
        // 显示状态列表
        if (message.equals("-l")) {
            List<StatusChecker> checkers = extensionLoader.getActivateExtension(channel.getUrl(), "status");
            String[] header = new String[]{"resource", "status", "message"};
            List<List<String>> table = new ArrayList<List<String>>();
            Map<String, Status> statuses = new HashMap<String, Status>();
            if (checkers != null && !checkers.isEmpty()) {
                // 遍历各个资源的状态，如果一个当全部 OK 时则显示 OK，只要有一个 ERROR 则显示 ERROR，只要有一个 WARN 则显示 WARN
                for (StatusChecker checker : checkers) {
                    String name = extensionLoader.getExtensionName(checker);
                    Status stat;
                    try {
                        stat = checker.check();
                    } catch (Throwable t) {
                        stat = new Status(Status.Level.ERROR, t.getMessage());
                    }
                    statuses.put(name, stat);
                    if (stat.getLevel() != null && stat.getLevel() != Status.Level.UNKNOWN) {
                        List<String> row = new ArrayList<String>();
                        row.add(name);
                        row.add(String.valueOf(stat.getLevel()));
                        row.add(stat.getMessage() == null ? "" : stat.getMessage());
                        table.add(row);
                    }
                }
            }
            Status stat = StatusUtils.getSummaryStatus(statuses);
            List<String> row = new ArrayList<String>();
            row.add("summary");
            row.add(String.valueOf(stat.getLevel()));
            row.add(stat.getMessage());
            table.add(row);
            return TelnetUtils.toTable(header, table);
        } else if (message.length() > 0) {
            return "Unsupported parameter " + message + " for status.";
        }
        String status = channel.getUrl().getParameter("status");
        Map<String, Status> statuses = new HashMap<String, Status>();
        if (status != null && status.length() > 0) {
            String[] ss = Constants.COMMA_SPLIT_PATTERN.split(status);
            for (String s : ss) {
                StatusChecker handler = extensionLoader.getExtension(s);
                Status stat;
                try {
                    stat = handler.check();
                } catch (Throwable t) {
                    stat = new Status(Status.Level.ERROR, t.getMessage());
                }
                statuses.put(s, stat);
            }
        }
        Status stat = StatusUtils.getSummaryStatus(statuses);
        return String.valueOf(stat.getLevel());
    }

}
```

### （八）Help

该接口是帮助文档接口

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface Help {

    String parameter() default "";

    String summary();

    String detail() default "";

}
```

可以看上在每个命令的实现类上都加上了@Help注解，为了添加一些帮助文案。

### （九）TelnetUtils

该类是Telnet命令的工具类，其中逻辑我就不介绍了。

### （十）TelnetCodec

该类继承了TransportCodec，是telnet的编解码类。

#### 1.属性

```java
private static final Logger logger = LoggerFactory.getLogger(TelnetCodec.class);

/**
 * 历史命令列表
 */
private static final String HISTORY_LIST_KEY = "telnet.history.list";

/**
 * 历史命令位置，就是用上下键来找历史命令
 */
private static final String HISTORY_INDEX_KEY = "telnet.history.index";

/**
 * 向上键
 */
private static final byte[] UP = new byte[]{27, 91, 65};

/**
 * 向下键
 */
private static final byte[] DOWN = new byte[]{27, 91, 66};

/**
 * 回车
 */
private static final List<?> ENTER = Arrays.asList(new Object[]{new byte[]{'\r', '\n'} /* Windows Enter */, new byte[]{'\n'} /* Linux Enter */});

/**
 * 退出
 */
private static final List<?> EXIT = Arrays.asList(new Object[]{new byte[]{3} /* Windows Ctrl+C */, new byte[]{-1, -12, -1, -3, 6} /* Linux Ctrl+C */, new byte[]{-1, -19, -1, -3, 6} /* Linux Pause */});
```

#### 2.getCharset

```java
private static Charset getCharset(Channel channel) {
    if (channel != null) {
        // 获得属性设置
        Object attribute = channel.getAttribute(Constants.CHARSET_KEY);
        // 返回指定字符集的charset对象。
        if (attribute instanceof String) {
            try {
                return Charset.forName((String) attribute);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        } else if (attribute instanceof Charset) {
            return (Charset) attribute;
        }
        URL url = channel.getUrl();
        if (url != null) {
            String parameter = url.getParameter(Constants.CHARSET_KEY);
            if (parameter != null && parameter.length() > 0) {
                try {
                    return Charset.forName(parameter);
                } catch (Throwable t) {
                    logger.warn(t.getMessage(), t);
                }
            }
        }
    }
    // 默认的编码是utf-8
    try {
        return Charset.forName(Constants.DEFAULT_CHARSET);
    } catch (Throwable t) {
        logger.warn(t.getMessage(), t);
    }
    return Charset.defaultCharset();
}
```

该方法是获得通道的字符集，根据url中编码来获得字符集，默认是utf-8。

#### 3.encode

```java
@Override
public void encode(Channel channel, ChannelBuffer buffer, Object message) throws IOException {
    // 如果需要编码的是 telnet 命令结果
    if (message instanceof String) {
        //如果为客户端侧的通道message直接返回
        if (isClientSide(channel)) {
            message = message + "\r\n";
        }
        // 获得字节数组
        byte[] msgData = ((String) message).getBytes(getCharset(channel).name());
        // 写入缓冲区
        buffer.writeBytes(msgData);
    } else {
        super.encode(channel, buffer, message);
    }
}
```

该方法是编码方法。

#### 4.decode

```java
@Override
public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
    // 获得缓冲区可读的字节
    int readable = buffer.readableBytes();
    byte[] message = new byte[readable];
    // 从缓冲区读数据
    buffer.readBytes(message);
    return decode(channel, buffer, readable, message);
}

@SuppressWarnings("unchecked")
protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] message) throws IOException {
    // 如果是客户端侧，直接返回结果
    if (isClientSide(channel)) {
        return toString(message, getCharset(channel));
    }
    // 检验消息长度
    checkPayload(channel, readable);
    if (message == null || message.length == 0) {
        return DecodeResult.NEED_MORE_INPUT;
    }

    // 如果回退
    if (message[message.length - 1] == '\b') { // Windows backspace echo
        try {
            boolean doublechar = message.length >= 3 && message[message.length - 3] < 0; // double byte char
            channel.send(new String(doublechar ? new byte[]{32, 32, 8, 8} : new byte[]{32, 8}, getCharset(channel).name()));
        } catch (RemotingException e) {
            throw new IOException(StringUtils.toString(e));
        }
        return DecodeResult.NEED_MORE_INPUT;
    }

    // 如果命令是退出
    for (Object command : EXIT) {
        if (isEquals(message, (byte[]) command)) {
            if (logger.isInfoEnabled()) {
                logger.info(new Exception("Close channel " + channel + " on exit command: " + Arrays.toString((byte[]) command)));
            }
            // 关闭通道
            channel.close();
            return null;
        }
    }
    boolean up = endsWith(message, UP);
    boolean down = endsWith(message, DOWN);
    // 如果用上下键找历史命令
    if (up || down) {
        LinkedList<String> history = (LinkedList<String>) channel.getAttribute(HISTORY_LIST_KEY);
        if (history == null || history.isEmpty()) {
            return DecodeResult.NEED_MORE_INPUT;
        }
        Integer index = (Integer) channel.getAttribute(HISTORY_INDEX_KEY);
        Integer old = index;
        if (index == null) {
            index = history.size() - 1;
        } else {
            // 向上
            if (up) {
                index = index - 1;
                if (index < 0) {
                    index = history.size() - 1;
                }
            } else {
                // 向下
                index = index + 1;
                if (index > history.size() - 1) {
                    index = 0;
                }
            }
        }
        // 获得历史命令，并发送给客户端
        if (old == null || !old.equals(index)) {
            // 设置当前命令位置
            channel.setAttribute(HISTORY_INDEX_KEY, index);
            // 获得历史命令
            String value = history.get(index);
            // 清除客户端原有命令，用查到的历史命令替代
            if (old != null && old >= 0 && old < history.size()) {
                String ov = history.get(old);
                StringBuilder buf = new StringBuilder();
                for (int i = 0; i < ov.length(); i++) {
                    buf.append("\b");
                }
                for (int i = 0; i < ov.length(); i++) {
                    buf.append(" ");
                }
                for (int i = 0; i < ov.length(); i++) {
                    buf.append("\b");
                }
                value = buf.toString() + value;
            }
            try {
                channel.send(value);
            } catch (RemotingException e) {
                throw new IOException(StringUtils.toString(e));
            }
        }
        // 返回，需要更多指令
        return DecodeResult.NEED_MORE_INPUT;
    }
    // 关闭命令
    for (Object command : EXIT) {
        if (isEquals(message, (byte[]) command)) {
            if (logger.isInfoEnabled()) {
                logger.info(new Exception("Close channel " + channel + " on exit command " + command));
            }
            channel.close();
            return null;
        }
    }
    byte[] enter = null;
    // 如果命令是回车
    for (Object command : ENTER) {
        if (endsWith(message, (byte[]) command)) {
            enter = (byte[]) command;
            break;
        }
    }
    if (enter == null) {
        return DecodeResult.NEED_MORE_INPUT;
    }
    LinkedList<String> history = (LinkedList<String>) channel.getAttribute(HISTORY_LIST_KEY);
    Integer index = (Integer) channel.getAttribute(HISTORY_INDEX_KEY);
    // 移除历史命令
    channel.removeAttribute(HISTORY_INDEX_KEY);
    // 将历史命令拼接
    if (history != null && !history.isEmpty() && index != null && index >= 0 && index < history.size()) {
        String value = history.get(index);
        if (value != null) {
            byte[] b1 = value.getBytes();
            byte[] b2 = new byte[b1.length + message.length];
            System.arraycopy(b1, 0, b2, 0, b1.length);
            System.arraycopy(message, 0, b2, b1.length, message.length);
            message = b2;
        }
    }
    // 将命令字节数组，转成具体的一条命令
    String result = toString(message, getCharset(channel));
    if (result.trim().length() > 0) {
        if (history == null) {
            history = new LinkedList<String>();
            channel.setAttribute(HISTORY_LIST_KEY, history);
        }
        if (history.isEmpty()) {
            history.addLast(result);
        } else if (!result.equals(history.getLast())) {
            history.remove(result);
            // 添加当前命令到历史尾部
            history.addLast(result);
            // 超过上限，移除历史的头部
            if (history.size() > 10) {
                history.removeFirst();
            }
        }
    }
    return result;
}
```

该方法是编码。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/telnet

该文章讲解了telnet的相关实现逻辑,本文有兴趣的朋友可以看看。下一篇我会讲解基于grizzly实现远程通信部分。