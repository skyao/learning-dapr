---
title: "ObjectSerializer"
linkTitle: "ObjectSerializer"
weight: 30
date: 2021-02-24
description: >
  ObjectSerializer 是 dapr 的默认对象序列化器
---

### 类定义

```java
public class ObjectSerializer {
  // 默认构造函数，以避免类在包外被实例化，但仍可以被继承。
  protected ObjectSerializer() {
  }
}
```

### jackson 相关设置

```java
  protected static final ObjectMapper OBJECT_MAPPER = new ObjectMapper()
      .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
      .setSerializationInclusion(JsonInclude.Include.NON_NULL);
```

### serialize() 方法实现

```java
public byte[] serialize(Object state) throws IOException {
    if (state == null) {
      return null;
    }

    if (state.getClass() == Void.class) {
      return null;
    }

    // Have this check here to be consistent with deserialization (see deserialize() method below).
    if (state instanceof byte[]) {
      return (byte[]) state;
    }

    // Proto buffer class is serialized directly.
    if (state instanceof MessageLite) {
      return ((MessageLite) state).toByteArray();
    }

    // Not string, not primitive, so it is a complex type: we use JSON for that.
    return OBJECT_MAPPER.writeValueAsBytes(state);
  }
```

### deserialize() 方法实现

这两个方法都是简单代理：

```java
  public <T> T deserialize(byte[] content, TypeRef<T> type) throws IOException {
    return deserialize(content, OBJECT_MAPPER.constructType(type.getType()));
  }

  public <T> T deserialize(byte[] content, Class<T> clazz) throws IOException {
    return deserialize(content, OBJECT_MAPPER.constructType(clazz));
  }
```

具体实现在这里：

```java
  private <T> T deserialize(byte[] content, JavaType javaType) throws IOException {
    // 对应 serialize 的做法
    if ((javaType == null) || javaType.isTypeOrSubTypeOf(Void.class)) {
      return null;
    }

    // 如果是 java 基本类型，则交给 deserializePrimitives() 方法处理
    // 注意此时 content 有可能是 null 或者 空数组
    if (javaType.isPrimitive()) {
      return deserializePrimitives(content, javaType);
    }

    // 对应 serialize 的做法
    if (content == null) {
      return null;
    }

    // Deserialization of GRPC response fails without this check since it does not come as base64 encoded byte[].
    // 如果没有这个检查，GRPC响应的反序列化就会失败，因为它不是以 base64 编码的 byte[] 形式出现的。
    // TBD：这里有点不是太理解
    if (javaType.hasRawClass(byte[].class)) {
      return (T) content;
    }

    // // 对应 serialize 的做法，但长度为零的检测放在 byte[] 检测之后
    if (content.length == 0) {
      return null;
    }

    // 对 CloudEvent 的支持：如果是 CloudEvent，则单独序列化
    if (javaType.hasRawClass(CloudEvent.class)) {
      return (T) CloudEvent.deserialize(content);
    }

    // 对 grpc MessageLite 的支持：通过反射调用 parseFrom 方法
    if (javaType.isTypeOrSubTypeOf(MessageLite.class)) {
      try {
        Method method = javaType.getRawClass().getDeclaredMethod("parseFrom", byte[].class);
        if (method != null) {
          return (T) method.invoke(null, content);
        }
      } catch (NoSuchMethodException e) {
        // It was a best effort. Skip this try.
      } catch (Exception e) {
        throw new IOException(e);
      }
    }

    // 最后才通过 jackson 进行标准的 json 序列化
    return OBJECT_MAPPER.readValue(content, javaType);
  }
```

### deserializePrimitives() 方法

对原生类型的解析：

```java
private static <T> T deserializePrimitives(byte[] content, JavaType javaType) throws IOException {
    if ((content == null) || (content.length == 0)) {
      // content 为null或者空的特殊处理，相当于是缺省值
      if (javaType.hasRawClass(boolean.class)) {
        return (T) Boolean.FALSE;
      }

      if (javaType.hasRawClass(byte.class)) {
        return (T) Byte.valueOf((byte) 0);
      }

      if (javaType.hasRawClass(short.class)) {
        return (T) Short.valueOf((short) 0);
      }

      if (javaType.hasRawClass(int.class)) {
        return (T) Integer.valueOf(0);
      }

      if (javaType.hasRawClass(long.class)) {
        return (T) Long.valueOf(0L);
      }

      if (javaType.hasRawClass(float.class)) {
        return (T) Float.valueOf(0);
      }

      if (javaType.hasRawClass(double.class)) {
        return (T) Double.valueOf(0);
      }

      if (javaType.hasRawClass(char.class)) {
        return (T) Character.valueOf(Character.MIN_VALUE);
      }

      return null;
    }

    // 对于非空值，通过 jackson 进行反序列化
    return OBJECT_MAPPER.readValue(content, javaType);
  }
  ```

  ## 总结

  这个代码中，在 jackson 处理之前有很多特殊逻辑，这些逻辑理论上应该是独立于 jackson 序列化方案的，如果要引入其他 DaprObjectSerializer 的实现，这些特殊逻辑都要重复 n 次，有代码重复和逻辑不一致的风险。

  最好是能把这些逻辑提取出来，在序列化和反序列化时先用这些特殊逻辑出来一遍，最后再交给 DaprObjectSerializer ，会比较合理。

  再有就是依赖冲突问题，目前的 DaprObjectSerializer 方案没有给出完整的解决方案。jackson 的依赖还是写死的。

  