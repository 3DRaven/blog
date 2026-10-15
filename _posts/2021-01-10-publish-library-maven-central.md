---
layout: post
title: "Публикуем либку в maven central"
date: 2021-01-10 20:10:37 +0000
tags: ["java", "maven", "sonatype nexus", "pairwise"]
description: "Предисловие Решил я что то сделать полезное. Был проект в котором мы должны были вместо старого сервиса сделать новый, но лучше. Сервис большой получился, 10к строк бизнесс-правил+код на джаве 15к строк. Сейчас уже раза в два больше. Но так как логики много и никто толком уже многого не помнит (за тем нас и призвали, что бы были тесты, доки и минимум багов) надо было сравнить старый сервис и новый. Что бы наши тестеры не умерли от старости, пытаясь все проверить я подумал и написал сервис, который дергал апи старого и нового, сравнивал и сливал разницу ответов в базу, что бы тестеры проверили и мы пофиксили разницу. Но вот проблема, на входе у нас много параметров и возможных их значений, это же миллиарды комбинаций будут, проверять несколько месяцев с полной нагрузкой, да и похожих много. В итоге сначала была рождена идея взять и сократить возможные изменения параметров и вообще выкинуть некоторые из перебора, а потом была взята на вооружение Pairwise testing теория и она позволила закинуть все параметры и все значенеия и все проверить целиком."
---

# Предисловие

  

Решил я что то сделать полезное. Был проект в котором мы должны были вместо старого сервиса сделать новый, но лучше. Сервис большой получился, 10к строк бизнесс-правил+код на джаве 15к строк. Сейчас уже раза в два больше. Но так как логики много и никто толком уже многого не помнит (за тем нас и призвали, что бы были тесты, доки и минимум багов) надо было сравнить старый сервис и новый. Что бы наши тестеры не умерли от старости, пытаясь все проверить я подумал и написал сервис, который дергал апи старого и нового, сравнивал и сливал разницу ответов в базу, что бы тестеры проверили и мы пофиксили разницу. Но вот проблема, на входе у нас много параметров и возможных их значений, это же миллиарды комбинаций будут, проверять несколько месяцев с полной нагрузкой, да и похожих много.  
В итоге сначала была рождена идея взять и сократить возможные изменения параметров и вообще выкинуть некоторые из перебора, а потом была взята на вооружение Pairwise testing теория и она позволила закинуть все параметры и все значенеия и все проверить целиком.

  

Эта нехитрая теория утверждает, что если мы создадим тесткейсы где каждая пара параметров встречается хотя бы однажды, то так покроем чуть ли не 90% багов. При этом можно все так подобрать, что тест кейсов будут не миллиарды, а может несколько тысяч. Отличный результат! В итоге мы нашли сотни расхождений и даже кучу багов в легаси сервисе, а так же надоумлили сочинителей бизнесс-логики подумать о том, о чем они ранее вообще не задумывались, так как мы им просто подкинули новые интересные тест кейсы в которых было непонятно "как правильно".

  

Код написан, все работает, все счастливы, но выкидывать генератор тест кейсов как то не хотелось, потому я сел на выходных и написал либку, которая использовала тот же алгоритм, что и в рабочем сервисе, но была полностью оторвана от всего кроме lombok и написана в нерабочее время. Потому, захотелось ее опубликовать под Apache 2.0 лицензией в мавен централе. Но это было настолько муторно и плохо описано, что я решил оставить себе памятку на Хабре, что бы потом если вдруг, не мучиться второй раз.

  

