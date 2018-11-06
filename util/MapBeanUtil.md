
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

import java.io.UnsupportedEncodingException;
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
            if (cls != null) {
                Field[] fileds = cls.getDeclaredFields();
                for (Field field : fileds) {
                    String fieldName = field.getName();
                    fieldMap.put(fieldName, field);
                }
                cacheFieldMap.put(cls, fieldMap);
            }
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
                        if (field.getType().isArray()) {
                            Object array = getterMethod.invoke(target);
                            if (array == null) {
                                throw new NullPointerException(" array is null");
                            }
                            target = Array.get(array, arrayIndex);
                            if (target == null) {
                                throw new IndexOutOfBoundsException("no such index array");
                            }
                        }
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