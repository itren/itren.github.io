---
title: Dubbo 中过滤器（filter）加载流程解析以及建议
categories:
  - microservices
tags:
  - dubbo
abbrlink: 1ec2949
date: 2019-05-29 23:29:53
---

Dubbo 在远程调用的过程中会涉及到服务提供方（Provider）与 服务消费方（Consumer），如果我们希望在这两边调用的过程中添加一些额外的逻辑可以使用 Dubbo 提供给我们的过滤器来实现。

<!--more-->

Dubbo 的拦过滤器使用的是 SPI 机制来实现的，如果要编写过滤器需要先实现 Dubbo 提供的 Filter 接口，然后在“META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Filter”文件夹下配置自己实现的过滤器，Dubbo 为我们提供了一些自带的过滤器，如下所示：

```
cache=com.alibaba.dubbo.cache.filter.CacheFilter
validation=com.alibaba.dubbo.validation.filter.ValidationFilter
echo=com.alibaba.dubbo.rpc.filter.EchoFilter
generic=com.alibaba.dubbo.rpc.filter.GenericFilter
genericimpl=com.alibaba.dubbo.rpc.filter.GenericImplFilter
token=com.alibaba.dubbo.rpc.filter.TokenFilter
accesslog=com.alibaba.dubbo.rpc.filter.AccessLogFilter
activelimit=com.alibaba.dubbo.rpc.filter.ActiveLimitFilter
classloader=com.alibaba.dubbo.rpc.filter.ClassLoaderFilter
context=com.alibaba.dubbo.rpc.filter.ContextFilter
consumercontext=com.alibaba.dubbo.rpc.filter.ConsumerContextFilter
exception=com.alibaba.dubbo.rpc.filter.ExceptionFilter
executelimit=com.alibaba.dubbo.rpc.filter.ExecuteLimitFilter
deprecated=com.alibaba.dubbo.rpc.filter.DeprecatedFilter
compatible=com.alibaba.dubbo.rpc.filter.CompatibleFilter
timeout=com.alibaba.dubbo.rpc.filter.TimeoutFilter
trace=com.alibaba.dubbo.rpc.protocol.dubbo.filter.TraceFilter
future=com.alibaba.dubbo.rpc.protocol.dubbo.filter.FutureFilter
monitor=com.alibaba.dubbo.monitor.support.MonitorFilter
```

**Dubbo 中过滤器的定义会结合“@Activate”注解来使用，如果我们定义了一个过滤器，但是没有在该类上添加“@Activate”的注解，那么默认情况下该注解不会被激活，例如 Dubbo 中的 “CompatibleFilter”就没有添加激活的注解，所以该过滤器默认情况下不会被启用。**

在 Dubbo 中获取过滤器的实现类是 ExtentationLoader，该类可以根据 type 来获取相应的插件，我们这里想获取过滤器的插件，所以 type 的类型是 Filter.class。获取激活的过滤器的入口代码如下：

```
public List<T> getActivateExtension(URL url, String[] values, String group) {
        List<T> exts = new ArrayList<T>();
        List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values);
        if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
            getExtensionClasses();
            for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Activate activate = entry.getValue();
                if (isMatchGroup(group, activate.group())) {
                    T ext = getExtension(name);
                    if (!names.contains(name)
                            && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                            && isActive(activate, url)) {
                        exts.add(ext);
                    }
                }
            }
            Collections.sort(exts, ActivateComparator.COMPARATOR);
        }
        List<T> usrs = new ArrayList<T>();
        for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                    && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
                if (Constants.DEFAULT_KEY.equals(name)) {
                    if (!usrs.isEmpty()) {
                        exts.addAll(0, usrs);
                        usrs.clear();
                    }
                } else {
                    T ext = getExtension(name);
                    usrs.add(ext);
                }
            }
        }
        if (!usrs.isEmpty()) {
            exts.addAll(usrs);
        }
        return exts;
    }
```

该方法会根据 Provider 或者 Consumer 的 url，插件列表以及分组来配置插件，该代码的逻辑主要分为两部分，一部分是配置默认激活的插件，另外一部分是配置用户指定的过滤器。

## 1. 装载默认激活的过滤器

上述代码中有一行代码是“getExtensionClasses()”，该方法的作用是获取所有配置过的 Filter，它会扫描“META-INF/dubbo/internal/”、“META-INF/dubbo/”、“META-INF/services/” 目录下的配置文件，getExtensionClasses() 方法代码如下：

```
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

可以看到 Dubbo 并不会每次都去扫描这些插件，如果在缓存中找不到这些插件才会去调用 loadExtensionClasses()，loadExtensionClasses() 代码如下：

```
// synchronized in getExtensionClasses
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

我们可以注意到装载插件时会去三个目录下面去装载插件，这就是开头所说的三个目录的依据。当然扫描完三个目录中所有的过滤器配置之后还需要进行筛选才会进行应用，筛选的逻辑代码如下：

