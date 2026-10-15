---
layout: post
title: "Jackson ObjectMapper Streaming API без возни"
date: 2023-09-01 15:11:04 +0000
tags: ["java", "mapper", "json"]
description: "Маппинг джейсонов или еще чего в модели чаще всего головная боль. Много мелочей, модели сделай, все подгони, аннотации расставь и прочее. Далее код примерно наколеночный, кому надо идею, поймет. Маппинг еще и памяти ест очень много, так как обычно ObjectMapper применяют примерно так: mapper.readValue(inputStream,Model.class) В итоге если модель большая маппер ее всю в памяти построит за раз, прочитав опять же весь json из стрима. Хуже когда даже json сначала в строку читают конечно. Потом приходит очередной ругатель и заявляет, что это java виновата. Что бы этого не делать, придумали ObjectMapper Streaming API. Что то вроде такого: while (jParser.nextToken() != JsonToken.END_OBJECT) { String fieldname = jParser.getCurrentName(); if (\"name\".equals(fieldname)) { jParser.nextToken(); parsedName = jParser.getText(); } Но фактически руками парсить json это тоже головняк. Есть хак, который позволяет и модели сразу получать и стриминг использовать. Может кому пригодится."
---

Маппинг джейсонов или еще чего в модели чаще всего головная боль. Много мелочей, модели сделай, все подгони, аннотации расставь и прочее. Далее код примерно наколеночный, кому надо идею, поймет. Маппинг еще и памяти ест очень много, так как обычно ObjectMapper применяют примерно так:

`mapper.readValue(inputStream,Model.class)`

В итоге если модель большая маппер ее всю в памяти построит за раз, прочитав опять же весь json из стрима. Хуже когда даже json сначала в строку читают конечно. Потом приходит очередной ругатель и заявляет, что это java виновата. Что бы этого не делать, придумали ObjectMapper Streaming API. Что то вроде такого:  
  
`while (jParser.nextToken() != JsonToken.END_OBJECT) {`  
`String fieldname = jParser.getCurrentName();`  
`if ("name".equals(fieldname)) {`  
`jParser.nextToken();`  
`parsedName = jParser.getText();`  
`}`

Но фактически руками парсить json это тоже головняк. Есть хак, который позволяет и модели сразу получать и стриминг использовать. Может кому пригодится. Предположим у нас есть json, который содержит в себе массив объектов:  
  
`{result:[{"name":"test"}]}`

Делаем две модели. Первая это общий объект:  
  
`public class Model {`  
`private Set<NestedModel> result;`  
`}`  
Вторая это вложенный объект:  
  
`public class NestedModel {`  
`private String name;`  
`}`

Далее делаем десериализатор, который десериализует модель класса NestedModel. При этом данный десериализатор должен в конструкторе принимать обработчик моделей NestedModel и возвращать null вместо результата. То есть он обработчиком модель обработает и вернет пустоту. В итоге ObjectMapper вернет Model с одним null элементом, который нам и не нужен, так как в процессе десериализации всех NestedModel мы их уже и так все обработали. В памяти при этом в момент времени хранится всего одна NestedModel и писать ручного кода не нужно вовсе. Десериализатор:  
  
`public class NestedModelDeserializer extends StdDeserializer<NestedModel> {`  
`private final Consumer<NestedModel> nestedModelConsumer;`  
`private final ObjectMapper innerMapper;`  
`protected NestedModelDeserializer(Class<NestedModel> vc, Consumer<NestedModel> nestedModelConsumer) {`  
`super(vc);`  
`this.nestedModelConsumer = nestedModelConsumer;`  
`this.innerMapper = new ObjectMapper();`  
`}`  
`@Override public NestedModel deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JacksonException {`  
`nestedModelConsumer.accept(innerMapper.readValue(p,NestedModel.class));`  
`return null;`  
`}`  
`} `

Второй ObjectMapper нужен потому, что если отдать десериализацию NestedModel первому, он в рекурсию попадет бесконечную. В итоге ручного парсинга джейсона не надо, десериализатор пишется очень просто, и памяти ест мало, по дороге можно как угодно обработать данные. Десериализатор конечно надо зарегистрировать в первом ObjectMapper. Это экономит много сил и упрощает код, так как ручной парсинг дело неудобное и там где родной Streaming API вынуждает тебя спуститься на уровень токенов данный подход позволяет остаться на уровне моделей и не задумываться как он там разберет json.
