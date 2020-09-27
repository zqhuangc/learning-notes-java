* JsonNode
  *　BaseJsonNode
    * ContainerNode
      * ArrayNode
      * ObjectNode
    * ValueNode
      * ...



ObjectMapper#writeValueAsString        JSON.toJsonString

ArrayNode ---------------    JsonArray

ObjectNode -----------------JsonNode

ObjectMapper#writeValue

JsonGenerator#writeObject

JsonParser



objectMapper   

readTree    

readValue

ArrayNode#elements：map.entry

JsonNode#filedNames：key

jsonNode#get(filedName) ：value

 JsonNode#elements ：json array or value

objectMapper 配置

```java
ObjectMapper objectMapper = new ObjectMapper();
//去掉默认的时间戳格式     
objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
//设置为东八区
objectMapper.setTimeZone(TimeZone.getTimeZone("GMT+8"));
// 设置输入:禁止把POJO中值为null的字段映射到json字符串中
objectMapper.configure(SerializationFeature.WRITE_NULL_MAP_VALUES, false);
 //空值不序列化
objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
//反序列化时，属性不存在的兼容处理
objectMapper.getDeserializationConfig().withoutFeatures(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
//序列化时，日期的统一格式
objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
//序列化日期时以timestamps输出，默认true
objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
//序列化枚举是以toString()来输出，默认false，即默认以name()来输出
objectMapper.configure(SerializationFeature.WRITE_ENUMS_USING_TO_STRING,true);
//序列化枚举是以ordinal()来输出，默认false
objectMapper.configure(SerializationFeature.WRITE_ENUMS_USING_INDEX,false);
//类为空时，不要抛异常
objectMapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
//反序列化时,遇到未知属性时是否引起结果失败
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
 //单引号处理
objectMapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);
//解析器支持解析结束符
objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
```





```java
package com.melody.spring.upload;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.assertj.core.util.Streams;

import java.lang.reflect.Type;
import java.sql.SQLOutput;
import java.util.*;
import java.util.stream.Stream;

/**
 * TODO
 * @author zqhuangc
 */
public class ObjectMapperTest {
    public static final ObjectMapper objectMapper = new ObjectMapper();

    public static void main(String[] args) throws JsonProcessingException {
        testArrayNode();
    }

    public static void testObjectNode() throws JsonProcessingException {
        ObjectNode objectNode = objectMapper.createObjectNode();
        objectNode.put("mingtian", 1);
        objectNode.put("haoxiang", 12L);
        objectNode.put("qishi", "oop");
        String s = objectMapper.writeValueAsString(objectNode);
        System.out.println(s);
    }

    public static void testArrayNode() throws JsonProcessingException {
        ArrayNode arrayNode = objectMapper.createArrayNode();
        for (int i = 0; i < 3; i++) {
            ObjectNode objectNode = objectMapper.createObjectNode();
            objectNode.put("a"+i, 1);
            objectNode.put("b"+i, 2L);
            objectNode.put("c"+i, "3");
            ObjectNode objectNode1 = objectMapper.createObjectNode();
            objectNode1.set("testnode"+i, objectNode);
            ObjectNode objectNode2 = objectMapper.createObjectNode();
            objectNode2.set("test_node"+i, objectNode1);
            arrayNode.add(objectNode2);
        }
        String s = objectMapper.writeValueAsString(arrayNode);
        System.out.println("json:" + s);

        ArrayNode jsonNode = objectMapper.readValue(s, new TypeReference<ArrayNode>() {
            @Override
            public Type getType() {
                return super.getType();
            }
        });
        //JsonNode jsonNode = objectMapper.readTree(s);
        LinkedHashMap<String, Object> map = new LinkedHashMap<>();
        readDeepestValue(jsonNode, map);
        System.out.println("json parse:");
        Stream.of(map).forEach(System.out::println);
    }

    /**
     * 获取最底层的值
     * @param jsonNode
     * @param map
     */
    private static Map<String, Object> readDeepestValue(JsonNode jsonNode, Map<String, Object> map){

        Iterator<JsonNode> elements = jsonNode.elements();
        Iterator<String> fields = jsonNode.fieldNames();
        while(elements.hasNext()){
            JsonNode next = elements.next();
            //System.out.println(next.getNodeType());
            if(next.getNodeType().name().equals("OBJECT")){
                if(fields.hasNext()){
                    map.put(fields.next(), readDeepestValue(next, map));
                }else {
                    map=readDeepestValue(next, map);
                }

            }else {
                LinkedHashMap<String, Object> hashMap = new LinkedHashMap<>();
                while (fields.hasNext()) {
                    String field = fields.next();
                    hashMap.put(field, jsonNode.get(field));
                }
                return hashMap;
            }
        }
        return map;
    }

    private static void testReadTree(String json) throws JsonProcessingException {
        ArrayNode arrayNode =(ArrayNode) objectMapper.readTree(json);
        Iterator<JsonNode> elements = arrayNode.elements();
        while (elements.hasNext()){
            ObjectNode objectNode = (ObjectNode)elements.next();
            Iterator<String> eleIterator = objectNode.fieldNames();
            while (eleIterator.hasNext()){
                String key = eleIterator.next();
                ObjectNode ObjectNode = (ObjectNode)objectNode.get(key);
                Iterator<String> jsonIterator = ObjectNode.fieldNames();
                while(jsonIterator.hasNext()){
                    String fieldName = jsonIterator.next();
                    System.out.println(fieldName + ":" + ObjectNode.get(fieldName));
                }
            }
        }
    }

    public static void testMap() throws JsonProcessingException {

        HashMap<Object, Object> map = new HashMap<>();
        map.put("qian", "123");
        map.put("money", "234");
        map.put("dollar", "456");
        String s = objectMapper.writeValueAsString(map);
        System.out.println(s);
    }
}

```













ObjectMapper mapper = new ObjectMapper();
ClassLoader loader = getClass().getClassLoader();
List<Module> modules = SecurityJackson2Modules.getModules(loader);
mapper.registerModules(modules);
// ... use ObjectMapper as normally ...
SecurityContext context = new SecurityContextImpl();
// ...
String json = mapper.writeValueAsString(context);