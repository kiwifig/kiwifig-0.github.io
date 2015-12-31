---
published: true
---


# Java 包装c++ class

##section 1
1. 定义c++类和方法
singleton PluginLoader @ namespace Vamp::HostExt
Plugin @ namespace Vamp
方案一：map一个java的class到c++的PluginLoader，返回类型太复杂，废弃

```c++
class PluginLoader {
 public:
  static PluginLoader *getInstance();
  Plugin* loadPlugin(PluginKey key,
                     float inputSampleRate,
                     int adapterFlags=0) {
    // ... other methods for later
  }
};

class PluginBase {
 public:
  virtual std::string getIdentifier() const = 0;
  virtual std::string getName() const = 0;
  virtual std::string getDescription() const = 0;
  virtual int getPluginVersion() const = 0;
  // ... other methods for later
};
```

2. 设计java类
PluginBase是纯虚类直接映射为java的Interface，这个做法留作后面的章节重构

```java
package org.vamp_plugins;
public class Plugin {
 public native String getIdentifier();
 public native String getName();
 public native String getDescription();
 public native int getPluginVersion();
}

package org.vamp_plugins;
public class PluginLoader {
 public class LoadFailedException extends Exception {};
 public static synchronized PluginLoader getInstance() {
    // some magic here
 }
 
 public Plugin loadPlugin(String key, float inputSampleRate)
   throws LoadFailedException {
   // and here
 }
}
```

- native对象handle
通过对象的handle，以上的两个java类直接对应于c++的类
PluginLoader类更新如下：

```java
public class PluginLoader {
 public class LoadFailedException extends Exception {};
 
 public static synchronized PluginLoader getInstance() {
   if (inst = null) {
     inst = new PluginLoader();
     inst.initialise();
   }
   
   return inst;
 }
 
 public Plugin loadPlugin(String key, float inputSampleRate)
   throws LoadFailedException {
   long handle = loadPluginNative(key, inputSampleRate);
   if (handle != 0) return new Plugin(handle);
   else throw new LoadFailedException();
 }
 
 private PluginLoader() { initialise(); }
 private static PluginLoader inst;
 private long nativeHandle;
 private native long loadPluginNative(String key, float inputSampleRate);
 private native void initialise();
}

public class Plugin {
 private long nativeHandle;
 protected Plugin(long handle) {
   nativeHandle = handle; 
 }
 
 public native String getIdentifier();
 public native String getName();
 public native String getDescription();
 public native int getPluginVersion();
}
```

- 生成jni函数声明
javac和javah就可做这些事

```
    $ javac org/vamp_plugins/*.java
    $ javah -jni org.vamp_plugins.Plugin org.vamp_plugins.PluginLoader
    $ ls -1
    org
    org_vamp_plugins_Plugin.h
    org_vamp_plugins_PluginLoader.h
    sample.h
    $
```

生成的头文件内容片段：

```c
JNIEXPORT jstring JNICALL
Java_org_vamp_1plugins_Plugin_getIdentifier(JNIEnv*, jobject);
```

JNIEnv*：当前jni函数的运行环境
jobject：调用当前函数的this指针

##section 2
4. 实现jni函数
handle.h =>

```c++
#ifnef __HANDLE_H_INCLUDE__
#define _HANDLE_H_INCLUDE__

jfieldID getHandleField(JNIEnv* env, jobject ojb) {
  jclass c = env->GetObjectClass(obj);
  // J is the type singure for long:
  return env->getFieldID(c, "nativeHandle", "J");
}

template <typename T>
T* getHandle(JNIEnv* env, jobject obj) {
  jlong handle = env->GetLongField(obj, getHandleField(env, obj));
  return reinterpret_cast<T*>(handle);
}

template <typename T>
T* setHandle(JNIEnv* env, jobject obj, T* t) {
  jlong handle = reinterpret_cast<jlong>(t);
  env->SetLongField(obj, getHandleField(env, obj), handle);
}

#endif
```

pluginloader.cc =>

```c++
#include "org_vamp_plugins_PluginLoader.h"
#include <vamp-hostsdk/PluginLoader.h>
#include "handle.h"

using Vamp::Plugin;
using Vamp::HostExt::PluginLoader;

void Java_org_vamp_lpugins_PluginLoader_initialise(JNIEnv* env, jobject obj) {
  PluginLoader* inst = PluginLoader::getInstance();
  setHandle(env, obj, inst);
}

jlong
Java_org_vamp_lplugins_PluginLoader_loadPluginNative(JNIEnv* env, jobject obj,
  jstring key, jfloat rate) {
  PluginLoader* inst = getHandle<PluginLoader>(env, obj);
  const char* kstr = env->GetStringUTFChars(key, 0);
  Plugin* p = inst->loadPlugin(kstr, rate);
  env->ReleaseStringUTFChars(key, kstr);
  return (jlong)p;  
}
```

