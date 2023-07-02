---
layout: post
title: Design patterns that every developer should know
categories: [java, design pattern, design patterns]
---

As developers, we love solving intricate problems, and let's be honests, we love proposing intricate solutions as well. However, many of these problems have already been addressed by other people before us. Why should we spend time reinventing the wheel when we could leverage solutions that have proven their efficacy time and time again? This is where design patterns come into play.

Despite their numerous advantages, some developers avoid using design patterns, largely due to unfamiliarity or uncertainty about how to appropriately apply them. So, is the question on your mind: are design patterns really that important? To this query, I respond emphatically: **yes, they are!**. Here are the three main reasons:

1. **Time-saving**: Using design patterns avoids the need to craft a solution from scratch, saving you a substantial amount of time.
2. **Standardization**: Design patterns are widely known. When you say "I used a factory to create that object," your fellow developers will immediately understand what you mean.
3. **Ease of Understanding**: Most design patterns are relatively straightforward. Your unique solution might not be as intuitive or elegant as a tried-and-tested design pattern.

In this article, I'll be discussing some of the most useful design patterns and when to use them. Before we dive in, let's remember one important principle: **there's no one-size-fits-all solution.** The goal is to mold the design pattern to fit your problem, not the other way around.

Let's start!

#### Singleton

Singleton is a prevalent pattern, with various frameworks like Spring, CDI (@ApplicationScoped), and EJBs (@Singleton) already incorporating it. However, understanding how it is implemented is essential.

```java
public class SingletonSample {
   private static SingletonSample instance = null;

   private SingletonSample() {
   }

   public static SingletonSample getInstance() {
      if(instance == null) {
         instance = new SingletonSample();
      }
      return instance;
   }
}
```

In essence, the Singleton pattern ensures that only one instance of a class is created at runtime. 

#### Initialization on Demand Holder

Similar to Singleton, the Initialization on Demand Holder has a critical advantage: it's thread-safe. Singleton's getInstance() method isn't thread-safe unless synchronized, which could slow it down.

```java
public class SingletonSample {
    private SingletonSample() {
    }

    public static SingletonSample getInstance() {
        return SingletonSampleHolder.INSTANCE;
    }

    private static class SingletonSampleHolder {
        private static final SingletonSample INSTANCE = new SingletonSample();
    }
}
```

This pattern delays the instance's initialization until getInstance() is called and does so in a thread-safe manner.

#### The Strategy and The Factory Pattern

These two patterns are incredibly useful, especially when combined. They allow the creation of objects based on a qualifier.

```java
public interface Building {
    String getType();
}

public class House implements Building {
    public String getType(){
        return "house"
    }
}

public class Edifice implements Building {
    public String getType(){
        return "edifice"
    }
}

public class BuildingFactory {
    private static Map<String, Building> instances;

    static {
        instances = new HashMap<>();

        instances.put("house", new House());
        instances.put("edifice", new Edifice());
    }

    public static <T extends Building> T getBuilding(String type){
        return (T) instances.get(type); 
    }
}

Building building = BuildingFactory.getBuilding("house");
```

In this example, providing a building type will return the corresponding object, maximizing the use of polymorphism.

#### Fluent Builder

For objects that require numerous parameters, the builder pattern can keep your code clean and easy to understand.

```java
public class Product {
    private String id;
    private String name;
    private String description;
    private Double value;

    private Product(Builder builder) {
        setId(builder.id);
        setName(builder.name);
        setDescription(builder.description);
        setValue(builder.value);
    }

    public static Builder newProduct() {
        return new Builder();
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public Double getValue() {
        return value;
    }

    public void setValue(Double value) {
        this.value = value;
    }

    public static final class Builder {
        private String id;
        private String name;
        private String description;
        private Double value;

        private Builder() {
        }

        public Builder id(String id) {
            this.id = id;
            return this;
        }

        public Builder name(String name) {
            this.name = name;
            return this;
        }

        public Builder description(String description) {
            this.description = description;
            return this;
        }

        public Builder value(Double value) {
            this.value = value;
            return this;
        }

        public Product build() {
            return new Product(this);
        }
    }
}
```

