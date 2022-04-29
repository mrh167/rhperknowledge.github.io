### **一、redis基础类**

```java
package com.qlchat.component.redis.template;

import javax.annotation.PostConstruct;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

 import redis.clients.jedis.ShardedJedis;
 import redis.clients.jedis.ShardedJedisPool;

 import com.qlchat.component.redis.RedisConnectionFactory;
 import com.qlchat.component.redis.utils.JsonUtil;

 public abstract class BasicRedisTemplate {
    private static Logger logger = LoggerFactory.getLogger(BasicRedisTemplate.class); 
protected RedisConnectionFactory connectionFactory;
 
 public void setConnectionFactory(RedisConnectionFactory connectionFactory) {
     this.connectionFactory = connectionFactory;
 }
 
 protected ShardedJedisPool pool;
 
 @PostConstruct
 protected void initPool() {
     this.pool = connectionFactory.getConnectionPool();
 }
 
 @SuppressWarnings("deprecation")
 protected void closeRedis(ShardedJedis jedis) {
     if (jedis != null) {
         try {
             pool.returnResource(jedis);
         } catch (Exception e) {
             logger.error("Error happen when return jedis to pool, try to close it directly.", e);
             if (jedis != null) {
                 try {
                     jedis.disconnect();
                 } catch (Exception e1) {
                 }
             }
         }
     }
 }
 
 /**
  \* 删除key, 如果key存在返回true, 否则返回false。
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public boolean del(String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.del(key) == 1 ? true : false;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* true if the key exists, otherwise false
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean exists(String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.exists(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* set key expired time
  *
  \* @param key
  \* @param seconds
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean expire(String key, int seconds) {
     ShardedJedis jedis = null;
     if (seconds == 0) {
         return true;
     }
     try {
         jedis = pool.getResource();
         return jedis.expire(key, seconds) == 1 ? true : false;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  *
  \* 把object转换为json byte array
  *
  \* @param o
  \* @return
  */
 protected byte[] toJsonByteArray(Object o) {
     String json = JsonUtil.toJsonString(o) != null ? JsonUtil.toJsonString(o) : "";
     return json.getBytes();
 }
 
 /**
  *
  \* 把json byte array转换为T类型object
  *
  \* @param b
  \* @param clazz
  \* @return
  */
 protected <T> T fromJsonByteArray(byte[] b, Class<T> clazz) {
     if (b == null || b.length == 0) {
         return null;
     }
     return JsonUtil.parseJson(new String(b), clazz);
 }
 
 public Long ttl(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.ttl(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
  }
```

 ### **二、String类型redis工具类**