```
 for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Activate activate = entry.getValue();
                if (isMatchGroup(group, activate.group())) {
                    T ext = getExtension(name);
                    if (!names.contains(name)
                            && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                            && isActive(activate, url)) {
                        exts.add(ext);
                    }
                }
            }
```

首先第一个判断条件是 isMatchGroup，如果过滤器的分组与当前需要的分组不匹配那么会被排除掉，第二个条件是用户在 @Reference 注解上显示配置的过滤器也会排除掉，第三个条件是如果用户显示配置的插件名前面加上了“-”前缀，那么也会进行排除，第四个条件是如果过滤器配置了关键字，那么会对 url 中的参数进行关键字匹配，只有匹配上的过滤器才会保留下来。

## 2. 对激活的过滤器进行排序

上面已经讲解了默认激活的过滤器的装载，Dubbo 中的过滤器也是有优先级之分的，我们需要弄清楚不同过滤器执行的先后顺序：

```
 Collections.sort(exts, ActivateComparator.COMPARATOR);
```

上面这行代码中 exts 是 List 类型的，Dubbo 使用 Collections 中的方法进行排序，排序的逻辑在 ActivateComparator.COMPARATOR 这个类中：

```
public class ActivateComparator implements Comparator<Object> {

    public static final Comparator<Object> COMPARATOR = new ActivateComparator();

    @Override
    public int compare(Object o1, Object o2) {
        if (o1 == null && o2 == null) {
            return 0;
        }
        if (o1 == null) {
            return -1;
        }
        if (o2 == null) {
            return 1;
        }
        if (o1.equals(o2)) {
            return 0;
        }
        Activate a1 = o1.getClass().getAnnotation(Activate.class);
        Activate a2 = o2.getClass().getAnnotation(Activate.class);
        if ((a1.before().length > 0 || a1.after().length > 0
                || a2.before().length > 0 || a2.after().length > 0)
                && o1.getClass().getInterfaces().length > 0
                && o1.getClass().getInterfaces()[0].isAnnotationPresent(SPI.class)) {
            ExtensionLoader<?> extensionLoader = ExtensionLoader.getExtensionLoader(o1.getClass().getInterfaces()[0]);
            if (a1.before().length > 0 || a1.after().length > 0) {
                String n2 = extensionLoader.getExtensionName(o2.getClass());
                for (String before : a1.before()) {
                    if (before.equals(n2)) {
                        return -1;
                    }
                }
                for (String after : a1.after()) {
                    if (after.equals(n2)) {
                        return 1;
                    }
                }
            }
            if (a2.before().length > 0 || a2.after().length > 0) {
                String n1 = extensionLoader.getExtensionName(o1.getClass());
                for (String before : a2.before()) {
                    if (before.equals(n1)) {
                        return 1;
                    }
                }
                for (String after : a2.after()) {
                    if (after.equals(n1)) {
                        return -1;
                    }
                }
            }
        }
        int n1 = a1 == null ? 0 : a1.order();
        int n2 = a2 == null ? 0 : a2.order();
        // never return 0 even if n1 equals n2, otherwise, o1 and o2 will override each other in collection like HashSet
        return n1 > n2 ? 1 : -1;
    }

}
```

从这个类中我们可以看到 @Active 中的 order、before、after 等注解会影响排序的结果，其中 before、after 的优先级较高，当然如果没有显示配置 order 的话，默认的优先级是 0，根据上述代码也可以知道 order 越小优先级越高，这个 Servlet 中的规则是类似的。

## 3. 装载用户指定的过滤器

当系统默认的过滤器加载完之后，Dubbo 会去加载用户配置的 filters，代码如下：

```
for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                    && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
                if (Constants.DEFAULT_KEY.equals(name)) {
                    if (!usrs.isEmpty()) {
                        exts.addAll(0, usrs);
                        usrs.clear();
                    }
                } else {
                    T ext = getExtension(name);
                    usrs.add(ext);
                }
            }
        }
```

用户指定类型的过滤器加载过程比较简单，主要是配置中没有包含“-”就可以了，当然 Dubbo 还进行一些特殊的处理，例如“-filter”进行了优化，如果用户在前面加上了空格之类的字符也是可以进行识别的。当加载用户指定的过滤器时，里面有一个关键字“default”，在“default”前面的过滤器会装载到系统过滤器的前面，反之就在后面，上面代码中逻辑可以验证官网的说法。

## 4. 自定义 filter 建议

一般这些 filter 可以按功能分为中间件的 filter 和 业务 filter，中间件的 filter 会设计得更加通用一些，业务的 filter 更加具有针对性。这两种 filter 在设计的时候建议区别对待，一般中间件的 filter 我们可以定义成系统级别的 filter，也就是在 filter 上面加上“@Active” 注解，而业务使用的 filter 不建议加上这个注解。这样做的好处就是业务的 filter 会始终在系统级别 filter 的前面或者后面，不会破坏中间件原有 filter 的逻辑，也有利于问题的排查，业务人员如果要激活这个 filter，需要自行在配置文件中显示的进行配置，可以使用“default”来控制业务 filter 生效的顺序。