Если вдруг надо, то [либка опубликована тут](https://github.com/3DRaven/pairwiser).

  

Итак, поехали.

  

## Регистрируемся на sonatype.org

  

Напрямую в Maven Central опубликовать ничего не выйдет просто так, потому я воспользовался промежуточным сервисом sonatype.org Сначала нужно там зарегистрироваться.

  

[У них есть документация, как все зарелизить](https://central.sonatype.org/pages/releasing-the-deployment.html).

  

[Есть дока](https://central.sonatype.org/pages/requirements.html), которую надо обязательно посмотреть с требованиями к проекту и особенно к pom.xml

  

## Готовим проект

  

Для этого нам понадобится: gnupg, maven release plugin да собственно и все. Причем в принципе без второго обойтись можно, да еще и релизы он делает в той же ветке git репы, в которой версию прибавил. Во всяком случае без возни. Так что второе по желанию. Возиться я не стал и его все же использовал.

  

## GNUpg

  

Это много кому известная штука для генерации ключей и много чего еще. И главная засада была в том, что ключи, которые по инструкции сделать можно из командной строки у меня потом не принял ни один сервер ключей публичных. Принимают и либо ошибку выдают, либо ключей там нет в итоге. Потому, вместо того что бы мучится с этим я рекомендую просто использовать GUI утилитку, хотя сначала я все делал с консоли и по докам. В итоге взял gpa и в два клика сделал себе ключи, которые сервера прожевали.

  

Там все просто:

  

1.  Открыть диспетчер ключей.  
    [Imgur](https://imgur.com/RCrGQm8)
2.  Создание нового ключа Ctrl+N.
3.  Далее заполняем свое имя и это все.
4.  Это важный этап, что бы sonatype.org мог проверить после публикации подпись вашей либки, надо что бы ваш публичный ключ был на сервере публичных ключей. Для этого в меню "Сервер" выбираем "Отправить ключи".  
    [Imgur](https://imgur.com/TU41cCe)  
    Ваш публичный ключ будет опубликован и его можно любой желающий взять что бы проверить подпись.

  

## Готовим settings.xml

  

В этом файле хранятся глобальные настройки Maven. В него нам надо добавить пароль к нашей связке ключей и репозиторию sonatype.org (вашему аккаунту там), которую мы сделали на прошлом шаге.

  

``` xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository/>
  <interactiveMode/>
  <usePluginRegistry/>
  <offline/>
  <servers>
    <server>
      <id>ossrh</id>
      <username>3DRaven</username>
      <password>passwordForYourAccountInSonata</password>
    </server>
  </servers>
  <mirrors/>
  <proxies/>
  <profiles>
    <profile>
      <id>ossrh</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <gpg.executable>gpg2</gpg.executable>
        <gpg.passphrase>passwordForGPGKey</gpg.passphrase>
      </properties>
    </profile>
  </profiles>
  <activeProfiles/>
</settings>
```

  

Обратите внимние на ossrh этот айдишник мы встретим потом в помнике.

  

### Готовим pom.xml

  

Далее нужно настроить помник проекта так что бы он подписал проект, сделал javadoc, собрал source. Это обязательные требования к публикации. Пример будет кусками, в помнике либки можете полный глянуть

  

Обязательные элементы

  

``` xml
<description>Generate array of minimal size of pairwise tests cover all pairs of changing params</description>
    <url>https://github.com/3DRaven/pairwiser</url>
    <inceptionYear>2020</inceptionYear>
    <licenses>
        <license>
            <name>Apache License, Version 2.0</name>
            <url>https://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
            <comments>A business-friendly OSS license</comments>
        </license>
    </licenses>
    <developers>
        <developer>
            <id>3DRaven</id>
            <name>Renat Eskenin</name>
            <email>email</email>
        </developer>
    </developers>
    <scm>
        <connection>scm:git:ssh://git@github.com:3DRaven/pairwiser.git</connection>
        <developerConnection>scm:git:ssh://git@github.com:3DRaven/pairwiser.git</developerConnection>
        <tag>HEAD</tag>
        <url>https://github.com/3DRaven/pairwiser</url>
    </scm>

    <issueManagement>
        <system>GitHub</system>
        <url>https://github.com/3DRaven/pairwiser/issues</url>
    </issueManagement>
    <distributionManagement>
        <snapshotRepository>
            <id>ossrh</id>
            <url>https://oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
        <repository>
            <id>ossrh</id>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>
```

  

Самое важное тут это distributionManagement. В нем указан как сервер, на который будет опубликован итоговый артефакт, так и id ossrh по которому будет найден в settings.xml логин и пароль на сервере. Далее нам нужно что бы наш артефакт был подписан, собран javadoc и source. Если три эти плагина не поместить именно в profiles раздел, то релиз плагин их запускать не будет. Так что они вписаны отдельно.

  

``` xml
<profiles>
        <!-- GPG Signature on release -->
        <profile>
            <id>release-sign-artifacts</id>
            <activation>
                <property>
                    <name>performRelease</name>
                    <value>true</value>
                </property>
            </activation>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>${maven.gpg.plugin.version}</version>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-source-plugin</artifactId>
                        <version>${maven.source.plugin.version}</version>
                        <executions>
                            <execution>
                                <id>attach-sources</id>
                                <goals>
                                    <goal>jar-no-fork</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-javadoc-plugin</artifactId>
                        <version>${maven.javadoc.plugin.version}</version>
                        <executions>
                            <execution>
                                <id>attach-javadocs</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
```

  

Так же использован плагин от sonatype.org, который публикует staging репозитрий на их сайте. Этот репозиторий видите только вы. После его публикации он будет в статусе Open и на нем запустятся автопроверки. Если они все пройдут, то можно будет его "закрыть", это такой статус репозитория у них. Среди проверок будет и проверка подписи и проверки требований sonatype.org к проектам. Следующий плагин нажмет за вас пару кнопок на сайте sonatype.org (close + drop) и покажет в консольке прогресс релиза. Можно его юзать, можно нет.

  

``` xml
<plugin>
    <groupId>org.sonatype.plugins</groupId>
    <artifactId>nexus-staging-maven-plugin</artifactId>
    <version>${nexus.staging.maven.plugin.version}</version>
    <extensions>true</extensions>
    <configuration>
        <serverId>ossrh</serverId>
        <nexusUrl>https://oss.sonatype.org/</nexusUrl>
        <autoReleaseAfterClose>true</autoReleaseAfterClose>
    </configuration>
</plugin>
```

  

## Просим опубликовать наш проект

  

На этом этапе нужно в sonatype.org сделать задачку, о том, что вы хотите свой проект опубликовать. Jira задачку [типа такой](https://issues.sonatype.org/browse/OSSRH-57701).

  

## Публикуем staging репозиторий

  

Эти команды включат релиз плагин, он подпишет ваш проект, поднимет версию, соберет исходники и доки, а так же запустит отправку проекта на sonatype.org

  

``` bash
mvn release:clean release:prepare 
mvn release:perform
```

  

Если все хорошо, после того как ваш репозиторий [создан тут](https://oss.sonatype.org/#stagingRepositories), все проверки прошли, вы можете его выбрать и нажать "Close".

  

## Просим зарелизить наш проект

  

На этом этапе нужно в [задачке](https://issues.sonatype.org/browse/OSSRH-57701), которую мы ранее сделали, написать, что хотите релизиться. Дело в том, что синк релиз репозитория с Maven Central они включают по требованию. Так как у вас уже есть свой стеджинг (спрятанный и видимый только вами) репозиторий вы можете идти далее. Нужно его "Закрыть", нажав Close. После закрытия, можно нажать Release и Drop. С этого момента ваш стейджинг репозиторий попадет в релизный репозиторий и будет удален из стейджинга. Через десять минут после этого, если синк с централом для вашей репы включен, он будет доступен в Maven Central

  

Ура!

  

P.S. Обратите внимание для group надо пройти этот путь один раз, последующие проекты можно публиковать не делая тасок и никого не ожидая ;)

  

## Краткий итог

  

1.  Регаемся на sonatype.org
2.  Просим их ваш проект опубликовать
3.  Делаем ключи для подписи и публикуем их
4.  Закидываем логопасы в settings.xml
5.  Настраиваем pom.xml
6.  Запускаем публикацию
7.  Заходим в sonatype.org и закрываем, релизим и дропаем свой репозиторий
8.  PROFIT!!!
