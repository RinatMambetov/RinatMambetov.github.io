---
layout: single
title: "Spring IoC Container"
date: 2024-07-19 18:18:05 +0300
categories: spring
---

Интерфейс org.springframework.context.ApplicationContext представляет собой контейнер Spring IoC и отвечает за инстанцирование, конфигурацию и сборку бинов. Контейнер получает инструкции о компонентах, которые необходимо инстанцировать, конфигурировать и собирать, считывая метаданные конфигурации. Метаданные конфигурации могут быть представлены в виде аннотированных классов компонентов, классов конфигурации с фабричными методами или внешних XML-файлов или скриптов Groovy. В любом из этих форматов вы можете составить свое приложение и богатые взаимозависимости между этими компонентами.

## Обзор контейнера

Несколько реализаций интерфейса ApplicationContext являются частью основного Spring. В автономных приложениях обычно создается экземпляр AnnotationConfigApplicationContext или ClassPathXmlApplicationContext.

В большинстве сценариев приложений явный код пользователя не требуется для инстанцирования одного или нескольких экземпляров контейнера Spring IoC. Например, в простом веб-приложении достаточно простого шаблонного веб-дескриптора XML в файле web.xml приложения. В сценарии Spring Boot контекст приложения неявно инициализируется для вас на основе общих соглашений по настройке.

Следующая диаграмма показывает общее представление о том, как работает Spring. Ваши классы приложения комбинируются с метаданными конфигурации, так что после создания и инициализации ApplicationContext у вас есть полностью настроенная и исполняемая система или приложение.

