---
title: "Json"
date: 2022-03-06T12:08:26+08:00
draft: false
---

Json

+ ``@Seriali``,``@Expose``,``@Since``,``@Until``

+ ``@JsonAdapter``

+ ``GsonBuilder().setVersion()``

+ ``JsonWriter,JsonReader``

+ 自定义 ``TypeAdapter``,``JsonDeserializer``

+ ``JsonToken``

+ 一次性加载到内存 xml dom

+ 需要的时候再加载到内存 xml sax

+ JsonElement JsonObject JsonArray JsonNull JsonPrimitive

+ Type 基本类型 -> 创建对应的TypeAdapter读取Json串并返回

+ Type 对象 -> 创建ReflectiveTypeAdapter遍历对象的属性

+ ``TypeAdapter``任意一种类型对应一种``TypeAdapter``

+ ``class.getGenericSuperClass()``获取父类泛型类型

+ ``Excluder``排除器,自定义TypeAdapter,gson自带的TypeAdapter,ReflectiveTypeAdapter

+ ``GsonBuilder.registerTypeAdapter``