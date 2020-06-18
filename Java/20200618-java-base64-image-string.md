# Overview
이미지 파일 검색을 위해 `Apache Common IO` 및 `Java8 native Base64` 기능을 이용해 Base64 String으로 인코딩하고,
디코딩하는 방법에 대해 알아본다.

해당 방식은 모든 이진 파일 또는 이진 배열에 적용될 수 있다. 바이너리 형식의 콘텐츠를 REST 앤드 포인트로 JSON
형식으로 전송해야 할 때 유용하다.

# Dependency
~~~groovy
compile group: 'commons-io', name: 'commons-io', version: '2.7'
~~~

# 이미지 파일을 Base64 문자열로 변환
파일 내용을 우선 바이트 배열로 읽고, Java8의 Base64 클래스를 이용하여 인코딩한다.
~~~java
byte[] fileContent = FileUtils.readFileToByteArray(new File(filePath));
String encodedString = Base64.getEncoder().encodeToString(fileContent);
~~~

# Base64 문자열을 이미지 파일로 변환
Base64 String을 다시 바이너리 형태로 디코딩하여 파일에 쓴다.
~~~java
byte[] decodedBytes = Base64.getDecoder().decode(encodedString);
FileUtils.writeByteArrayToFile(new File(outputFileName), decodedBytes);
~~~

# References
* https://www.baeldung.com/java-base64-image-string