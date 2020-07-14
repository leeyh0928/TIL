# Overview
***Flyway*** 프레임워크를 사용하여 애플리케이션의 데이터베이스 스키마를 안정적이고 쉽게 지속적으로 리모델링하는 방법에 대해
알아본다. Flyway 플러그인을 사용하여 메모리 내 H2 데이터베이스를 관리하는 예를 살펴볼 것이다.

***Flyway***는 마이그레이션을 사용하여 한 버전에서 다음 버전으로 데이터베이스를 업데이트한다. 데이터베이스 별 구문을 사용하여 
SQL 또는 고급 데이터베이스 변환을 위해 Java로 마이그레이션을 작성할 수 있다.

# Setup
## 1. Plugin 설정
~~~groovy
plugins {
    ...
    id "org.flywaydb.flyway" version "6.5.1"
    ...
}
~~~
## 2. Dependency 설정
~~~groovy
dependencies {
    ...

    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'org.flywaydb:flyway-core'

    runtimeOnly 'com.h2database:h2'

    ...
}
~~~
## 3. Flayway 설정
~~~groovy
flyway {
    user='sa'
    password=
    schemas='app-db'
    url='jdbc:h2:mem:app-db;MODE=MYSQL'
    locations='db/migration'
}
~~~

# 마이그레이션 정의
Flyway는 마이그레이션 스크립트에 대한 다음의 이름 지정 규칙을 준수한다.
> <접두사> <버전> __ <설명> .sql
- <접두사> – 기본 접두사는 V 이며, flyway.sqlMigrationPrefix 등록 정보를 사용하여 위 구성 파일에서 구성한다.
- <버전> – 마이그레이션 버전 번호. 주 버전과 부 버전은 밑줄 로 구분 될 수 있다. 마이그레이션 버전은 항상 1로 시작해야 한다.
- <설명> – 마이그레이션에 대한 텍스트 설명. 설명은 이중 밑줄로 버전 번호와 구분해야 한다.

`V1_0__create_employee_schema.sql` 이라는 이주 스크립트 파일명으로 `$PROJECT_ROOT`에 db/migration 디렉토리 아래에 작성한다.
~~~sql
CREATE TABLE IF NOT EXISTS `employee` (
    `id` int NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `name` varchar(20),
    `email` varchar(50),
    `date_of_birth` timestamp
) ENGINE=InnoDB DEFAULT CHARSET=UTF8;
~~~

# 마이그레이션 실행
~~~bash
./gradlew clean flywayMigrate
~~~
데이터베이스 스키마는 이제 다음과 같은 형태가 될 것이다.

employee:
+----+------+-------+---------------+
| id | name | email | date_of_birth |
+----+------+-------+---------------+
정의 및 실행 단계를 반복하여 더 많은 마이그레이션을 수행 할 수 있다.
