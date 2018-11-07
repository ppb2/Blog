
1. 简介

根据传来的Map参数，key 为更新路径，value 为对应的值（任意基本类型，对象（json字符串））


2. 数据示例

```java
 public class UserInfo {
        private int age;
        private Integer role;
        private String name;
        private Boolean married;
        private UserInfo.Sex sex;
        private Address address;
        private String[] hobbys;
        private Address[] addresses;
        private List<String> hobbyList;
        private List<Address> addressList;
        private Map<String, String> keyMap;
        private Map<String, Address> addressMap;

        public enum Sex {
            男, 女
        }
      
    }

public class Address {
        private Integer floor;
        private String name;
        private Address address;
    }
```

3. 基本类型数据更新

- 基本对象
    - int  
        - key : "age", value : 18
    - Integer
        - key : "role", value : null
    - String
        - key : "name", value : "zhansan"
    - Enum 
        - key : "sex", value : "男"

4. 复杂类型数据更新

- 数组

    - 基本类型数组
        -   String[]
             - 创建数组
                - key： "hobbys", value : "[\\"打排球\\"]"
            - 修改数组
                - key: "hobbys[0]",  value : "登山"
            - 删除数组
                - key: "hobbys[0]",  value : null
            - 添加数组
                - key: "hobbys[]",  value : "吃饭"
            
    - 复杂类型数组
        -   Address[]
             - 创建数组
                - key： "address", value : "[{\\"floor\\":8,\\"name\\":\\"上海\\"}]"
            - 修改数组
                - key: "address[0]",  value : "[{\\"floor\\":8,\\"name\\":\\南京\\"}]"
            - 删除数组
                - key: "address[0]",  value : null
            - 添加数组
                - key: "address[]",  value : "[{\\"floor\\":8,\\"name\\":\\"北京\\"}]"
            - 修改某个数组对象的字段
                -  - key: "address[0].floor",  value : 10

- List

    - 基本类型List
        -   String[]
             - 创建数组
                - key： "hobbyList", value : "[\\"打排球\\"]"
            - 修改数组
                - key: "hobbyList[0]",  value : "登山"
            - 删除数组
                - key: "hobbyList[0]",  value : null
            - 添加数组
                - key: "hobbyList[]",  value : "吃饭"

     - 复杂类型List
        -   Address[]
             - 创建数组
                - key： "addressList", value : "[{\\"floor\\":8,\\"name\\":\\"上海\\"}]"
            - 修改数组
                - key: "addressList[0]",  value : "[{\\"floor\\":8,\\"name\\":\\南京\\"}]"
            - 删除数组
                - key: "addressList[0]",  value : null
            - 添加数组
                - key: "addressList[]",  value : "[{\\"floor\\":8,\\"name\\":\\"北京\\"}]"
            - 修改某个数组对象的字段
                - key: "addressList[0].floor",  value : 10

- Map

    - 基本类型Map
        - Map<String, String> 
            - 创建Map
                - key: "keyMap" , value : "{\\"test02\\":\\"w\\"}"
            - 修改Map
                - key: "keyMap["test02"]" . value :  "hello world"


    - 复杂类型Map
        - Map<String, Address> 
            - 创建Map
                - key: "addressMap" , value : "{\\"test01\\":{\\"floor\\":10}}"
            - 修改Map
                - key: "addressMap["test01"]" , value : "{\\"floor\\":10}"


### 代码实现

