## Project Bootstrap

1. Use all small letter naming conventions for project naming. Do not use any underscore(_) or hyphen(-) for multiple word project names. i.e rokomari, orderservice, quotationservice etc. This will keep the package names clean.

1. Place the Hibernate entity classes in the entity package.

1. Use repository and service packages for Repository and Service classes.

1. Place template based controllers in the web.controller package and rest controllers in the web.restcontroller package.

1. For cloud projects (config server, eureka server, cloud services etc) and services i.e you need to add cloud dependencies in gradle.build file. Make sure you have these property blocks in your build.gradle file - 

    ```yaml
    ext {
        set('springCloudVersion', "Hoxton.SR3")
    }

    dependencyManagement {
        imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
    ```



## Application Properties

1. For Monolithic application always use these properties as first two. Although application.name property is not that useful in monolithic case, but we will use it for the sake of future microservice migration. Server port is not fixed, it better to mention it explicitly. 

    ```yaml
    spring.application.name=spring-boot-mysql
    server.port=8080
    ```

1. Use these properties for MySQL -

    ```yaml
    spring.datasource.url=jdbc:mysql://localhost:3306/spring-boot-mysql_db
    spring.datasource.username=root
    spring.datasource.password=root
    ```

1. Remove MySQL properties from production. Try to maintain this type of difference using profile. ddl-auto's default value is none, which does to do any database changes from hibernate entity. Production database change should be executed separately. 

    ```yml
    spring.jpa.show-sql = true
    spring.jpa.hibernate.ddl-auto = update
    ```


## Application Folder Structure




## Database

1. Whatever database you choose just add ‘_db’ after the application name and use that as the database name. For the quotationservice application, the database name will be quotationservice_db. If it is a microservice add the application name in from of it. If quotationservice is a microservice and part of notebook application, use notebook_quotationservice_db as the database name.

1. Use underscore based small letter naming convention for table names and table column names. i.e customer_order, product_purchase. Avoid any sort of short name or acronym for table names.


## Docker

1. Use container_name property in docker_compose.yml file. Naming format would be [application_name]_[service_name]_[container_service_name]. If redis is used for Rokomari application's quotationservice service -

    ```yml
    version: "3.5"

    services:

        redis:
            container_name: rokomari_quotationservice_redis
            image: redis
            ports:
                - "6380:6379"
            restart: on-failure
    ```


## Hibernate

#### This guide is specifically for Spring Boot projects.

1. Use just the @Entity annotation on top of a Entity class. Spring Boot automatically derives table name from class name (i.e PostDoc to post_doc), so @Table annotation is not needed.

    ```java
    @Entity
    public class Tag {
        ...
    }
    ```

1. Use Enum for multi valued properties (i.e Status with value ACTIVE and INACTIVE). Place those at the bottom of the Entity class. Create an enum in enumeration package only if it is usable in more than one classes.

    ```java
    @Entity
    public class Tag {
        ...
        ...

        public enum Status {ACTIVE, INACTIVE}
    }
    ```

1. These are 5 must have properties for every Domain - id, createdBy, updatedBy, createdAt, updatedAt. Use AbstractAuditingEntity. All the fields except id is managed by the AuditingEntityListener class.

