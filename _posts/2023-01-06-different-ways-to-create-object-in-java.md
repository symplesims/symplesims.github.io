---
layout: post
title: "Different ways to create object in Java"
date:  2023-01-06 18:00:00 +0900
categories:
- Java
- Programming
---

Java Class 를 동작 하는 객체로 생성 하는 작업은 생각 보다 귀찮은 작업 입니다. 이 과정은 Java 뿐만 아니라 다른 프로그래밍 언어 역시 동일 합니다.  

Person 클래스를 정의하고 오브젝트 인스턴스를 생성하는 과정을 몇가지 예제로 살펴 봅니다.

<br>

## Way 1 - 생성자를 통한 객체 생성

가장 일반적인 방법으로 생성자를 통해 객체를 생성 합니다. 

**Person.java** 클래스는 다음과 같습니다.

```java
public class Person {

    private String firstName;
    private String lastName;
    private String birthDay;
    private String gender;
    private String email;
    private String cellphone;

    public Person() {
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public String getBirthDay() {
        return birthDay;
    }

    public void setBirthDay(String birthDay) {
        this.birthDay = birthDay;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getCellphone() {
        return cellphone;
    }

    public void setCellphone(String cellphone) {
        this.cellphone = cellphone;
    }

}
```

우리는 아래와 같이 객체를 가장 일반적인 생성자를 통해 객체를 생성 하고 필요로 하는 속성 값을 setter 메서드를 통해 설정 합니다. 값을 가져올 때는 getter 메서드를 통해 가져 옵니다.  

```java
class Main {

    public static void main(String[] args) {
        Person person = new Person();
        person.setFirstName("Symple");
        person.setLastName("Sim");
        person.setBirthDay("2002-01-01");
        person.setGender("M");
        person.setEmail("symple.sim@yourdomain.com");
        person.setCellphone("+82 10 1111 1111");

        System.out.print(person.getFirstName().equals("Symple"));
    }

}
```

<br>

## Way 2 - clone 메서드를 통한 객체 생성

clone 메서드를 사용하려면 아래와 같이 `Cloneable` 인터페이스를 구현 하여야 합니다. 

```java
public class Person implements Cloneable {

    // 중략 

    public Person clone() throws CloneNotSupportedException {
        return (Person) super.clone();
    }
}

class Main {

    public static void main(String[] args) throws Exception {
        Person person = new Person();
        person.setFirstName("Symple");
        person.setLastName("Sim");
        person.setBirthDay("2002-01-01");
        person.setGender("M");
        person.setEmail("symple.sim@yourdomain.com");
        person.setCellphone("+82 10 1111 1111");

        Person person2 = person.clone();
        System.out.println(person2.getFirstName().equals("Symple"));
    }

}

```

아래와 같이 person2 객체는 person 객체로부터 간단하게 복제할 수 있습니다. 
```
Person person2 = person.clone();
```

<br>

## Way 3 - ClassLoader 을 통한 객체 생성 

"Class.forName" 을 통한 방법은 동적 클래스로딩 방식으로 JDBC Driver 같이 사전 정의된 플러그인 방식으로 객체를 생성하는 경우 유용 합니다.    
```java
class Main {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("Person");
        Person person = (Person) clazz.newInstance();
        person.setFirstName("Symple");
        person.setLastName("Sim");
        person.setBirthDay("2002-01-01");
        person.setGender("M");
        person.setEmail("symple.sim@yourdomain.com");
        person.setCellphone("+82 10 1111 1111");

        System.out.println(person.getFirstName().equals("Symple"));
    }
}
```

<br>

## Way 4 - Person Class 정의를 통한 객체 생성 

Person 클래스에 정의된 메타데이터와 Java reflection 기능을 활용하여 객체를 생성하는 방식 입니다.  
역시 동적으로 클래스를 생성 하며 Spring 프레임워크와 같이 사전 정의된 Bean 을 통해 Real 객체를 생성 할 때 유용합니다.  
```java
class Main {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Person.class;
        Constructor[] constructors = clazz.getDeclaredConstructors();
        Person person = (Person) constructors[0].newInstance(null);
        person.setFirstName("Symple");
        person.setLastName("Sim");
        person.setBirthDay("2002-01-01");
        person.setGender("M");
        person.setEmail("symple.sim@yourdomain.com");
        person.setCellphone("+82 10 1111 1111");

        System.out.println(person.getFirstName().equals("Symple"));
    }
}
```

<br>

## Way 5 - Object Stream 을 통한 객체 생성 (Serialization 과 Deserialization)
Java 클래스를 인스턴스 객체로 생성하고 그 객체를 File 로 작성(Serialization) 합니다.  
이렇게 작성된 파일 스트림을 읽어들여서(Deserialization) 원래의 객체로 생성 할 수 있습니다.  

객체의 무결성을 보장하기 위해 직렬화 처리를 위해 Person 클래스에 아래와 같이 Serializable 인터페이스를 구현 하여야 합니다.  


```java
public class Person implements Serializable {
    // 중략 
}
```


```java
class Main {
    public static void main(String[] args) throws Exception {
        Person person = new Person();
        person.setFirstName("Symple");
        person.setLastName("Sim");
        person.setBirthDay("2002-01-01");
        person.setGender("M");
        person.setEmail("symple.sim@yourdomain.com");
        person.setCellphone("+82 10 1111 1111");

        // person 객체를 person.ser 파일스트림 으로 기록 
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(new File("person.ser")))) {
            out.writeObject(person);
        } catch (IOException ioe) {
            ioe.printStackTrace();
        }

        // person.ser 파일스트림 으로부터 person2 객체 생성 
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(new File("person.ser")))) {
            Person person2 = (Person) in.readObject();
            System.out.println(person2.getFirstName().equals("Symple"));
        } catch (IOException ioe) {
            ioe.printStackTrace();
        }

    }
}
```

이 방법은 작은 모듈을 온라인을 통해 동시에 패치하는 경우 효과적일 수 있습니다. 예를 들면 IoT 센서와 같은 다수의 디바이스들을 한번에 패치 할 수 있습니다.  
하지만 해킹과 같은 나쁜 용도로 악용될 수 있으므로 주의가 필요 합니다. 

<br>
<br>

## Builder for Java Object
Person 객체를 생성 하기 위한 5가지 방법을 알아 보았습니다. 하지만 클래스가 여러 속성을 가지고 있는 경우 그 객체를 생성 하는 방법은 번거롭기만 합니다.  
이 문제를 해결 하기 위해 생성자를 오버로딩 하거나, Factory 클래스를 활용 하거나, PoJo Builder 를 만들 수 있습니다.  