```java


 package com.qlchat.component.redis.template;

 import redis.clients.jedis.ShardedJedis;

 import com.qlchat.component.redis.RedisException;

 public class ValueRedisTemplate extends BasicRedisTemplate {

  /**
      \* set key-value
      *
      \* @param key
      \* @param value String
      \* @throws RedisException
      \* @since qlchat 1.0
      */
     public void set(String key, String value) {
         set(key, value, 0);
     } 
/**
  \* set key-value
  *
  \* @param key
  \* @param value String
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public void set(String key, String value, int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         jedis.set(key, value);
         if (ttl > 0) {
             jedis.expire(key.getBytes(), ttl);
         }
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* set key-value For Object(NOT String)
  *
  \* @param key
  \* @param value Object
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public void set(String key, Object value) {
     set(key, value, 0);
 }
 
 /**
  \* set key-value For Object(NOT String)
  *
  \* @param key
  \* @param value Object
  \* @param ttl int
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public void set(String key, Object value, int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         // jedis.set(key.getBytes(), HessianSerializer.serialize(value));
         jedis.set(key.getBytes(), toJsonByteArray(value));
         if (ttl > 0) {
             jedis.expire(key.getBytes(), ttl);
         }
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* set key-value with expired time(s)
  *
  \* @param key
  \* @param seconds
  \* @param value
  \* @return
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public String setex(String key, int seconds, String value) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.setex(key, seconds, value);
 
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* set key-value For Object(NOT String) with expired time(s)
  *
  \* @param key
  \* @param seconds
  \* @param value
  \* @return
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public String setex(String key, int seconds, Object value) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         // return jedis.setex(key.getBytes(), seconds, HessianSerializer.serialize(value));
         return jedis.setex(key.getBytes(), seconds, toJsonByteArray(value));
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 如果key还不存在则进行设置，返回true，否则返回false.
  *
  \* @param key
  \* @param value
  \* @return
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public boolean setnx(String key, String value, int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Long reply = jedis.setnx(key, value);
         if(reply == null){
             reply = 0L;
         }
         if (ttl > 0 && reply.longValue() == 1) {
             jedis.expire(key, ttl);
         }
         return reply.longValue() == 1;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 如果key还不存在则进行设置，返回true，否则返回false.
  *
  \* @param key
  \* @param value
  \* @return
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public boolean setnx(String key, String value) {
     return setnx(key, value, 0);
 }
 
 /**
  \* 如果key还不存在则进行设置 For Object，返回true，否则返回false.
  *
  \* @param key
  \* @param value
  \* @return
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public boolean setnx(String key, Object value) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         // return jedis.setnx(key.getBytes(), HessianSerializer.serialize(value)) == 1 ? true : false;
         return jedis.setnx(key.getBytes(), toJsonByteArray(value)) == 1 ? true : false;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 如果key还不存在则进行设置 For Object，返回true，否则返回false.
  *
  \* @param key
  \* @param value
  \* @return
  \* @throws RedisException
  \* @since qlchat 1.0
  */
 public boolean setnx(String key, Object value, int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         // Long reply = jedis.setnx(key.getBytes(), HessianSerializer.serialize(value));
         Long reply = jedis.setnx(key.getBytes(), toJsonByteArray(value));
         if (ttl > 0) {
             jedis.expire(key.getBytes(), ttl);
         }
         return reply == 1;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 如果key不存在, 返回null.
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public String get(String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.get(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* For Object, 如果key不存在, 返回null.
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public <T> T get(String key, Class<T> clazz) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         // return (T) HessianSerializer.deserialize(jedis.get(key.getBytes()));
         return fromJsonByteArray(jedis.get(key.getBytes()), clazz);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 自增 +1
  *
  \* @param key
  \* @return 返回自增后结果
  \* @since qlchat 1.0
  */
 public Long incr(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.incr(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 自增 +1
  *
  \* @param key key
  \* @param integer 起始值
  \* @return 返回自增后结果
  \* @since qlchat 1.0
  */
 public Long incrBy(final String key, long integer) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.incrBy(key, integer);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* 自减 -1
  *
  \* @param key
  \* @return 返回自减后结果
  \* @since qlchat 1.0
  */
 public Long decr(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.decr(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
  }
```


 
### **三、Hash类型redis工具类**
```java


 package com.qlchat.component.redis.template;

 import java.util.ArrayList;
 import java.util.HashMap;
 import java.util.HashSet;
 import java.util.List;
 import java.util.Map;
 import java.util.Map.Entry;
 import java.util.Set;

 import redis.clients.jedis.ShardedJedis;

 public class HashRedisTemplate extends BasicRedisTemplate {

   /**
      \* Set the string value of a hash field
      *
      \* @param key
      \* @param field
      \* @param value
      \* @return
      \* @since qlchat 1.0
      */
     public Boolean hset(final String key, final String field, final String value) {
         return hset(key, field, value, 0);
     } 
/**
  \* Set the string value of a hash field
  *
  \* @param key
  \* @param field
  \* @param value
  \* @param ttl
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean hset(final String key, final String field, final String value, final int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Long reply = jedis.hset(key, field, value);
         if (ttl > 0) {
             jedis.expire(key, ttl);
         }
         return reply == 1;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Set the Object value of a hash field
  *
  \* @param key
  \* @param field
  \* @param value
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean hset(final String key, final String field, final Object value) {
     return hset(key, field, value, 0);
 }
 
 /**
  \* Set the Object value of a hash field
  *
  \* @param key
  \* @param field
  \* @param value
  \* @param ttl
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean hset(final String key, final String field, final Object value, final int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Long reply = jedis.hset(key.getBytes(), field.getBytes(), toJsonByteArray(value));
         if (ttl > 0) {
             jedis.expire(key, ttl);
         }
         return reply == 1;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get the value of a hash field
  *
  \* @param key
  \* @param field
  \* @return
  \* @since qlchat 1.0
  */
 public String hget(final String key, final String field) {
     return hget(key, field, 0);
 }
 
 /**
  \* Get the value of a hash field
  *
  \* @param key
  \* @param field
  \* @return
  \* @since qlchat 1.0
  */
 public String hget(final String key, final String field, int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         String res = jedis.hget(key, field);
         if (ttl > 0) {
             jedis.expire(key, ttl);
         }
         return res;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get the value of a hash field
  *
  \* @param key
  \* @param field
  \* @return
  \* @since qlchat 1.0
  */
 public <T> T hget(final String key, final String field, final Class<T> clazz) {
     return hget(key, field, clazz, 0);
 }
 
 /**
  \* Get the value of a hash field
  *
  \* @param key
  \* @param field
  \* @return
  \* @since qlchat 1.0
  */
 public <T> T hget(final String key, final String field, final Class<T> clazz, final int ttl) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         T res = fromJsonByteArray(jedis.hget(key.getBytes(), field.getBytes()), clazz);
         if (ttl > 0) {
             this.expire(key, ttl);
         }
         return res;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Delete one or more hash fields
  *
  \* @param key
  \* @param fields
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean hdel(final String key, final String... fields) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hdel(key, fields) == 1 ? true : false;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Check if a hash field exists
  *
  \* @param key
  \* @param field
  \* @return
  \* @since qlchat 1.0
  */
 public Boolean hexists(final String key, final String field) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hexists(key, field);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get all the fields and values in a hash
  \* 当Hash较大时候，慎用！
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public Map<String, String> hgetAll(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hgetAll(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get all the fields and values in a hash
  \* 当Hash较大时候，慎用！
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public Map<String, Object> hgetAllObject(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Map<byte[], byte[]> byteMap = jedis.hgetAll(key.getBytes());
         if (byteMap != null && byteMap.size() > 0) {
             Map<String, Object> map = new HashMap<String, Object>();
             for (Entry<byte[], byte[]> e : byteMap.entrySet()) {
                 map.put(new String(e.getKey()), fromJsonByteArray(e.getValue(), Object.class));
             }
             return map;
         }
         return null;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get all the fields and values in a hash
  \* 当Hash较大时候，慎用！
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public <T> Map<String, T> hgetAllObject(final String key, Class<T> clazz) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Map<byte[], byte[]> byteMap = jedis.hgetAll(key.getBytes());
         if (byteMap != null && byteMap.size() > 0) {
             Map<String, T> map = new HashMap<String, T>();
             for (Entry<byte[], byte[]> e : byteMap.entrySet()) {
                 map.put(new String(e.getKey()), fromJsonByteArray(e.getValue(), clazz));
             }
             return map;
         }
         return null;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get the values of all the given hash fields.
  *
  \* @param key
  \* @param fields
  \* @return
  \* @since qlchat 1.0
  */
 public List<String> hmget(final String key, final String... fields) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hmget(key, fields);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  *
  \* Get the value of a mulit fields
  *
  \* @param key
  \* @param ttl
  \* @param fields
  \* @return
  \* @since Wifi 1.0
  */
 public Map<String, Object> hmgetObject(final String key, final int ttl, final String... fields) {
     ShardedJedis jedis = null;
     try {
         if (null == fields) {
             return null;
         }
         jedis = pool.getResource();
         List<byte[]> byteList = new ArrayList<byte[]>();
         for (String field : fields) {
             byteList.add(field.getBytes());
         }
         List<byte[]> resBytes = jedis.hmget(key.getBytes(), byteList.toArray(new byte[byteList.size()][]));
         Map<String, Object> resMap = null;
         if (null != resBytes) {
             resMap = new HashMap<String, Object>();
             for (int i = 0; i < resBytes.size(); i++) {
                 resMap.put(fields[i], fromJsonByteArray(resBytes.get(i), Object.class));
             }
         }
         if (ttl > 0) {
             this.expire(key, ttl);
         }
         return resMap;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  *
  \* Get the value of a mulit fields
  *
  \* @param key
  \* @param ttl
  \* @param fields
  \* @return
  \* @since Wifi 1.0
  */
 public Map<String, Object> hmgetObject(final String key, final String... fields) {
     return hmgetObject(key, 0, fields);
 }
 
 /**
  \* Set multiple hash fields to multiple values.
  *
  \* @param key
  \* @param hash
  \* @return
  \* @since qlchat 1.0
  */
 public String hmset(final String key, final Map<String, String> hash) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hmset(key, hash);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Set multiple hash fields to multiple values.
  *
  \* @param key
  \* @param hash
  \* @return
  \* @since qlchat 1.0
  */
 public String hmsetObject(final String key, final Map<String, Object> hash) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Map<byte[], byte[]> byteMap = new HashMap<byte[], byte[]>(hash.size());
         for (Entry<String, Object> e : hash.entrySet()) {
             byteMap.put(e.getKey().getBytes(), toJsonByteArray(e.getValue()));
         }
         return jedis.hmset(key.getBytes(), byteMap);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Increment the integer value of a hash field by the given number.
  *
  \* @param key
  \* @param field
  \* @param value
  \* @return
  \* @since qlchat 1.0
  */
 public Long hincrBy(final String key, final String field, final long value) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hincrBy(key, field, value);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get all the fields in a hash.
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public Set<String> hkeys(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hkeys(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get all the fields in a hash.
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public <T> Set<T> hkeys(final String key, final Class<T> clazz) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         Set<byte[]> set = jedis.hkeys(key.getBytes());
         Set<T> objectSet = new HashSet<T>();
         if (set != null && set.size() != 0) {
             for (byte[] b : set) {
                 objectSet.add(fromJsonByteArray(b, clazz));
             }
         }
         return objectSet;
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Get the number of fields in a hash.
  *
  \* @param key
  \* @return
  \* @since qlchat 1.0
  */
 public Long hlen(final String key) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hlen(key);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 public Long hsetnx(final String key, final String field, final String value) {
     ShardedJedis jedis = null;
     try {
         jedis = pool.getResource();
         return jedis.hsetnx(key, field, value);
     } finally {
         this.closeRedis(jedis);
     }
 
 }
  }
```

 
### **四、List基础类型工具类**
```java


 package com.qlchat.component.redis.template;

 import java.util.ArrayList;
 import java.util.List;

 import redis.clients.jedis.ShardedJedis;

 public class ListRedisTemplate extends BasicRedisTemplate {
/**
  \* Prepend one or multiple values to a list
  *
  \* @param key
  \* @param values
  \* @since qlchat 1.0
  */
 public void lpush(String key, String... values) {
     ShardedJedis jedis = null;
     if (values == null) {
         return;
     }
     try {
         jedis = pool.getResource();
         jedis.lpush(key, values);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* For Object,Prepend one or multiple values to a list
  *
  \* @param key
  \* @param values
  \* @since qlchat 1.0
  */
 public void lpush(String key, Object... values) {
     ShardedJedis jedis = null;
     if (values == null) {
         return;
     }
     try {
         jedis = pool.getResource();
         byte[][] strings = new byte[values.length][];
         for (int j = 0; j < values.length; j++) {
             // strings[j] = HessianSerializer.serialize(values[j]);
             strings[j] = toJsonByteArray(values[j]);
         }
         jedis.lpush(key.getBytes(), strings);
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* For single Object,Prepend one or multiple values to a list
  *
  \* @param key
  \* @param values
  \* @since qlchat 1.0
  */
 public void lpushForObject(String key, Object value) {
     ShardedJedis jedis = null;
     if (value == null) {
         return;
     }
     try {
         jedis = pool.getResource();
         // jedis.lpush(key.getBytes(), HessianSerializer.serialize(value));
         jedis.lpush(key.getBytes(), toJsonByteArray(value));
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* For single Object,Prepend one or multiple values to a list
  *
  \* @param key
  \* @param values
  \* @since qlchat 1.0
  */
 public void rpushForObject(String key, Object value) {
     ShardedJedis jedis = null;
     if (value == null) {
         return;
     }
     try {
         jedis = pool.getResource();
         // jedis.rpush(key.getBytes(), HessianSerializer.serialize(value));
         jedis.rpush(key.getBytes(), toJsonByteArray(value));
     } finally {
         this.closeRedis(jedis);
     }
 }
 
 /**
  \* Append one or multiple values to a list
  *
  \* @param key
  \* @param values
  \* @since qlchat 1.0
  */
 public void rpush(String key, String... values) {
     ShardedJedis jedis = null;
     if (values == null) {
        return;
    }
    try {
        jedis = pool.getResource();
        jedis.rpush(key, values);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Append one or multiple values to a list
 *
 \* @param key
 \* @param values
 \* @since qlchat 1.0
 */
public void rpush(String key, Object... values) {
    ShardedJedis jedis = null;
    if (values == null) {
        return;
    }
    try {
        jedis = pool.getResource();
        byte[][] strings = new byte[values.length][];
        for (int j = 0; j < values.length; j++) {
            // strings[j] = HessianSerializer.serialize(values[j]);
            strings[j] = toJsonByteArray(values[j]);
        }
        jedis.rpush(key.getBytes(), strings);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Remove and get the last element in a list
 *
 \* @param key
 \* @return
 \* @since qlchat 1.0
 */
public String rpop(String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.rpop(key);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Remove and get the last element in a list
 *
 \* @param key
 \* @param clazz
 \* @return
 \* @since qlchat 1.0
 */
public <T> T rpop(String key, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        // return (T) HessianSerializer.deserialize(jedis.rpop(key.getBytes()));
        return fromJsonByteArray(jedis.rpop(key.getBytes()), clazz);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Remove and get the first element in a list
 *
 \* @param key
 \* @return
 \* @since qlchat 1.0
 */
public String lpop(String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.lpop(key);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Remove and get the first element in a list
 *
 \* @param key
 \* @param clazz
 \* @return
 \* @since qlchat 1.0
 */
public <T> T lpop(String key, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        // return (T) HessianSerializer.deserialize(jedis.lpop(key.getBytes()));
        return fromJsonByteArray(jedis.lpop(key.getBytes()), clazz);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Get the length of a list
 *
 \* @param key
 \* @return
 \* @since qlchat 1.0
 */
public Long llen(String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.llen(key);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* 删除List中的等于value的元素
 *
 \* count = 1 :删除第一个； count = 0 :删除所有； count = -1:删除最后一个；
 *
 \* @param key
 \* @param count
 \* @param value
 \* @return
 \* @since qlchat 1.0
 */
public Long lrem(String key, long count, String value) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.lrem(key, count, value);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* FOR Object, 删除List中的等于value的元素
 *
 \* count = 1 :删除第一个； count = 0 :删除所有； count = -1:删除最后一个；
 *
 \* @param key
 \* @param count
 \* @param value
 \* @return
 \* @since qlchat 1.0
 */
public Long lrem(String key, long count, Object value) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        // return jedis.lrem(key.getBytes(), count, HessianSerializer.serialize(value));
        return jedis.lrem(key.getBytes(), count, toJsonByteArray(value));
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Get a range of elements from a list.
 \* <P>
 \* For example LRANGE foobar 0 2 will return the first three elements of the list.
 \* </p>
 \* <P>
 \* For example LRANGE foobar -1 -2 will return the last two elements of the list.
 \* </p>
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return
 \* @since qlchat 1.0
 */
public List<String> lrange(String key, long start, long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.lrange(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Get a range of elements from a list.
 \* <P>
 \* For example LRANGE foobar 0 2 will return the first three elements of the list.
 \* </p>
 \* <P>
 \* For example LRANGE foobar -1 -2 will return the last two elements of the list.
 \* </p>
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return
 \* @since qlchat 1.0
 */
public <T> List<T> lrange(String key, long start, long end, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        List<byte[]> list = jedis.lrange(key.getBytes(), start, end);
        if (list != null && list.size() > 0) {
            List<T> results = new ArrayList<>();
            for (byte[] bytes : list) {
                results.add(fromJsonByteArray(bytes, clazz));
            }
            return results;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

public String ltrim(String key, long start, long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.ltrim(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}
}
```