```java

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import org.springframework.util.StringUtils;

import java.lang.reflect.*;
import java.util.*;

public class MapBeanUtil<T> {
    private Map<Class<?>, Map<String, Field>> cacheFieldMap = new HashMap<>();
    private Map<Class<?>, Map<String, Method>> cacheGetterMethodMap = new HashMap<>();
    private Map<Class<?>, Map<String, Method>> cacheSetterMethodMap = new HashMap<>();
    private Map<String, Object> param;
    private Map<String, Class<?>> commonClassMap = new HashMap<>();

    public MapBeanUtil() {
        initCommonClassMap();
    }

    private void initCommonClassMap() {
        commonClassMap.put("String", String.class);
        commonClassMap.put("Enum", Enum.class);
        commonClassMap.put("int", int.class);
        commonClassMap.put("Integer", Integer.class);
        commonClassMap.put("boolean", boolean.class);
        commonClassMap.put("Boolean", Boolean.class);
        commonClassMap.put("Float", Float.class);
        commonClassMap.put("float", float.class);
        commonClassMap.put("Long", Long.class);
        commonClassMap.put("long", long.class);
        commonClassMap.put("Double", Double.class);
        commonClassMap.put("double", double.class);
        commonClassMap.put("Short", Short.class);
        commonClassMap.put("short", short.class);
        commonClassMap.put("sql.Date", java.sql.Date.class);
        commonClassMap.put("Date", Date.class);
        commonClassMap.put("char", char.class);
        commonClassMap.put("byte", byte.class);
        commonClassMap.put("Byte", Byte.class);
        commonClassMap.put("Byte[]", Byte[].class);
        commonClassMap.put("byte[]", byte[].class);
    }

    private static class EnumConverter {
        public static Object convert(Class c, Object value) {
            if (value != null) {
                String strVal = (String) value;
                return Enum.valueOf(c, strVal);
            } else {
                return null;
            }
        }
    }

    private Map<String, Field> getFiledMap(Class<?> cls) {
        if (!cacheFieldMap.containsKey(cls)) {
            Map<String, Field> fieldMap = new HashMap<String, Field>();
            Class tempCls = cls;
            while (tempCls.getSuperclass() != null && !tempCls.getName().toLowerCase().equals("java.lang.object")) {
                if (tempCls != null) {
                    Field[] fileds = tempCls.getDeclaredFields();
                    for (Field field : fileds) {
                        String fieldName = field.getName();
                        fieldMap.put(fieldName, field);
                    }

                }
                tempCls = tempCls.getSuperclass();
            }
            cacheFieldMap.put(cls, fieldMap);
        }
        return cacheFieldMap.get(cls);
    }

    private Map<String, Method> getGetterMethod(Class<?> cls) {
        if (!cacheGetterMethodMap.containsKey(cls)) {
            Map<String, Method> methosMap = new HashMap<String, Method>();
            if (cls != null) {
                Method[] methods = cls.getMethods();
                for (Method method : methods) {
                    String methodName = method.getName();
                    if (methodName.startsWith("get")) {
                        methodName = methodName.replace("get", "");
                        if (methodName.length() > 1) {
                            String key = methodName.substring(0, 1).toLowerCase() + methodName.substring(1);
                            methosMap.put(key, method);
                        }
                        if (methodName.length() == 1) {
                            String key = methodName.substring(0, 1).toLowerCase();
                            methosMap.put(key, method);
                        }
                    }
                }
                cacheGetterMethodMap.put(cls, methosMap);
            }
        }
        return cacheGetterMethodMap.get(cls);
    }

    private void convertType(Class<?> type, Object object, Method method, String key) {
        JSONObject parseObject = JSONObject.parseObject(JSON.toJSONString(this.param));
        try {
            if (type.isEnum()) {
                method.invoke(object, MapBeanUtil.EnumConverter.convert(type, parseObject.getString(key)));
            }
            if (type == String.class) {
                method.invoke(object, parseObject.getString(key));
            }
            if (type == int.class) {
                method.invoke(object, parseObject.getIntValue(key));
            }
            if (type == Integer.class) {
                method.invoke(object, parseObject.getInteger(key));
            }
            if (type == Boolean.class) {
                method.invoke(object, parseObject.getBoolean(key));
            }
            if (type == boolean.class) {
                method.invoke(object, parseObject.getBooleanValue(key));
            }
            if (type == Float.class) {
                method.invoke(object, parseObject.getFloat(key));
            }
            if (type == float.class) {
                method.invoke(object, parseObject.getFloatValue(key));
            }
            if (type == Long.class) {
                method.invoke(object, parseObject.getLong(key));
            }
            if (type == long.class) {
                method.invoke(object, parseObject.getLongValue(key));
            }
            if (type == Short.class) {
                method.invoke(object, parseObject.getShort(key));
            }
            if (type == short.class) {
                method.invoke(object, parseObject.getShortValue(key));
            }
            if (type == java.sql.Date.class) {
                method.invoke(object, parseObject.getSqlDate(key));
            }
            if (type == Date.class) {
                String date = parseObject.getString(key);
                if (date.matches("[\\d]{4}-[\\d]{2}")) {
                    date = date + "-01";
                    parseObject.put(key, date);
                }
                method.invoke(object, parseObject.getDate(key));
            }
            if (type == Double.class) {
                method.invoke(object, parseObject.getDouble(key));
            }
            if (type == double.class) {
                method.invoke(object, parseObject.getDoubleValue(key));
            }
            if (type == char.class) {
                method.invoke(object, parseObject.get(key));
            }
            if (type == byte.class || type == Byte.class) {
                Byte result = null;
                result = parseObject.getByte(key);
                method.invoke(object, result);
            }
            if (type == byte[].class || type == Byte[].class) {
                byte[] result = parseObject.getBytes(key);
                method.invoke(object, result);
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

    }

    private Map<String, Method> getSetterMethod(Class<?> cls) {
        if (!cacheSetterMethodMap.containsKey(cls)) {
            Map<String, Method> methosMap = new HashMap<String, Method>();
            if (cls != null) {
                Method[] methods = cls.getMethods();
                for (Method method : methods) {
                    String methodName = method.getName();
                    methosMap.put(methodName, method);
                    if (methodName.startsWith("set")) {
                        methodName = methodName.replace("set", "");
                        if (methodName.length() > 1) {
                            String key = methodName.substring(0, 1).toLowerCase() + methodName.substring(1);
                            methosMap.put(key, method);
                        }
                        if (methodName.length() == 1) {
                            String key = methodName.substring(0, 1).toLowerCase();
                            methosMap.put(key, method);
                        }
                    }
                    cacheSetterMethodMap.put(cls, methosMap);
                }
            }
        }
        return cacheSetterMethodMap.get(cls);
    }

    private static Class<?> getComponentType(Class<?> arrayClass) {
        return arrayClass.getComponentType();
    }

    private Class<?> getListClazz(Field list) {
        ParameterizedType listGenericType = (ParameterizedType) list.getGenericType();
        Type[] listActualTypeArguments = listGenericType.getActualTypeArguments();
        return (Class<?>) listActualTypeArguments[0];
    }

    private Class<?> getMapClazz(Field field) {
        ParameterizedType mapGenericType = (ParameterizedType) field.getGenericType();
        Type[] mapActualTypeArguments = mapGenericType.getActualTypeArguments();
        return (Class<?>) mapActualTypeArguments[1];
    }

    private void doAnalyzeNew(String key, T object) {
        try {
            Map<String, Field> fieldMap = getFiledMap(object.getClass());
            Map<String, Method> setterMethodMap = getSetterMethod(object.getClass());
            Map<String, Method> getterMethodMap = getGetterMethod(object.getClass());
            Object target = object;
            String[] keys = key.split("\\.");
            int j = 0;
            for (String keyMember : keys) {
                String keyName = keyMember;
                //do update
                if (j == keys.length - 1) {
                    //对象直接赋值
                    if (!keyMember.matches("\\S+\\[[a-z\"\\d]{0,}\\]")) {
                        Field field = getFieldFromMap(fieldMap, keyName);
                        if (field == null)
                            throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                        Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                        if (getterMethod == null)
                            throw new NoSuchMethodException("no such method");
                        // common type
                        if (field.getType().isEnum() || commonClassMap.containsValue(field.getType())) {
                            convertType(field.getType(), target, getSetterMethodFromMap(setterMethodMap, keyName), key);
                            return;
                        } else {
                            // not common type
                            if (field.getType() == List.class) {
                                //list
                                Method m = getSetterMethodFromMap(setterMethodMap, keyName);
                                if (m != null) {
                                    List<?> list = JSONObject.parseArray(param.get(key).toString(), getListClazz(field));
                                    m.invoke(target, list);
                                    return;
                                } else {
                                    throw new NoSuchMethodException("list: no such method" + " field:" + keyName);
                                }
                            } else if (field.getType() == Map.class) {
                                //map
                                Method m = getSetterMethodFromMap(setterMethodMap, keyName);
                                if (m != null) {
                                    JSONObject jsonObject = JSON.parseObject(param.get(key).toString());
                                    Map<String, Object> map2 = new HashMap<>();
                                    Class mapcls = getMapClazz(field);
                                    for (JSONObject.Entry<String, Object> entry : jsonObject.entrySet()) {
                                        if (!commonClassMap.containsValue(mapcls)) {
                                            JSONObject json = (JSONObject) entry.getValue();
                                            if (!StringUtils.isEmpty(entry.getValue().toString())) {
                                                map2.put(entry.getKey(), JSON.parseObject(JSON.toJSONString(entry.getValue()), mapcls));
                                            }
                                        } else {
                                            map2.put(entry.getKey(), entry.getValue());
                                        }

                                    }

                                    m.invoke(target, map2);
                                    return;
                                } else {
                                    throw new NoSuchMethodException("map: no such method" + " field:" + keyName);
                                }
                            } else if (field.getType().isArray()) {
                                //array
                                Method m = getSetterMethodFromMap(setterMethodMap, keyName);
                                if (m != null) {
                                    Class cls = field.getType();
                                    Class cls2 = getComponentType(cls);
                                    Object array = JSONObject.parseObject(param.get(key).toString(), cls);

                                    m.invoke(target, array);
                                    return;
                                } else {
                                    throw new NoSuchMethodException("array: no such method" + " field:" + keyName);
                                }
                            } else {
                                //object
                                Method m = getSetterMethodFromMap(setterMethodMap, keyName);
                                if (m != null) {
                                    m.invoke(target, JSONObject.parseObject(param.get(key).toString(), field.getClass()));
                                    return;
                                } else {
                                    throw new NoSuchMethodException("object: no such method" + " field:" + keyName);
                                }
                            }
                        }
                    } else {
                        //非对象直接赋值： 修改数组， 修改map
                        boolean isArray = keyMember.matches("\\S+\\[\\d+\\]");
                        boolean isMap = keyMember.matches("\\S+\\[\"\\S*\"\\]");
                        boolean addArray = keyMember.matches("\\S+\\[\\]");
                        if (!isArray && !isMap && !addArray) {
                            Field field = getFieldFromMap(fieldMap, keyName);
                            if (field == null)
                                throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                            Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                            if (getterMethod == null)
                                throw new NoSuchMethodException("no such method");
                            if (field.getType().isEnum() || commonClassMap.containsValue(field.getType())) {
                                convertType(field.getType(), target, getSetterMethodFromMap(setterMethodMap, keyName), key);
                                return;
                            }
                        }
                        //新增数组

                        if (addArray) {
                            keyName = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[\\]", 1);
                            Field field = getFieldFromMap(fieldMap, keyName);
                            if (field == null)
                                throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                            Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                            if (getterMethod == null)
                                throw new NoSuchMethodException("no such method");

                            if (field.getType() == List.class) {
                                List arrayList = (List) getterMethod.invoke(target);
                                if (arrayList != null) {
                                    Object o = null;
                                    if (param.get(key) != null) {
                                        o = JSONObject.parseObject(param.get(key).toString(), getListClazz(field));
                                    }
                                    List.class.getMethod("add", Object.class).invoke(arrayList, o);
                                    return;
                                } else {
                                    throw new NullPointerException("list is null");
                                }
                            }

                            if (field.getType().isArray()) {
                                Object targetArray = getterMethod.invoke(target);
                                Class<?> cls = getComponentType(targetArray.getClass());
                                if (targetArray != null) {
                                    Method m = getSetterMethodFromMap(setterMethodMap, keyName);
                                    int length = Array.getLength(targetArray);
                                    Object newArray = Array.newInstance(cls, length + 1);
                                    System.arraycopy(targetArray, 0, newArray, 0, length);
                                    if (commonClassMap.containsValue(cls)) {
                                        Array.set(newArray, length, param.get(key));
                                        m.invoke(target, newArray);
                                        return;
                                    } else {
                                        if (param.get(key) != null) {
                                            Array.set(newArray, length, JSONObject.parseObject(param.get(key).toString(), cls));
                                        }
                                        m.invoke(target, newArray);
                                        return;
                                    }
                                } else {
                                    throw new NullPointerException("array is null");
                                }
                            }

                        }

                        // 修改array
                        if (isArray) {
                            Integer arrayIndex = null;
                            keyName = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[(\\d+)\\]", 1);
                            arrayIndex = Integer.parseInt(RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[(\\d+)\\]", 2));
                            Field field = getFieldFromMap(fieldMap, keyName);
                            if (field == null)
                                throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                            Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                            if (getterMethod == null)
                                throw new NoSuchMethodException("no such method");

                            if (field.getType() == List.class) {
                                Object arrayList = getterMethod.invoke(target);
                                if (arrayList != null) {
                                    int length = (int) List.class.getDeclaredMethod("size").invoke(arrayList);
                                    if (arrayIndex < length) {
                                        if (param.get(key) != null) {
                                            Object o = JSONObject.parseObject(param.get(key).toString(), getListClazz(field));
                                            List.class.getMethod("remove", int.class).invoke(arrayList, arrayIndex);
                                            List.class.getMethod("add", int.class, Object.class).invoke(arrayList, arrayIndex, o);
                                        } else {
                                            List.class.getMethod("remove", int.class).invoke(arrayList, arrayIndex);
                                        }
                                        return;
                                    } else {
                                        throw new IndexOutOfBoundsException();
                                    }
                                } else {
                                    throw new NullPointerException("list is null");
                                }
                            }

                            if (field.getType().isArray()) {
                                Object array = getterMethod.invoke(target);
                                Class<?> cls = getComponentType(array.getClass());
                                if (array != null) {
                                    int length = Array.getLength(array);
                                    Method m = getSetterMethodFromMap(setterMethodMap, keyName);
                                    if (arrayIndex < length) {
                                        if (commonClassMap.containsValue(cls)) {
                                            if (param.get(key) != null) {
                                                Array.set(array, arrayIndex, param.get(key));
                                            } else {
                                                Object newArray = Array.newInstance(cls, length - 1);
                                                System.arraycopy(array, 0, newArray, 0, arrayIndex);
                                                System.arraycopy(array, arrayIndex + 1, newArray, 0, length - 1 - arrayIndex);
                                                m.invoke(target, newArray);
                                                return;
                                            }
                                        } else {
                                            if (param.get(key) != null) {
                                                Array.set(array, arrayIndex, JSONObject.parseObject(param.get(key).toString(), cls));
                                            } else {

                                                Object newArray = Array.newInstance(cls, length - 1);
                                                System.arraycopy(array, 0, newArray, 0, arrayIndex);
                                                System.arraycopy(array, arrayIndex + 1, newArray, 0, length - 1 - arrayIndex);
                                                m.invoke(target, newArray);
                                            }
                                        }
                                        return;
                                    } else {
                                        throw new NullPointerException("array is null");
                                    }
                                }
                            }

                        }

                        if (isMap) {
                            keyName = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[\"(\\S*)\"\\]", 1);
                            String mapKey = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[\"(\\S*)\"\\]", 2);
                            Field field = getFieldFromMap(fieldMap, keyName);
                            if (field == null)
                                throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                            Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                            if (getterMethod == null)
                                throw new NoSuchMethodException("no such method");
                            if (field.getType() == Map.class) {
                                if (getterMethod.invoke(target) == null) {
                                    throw new NullPointerException("such object is null");
                                } else {
                                    boolean containsKey = (boolean) Map.class.getMethod("containsKey", Object.class).invoke(getterMethod.invoke(target), mapKey);
                                    if (containsKey) {
                                        Object o = getterMethod.invoke(target);
                                        Class cls = getMapClazz(field);
                                        if (commonClassMap.containsValue(getMapClazz(field))) {
                                            Object p = param.get(key);
                                            HashMap.class.getMethod("put", Object.class, Object.class).invoke(o, mapKey, p);
                                        } else {
                                            Object p = null;
                                            if (param.get(key) != null) {
                                                p = JSONObject.parseObject(param.get(key).toString(), cls);
                                            }
                                            HashMap.class.getMethod("put", Object.class, Object.class).invoke(o, mapKey, p);
                                        }
                                        return;
                                    } else {
                                        throw new RuntimeException("map no such key" + " :" + mapKey);
                                    }
                                }
                            }
                        }

                    }

                } else {
                    boolean isArray = keyMember.matches("\\S+\\[\\d+\\]");
                    boolean isMap = keyMember.matches("\\S+\\[\"\\S*\"\\]");
                    //map
                    if (isMap) {
                        keyName = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[\"(\\S*)\"\\]", 1);
                        String mapKey = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[\"(\\S*)\"\\]", 2);
                        Field field = getFieldFromMap(fieldMap, keyName);
                        if (field == null)
                            throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                        Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                        if (getterMethod == null)
                            throw new NoSuchMethodException("no such method");
                        if (field.getType() == Map.class) {
                            if (getterMethod.invoke(target) == null) {
                                throw new NullPointerException("such object is null");
                            } else {
                                boolean containsKey = (boolean) Map.class.getMethod("containsKey", Object.class).invoke(getterMethod.invoke(target), mapKey);
                                if (containsKey) {
                                    target = Map.class.getMethod("get", String.class).invoke(getterMethod.invoke(target), mapKey);
                                } else {
                                    throw new RuntimeException("map no such key" + " :" + mapKey);
                                }
                            }
                        }
                    }
                    // array  -> list  & Array
                    else if (isArray) {
                        Integer arrayIndex = null;
                        keyName = RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[(\\d+)\\]", 1);
                        arrayIndex = Integer.parseInt(RegexUtil.fetchDataFromDocumentText(keyMember, "(\\S+)\\[(\\d+)\\]", 2));
                        Field field = getFieldFromMap(fieldMap, keyName);
                        if (field == null)
                            throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                        Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                        if (getterMethod == null)
                            throw new NoSuchMethodException("no such method");
                        // List Array
                        if (field.getType() == List.class) {
                            if (getterMethod.invoke(target) == null) {
                                throw new NullPointerException("such object is null");
                            } else {
                                int index = (int) List.class.getDeclaredMethod("size").invoke(getterMethod.invoke(target));
                                if (j != keys.length - 1) {
                                    if (arrayIndex != null && arrayIndex <= index - 1) {
                                        target = List.class.getMethod("get", int.class).invoke(getterMethod.invoke(target), arrayIndex);
                                    } else {
                                        throw new RuntimeException("路径更新");
                                    }
                                }
                            }
                        }
                        // Array Type
                        else if (field.getType().isArray()) {
                            Object array = getterMethod.invoke(target);
                            if (array == null) {
                                throw new NullPointerException(" array is null");
                            }
                            target = Array.get(array, arrayIndex);
                            if (target == null) {
                                throw new IndexOutOfBoundsException("no such index array");
                            }
                        }
                    } else {
                        Field field = getFieldFromMap(fieldMap, keyName);
                        if (field == null)
                            throw new NoSuchFieldException("key:" + key + " no such field " + " :" + keyName);
                        Method getterMethod = getGetterMethodFromMap(getterMethodMap, keyName);
                        if (getterMethod == null)
                            throw new NoSuchMethodException("no such method");
                        target = getterMethod.invoke(target);
                    }

                    fieldMap = getFiledMap(target.getClass());
                    setterMethodMap = getSetterMethod(target.getClass());
                    getterMethodMap = getGetterMethod(target.getClass());
                    j++;
                }

            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }


    private Method getSetterMethodFromMap(Map<String, Method> setterMethodMap, String key) {
        if (setterMethodMap.containsKey(key)) {
            return setterMethodMap.get(key);
        } else {
            return null;
        }
    }

    private Field getFieldFromMap(Map<String, Field> fieldMap, String key) {
        if (fieldMap.containsKey(key)) {
            return fieldMap.get(key);
        } else {
            return null;
        }
    }

    private Method getGetterMethodFromMap(Map<String, Method> getterMethodMap, String key) {
        if (getterMethodMap.containsKey(key)) {
            return getterMethodMap.get(key);
        } else {
            return null;
        }
    }

    public void doUpdate(Map<String, Object> param, T object) {
        this.param = param;
        for (String key : param.keySet()) {
            doAnalyzeNew(key, object);
        }
    }
}



```