1. id, createdBy, updatedBy, createdAt, updatedAt - these are 5 must have properties for all the Entity classes. Do not define these in all classes. Use an AbstractAuditingEntity class and extend this from all the Entity classes. There could be some special cases, when primary key generation strategy is not same all over the project. But this is very unlikely.

    ```java
    import org.springframework.data.jpa.domain.support.AuditingEntityListener;
    import org.springframework.format.annotation.DateTimeFormat;

    import javax.persistence.EntityListeners;
    import javax.persistence.GeneratedValue;
    import javax.persistence.GenerationType;
    import javax.persistence.Id;
    import javax.persistence.MappedSuperclass;
    import javax.validation.constraints.NotNull;
    import java.time.Instant;

    @MappedSuperclass
    @EntityListeners(AuditingEntityListener.class)
    public abstract class AbstractAuditingEntity {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @NotNull
        @DateTimeFormat(pattern = "dd-MMM-yyyy")
        private Instant createdAt;

        @DateTimeFormat(pattern = "dd-MMM-yyyy")
        private Instant updatedAt;

        @NotNull
        private String createdBy;

        private String updatedBy;

        public Long getId() {
            return id;
        }

        public void setId(Long id) {
            this.id = id;
        }

        public Instant getCreatedAt() {
            return createdAt;
        }

        public void setCreatedAt(Instant createdAt) {
            this.createdAt = createdAt;
        }

        public Instant getUpdatedAt() {
            return updatedAt;
        }

        public void setUpdatedAt(Instant updatedAt) {
            this.updatedAt = updatedAt;
        }

        public String getCreatedBy() {
            return createdBy;
        }

        public void setCreatedBy(String createdBy) {
            this.createdBy = createdBy;
        }

        public String getUpdatedBy() {
            return updatedBy;
        }

        public void setUpdatedBy(String updatedBy) {
            this.updatedBy = updatedBy;
        }
    }
    ```

    ```java
    @Entity
    public class Tag extends AbstractAuditingEntity {
        ...
    }
    ```

1. Use Hibernate/JPA annotations on property level. Mixing up annotation on property and setter levels creates compilation issue.

1. Use Long data type and IDENTITY GenerationType for the id property (primary key). Do not use TABLE or SEQUENCE GenerationType. Those strategies maintains an extra table in the database. Define the id property in AbstractAuditingEntity class. This is applicable for SQL databases. Need to figure out the way for no-SQL databases.

    ```java
    @@Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    ```

1. Use @NotNull annotation to define not null validation for a property. Using this annotation will also create the column with nullable in database. So no need to use @Column(nullable = true) annotation. Use @Size annotaion to define minimum and maximum length limit. This will be used for entity validation. max value will be used as varchar length in database. Do not use @NotEmpty or @NotBlank annotations. Using those will not create not nullable column in database.

    ```java
    @NotNull
    @Size(min = 5, max = 1023)
    private String title;
    ```

1. Use following for huge texts. This will create text type column in database. Drop the @NotNull annotation if not neede.

    ```java
    @NotNull
    @Column(columnDefinition = "TEXT")
    private String content;
    ```

1. Use @Enumerated annotation for enum properties. Most of the cases @NotNull property will be needed.

    ```java
    @NotNull
    @Enumerated(EnumType.STRING)
    private Status status;
    ```

1. If nullable=false needed in database column (which makes not nullable in application), use @NotNull annotion in Entity.

1. Use 127, 255, 511, 1023, 2047, 4095 as String length. For lengthier cases use TEXT. These are few examples. 

    ```java
    @NotNull
    @Size(min = 1, max = 255)
    private String name;

    @NotNull
    @Size(min = 5, max = 255)
    private String password;

    @NotNull
    @Size(min = 5, max = 255)
    @Column(unique = true)
    private String email;

    @NotNull
    @Size(min = 5, max = 255)
    @Column(unique = true)
    private String phone;

    @DateTimeFormat(pattern = "dd-MMM-yyyy")
    private Date dateOfBirth;

    @NotNull
    @Size(min = 5, max = 1023)
    private String title;

    @NotNull
    @Size(min = 5, max = 2047)
    private String topSummary;

    @NotNull
    @Column(columnDefinition = "TEXT")
    private String content;
    ```

1. JOINs - yet to come....

1. Explicitly disable Entity class default constructor with a private access modifier. Use a static method for instance creation. Use newObjectWithDefaults as method name. Assign all the default properties in this method.

    ```java
    private Tag() {
    }

    public static Tag newObjectWithDefaults() {
        Tag tag = new Tag();
        tag.postCount = 0;
        return tag;
    }
    ```



## Port Arrangement

1. Use Config server’s (configserver) port as 8888.

1. Use Eureka server’s (eurekaserver) port as 8761.

1. Start service application port from 8080. Then 8081, 8082 etc. If it reaches 8099 go back to 8060 and increase upto 8079. In the same fashion go back to 8040, 8020, 8000. For now 100 services should be enough for us.
