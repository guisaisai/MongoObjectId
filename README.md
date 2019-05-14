# MongoObjectId

## 环境

springboot 2.1.0

spring-boot-starter-data-mongodb

## 概述
mongo string 类型的主键不插入值会默认创建,以为是mongo数据库创建的 查看源码才发现是客户端调用insert的时候创建的,数据库类型是ObjectId

下面看源码:
```
com.mongodb.DBCollection
public class DBCollection {
...

public WriteResult insert(final List<? extends DBObject> documents, final InsertOptions insertOptions) {
        WriteConcern writeConcern = insertOptions.getWriteConcern() != null ? insertOptions.getWriteConcern() : getWriteConcern();
        Encoder<DBObject> encoder = toEncoder(insertOptions.getDbEncoder());

        List<InsertRequest> insertRequestList = new ArrayList<InsertRequest>(documents.size());
        for (DBObject cur : documents) {
            if (cur.get(ID_FIELD_NAME) == null) {
                cur.put(ID_FIELD_NAME, new ObjectId());
            }
            insertRequestList.add(new InsertRequest(new BsonDocumentWrapper<DBObject>(cur, encoder)));
        }
        return insert(insertRequestList, writeConcern, insertOptions.isContinueOnError(), insertOptions.getBypassDocumentValidation());
    }

...
}

```
其中insert方法
```
if (cur.get(ID_FIELD_NAME) == null) {
                cur.put(ID_FIELD_NAME, new ObjectId());
            }
```
判断id不存在就实例一个ObjectId进去

## 看一下ObjectId
```
看一下注释
/**
 * <p>A globally unique identifier for objects.</p>
 *
 * <p>Consists of 12 bytes, divided as follows:</p>
 * <table border="1">
 *     <caption>ObjectID layout</caption>
 *     <tr>
 *         <td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td><td>10</td><td>11</td>
 *     </tr>
 *     <tr>
 *         <td colspan="4">time</td><td colspan="3">machine</td> <td colspan="2">pid</td><td colspan="3">inc</td>
 *     </tr>
 * </table>
 *
 * <p>Instances of this class are immutable.</p>
 *
 * @mongodb.driver.manual core/object-id ObjectId
 */
public final class ObjectId implements Comparable<ObjectId>, Serializable {
}

```
类里面有几个属性是存储id格式的
```

    private final int timestamp;
    private final int machineIdentifier;
    private final short processIdentifier;
    private final int counter;

```
看一下toString方法
```
    @Override
    public String toString() {
        return toHexString();
    }

    public String toHexString() {
      char[] chars = new char[24];
      int i = 0;
      for (byte b : toByteArray()) {
        chars[i++] = HEX_CHARS[b >> 4 & 0xF];
        chars[i++] = HEX_CHARS[b & 0xF];
      }
      return new String(chars);
    }
    
    public byte[] toByteArray() {
        ByteBuffer buffer = ByteBuffer.allocate(12);
        putToByteBuffer(buffer);
        return buffer.array();  // using .allocate ensures there is a backing array that can be returned
    }
    
    public void putToByteBuffer(final ByteBuffer buffer) {
        notNull("buffer", buffer);
        isTrueArgument("buffer.remaining() >=12", buffer.remaining() >= 12);

        buffer.put(int3(timestamp));
        buffer.put(int2(timestamp));
        buffer.put(int1(timestamp));
        buffer.put(int0(timestamp));
        buffer.put(int2(machineIdentifier));
        buffer.put(int1(machineIdentifier));
        buffer.put(int0(machineIdentifier));
        buffer.put(short1(processIdentifier));
        buffer.put(short0(processIdentifier));
        buffer.put(int2(counter));
        buffer.put(int1(counter));
        buffer.put(int0(counter));
   }

一层层追进去结果很明显了
前四位是时间戳 三位是机器id 两位是pid(同一个服务不同进程) 最后三位是流水号
(是转换到16进制之后的数据)
```

## 结论
mongo 主键 ObjectId 是客户端自己生成的类似于雪花id
大体上看上去是有序的但是如果有实际业务用到不能用id作为排序字段并发情况下id会很乱

相关参考:
https://blog.csdn.net/xiamizy/article/details/41521025
解释的更细致,详细的内容根据版本不同会有改变还是需要自己去看代码