```java
public class CommonUtilTest {
    private Map<String, Object> input = new HashMap<>();

    @Test
    public void toMap() throws IOException {
        String source = "{\n" +
                "    \"personalInfo\": {\n" +
                "        \"fullName\": \"马苗苗\",\n" +
                "        \"gender\": \"female\",\n" +
                "        \"birthDate\": \"1992-10-01\",\n" +
                "        \"houseHoldRegistration\": \"乌兰察布\",\n" +
                "        \"residence\": \"北京\",\n" +
                "        \"email\": \"15801098596@163.com\",\n" +
                "        \"mobile\": \"15801098596\",\n" +
                "        \"nationality\": \"中国\",\n" +
                "        \"ethnicity\": \"汉\"\n" +
                "    },\n" +
                "    \"expectedJob\": {\n" +
                "        \"location\": \"北京\",\n" +
                "        \"totalPackage\": \"8001-10000元/月\",\n" +
                "        \"jobType\": \"fullTime\",\n" +
                "        \"profession\": \"市场主管、市场营销主管、市场文案策划\",\n" +
                "        \"industry\": \"医疗设备/器械、医疗/护理/美容/保健/卫生服务、医药/生物工程\"\n" +
                "    },\n" +
                "    \"educationInfo\": [\n" +
                "        {\n" +
                "            \"highestDegree\": \"master\",\n" +
                "            \"startDate\": \"2014-09\",\n" +
                "            \"endDate\": \"2017-06\",\n" +
                "            \"college\": \"北京化工大学\",\n" +
                "            \"major\": \"制药工程\"\n" +
                "        },\n" +
                "        {\n" +
                "            \"highestDegree\": \"bachelor\",\n" +
                "            \"startDate\": \"2010-09\",\n" +
                "            \"endDate\": \"2014-06\",\n" +
                "            \"college\": \"北京化工大学\",\n" +
                "            \"major\": \"制药工程\"\n" +
                "        }\n" +
                "    ],\n" +
                "    \"careerInfo\": [\n" +
                "        {\n" +
                "            \"aboutOrganization\": \"合资\",\n" +
                "            \"startDate\": \"2017-05\",\n" +
                "            \"summary\": \"1. 公众号运营：包括日常发稿、运维、广告投放收益等。 运营公司公众号期间，粉丝增长7K+，最高阅读量2W+ （粉丝数5W+） 2. 活动执行：包括活动前准备工作 （方案策划、项目预算、人员安排及相关事项对接），活动过程中现场后勤及活动结束后的收尾工作。 2017年昆明国际旅游交易参展会体育馆展： 主办方为云南省旅发委，会议规模约1万人，公司参与展位108m ，展期三天，成功展示公司强大的救援能力及服务平台，并与云南省旅发委签订战略协议，为公司在云南渠道铺设打下坚实基础。 2017年公司年会筹划：参与人数约为500人，主要负责现场微信上墙互动、节目评选及抽奖娱乐环节，圆满完成工作。 2018年公司与马鞍山市民卡公司共同举办的“健康中国行“活动，规模约200人，主负责活动方案策划、预算评估及新闻稿修饰，保证活动顺利进行和后期宣传。 3. 公司线上宣传：包括各类媒体信息搜集和对接。 主要完成官网百度及360认证、品牌商标注册。 4. 规范撰写：包括市场部公众号文章发布及线上OA审文、活动落地执行、内部管理规范等。规范经审核后全员发送，确保后续工作进行以规范内容为准。\"\n" +
                "        }\n" +
                "    ],\n" +
                "    \"projectInfo\": [],\n" +
                "    \"languageSkill\": [],\n" +
                "    \"certificates\": [],\n" +
                "    \"honors\": [],\n" +
                "    \"patents\": [],\n" +
                "    \"papers\": [],\n" +
                "    \"books\": [],\n" +
                "    \"candidateId\": \"Xa9M7GYBpaluvtCwBCL1\",\n" +
                "    \"name\": \"智联招聘_马苗苗_中文_20180605_1528181385912.doc\",\n" +
                "    \"language\": \"SimplifiedChinese\",\n" +
                "    \"aboutSelf\": \"综合能力：做事情执行力强，有较强的 人际交流和学习的能力 做事风格：做事直接，目的导向性强 个人性格：乐观开朗，热情友善 来电测试\\n \",\n" +
                "    \"workingStatus\": null,\n" +
                "    \"tags\": [],\n" +
                "    \"id\": \"Xq9M7GYBpaluvtCwBSI-\",\n" +
                "    \"certificates\": [\n" +
                "        {\n" +
                "            \"name\": \"大学英语六级\",\n" +
                "            \"date\": \"2013-12\"\n" +
                "        }\n" +
                "    ]\n" +
                "}";

        ObjectMapper mapper = new ObjectMapper();
        HashMap<String, Object> result = mapper.readValue(source, HashMap.class);
        travelMap("", result);

        Resume resume =  mapper.readValue(source, Resume.class);
        MapBeanUtil<Resume> beanUtil = new MapBeanUtil<>();
        beanUtil.doUpdate(input, resume);
        assertNotNull(resume);
    }

    private void travelMap(String path, Map<String, Object> object) {
        for (Map.Entry<String, Object> e: object.entrySet()) {
            String thisPath;
            String key = e.getKey();
            if (path.length() == 0) {
                thisPath = key;
            } else {
                thisPath = path + "." + key;
            }
            Object value = e.getValue();
            if (value instanceof Map) {
                travelMap(thisPath, (Map<String, Object>) value);
            } else if (value instanceof List) {
                travelList(thisPath, (List<Object>)value);
            } else {
                travelValue(thisPath, value);
            }
        }
    }

    private void travelList(String path, List<Object> object) {
        int i = 0;

        for (Object o: object) {
            String cur = String.format("%s[%d]", path, i);
            if (o instanceof Map) {
                travelMap(cur, (Map<String, Object>) o);
            } else if (o instanceof List) {
                travelList(cur, (List<Object>)o);
            } else {
                travelValue(cur, o);
            }
            ++i;
        }
    }

    private void travelValue(String path, Object object) {
        String result = String.format("%s = %s", path, object == null? null : object.toString());
        System.out.println(result);
        input.put(path, object);
    }
```