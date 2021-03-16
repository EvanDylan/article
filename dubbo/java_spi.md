# 1.SPI的定义

	A service provider interface(SPI) is the set of public interfaces and abstract classes that a service defines.
	SPI是定义了一组公共的接口和抽象类的服务类。其实就是定义了一组公共的接口或者抽象方法，例如Java Database Connectivity (JDBC) 规范，对于不同的数据库厂商而言是SPI，但是对于使用JDBC的开发者而言又是API。
	SPI描述了服务厂商需要拓展实现的内容规范，API则描述了如何、怎么使用的内容。

# 2. 入门示例	
````javax.sound.sampled````和````javax.sound.midi````包是提供给那些需要处理音频工作的开发者使用，软件包提供了信息获取、控制和访问音频相关接口。此外Java Sound API同样提供了````javax.sound.sampled.spi````和 ````javax.sound.midi.spi````相关软件包，其定义了音频相关的抽象类。开发者想要提供新的音频相关的服务，只需要实现在SPI包下的具体类的子类即可。 这些类以及支持新服务的附加类，都被放到Jar文件中，并包含了对所附加了对一个或者多个服务的描述。当这个Jar包被载入到用户的CLASSPATH下，运行时系统就会自动的载入将新的服务类(扩展了Java运行时系统的功能)。一旦新的服务类被载入，它可以像其他已经预先存在的类一样被正常访问。通过调用````AudioSystem````和````MidiSystem````类下的方法，调用者可以获取服务类的相关信息，也可以创建出服务类的实例。**应用程序不需要也不应该直接引用位于SPI包中的类来使用已经载入的服务。**
      
   举个栗子：现在有一个Acme软件公司提供了新的服务类，允许应用程序读取新格式的音频文件。新的类就假设称为````AcmeAudioFileReader````，其提供实现了````AudioFileReader````类的所有定义的方法（````AudioFileReader````仅有两个方法````getAudioFileFormat````和````getAudioInputStream````）。当一个应用程序试图读取音频文件，碰巧格式就是Acme时，就会调用````AudioSystem````类的方法来访问文件及其相关信息。````AudioSystem.getAudioInputStream````和````AudioSystem.getAudioFileFormat````方法提供了一个读取音频流的标准API，在载入了AcmeAudioFileReader类的情况下，该接口被扩展为透明地支持读取新格式的音频文件。开发者不需要直接访问新注册的SPI类：通过````AudioSystem````对象的方法将参数传递给````AcmeAudioFileReader````。

例子中使用这些**工厂类**的目的是什么？为什么不让应用开发者直接访问新的服务类？这虽然是一种可行的途径，但是通过统一的入口去管理和载入这些服务类，使得开发人员不必知道每个载入的服务类是做什么的。开发者只需要通过统一的入口使用在这些服务类拿到想要的结果即可，与此同时厂商也可以借助这种架构有效的去管理包下可用资源。

通常使用新的音频类对应用程序是透明的。 例如，想象一下，应用程序开发人员想要从文件中读取音频流的情况。 假设````thePathName````标识一个音频输入文件，程序大概是这样子的：
	
````
File theInFile = new File(thePathName);
AudioInputStream theInStream = AudioSystem.getAudioInputStream(theInFile); 
````
在这个场景下，````AudioSystem````检测到哪些被载入的服务可以读取这个文件，并要求对应的服务类将音频数据以一个````AudioInputStream````对象返回.开发者可能并不知道甚至不在意输入的音频文件使用了新的第三方提供的格式。程序的第一次使用到的流对象是通过````AudioSystem````来完成，而后续的所有的访问流和流属性的都是通过````AudioInputStream ````的方法来完成。这两个都是````javax.sound.sampled API````的标准对象；所有可能的针对新文件格式的特殊处理的细节都已经被完全的屏蔽了。
# 3. 窥探AudioSystem的秘密
上面例子中````AudioSystem````究竟有什么神奇的魔法，可以知道哪些被载入的服务类可以读取这个文件，所谓源码面前无秘密，直接"上菜"。

## 3.1 AudioSystem是如何找到对应解析类
````
public static AudioInputStream getAudioInputStream(File file)
        throws UnsupportedAudioFileException, IOException {

        List providers = getAudioFileReaders();
        AudioInputStream audioStream = null;

        for(int i = 0; i < providers.size(); i++ ) {
            AudioFileReader reader = (AudioFileReader) providers.get(i);
            try {
                audioStream = reader.getAudioInputStream( file ); // throws IOException
                break;
            } catch (UnsupportedAudioFileException e) {
                continue;
            }
        }

        if( audioStream==null ) {
            throw new UnsupportedAudioFileException("could not get audio input stream from input file");
        } else {
            return audioStream;
        }
    }