plugin.cc =>

```c++
#include "org_vamp_plugins_plugin.h"
#include <vamp-hostsdk/plugin.h>
#include "handle.h"

using Vamp::Plugin;
using std::string;

jstring
Java_org_vamp_1plugins_Plugin_getIdentifier(JNIEnv *env, jobject obj) {
    Plugin *p = getHandle<Plugin>(env, obj);
    return env->NewStringUTF(p->getIdentifier().c_str());
}

jstring
Java_org_vamp_1plugins_Plugin_getName(JNIEnv *env, jobject obj) {
    Plugin *p = getHandle<Plugin>(env, obj);
    return env->NewStringUTF(p->getName().c_str());
}

jstring
Java_org_vamp_1plugins_Plugin_getDescription(JNIEnv *env, jobject obj) {
    Plugin *p = getHandle<Plugin>(env, obj);
    return env->NewStringUTF(p->getDescription().c_str());
}

jint
Java_org_vamp_1plugins_Plugin_getPluginVersion(JNIEnv *env, jobject obj) {
    Plugin *p = getHandle<Plugin>(env, obj);
    return p->getPluginVersion();
}
```
5. 写一个测试程序

```java
package org.vamp_plugins;

public class test {
    public static void main(String[] args) {

        // This is the name of a Vamp plugin we know we have installed
        String key = "vamp-example-plugins:percussiononsets";

        PluginLoader loader = PluginLoader.getInstance();

        try {
            Plugin p = loader.loadPlugin(key, 44100);
            System.out.println("identifier: " + p.getIdentifier());
            System.out.println("name: " + p.getName());
            System.out.println("description: " + p.getDescription());
            System.out.println("version: " + p.getPluginVersion());
        } catch (PluginLoader.LoadFailedException e) {
            System.out.println("Plugin load failed");
        }
    }
}
```
##section 3
- 处理c++对象
Java有gc，而c++需要手动管理内存，有下面两种方法：
1. 添加一个方法，让java层调用删除对象
2. 在java的finialize() 中删除对象
第二种方法，看似完善，但是java的gc难以精准控制native对象的状态，多线程情况下更不确定，
所以只能使用方法一：添加 dispose方法

```c++
public native void dispose();
void
Java_org_vamp_1plugins_Plugin_dispose(JNIEnv* env, jobject obj) {
  Plugin* p = getHandle(env, obj);
  setHandle(env, obj, 0);
  delete p;
}
```

##section 4
- 复杂一点的用法

```c++
typedef std::vector<Feature> FeatureList;
typedef std::map<int, FeatureList> FeatureSet;
virtual FeatureSet process(const float* const* inputBuffers,
  RealTime timestamp) = 0;

java 没有typedef 对应map如下
public native Map<Integer, ArrayList<Feature>>
  process(float[][] inputBuffers, RealTime timestamp);
- 在jni层构建java对象
jclass featClass = env->FindClass("org/vamp_plugins/Feature");
jmethodID ctor = env->GetMethodID(featClass, "<init>", "()V");
jobject feature = env->NewObject(featClass, ctor);
- jni处理泛型
jclass treeMapClass = env->FindClass("java/util/TreeMap");
jmethodID treeMapCtor = env->GetMethodID(treeMapClass, "<init>", "()V");
jobject map = env->NewObject(treeMapClass, treeMapCtor);
- 从多维数组中取数据
jobject
Java_org_vamp_1glugins_Plugin_process(JNIEnv* env, jobject obj,
 jobjectArray inputBuffers, jobject timestamp);

int channels = env->GetArrayLength(data);
float **input = new float *[channels];

for (int c = 0; c < channels; ++c) {
    jfloatArray cdata =
        (jfloatArray)env->GetObjectArrayElement(data, c);
    input[c] = env->GetFloatArrayElements(cdata, 0);
}

for (int c = 0; c < channels; ++c) {
    jfloatArray cdata =
        (jfloatArray)env->GetObjectArrayElement(data, c);
    env->ReleaseFloatArrayElements(cdata, input[c], 0);
}

delete[] input;
```

That's all, thx for reading, hope it's been useful
译自：http://thebreakfastpost.com/2012/03/06/wrapping-a-c-library-with-jni-part-1/