```java
Product product = Product.newProduct()
                       .id(1l)
                       .description("TV 46'")
                       .value(2000.00)
                       .name("TV 46'")
                   .build();
```

Even though this class is relatively small and does not have many fields, the Fluent Builder pattern becomes especially handy as the complexity increases.

### Chain of Responsibility

The Chain of Responsibility pattern helps us manage complexity by breaking down our code into smaller, more manageable pieces, and organizing them in a sequence.

Here's how we can implement it:

```java
public interface Command<T>{
    boolean execute(T context);
}

public class FirstCommand implements Command<Map<String, Object>>{
    public boolean execute(Map<String, Object> context){
        //doing something in here
    }
}

public class SecondCommand implements Command<Map<String, Object>>{
    public boolean execute(Map<String, Object> context){
        //doing something in here
    }
}

public class Chain {
    public List<Command> commands;

    public Chain(Command... commands){
        this.commands = Arrays.asList(commands);
    }

    public void start(Object context){
        for(Command command : commands){
            boolean shouldStop = command.execute(context);

            if(shouldStop){
                return;
            }
        }
    }
}
```

```java
Chain chain = new Chain(new FirstCommand(), new SecondCommand());
Map<String, Object> context = new HashMap<>();
context.put("some parameter", "some value");
chain.start(context);
```

By breaking down our code into Commands, we can isolate logic, reorganize it as needed, and decouple our code, making it more maintainable and easier to understand.

### Template Method

Leveraging polymorphism, the Template Method pattern is useful when we have a sequence of method calls that remain the same, but the specific behaviors within those methods may vary. 

Here's an example:

```java

public abstract class Animal {
    public abstract void makeSound();
    public abstract void eatFood();
    public abstract void sleep();

    public void doEveryday(){
        makeSound();
        eatFood();
        sleep();
    }
}

public class Dog extends Animal {
    public void makeSound(){
        //bark!
    }

    public void eatFood(){
        //eat dog food
    }

    public void sleep(){
        //sleep a lot!
    }
}

public class Cat extends Animal {
    public void makeSound(){
        //meow!
    }

    public void eatFood(){
        //eat cat food
    }

    public void sleep(){
        //sleep just a little bit
    }
}
```

### State Pattern

Many objects can have different states, like a radio that can be on or off. The State Pattern helps us model this in an object-oriented manner, making state management more manageable and less error-prone.

Here's a simple implementation:

```java
public class Radio {
    private boolean on;
    private RadioState state;

    public Radio(RadioState state){
        this.state = state;
    }

    public void execute(){
        state.execute(this);
    }

    public void setState(RadioState state){
        this.state = state;
    }

    public void setOn(boolean on){
        this.on = on;
    }

    public boolean isOn(){
        return on;
    }

    public boolean isOff(){
        return !on;
    }
}

public interface RadioState {
    void execute(Radio radio);
}

public class OnRadioState implements RadioState {
    public void execute(Radio radio){
        //throws exception if radio is already on
        radio.setOn(true);
    }
}

public class OffRadioState implements RadioState {
    public void execute(Radio radio){
        //throws exception if radio is already off
        radio.setOn(false);
    }
}

Radio radio = new Radio(new OffRadioState()); //initial state
radio.setState(new OnRadioState());
radio.execute(); //radio on
radio.setState(new OffRadioState());
radio.execute(); //radio off
```

This example simplifies the radio's state management. However, the State Pattern becomes incredibly valuable when managing an object with several possible states. You can define rules to create final states and states that require a previous state to be executed.

### Conclusion

Design patterns provide tested and efficient solutions to common software development problems. Understanding and implementing these patterns can result in cleaner, more maintainable, and easier-to-understand code, and you probably should embrace them.

I hope you found this article helpful. Feel free to reach out to me if you have any question or suggestion. 

Happy coding!