<!-- ![container-magic](https://docs.spring.io/spring-framework/reference/_images/container-magic.png) -->
![container-magic](/assets/images/container-magic.png)

## Метаданные конфигурации

Как показано на предыдущей диаграмме, контейнер Spring IoC использует форму метаданных конфигурации. Эти метаданные конфигурации представляют собой способ, которым вы, как разработчик приложения, сообщаете контейнеру Spring, как инстанцировать, конфигурировать и собирать компоненты в вашем приложении.

Сам контейнер Spring IoC полностью отделен от формата, в котором эти метаданные конфигурации фактически написаны. В настоящее время многие разработчики выбирают конфигурацию на основе Java для своих приложений Spring:

- Конфигурация на основе аннотаций: определяйте бины, используя метаданные конфигурации на основе аннотаций в классах компонентов вашего приложения.

- Конфигурация на основе Java: определяйте бины вне классов вашего приложения, используя классы конфигурации на основе Java. Для использования этих функций смотрите аннотации @Configuration, @Bean, @Import и @DependsOn.

Конфигурация Spring состоит как минимум из одного, а обычно из более чем одного определения бина, которые контейнер должен управлять. Конфигурация на основе Java обычно использует методы, аннотированные @Bean, внутри класса @Configuration, каждый из которых соответствует одному определению бина.

Эти определения бинов соответствуют фактическим объектам, которые составляют ваше приложение. Обычно вы определяете объекты уровня сервиса, объекты уровня персистенции, такие как репозитории или объекты доступа к данным (DAO), объекты представления, такие как веб-контроллеры, инфраструктурные объекты, такие как JPA EntityManagerFactory, JMS-очереди и так далее. Обычно не настраивают детализированные доменные объекты в контейнере, поскольку обычно это ответственность репозиториев и бизнес-логики — создавать и загружать доменные объекты.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="..." class="...">
		<!-- collaborators and configuration for this bean go here -->
	</bean>

	<bean id="..." class="...">
		<!-- collaborators and configuration for this bean go here -->
	</bean>

	<!-- more bean definitions go here -->

</beans>
```

## XML как внешний DSL конфигурации

Метаданные конфигурации на основе XML настраивают эти бины как элементы `<bean/>` внутри верхнего уровня элемента `<beans/>`. Следующий пример показывает основную структуру метаданных конфигурации на основе XML:

Атрибут id — это строка, которая идентифицирует отдельное определение бина.
Атрибут class определяет тип бина и использует полное имя класса.
Значение атрибута id может быть использовано для ссылки на объекты-сотрудники. XML для ссылки на объекты-сотрудники в этом примере не показан.

Для инстанцирования контейнера необходимо указать путь или пути к XML-ресурсам в конструкторе ClassPathXmlApplicationContext, который позволяет контейнеру загружать метаданные конфигурации из различных внешних ресурсов, таких как локальная файловая система, Java CLASSPATH и так далее.

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

Следующий пример показывает файл конфигурации объектов уровня сервиса (services.xml):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<!-- services -->

	<bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
		<property name="accountDao" ref="accountDao"/>
		<property name="itemDao" ref="itemDao"/>
		<!-- additional collaborators and configuration for this bean go here -->
	</bean>

	<!-- more bean definitions for services go here -->

</beans>
```

Следующий пример показывает файл daos.xml для объектов доступа к данным:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="accountDao"
		class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
		<!-- additional collaborators and configuration for this bean go here -->
	</bean>

	<bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
		<!-- additional collaborators and configuration for this bean go here -->
	</bean>

	<!-- more bean definitions for data access objects go here -->

</beans>
```

В предыдущем примере уровень сервиса состоит из класса PetStoreServiceImpl и двух объектов доступа к данным типов JpaAccountDao и JpaItemDao (основанных на стандарте JPA для объектно-реляционного отображения). Элемент свойства name ссылается на имя свойства JavaBean, а элемент ref ссылается на имя другого определения бина. Эта связь между элементами id и ref выражает зависимость между сотрудничающими объектами.

## Составление метаданных конфигурации на основе XML

Полезно, когда определения бинов охватывают несколько XML-файлов. Часто каждый отдельный файл конфигурации XML представляет собой логический уровень или модуль в вашей архитектуре.

Вы можете использовать конструктор ClassPathXmlApplicationContext для загрузки определений бинов из фрагментов XML. Этот конструктор принимает несколько местоположений ресурсов, как было показано в предыдущем разделе. В качестве альтернативы вы можете использовать одно или несколько вхождений элемента `****<import/>` для загрузки определений бинов из другого файла или файлов. Следующий пример показывает, как это сделать:

```xml
<beans>
	<import resource="services.xml"/>
	<import resource="resources/messageSource.xml"/>
	<import resource="/resources/themeSource.xml"/>

	<bean id="bean1" class="..."/>
	<bean id="bean2" class="..."/>
</beans>
```

В предыдущем примере внешние определения бинов загружаются из трех файлов: services.xml, messageSource.xml и themeSource.xml. Все пути расположения относительны к файлу определения, который выполняет импорт, поэтому services.xml должен находиться в той же директории или класспасе, что и файл, выполняющий импорт, в то время как messageSource.xml и themeSource.xml должны находиться в расположении ресурсов ниже расположения импортирующего файла. Как видно, начальный слэш игнорируется. Однако, учитывая, что эти пути относительны, лучше вообще не использовать слэш. Содержимое импортируемых файлов, включая верхний уровень элемента `<beans/>`, должно быть действительными XML-определениями бинов в соответствии со схемой Spring.

Возможно, но не рекомендуется, ссылаться на файлы в родительских директориях, используя относительный путь "../". Это создает зависимость от файла, который находится вне текущего приложения. В частности, такая ссылка не рекомендуется для URL-адресов classpath: (например, classpath:../services.xml), где процесс разрешения во время выполнения выбирает «ближайший» корень класспаса, а затем ищет в его родительской директории. Изменения конфигурации класспаса могут привести к выбору другой, неверной директории.

Вы всегда можете использовать полностью квалифицированные местоположения ресурсов вместо относительных путей: например, file:C:/config/services.xml или classpath:/config/services.xml. Однако имейте в виду, что вы связываете конфигурацию вашего приложения с конкретными абсолютными местоположениями. Обычно предпочтительнее сохранять косвенность для таких абсолютных местоположений — например, через заполнители "${…​}", которые разрешаются в свойствах системы JVM во время выполнения.

Само пространство имен предоставляет возможность директивы импорта. Дополнительные функции конфигурации, помимо простых определений бинов, доступны в ряде XML-пространств имен, предоставляемых Spring, например, в пространствах имен context и util.

## DSL определения бинов на Groovy

В качестве еще одного примера для внешних метаданных конфигурации определения бинов также могут быть выражены в DSL определения бинов Groovy от Spring, известном из фреймворка Grails. Обычно такая конфигурация хранится в файле с расширением ".groovy" со структурой, показанной в следующем примере:

```groovy
beans {
	dataSource(BasicDataSource) {
		driverClassName = "org.hsqldb.jdbcDriver"
		url = "jdbc:hsqldb:mem:grailsDB"
		username = "sa"
		password = ""
		settings = [mynew:"setting"]
	}
	sessionFactory(SessionFactory) {
		dataSource = dataSource
	}
	myService(MyService) {
		nestedBean = { AnotherBean bean ->
			dataSource = dataSource
		}
	}
}
```

Этот стиль конфигурации в значительной степени эквивалентен XML-определениям бинов и даже поддерживает XML-пространства имен конфигурации Spring. Он также позволяет импортировать файлы определения бинов XML через директиву importBeans.

Использование контейнера
ApplicationContext — это интерфейс для продвинутой фабрики, способной поддерживать реестр различных бинов и их зависимостей. Используя метод T getBean(String name, Class\<T> requiredType), вы можете извлекать экземпляры ваших бинов.

ApplicationContext позволяет вам читать определения бинов и получать к ним доступ, как показано в следующем примере:

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

С конфигурацией на Groovy процесс инициализации выглядит очень похоже. У него есть другой класс реализации контекста, который осведомлен о Groovy (но также понимает определения бинов XML). Следующий пример демонстрирует конфигурацию на Groovy:

```java
ApplicationContext context = new GenericGroovyApplicationContext("services.groovy", "daos.groovy");
```

Самый гибкий вариант — это GenericApplicationContext в сочетании с делегатами чтения, например, с XmlBeanDefinitionReader для XML-файлов, как показано в следующем примере:

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();
```

Вы также можете использовать GroovyBeanDefinitionReader для файлов Groovy, как показано в следующем примере:

```java
GenericApplicationContext context = new GenericApplicationContext();
new GroovyBeanDefinitionReader(context).loadBeanDefinitions("services.groovy", "daos.groovy");
context.refresh();
```

Вы можете комбинировать такие делегаты чтения в одном ApplicationContext, считывая определения бинов из различных источников конфигурации.

Затем вы можете использовать метод getBean для извлечения экземпляров ваших бинов. Интерфейс ApplicationContext имеет несколько других методов для получения бинов, но, в идеале, ваш код приложения не должен их использовать. Действительно, ваш код приложения не должен содержать вызовов метода getBean() и, таким образом, не должен зависеть от API Spring. Например, интеграция Spring с веб-фреймворками предоставляет внедрение зависимостей для различных компонентов веб-фреймворков, таких как контроллеры и управляемые JSF-бины, позволяя вам объявлять зависимость от конкретного бина через метаданные (такие как аннотация автозаполнения).

[оригинал](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)
