# Tomcat JNDI ve Gömülü Tomcat JNDI

Bir Spring Boot uygulamasının veritabanı bağlantısını JNDI üzerinden alması için iki yol: harici Tomcat'te `server.xml`/`context.xml` ile tanımlanan kaynak ve Spring Boot'un gömülü Tomcat'inde kod ile tanımlanan kaynak.

> **Not:** Bu depo 2018 yılında örnek amaçlı yazıldı ve artık güncellenmiyor. Bağımlılıkları eskidir ve bilinen güvenlik açıkları içerir; üretimde kullanmayın.

## İçerik

- `TomcatjdniApplication.java` – Gömülü Tomcat'te JNDI'yi açan (`tomcat.enableNaming()`) ve `jdbc/blabla` adlı `DataSource` kaynağını tanımlayan `TomcatServletWebServerFactory` bean'i.
- `ServletInitializer.java` – Uygulamanın `war` olarak harici Tomcat'e kurulabilmesi için.
- `application.properties` – `spring.datasource.jndi-name=java:comp/env/jdbc/blabla` ile veri kaynağını JNDI'den okur.

## Teknolojiler

- Java 8, Maven (`war` paketleme)
- Spring Boot 2.0.6 (Web, Data JPA)
- Tomcat DBCP 9.0.12, Oracle JDBC (`ojdbc6` 12.1.0.2), H2

## Çalıştırma

`ojdbc6` Maven Central'da bulunmaz; Oracle JDBC sürücüsünü kendiniz indirip `mvn install:install-file` ile yerel Maven deposuna eklemeniz gerekir. Aşağıdaki örneklerdeki adres, kullanıcı adı ve şifre yer tutucudur; kendi veritabanı bilgilerinizi yazın.

- Gömülü Tomcat ile: `TomcatjdniApplication` içindeki bağlantı bilgilerini düzenleyip `mvn spring-boot:run`.
- Harici Tomcat ile: `mvn package` ile oluşan `war` dosyasını Tomcat'e kurun ve kaynağı aşağıdaki gibi tanımlayın.

## Harici Tomcat'te JNDI kaynağı

Tomcat üzerinde `conf\server.xml` altında `<GlobalNamingResources>` etiketinin içine aşağıdaki kaynak eklenir.

```xml
<Resource name="jdbc/blabla"
  auth="Container"
  type="javax.sql.DataSource"
  factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
  driverClassName="oracle.jdbc.driver.OracleDriver"
  url="jdbc:oracle:thin:@bla.bla.COM:1521/bla"
  username="kul"
  password="sif"
  maxActive="20"
  maxIdle="10"
  maxWait="-1"
  validationQuery="select 1 from dual" />
```

Ardından `conf\context.xml` altında `<Context>` etiketinin içine `ResourceLink` eklenir.

```xml
<ResourceLink
  name="jdbc/blabla"
  global="jdbc/blabla"
  type="javax.sql.DataSource"/>
```

Kod içinde kaynağa `java:comp/env/jdbc/blabla` adıyla JNDI üzerinden erişilir.

## Gömülü Tomcat'te JNDI kaynağı

Spring Boot uygulaması gömülü Tomcat ile JNDI kullanacaksa kaynak tanımları `TomcatServletWebServerFactory` üzerinde yapılır.

```java
@Bean
public TomcatServletWebServerFactory tomcatFactory() {
    return new TomcatServletWebServerFactory() {
        @Override
        protected TomcatWebServer getTomcatWebServer(org.apache.catalina.startup.Tomcat tomcat) {
            tomcat.enableNaming();
            return super.getTomcatWebServer(tomcat);
        }

        @Override
        protected void postProcessContext(Context context) {
            ContextResource resource = new ContextResource();
            resource.setName("jdbc/blabla");
            resource.setAuth("Container");
            resource.setType("javax.sql.DataSource");
            resource.setProperty("driverClassName", "oracle.jdbc.driver.OracleDriver");
            resource.setProperty("url", "jdbc:oracle:thin:@bla.bla.COM:1521/bla");
            resource.setProperty("username", "kul");
            resource.setProperty("password", "sifre");
            context.getNamingResources().addResource(resource);
        }
    };
}
```

Bunun için gereken bağımlılıklar:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>

<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-dbcp</artifactId>
    <version>9.0.12</version>
</dependency>
```