### **五、set基础类型工具类**
```java


package com.qlchat.component.redis.template;

import java.util.List;
import java.util.Set;
import java.util.TreeSet;

import redis.clients.jedis.ShardedJedis;

/**
 *
 \* <b><code>SetRedisTemplate</code></b>
 \* <p>
 \* Set 数据结构的常用方法封装，支持Object 类型
 \* </p>
 \* <b>Creation Time:</b> 2016年10月12日 下午5:51:46
 *
 */
public class SetRedisTemplate extends BasicRedisTemplate {
/**
 \* Add one or more members to a set
 *
 \* @param key
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean sadd(final String key, final String... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.sadd(key, members) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Add one or more members to a set
 *
 \* @param key
 \* @param ttl
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean sadd(final String key,int ttl, final String... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Boolean ret = jedis.sadd(key, members) == 1 ? true : false;
        if (ret && ttl > 0) {
            jedis.expire(key, ttl);
        }
        return ret;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Add one or more members to a set
 *
 \* @param key
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean sadd(final String key, final Object... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        byte[][] strings = new byte[members.length][];
        for (int j = 0; j < members.length; j++) {
            strings[j] = toJsonByteArray(members[j]);
        }
        return jedis.sadd(key.getBytes(), strings) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Remove one or more members from a set
 *
 \* @param key
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean srem(final String key, final String... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.srem(key, members) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Remove one or more members from a set
 *
 \* @param key
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean srem(final String key, final Object... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        byte[][] strings = new byte[members.length][];
        for (int j = 0; j < members.length; j++) {
            strings[j] = toJsonByteArray(members[j]);
        }
        return jedis.srem(key.getBytes(), strings) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Get all the members in a set.
 *
 \* @param key
 \* @return
 \* @since qlchat 1.0
 */
public Set<String> smembers(final String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.smembers(key);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Get all the members in a set.
 *
 \* @param key
 \* @param clazz
 \* @return
 \* @since qlchat 1.0
 */
public <T> Set<T> smembers(final String key, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<byte[]> tempSet = jedis.smembers(key.getBytes());
        if (tempSet != null && tempSet.size() > 0) {
            TreeSet<T> result = new TreeSet<T>();
            for (byte[] value : tempSet) {
                result.add(fromJsonByteArray(value, clazz));
            }
            return result;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Get all the members is number in a set.
 *
 \* @param key
 \* @return
 \* @since qlchat 1.0
 */
public Long scard(final String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.scard(key);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* true if the meber exists in a set,else false
 *
 \* @param key
 \* @param member
 \* @return
 \* @since qlchat 1.0
 */
public Boolean sismember(final String key, final String member) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.sismember(key, member);
    } finally {
        this.closeRedis(jedis);
    }
}

public String spop(final String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.spop(key);
    } finally {
        this.closeRedis(jedis);
    }
}

public List<String> srandmember(final String key,final int count) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.srandmember(key, count);
    } finally {
        this.closeRedis(jedis);
    }
}
}
```



