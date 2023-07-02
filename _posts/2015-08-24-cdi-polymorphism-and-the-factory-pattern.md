---
layout: post
title: CDI, Polymorphism and The Factory Pattern
categories: [java, design pattern, java ee]
---

Polymorphism is one of my favorite things from object-oriented programming. It's the ability of an object to take on many forms, allowing for flexible and efficient code execution. The beauty of polymorphism is that it can facilitate various behaviors in software without the explicit need to define each one in the code.

Let's take an example of a credit card transaction. The code will execute specific actions depending on whether the transaction is processed via VISA or MASTERCARD. Your initial thought might be to write the following code:

```java
public void pay(Transaction transaction) throws UnsupportedCardNetworkException {
    if("visa".equals(transaction.getType())){
        //pay with visa
        return;
    }
      
    if("mastercard".equals(transaction.getType())){
        //pay with mastercard
        return;
    }
        
    throw new UnsupportedCardNetworkException(); 
}
```

This design, however, isn't very efficient. It requires repetitive code for each card network, and each new card network added necessitates another 'if' statement. This approach can lead to bloated and less maintainable code. So, what's the solution? The answer lies in the power of polymorphism.

Let's redesign this code using polymorphism. We'll start with an interface called 'Payment' with a 'pay' method:

```java
public interface Payment {
   void pay(Transaction transaction);
}
```

Next, we will implement this interface for each credit card network:

```java
public class VisaPayment implements Payment {
    @Override
    public void pay(Transaction transaction) {
        //paying with visa
    }
}

public class MasterCardPayment implements Payment {
    @Override
    public void pay(Transaction transaction) {
        //paying with mastercard
    }
}
```

Now, whenever we need to execute a payment, we can simply do the following:

```java
Payment payment = new VisaPayment();
payment.pay(transaction);

Payment payment = new MasterCardPayment();
payment.pay(transaction);
```

But what if we want the choice of payment method to be dynamic, based on a variable rather than hard-coded 'if' statements? Let's utilize a PaymentFactory:

```java
public class PaymentFactory {
    private static final Map<String, Payment> payments;
    
    private PaymentFactory(){}

    static {
        payments = new HashMap<String, Payment>();
        payments.put("visa", new VisaPayment());
        payments.put("mastercard", new MasterCardPayment());
    }

    public static Payment get(String type){
        return payments.get(type);
    } 
}
```

Now, we can get a Payment instance as follows:

```java
Payment payment = PaymentFactory.get(transaction.getType());
```

But what about CDI? Well, let's introduce it into our solution then! 

CDI allows us to create multiple beans of a super type and choose an instance by a qualifier. We'll create a qualifier called @Network, which identifies the name of the network. Then, we'll annotate our Payment implementations with the corresponding network name.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER, CONSTRUCTOR})
@Qualifier
public @interface Network {
    String value();
}

@Network("mastercard")
public class MasterCardPayment implements Payment {
    @Override
    public void pay(Transaction transaction) {
        //paying with mastercard
    }
}

@Network("visa")
public class VisaPayment implements Payment {
    @Override
    public void pay(Transaction transaction) {
        //paying with visa
    }
}
```

In order to dynamically fetch instances, you also need to create an annotation literal, as shown below:

```java
public class NetworkAnnotationLiteral extends AnnotationLiteral<Network> implements Network {

    private String value;

    private NetworkAnnotationLiteral(String value) {
        this.value = value;
    }

    @Override
    public String value() {
        return value;
    }

    public static NetworkAnnotationLiteral network(String value){
        return new NetworkAnnotationLiteral(value);
    }
}
```

Now, we can select a payment instance using CDI:

```java
@Inject @Any
private Instance<Payment> payments;
 
@Test
public void payWithVisa(){
    Transaction transaction = new Transaction();
    transaction.setType("visa");
    Payment payment = payments.select(NetworkAnnotationLiteral.network(transaction.getType())).get();
    payment.pay(transaction);
}
 
@Test
public void payWithMaster(){
    Transaction transaction = new Transaction();
    transaction.setType("mastercard");
    Payment payment = payments.select(NetworkAnnotationLiteral.network(transaction.getType())).get();
    payment.pay(transaction);
}
```

With this design, CDI controls the creation of objects. You don't need to write your own factory, and you can inject resources into your Payment subtypes and choose any CDI scope you want.

The capabilities of CDI are vast and powerful, yet many developers remain unaware of its full potential. Leveraging polymorphism with CDI creates flexible, maintainable, and a beautiful software design.
