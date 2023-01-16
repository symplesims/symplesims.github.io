---
layout: post
title: "Different ways to create object in Java"
date:  2023-01-06 18:00:00 +0900
categories:

- Java
- Programming

---

Java Class 를 동작 하는 객체로 생성 하는 작업은 생각 보다 귀찮은 작업 입니다. 이 과정은 Java 뿐만 아니라 다른 프로그래밍 언어 역시 동일 합니다.

여기서는 Person 클래스를 정의하고 어떻게 오브젝트 인스턴스를 생성하는 과정을 몇가지 예제로 살펴 봅니다.

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

## Builder Pattern

Person 객체를 생성 하기 위한 5가지 방법을 알아 보았습니다. 하지만 클래스가 여러 속성을 가지고 있는 경우 그 객체를 생성 하는 방법은 번거롭기만 합니다.  
이 문제를 해결 하기 위해 실제로 서비스에 적용할 경우에는 생성자를 오버로딩 하거나, Factory 클래스를 활용 하거나 Builder 를 만들 수 있습니다.

### Constructor Overloading

생성자의 파라미터를 오버로딩 하는 방식 입니다.

```java
public class Person {

    private String firstName;
    private String lastName;
    private String birthDay;
    private String gender;
    private String email;

    public Person() {
        super();
    }

    public Person(String firstName, String lastName) {
        this();
        this.firstName = firstName;
        this.lastName = lastName;
    }

    public Person(String firstName, String lastName, String birthDay, String gender, String email) {
        this(firstName, lastName);
        this.birthDay = birthDay;
        this.gender = gender;
        this.email = email;
    }

    // 중략 ...
}


```

<br>

아래 예시는 3개의 Person 객체를 생성자 오버로딩을 이용 하여 생성 하고 있습니다. 이렇게 하면 setter 메서드를 이용 하는 것 보다 쉽고 편리합니다.

```java
class Main {

    public static void main(String[] args) {
        Person person = new Person();
        person.setFirstName("Symple");
        person.setLastName("Sims");
        person.setGender("M");

        Person person2 = new Person("SunShin", "YI");
        person2.setGender("M");
        person2.setBirthDay("15450428");
        person2.setEmail("sunshinyi1545.heo@chosun.org");

        Person person3 = new Person("Nanseolheon", "Heo", "F", "", "nanseolheon1563.heo@chosun.org");
    }

}
```

<br>

### Factory 패턴

Person 객체를 쉽게 생성해주는 빌더를 구현할 수 있습니다.

```java
public class PersonFactory {

    static public Person create(String firstName, String lastName) {
        Person person = new Person();
        person.setFirstName(firstName);
        person.setLastName(lastName);
        return person;
    }

    static public Person createWith(String firstName, String lastName, String birthDay, String gender, String email) {
        Person person = new Person();
        person.setFirstName(firstName);
        person.setLastName(lastName);
        person.setGender(gender);
        person.setBirthDay(birthDay);
        person.setEmail(email);
        return person;
    }

}

```

<br>

PersonFactory 클래스를 통해 목적에 맞게 Person 객체를 생성할 수 있습니다.

```java
class Main {

    public static void main(String[] args) {
        Person person = PersonFactory.create("SunShin", "YI");
        Person person2 = PersonFactory.createWith("Nanseolheon", "Heo", "F", "", "nanseolheon1563.heo@chosun.org");
    }

}
```

<br>

### Builder 패턴

Person 객체를 안전하게 생성하고 필요한 속성을 쉽게 설정할 수 있습니다.

```java
public class PersonBuilder {

    private String firstName;
    private String lastName;
    private String birthDay;
    private String gender;
    private String email;

    private PersonFactory() {
        super();
    }

    public static PersonFactory create() {
        return new PersonFactory();
    }

    public PersonFactory firstName(String firstName) {
        this.firstName = firstName;
        return PersonFactory.this;
    }

    public PersonFactory lastName(String lastName) {
        this.lastName = lastName;
        return PersonFactory.this;
    }

    public PersonFactory birthDay(String birthDay) {
        this.birthDay = birthDay;
        return PersonFactory.this;
    }

    public PersonFactory gender(String gender) {
        this.gender = gender;
        return PersonFactory.this;
    }

    public PersonFactory email(String email) {
        this.email = email;
        return PersonFactory.this;
    }

    public Person build() {
        return new Person(this.firstName, this.lastName, this.birthDay, this.gender, this.email);
    }
}

```

<br>

아래와 같이 Builder 를 사용 하는 장점은 객체를 생성 하고 설정이 필요한 속성을 비교적 쉽게 구성이 가능한 점 입니다.

```java
public class PersonMain {

    public static void main(String[] args) {
        Person person = PersonBuilder.create()
                .firstName("f")
                .lastName("l")
                .gender("M")
                .email("myemail@mydomain.com")
                .build();
        System.out.println(person);
    }
}
```

<br>

### Functional Builder 패턴

Java 8 부터 Functional Interface 와 함수형 문법을 지원합니다. java.util.function.Consumer 인터페이스를 사용 하여 함수형 프로그래밍 방식으로 객체를 생성 하고 속성을 설정 할 수 있습니다.

```java
public class PersonBuilder {

    public String firstName;
    public String lastName;
    public String birthDay;
    public String gender;
    public String email;

    private PersonBuilder(Consumer<PersonBuilder> f) {
        f.accept(this);
    }

    public static PersonBuilder create(java.util.function.Consumer<PersonBuilder> f) {
        return new PersonBuilder(f);
    }

    public Person build() {
        return new Person(this.firstName, this.lastName, this.birthDay, this.gender, this.email);
    }
}
```

<br>

PersonBuilder 클래스를 통해 Person 객체를 하려면 setter 메서드를 wrapping 해야 하는 귀찮은 작업을 해야 하는데 Consumer 를 이용하면 Builder 클래스를 간편하게 생성할 수
있으며, 많은 속성을 보다 쉽게 확인 하고 재 구성 (copy & paste) 할 수 있는 장점이 있습니다. 

```java
public class PersonMain {
    public static void main(String[] args) {
        // @formatter:off
        Person p = PersonBuilder.create(v -> {
            v.firstName = "fn.f";
            v.lastName  = "fn.l";
            v.gender    = "M";
            v.birthDay  = "19760101";
            v.email     = "fn.my@email.com";
        }).build();
        // @formatter:on
        System.out.println(p);
    }
}

```

<br>
<br>


## Conclusion 

객체를 생성하는 다양한 방법과 객체의 속성을 보다 쉽게 구성하기 위한 빌더 패턴을 알아 보았습니다.  

객체를 정의 하고 생성하는 것은 가장 기본적인 것이므로 한번쯤 다시 생각하고 쓰임새에 맞게 사용하는 것이 중요 합니다. 

매번 new 생성자를 통해 객체를 생성하고 속성을 정의 하는 것 보다 비즈니스 목적에 맞게 속성 정보를 구성 할 경우가 많이 있습니다. 이럴 경우에는 Factory 패턴을 이용하는 것이 보다 명시적이고 정확할 수 있습니다. 
