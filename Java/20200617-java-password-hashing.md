# Overview
해싱은 ***암호화 해시 함수***라고 하는 수학 함수를 사용하여 지정된 메시지에서 문자열 또는 해시를 생성하는 프로세스이다.
안전한 해싱 암호 함수에는 4가지의 주요 속성이 있어야만 한다.
1. ***결정적 이어야 한다.*** 동일한 해시 함수로 처리 된 동일한 메시지는 항상 동일한 해시를 생성해야 한다.
2. ***되돌릴 수 없다.*** 해시에서 다시 원본 메시지로 복호화할 수 없다.
3. ***엔트로피가 높다.*** 메시지를 조금만 변경하면 크게 다른 해시가 생성된다.
4. ***충돌에 저항성이 있다.*** 두 개의 서로 다른 메시지가 동일한 해시를 생성해서는 안된다.

[[참고] 안전한 해시 알고리즘](https://en.wikipedia.org/wiki/Secure_Hash_Algorithms)

# MD5 (권장하지 않음)
Java의 `MessageDigest`를 사용하면 쉽게 생성할 수 있다. 하지만 4번째 속성에 위배되고, 빠른 알고리즘으로 인해 무작위 대입 공격에
취약하여 사용 권장하지 않는다.

# SHA-512즘 (권장하지 않음)
무작위 시퀀스인 Salt 값을 활용하여 해시의 엔트로피를 증가시키는 방법이다.
대략적인 방법은 아래와 같다.
~~~
salt <- generate-salt;
hash <- salt + ':' + sha512(salt + password)
~~~
## 1. Salt 생성
~~~java
SecureRandom random = new SecureRandom();
byte[] salt = new byte[16];
random.nextBytes(salt);
~~~
## 2. SHA-512 해싱
~~~java
MessageDigest md = MessageDigest.getInstance("SHA-512");
md.update(salt);
byte[] hashedPassword = md.digest(passwordToHash.getBytes(StandardCharsets.UTF_8));
~~~

# PBKDF2, BCrypt 및 SCrypt (권장 알고리즘)
이들 각각의 알고리즘은 느리고, 견고한 특징을 가지고 있는 해싱 알고리즘이다. 무작위 대입 공격에 강한 특징을 가진다.

## 1. PBKDF2 구현
Salt는 암호 해싱의 기본 원칙이다.
> Salt 생성
~~~java
SecureRandom random = new SecureRandom();
byte[] salt = new byte[16];
random.nextBytes(salt);
~~~
> PBEKeySpec 및 SecretKeyFactory 작성

3번째 파라미터(65536)는 해싱 강도에 해당하는 값이다. 알고리즘이 실행되는 반복 횟수를 의미하며, 
해당 값에 따라 해시 생성 시간이 늘어난다.
~~~java
KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 65536, 128);
SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
~~~
> SecretKeyFactory를 사용하여 해시 생성
~~~java
byte[] hash = factory.generateSecret(spec).getEncoded();
~~~

## 2. BCrypt 및 SCrypt 구현
Java에서는 아직 제공되지 않지만, Spring Security에서 제공되고 있다.
Spring Security는 PasswordEncoder 인터페이스를 통해 이러한 모든 권장 알고리즘을 지원한다.
* `MessageDigestPasswordEncoder` : MD5와 SHA-512를 제공
* `PasswordEncoder` : PBKDF2를 제공
* `BCryptPasswordEncoder` : BCrypt를 제공
* `SCryptPasswordEncoder` : SCrypt를 제공
해당 알고리즘들은 위의 다른 알고리즘과 달리 내부적으로 Salt를 생성하고, 나중에 암호를 확인할 때 사용할 수 있도록 
Salt를 출력 해시 내에 저장한다.

# References
* https://www.baeldung.com/java-password-hashing
 