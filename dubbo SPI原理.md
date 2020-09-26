​									

# 																						Dubbo SPI原理

这里以***Protocol***为例展开讲解。

```java
public class ServiceConfig<T> extends AbstractServiceConfig {

    private static final Protocol protocol = 
       //获取Protocol 的扩展加载器ExtensionLoader
        ExtensionLoader.getExtensionLoader(Protocol.class)
            .getAdaptiveExtension();

    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class)
            .getAdaptiveExtension();

```

```java
//ExtensionLoader

@SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            //生成并返回对应类的ExtensionLoader
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }

private ExtensionLoader(Class<?> type) {
        this.type = type;
    	//非ExtensionFactory类的ExtensionLoader都要先获取到ExtensionFactory的实例对象，这个对象用来获取对应类的扩展类各个属性的填充。
    	//自适应扩展对象，ExtensionFactory的ExtensionLoader不需要指定objectFactory
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

```java
 public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            // 创建自适应拓展
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }

 /***
     * 1. 调用 getAdaptiveExtensionClass 方法获取自适应拓展 Class 对象
     * 2. 通过反射进行实例化
     * 3. 调用 injectExtension 方法向拓展实例中注入依赖
     * @return
     */
    @SuppressWarnings("unchecked")
    private T createAdaptiveExtension() {
        try {
            // 获取自适应扩展类，并进行实例化、依赖注入ioc注入
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }


 /***
     * 1. 调用 getExtensionClasses 获取所有的拓展类
     * 2. 检查缓存，若缓存不为空，则返回缓存(若实现类中某子类类上有@Adaptive注解修饰，则将该类赋值给cachedAdaptiveClass)
     * 3. 若缓存为空，则调用 createAdaptiveExtensionClass 创建自适应拓展类
     * @return
     */
    private Class<?> getAdaptiveExtensionClass() {
        // 通过SPI获取所有的扩展类
        getExtensionClasses();
        // 检查缓存，若某扩展类上用@Adaptive修饰，则缓存不为空，直接返回缓存值。
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        // 创建自适应扩展类
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }


 // synchronized in getExtensionClasses
    private Map<String, Class<?>> loadExtensionClasses() {
        // 如果type所代表的接口被SPI注解
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            // 从注解获取默认配置项名，用来获取指定的扩展类名称
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
        // 构建<配置项名>-<配置类>的映射关系
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        // 加载指定文件夹下的配置文件
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error when load extension class(interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + "is not subtype of interface.");
        }
        // 目标类是否有Adaptive注解
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            if (cachedAdaptiveClass == null) {
                // 设置 cachedAdaptiveClass缓存
                cachedAdaptiveClass = clazz;
            } else if (!cachedAdaptiveClass.equals(clazz)) {
                throw new IllegalStateException("More than 1 adaptive class found: "
                        + cachedAdaptiveClass.getClass().getName()
                        + ", " + clazz.getClass().getName());
            }
        // 通过 clazz 的构造方法，检测 clazz 是否是 Wrapper 类型。 存储在cachedWrapperClasses缓存，用于AOP实现
        } else if (isWrapperClass(clazz)) {
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers == null) {
                cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                wrappers = cachedWrapperClasses;
            }
            // 存储 clazz 到 cachedWrapperClasses 缓存中
            wrappers.add(clazz);
        } else {
            // 检测 clazz 是否有默认的构造方法，如果没有，则抛出异常
            clazz.getConstructor();
            if (name == null || name.length() == 0) {
                // 如果 name 为空，则尝试从 Extension 注解中获取 name，或使用小写的类名作为 name
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }
            String[] names = NAME_SEPARATOR.split(name);
            if (names != null && names.length > 0) {
                Activate activate = clazz.getAnnotation(Activate.class);
                if (activate != null) {
                    // 如果类上有 Activate 注解，则使用 names 数组的第一个元素作为键，存储 name 到 Activate 注解对象的映射关系
                    cachedActivates.put(names[0], activate);
                }
                // 可支持配置项名用“,”分割，指定多个
                for (String n : names) {
                    if (!cachedNames.containsKey(clazz)) {
                        cachedNames.put(clazz, n);
                    }
                    Class<?> c = extensionClasses.get(n);
                    if (c == null) {
                        // 存储<配置项名>-<配置类>的映射关系
                        extensionClasses.put(n, clazz);
                    } else if (c != clazz) {
                        throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                    }
                }
            }
        }
    }
```

dubbo将从

###### META-INF/dubbo/internal/

###### META-INF/dubbo/

###### META-INF/services/

这些目录下面加载对应的配置文件，例如/META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory

##### 																dubbo spi加载配置文件

![image-20200923195824327](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200923195824327.png)

![image-20200925174703413](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200925174703413.png)

根据文件配置的类路径加载类。

![image-20200923194930374](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200923194930374.png)



Dubbo SPI **loadClass**对应接口的实现类时会判断三种类，一种是被**Adaptive**注解修饰的类，一种是包装类，除此之外的其他实现类走默认逻辑，在默认逻辑中会判断该类是否**Activate**注解修饰，如果有该注解，则缓存到**cachedActivates**，会将走默认逻辑的所有实现类加载到**extensionClasses**缓存中。

```java
#ExtensionLoader

 //从ObjectFactory取出adaptive对象依赖的实例的注入实例的属性	
private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                // 遍历所有方法
                for (Method method : instance.getClass().getMethods()) {
                    // 以set开头，且只有一个参数的public方法
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        // @DisableInject 修饰的方法不进行注入
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        // 获得参数类型
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            // 获取属性名
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            // 从ObjectFactor中获取依赖对象
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                // 赋值
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

**Protocol**接口没有对应的**adaptive**类，需要通过字节码生成技术生成对应的adaptive类。

![image-20200926181049633](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200926181049633.png)

```java
//运行时生成的Protocol接口的adaptive类

package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements Protocol {
    public void destroy() {throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }
    public int getDefaultPort() {throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }
    public Invoker refer(Class arg0, com.alibaba.dubbo.common.URL arg1) throws RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
    public Exporter export(Invoker arg0) throws RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```

Dubbo中生成的**Adaptive**对象其实就是目标接口的一个代理实现对象，如上面的**Protocol$Adaptive**就是实现了**Protocol**接口的代理类，真正处理**refer**和**export**是由**Protocol$Adaptive**根据url中的**extName**参数决定去**ExtensionLoader**中加载具体实例来执行对应的**refer**和**export**的。