### **六、Zset工具类**
```java


package com.qlchat.component.redis.template;

import com.alibaba.fastjson.JSON;
import redis.clients.jedis.ShardedJedis;
import redis.clients.jedis.Tuple;

import java.util.*;

/**
 *
 \* <b><code>ZSetRedisTemplate</code></b>
 \* <p>
 \* Sorted Sets 数据结构的常用方法封装，支持Object 类型
 \* </p>
 \* <b>Creation Time:</b> 2016年10月12日 下午5:54:09
 \* @author abin.yao
 \* @since qlchat 1.0
 */
public class ZSetRedisTemplate extends BasicRedisTemplate {
/**
 \* 加入Sorted set, 如果member在Set里已存在, 只更新score并返回false, 否则返回true.
 *
 \* @param key
 \* @param member
 \* @param score
 \* @return
 \* @since qlchat 1.0
 */
public Boolean zadd(String key, double score, String member) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zadd(key, score, member) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* 加入Sorted set, 如果member在Set里已存在, 只更新score并返回false, 否则返回true.
 *
 \* @param key
 \* @param member
 \* @param score
 \* @param ttl 过期时间，秒
 \* @return
 \* @since qlchat 1.0
 */
public Boolean zadd(final String key,final double score,final int ttl,final String member) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Boolean ret = jedis.zadd(key, score, member) == 1 ? true : false;
        if (ret && ttl > 0) {
            jedis.expire(key, ttl);
        }
        return ret;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, 加入Sorted set, 如果member在Set里已存在, 只更新score并返回false, 否则返回true.
 *
 \* @param key
 \* @param member
 \* @param score
 \* @return
 \* @since qlchat 1.0
 */
public Boolean zadd(String key, double score, Object member) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        // return jedis.zadd(key.getBytes(), score, HessianSerializer.serialize(member)) == 1 ? true : false;
        return jedis.zadd(key.getBytes(), score, toJsonByteArray(member)) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return a range of members in a sorted set, by index. Ordered from the lowest to the highest score.
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return Ordered from the lowest to the highest score.
 \* @since qlchat 1.0
 */
public Set<String> zrange(String key, long start, long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrange(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Return a range of members in a sorted set, by index.Ordered from the lowest to the highest score.
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return Ordered from the lowest to the highest score.
 \* @since qlchat 1.0
 */
public <T> List<T> zrange(String key, long start, long end, Class<T> clazz) {
    List<T> result = new ArrayList<>();
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<byte[]> tempSet = jedis.zrange(key.getBytes(), start, end);
        if (tempSet != null && tempSet.size() > 0) {
            for (byte[] value : tempSet) {
                // result.add((T) HessianSerializer.deserialize(value));
                result.add(fromJsonByteArray(value, clazz));
            }
            return result;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return a range of members in a sorted set, by index. Ordered from the highest to the lowest score.
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return Ordered from the highest to the lowest score.
 \* @since qlchat 1.0
 */
public Set<String> zrevrange(String key, long start, long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrevrange(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Return a range of members in a sorted set, by index. Ordered from the highest to the lowest score.
 *
 \* @param key
 \* @param start
 \* @param end
 \* @param clazz
 \* @return Ordered from the highest to the lowest score.
 \* @since qlchat 1.0
 */
public <T> List<T> zrevrange(String key, long start, long end, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<byte[]> tempSet = jedis.zrevrange(key.getBytes(), start, end);
        if (tempSet != null && tempSet.size() > 0) {
            List<T> result = new ArrayList<>();
            for (byte[] value : tempSet) {
                // result.add((T) HessianSerializer.deserialize(value));
                result.add(fromJsonByteArray(value, clazz));
            }
            return result;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return the all the elements in the sorted set at key with a score between
 \* min and max (including elements with score equal to min or max).
 *
 \* @param key
 \* @param min
 \* @param max
 \* @return Ordered from the lowest to the highest score.
 \* @since qlchat 1.0
 */
public Set<String> zrangeByScore(final String key, final double min, final double max) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrangeByScore(key, min, max);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Return the all the elements in the sorted set at key with a score between
 \* min and max (including elements with score equal to min or max).
 *
 \* @param key
 \* @param min
 \* @param max
 \* @return Ordered from the lowest to the highest score.
 \* @since qlchat 1.0
 */
public <T> Set<T> zrangeHashSetByScore(final String key, final double min, final double max, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<byte[]> tempSet = jedis.zrangeByScore(key.getBytes(), min, max);
        if (tempSet != null && tempSet.size() > 0) {
            HashSet<T> result = new HashSet<T>();
            for (byte[] value : tempSet) {
                // result.add((T) HessianSerializer.deserialize(value));
                result.add(fromJsonByteArray(value, clazz));
            }
            return result;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return the all the elements in the sorted set at key with a score between
 \* min and max (including elements with score equal to min or max).
 \* @param key
 \* @param min
 \* @param max
 \* @return Ordered from the highest to the lowest score.
 \* @since qlchat 1.0
 */
public Set<String> zrevrangeByScore(final String key, final double min, final double max) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrevrangeByScore(key, max, min);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Return the all the elements in the sorted set at key with a score between
 \* min and max (including elements with score equal to min or max).
 *
 \* @param key
 \* @param min
 \* @param max
 \* @return Ordered from the lowest to the highest score.
 \* @since qlchat 1.0
 */
public <T> List<T> zrangeByScore(final String key, final double min, final double max, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<byte[]> tempSet = jedis.zrangeByScore(key.getBytes(), min, max);
        if (tempSet != null && tempSet.size() > 0) {
            List<T> result = new ArrayList<>();
            for (byte[] value : tempSet) {
                result.add(fromJsonByteArray(value, clazz));
            }
            return result;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

public <T> List<T> zrangeByScore(final String key, final double min, final double max, final int offset, final int count, Class<T> clazz) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<byte[]> tempSet = jedis.zrangeByScore(key.getBytes(), min, max, offset, count);
        if (tempSet != null && tempSet.size() > 0) {
            List<T> result = new ArrayList<>();
            for (byte[] value : tempSet) {
                result.add(fromJsonByteArray(value, clazz));
            }
            return result;
        }
        return null;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return a range of members with scores in a sorted set, by index. Ordered from the lowest to the highest score.
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return
 \* @since qlchat 1.0
 */
public Set<Tuple> zrangeWithScores(final String key, final long start, final long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrangeWithScores(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return a range of members with scores in a sorted set, by index. Ordered from the highest  to the lowest score.
 *
 \* @param key
 \* @param start
 \* @param end
 \* @return
 \* @since qlchat 1.0
 */
public Set<Tuple> zrevrangeWithScores(final String key, final long start, final long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrevrangeWithScores(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return the all the elements in the sorted set at key with a score between
 \* min and max (including elements with score equal to min or max). Ordered from the lowest to the highest score.
 *
 \* @param key
 \* @param min
 \* @param max
 \* @return Ordered from the lowest to the highest score.
 \* @since qlchat 1.0
 */
public Set<Tuple> zrangeByScoreWithScores(final String key, final double min, final double max) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrangeByScoreWithScores(key, min, max);
    } finally {
        this.closeRedis(jedis);
    }
}

public Set<Tuple> zrangeByScoreWithScores(final String key, final double min, final double max, final int offset, final int count) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrangeByScoreWithScores(key, min, max, offset, count);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Return the all the elements in the sorted set at key with a score between
 \* min and max (including elements with score equal to min or max). Ordered from the highest to the lowest score.
 *
 \* @param key
 \* @param min
 \* @param max
 \* @return Ordered from the highest to the lowest score.
 \* @since qlchat 1.0
 */
public Set<Tuple> zrevrangeByScoreWithScores(final String key, final double min, final double max) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrevrangeByScoreWithScores(key, max, min);
    } finally {
        this.closeRedis(jedis);
    }
}

public Set<Tuple> zrevrangeByScoreWithScores(final String key, final double min, final double max, final int offset, final int count) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrevrangeByScoreWithScores(key, max, min, offset, count);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Remove one or more members from a sorted set
 *
 \* @param key
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean zrem(final String key, final String... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrem(key, members) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For Object, Remove one or more members from a sorted set
 *
 \* @param key
 \* @param members
 \* @return
 \* @since qlchat 1.0
 */
public Boolean zrem(final String key, final Object... members) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        byte[][] strings = new byte[members.length][];
        for (int j = 0; j < members.length; j++) {
            // strings[j] = HessianSerializer.serialize(members[j]);
            strings[j] = toJsonByteArray(members[j]);
        }
        return jedis.zrem(key.getBytes(), strings) == 1 ? true : false;
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Get the score associated with the given member in a sorted set
 *
 \* @param key
 \* @param member
 \* @return
 \* @since qlchat 1.0
 */
public Double zscore(final String key, final String member) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zscore(key, member);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* For ObjecGet the score associated with the given member in a sorted set
 *
 \* @param key
 \* @param member
 \* @return
 \* @since qlchat 1.0
 */
public Double zscore(final String key, final Object member) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        // return jedis.zscore(key.getBytes(), HessianSerializer.serialize(member));
        return jedis.zscore(key.getBytes(), toJsonByteArray(member));
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Remove all elements in the sorted set at key with rank between start and
 \* end. Start and end are 0-based with rank 0 being the element with the
 \* lowest score. Both start and end can be negative numbers, where they
 \* indicate offsets starting at the element with the highest rank. For
 \* example: -1 is the element with the highest score, -2 the element with
 \* the second highest score and so forth.
 \* <p>
 \* <b>Time complexity:</b> O(log(N))+O(M) with N being the number of
 \* elements in the sorted set and M the number of elements removed by the
 \* operation
 \* @param key
 \* @param start
 \* @param end
 \* @return
 \* @since qlchat 1.0
 */
public Long zremrangeByRank(String key, long start, long end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zremrangeByRank(key, start, end);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 *
 \* Get the length of a sorted set
 *
 \* @param key
 \* @param min
 \* @param max
 \* @return
 \* @since qlchat 1.0
 */
public Long zcount(final String key, final double min, final double max) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zcount(key, min, max);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* Get the number of members in a sorted set
 \* @param key key
 \* @return Long
 */
public Long zcard(final String key) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zcard(key);
    } finally {
        this.closeRedis(jedis);
    }
}

public <T> List<T> zrevrangeByScore(String key, double max, double min, Class<T> clazz) {
    List<T> result = new ArrayList<T>();
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<String> sets = jedis.zrevrangeByScore(key, max, min);
        if (null == sets || sets.size()  == 0) {
            return result;
        }
        for (String s : sets) {
            result.add(JSON.parseObject(s, clazz));
        }
    } catch (Exception e) {
        e.printStackTrace();

    } finally {
        if(jedis != null) {jedis.close();}
    }
    return result;
}

public <T> List<T> zrevrangeByScore(String key, double max, double min, int offset, int count, Class<T> clazz) {
    List<T> result = new ArrayList<T>();
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<String> sets = jedis.zrevrangeByScore(key, max, min, offset, count);
        if (null == sets || sets.size()  == 0) {
            return result;
        }
        for (String s : sets) {
            result.add(JSON.parseObject(s, clazz));
        }
    } catch (Exception e) {
        e.printStackTrace();

    } finally {
        if(jedis != null) {jedis.close();}
    }
    return result;
}

public List<String> zrevrangeByScore(String key, double max, double min, int offset, int count) {
    List<String> result = new ArrayList<>();
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        Set<String> sets = jedis.zrevrangeByScore(key, max, min, offset, count);
        if (null == sets || sets.size()  == 0) {
            return result;
        }
        for (String s : sets) {
            result.add(s);
        }
    } catch (Exception e) {
        e.printStackTrace();

    } finally {
        if(jedis != null) {jedis.close();}
    }
    return result;
}

public Set<String> zrangeByScore(String key, double max, double min, int offset, int count) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zrangeByScore(key, min, max, offset, count);
    } finally {
        this.closeRedis(jedis);
    }
}

public Long zremrangeByScore(String key, double start, double end) {
    ShardedJedis jedis = null;
    try {
        jedis = pool.getResource();
        return jedis.zremrangeByScore(key, start, end);
    }finally {
        this.closeRedis(jedis);
    }
}

public Map<String,Object> zrevrankWithScore(String key,String element){
    ShardedJedis jedis = null;
    Map<String,Object> map = new HashMap<>();
    try{
        Long rankIndex = jedis.zrevrank(key,element);
        Double score = jedis.zscore(key,element);
        if(rankIndex != null && score >0){
            map.put("rank",rankIndex+1);
            map.put("score",score);
        }
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        this.closeRedis(jedis);
    }
    return map;
}
}
```




### **七、HyperLogLog模板**
```java


package com.qlchat.component.redis.template;

import redis.clients.jedis.ShardedJedis;
/**
 \* HyperLogLog模板
 \* Created by liujunjie on 16-12-15.
 */
public class HyperLogLogTemplate extends BasicRedisTemplate {
/**
 \* 添加
 \* @param key key
 \* @param elements 元素
 */
public Long pfadd(String key, String... elements) {
    ShardedJedis jedis = null;
    try{
        jedis = pool.getResource();
        return jedis.pfadd(key, elements);
    } finally {
        this.closeRedis(jedis);
    }
}

/**
 \* 统计
 \* @param key key
 \* @return long
 */
public long pfcount(String key) {
    ShardedJedis jedis = null;
    try{
        jedis = pool.getResource();
        return jedis.pfcount(key);
    } finally {
        this.closeRedis(jedis);
    }
}
}
```