````
首先通过````getAudioFileReaders()````方法获取所有````AudioFileReader````第三方实现类的实例，然后循环遍历````AudioFileReader````对象，如果当前对象不支持当前文件格式则抛出````UnsupportedAudioFileException````异常，捕获异常继续循环。神奇的魔法已经被揭开：````AudioSystem````在这里其实隐式的约定，第三方实现类如果不支持当前文件格式的读取则需要抛出特定的异常作为信号。

## 3.2 解析类是如何被载入的
通过追踪````getAudioFileReaders()````，追溯到如下关键代码，

````
static synchronized <T> List<T> getProviders(final Class<T> var0) {
        ArrayList var1 = new ArrayList(7);
        PrivilegedAction var2 = new PrivilegedAction<Iterator<T>>() {
            public Iterator<T> run() {
                return ServiceLoader.load(var0).iterator();
            }
        };
        final Iterator var3 = (Iterator)AccessController.doPrivileged(var2);
        PrivilegedAction var4 = new PrivilegedAction<Boolean>() {
            public Boolean run() {
                return var3.hasNext();
            }
        };

        while(((Boolean)AccessController.doPrivileged(var4)).booleanValue()) {
            try {
                Object var5 = var3.next();
                if (var0.isInstance(var5)) {
                    var1.add(0, var5);
                }
            } catch (Throwable var6) {
                ;
            }
        }

        return var1;
    }
````
同样只需要关心主要代码````ServiceLoader.load(var0).iterator();````，借助于ServiceLoader来完成的载入实例的工作。

# 4. 使用Java的SPI机制
## 4.1 使用步骤

	1. SPI定义
	2. 统一访问入口
	3. 实现SPI接口并加以描述实现了哪个SPI。(描述文件约定如下：放到META-INF/services/这个路      
	   径下；并以接口全路径作为文件名称，以实现接口类的全路径作为文件内容)

## 4.2 实践	

### 4.2.1 SPI定义：

````
public interface ObjectResourceStore<K ,V> {

    V read(K k) throws UnsupportedOperationException;

    void store(K k, V v) throws UnsupportedOperationException;

}
````
定义了一个具有储取Object对象的功能的SPI，提供了````read()````和````store()````方法。

### 4.2.2 统一访问入口
````
public final class ObjectResourceStores<K,V> {

    private static synchronized List<ObjectResourceStore> loaderProviders() {
        List<ObjectResourceStore> providers = new ArrayList();
        Iterator iterator = ServiceLoader.load(ObjectResourceStore.class).iterator();
        while (iterator.hasNext()) {
            providers.add((ObjectResourceStore) iterator.next());
        }
        return providers;
    }

    public static<K, V> V read(K k) throws UnsupportedOperationException {
        List<ObjectResourceStore> providers = loaderProviders();
        for (int i = 0; i < providers.size(); i++) {
            ObjectResourceStore provider = providers.get(i);
            try {
                return (V) provider.read(k);
            } catch (UnsupportedOperationException e) {
                continue;
            }
        }
        throw new UnsupportedOperationException("can not find suitable provider");
    }

    public static<K, V> void store(K k, V v) throws UnsupportedOperationException {
        List<ObjectResourceStore> providers = loaderProviders();
        for (int i = 0; i < providers.size(); i++) {
            ObjectResourceStore provider = providers.get(i);
            try {
                provider.store(k, v);
                return;
            } catch (UnsupportedOperationException e) {
                continue;
            }
        }
        throw new UnsupportedOperationException("can not find suitable provider");
    }
}
````
### 4.2.3 实现SPI
````
public class ObjectResourceMemoryStore implements ObjectResourceStore{

    private static ConcurrentHashMap store = new ConcurrentHashMap<Object, Object>();


    @Override
    public Object read(Object o) throws UnsupportedOperationException {
        return store.get(o);
    }

    @Override
    public void store(Object o, Object o2) throws UnsupportedOperationException {
        store.put(o, o2);
    }
}
````
创建描述文件
	![](D:\source code\article\dubbo\img\java_spi.webp)
文件内容如下：````org.rhine.ObjectResourceMemoryStore````

## 4.3 测试
测试代码如下：

````
public class TestSPI {

    public static void main(String[] args) {

        ObjectResourceStores.store(1, 1);
        System.out.println(ObjectResourceStores.read(1).toString());
    }

}

程序能够正确的输出结果:1
````
至此一个完整的SPI的demo已经完成，大家如果有兴趣可以继续深入的研究下SPI在开源框架中的具体实践。比如Spring、Dubbo都有非常典型的应用场景。

# 5. 写在最后
希望博客的内容能给广大的Java道友提供一些的帮助和提升。由于笔者水平有限，如果内容有误，希望大家批评指出，索要源码的同学可以直接私信。


**参考文献**

1. [官方SPI介绍](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html)


​	
​	
​	