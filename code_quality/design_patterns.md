# Design Patterns - Complete Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Creational Patterns](#creational-patterns)
3. [Structural Patterns](#structural-patterns)
4. [Behavioral Patterns](#behavioral-patterns)
5. [Interview Questions](#interview-questions)

---

## Introduction

**Design Patterns** are reusable solutions to common problems in software design. They are templates that can be applied to real-world programming situations.

### Why Use Design Patterns?
- Proven solutions to common problems
- Improve code readability and maintainability
- Facilitate communication among developers
- Speed up development process

### Categories
1. **Creational** - Object creation mechanisms
2. **Structural** - Object composition and relationships
3. **Behavioral** - Communication between objects

---

## Creational Patterns

### 1. Singleton Pattern

**Purpose**: Ensures a class has only one instance and provides a global point of access to it.

**When to Use**: 
- Database connections
- Configuration managers
- Logging

**Java Example**:
```java
public class Singleton {
    // Private static instance
    private static Singleton instance;
    
    // Private constructor prevents instantiation
    private Singleton() {}
    
    // Public method to get instance
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}

// Thread-Safe Version
public class ThreadSafeSingleton {
    private static volatile ThreadSafeSingleton instance;
    
    private ThreadSafeSingleton() {}
    
    public static ThreadSafeSingleton getInstance() {
        if (instance == null) {
            synchronized (ThreadSafeSingleton.class) {
                if (instance == null) {
                    instance = new ThreadSafeSingleton();
                }
            }
        }
        return instance;
    }
}

// Enum Singleton (Best Practice)
public enum EnumSingleton {
    INSTANCE;
    
    public void doSomething() {
        // Your logic here
    }
}
```

---

### 2. Factory Pattern

**Purpose**: Creates objects without specifying the exact class to create.

**When to Use**:
- When you don't know ahead of time what class object you need
- When subclasses decide which class to instantiate

**Java Example**:
```java
// Product interface
interface Shape {
    void draw();
}

// Concrete products
class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing Circle");
    }
}

class Rectangle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing Rectangle");
    }
}

class Square implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing Square");
    }
}

// Factory class
class ShapeFactory {
    public Shape getShape(String shapeType) {
        if (shapeType == null) {
            return null;
        }
        if (shapeType.equalsIgnoreCase("CIRCLE")) {
            return new Circle();
        } else if (shapeType.equalsIgnoreCase("RECTANGLE")) {
            return new Rectangle();
        } else if (shapeType.equalsIgnoreCase("SQUARE")) {
            return new Square();
        }
        return null;
    }
}

// Usage
public class FactoryPatternDemo {
    public static void main(String[] args) {
        ShapeFactory factory = new ShapeFactory();
        
        Shape shape1 = factory.getShape("CIRCLE");
        shape1.draw();
        
        Shape shape2 = factory.getShape("RECTANGLE");
        shape2.draw();
    }
}
```

---

### 3. Abstract Factory Pattern

**Purpose**: Creates families of related objects without specifying their concrete classes.

**When to Use**: When you need to create multiple related objects that belong to a family.

**Java Example**:
```java
// Abstract products
interface Button {
    void paint();
}

interface Checkbox {
    void paint();
}

// Concrete products - Windows
class WindowsButton implements Button {
    @Override
    public void paint() {
        System.out.println("Windows Button");
    }
}

class WindowsCheckbox implements Checkbox {
    @Override
    public void paint() {
        System.out.println("Windows Checkbox");
    }
}

// Concrete products - Mac
class MacButton implements Button {
    @Override
    public void paint() {
        System.out.println("Mac Button");
    }
}

class MacCheckbox implements Checkbox {
    @Override
    public void paint() {
        System.out.println("Mac Checkbox");
    }
}

// Abstract Factory
interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete Factories
class WindowsFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
}

class MacFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
}

// Usage
public class Application {
    private Button button;
    private Checkbox checkbox;
    
    public Application(GUIFactory factory) {
        button = factory.createButton();
        checkbox = factory.createCheckbox();
    }
    
    public void paint() {
        button.paint();
        checkbox.paint();
    }
}
```

---

### 4. Builder Pattern

**Purpose**: Constructs complex objects step by step, separating construction from representation.

**When to Use**:
- When object has many optional parameters
- When you want immutable objects
- When construction process is complex

**Java Example**:
```java
public class User {
    // Required parameters
    private final String firstName;
    private final String lastName;
    
    // Optional parameters
    private final int age;
    private final String phone;
    private final String address;
    
    private User(UserBuilder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
    }
    
    // Getters
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }
    public String getPhone() { return phone; }
    public String getAddress() { return address; }
    
    // Builder class
    public static class UserBuilder {
        private final String firstName;
        private final String lastName;
        private int age;
        private String phone;
        private String address;
        
        public UserBuilder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }
        
        public UserBuilder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public UserBuilder address(String address) {
            this.address = address;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Usage
public class BuilderDemo {
    public static void main(String[] args) {
        User user = new User.UserBuilder("John", "Doe")
                        .age(30)
                        .phone("1234567890")
                        .address("123 Main St")
                        .build();
    }
}
```

---

### 5. Prototype Pattern

**Purpose**: Creates new objects by copying existing objects (cloning).

**When to Use**: When object creation is expensive and you want to avoid it.

**Java Example**:
```java
public abstract class Shape implements Cloneable {
    private String id;
    protected String type;
    
    abstract void draw();
    
    public String getType() {
        return type;
    }
    
    public String getId() {
        return id;
    }
    
    public void setId(String id) {
        this.id = id;
    }
    
    @Override
    public Object clone() {
        Object clone = null;
        try {
            clone = super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return clone;
    }
}

class Rectangle extends Shape {
    public Rectangle() {
        type = "Rectangle";
    }
    
    @Override
    void draw() {
        System.out.println("Drawing Rectangle");
    }
}

class Circle extends Shape {
    public Circle() {
        type = "Circle";
    }
    
    @Override
    void draw() {
        System.out.println("Drawing Circle");
    }
}

// Cache
class ShapeCache {
    private static Map<String, Shape> shapeMap = new HashMap<>();
    
    public static Shape getShape(String shapeId) {
        Shape cachedShape = shapeMap.get(shapeId);
        return (Shape) cachedShape.clone();
    }
    
    public static void loadCache() {
        Circle circle = new Circle();
        circle.setId("1");
        shapeMap.put(circle.getId(), circle);
        
        Rectangle rectangle = new Rectangle();
        rectangle.setId("2");
        shapeMap.put(rectangle.getId(), rectangle);
    }
}
```

---

## Structural Patterns

### 1. Adapter Pattern

**Purpose**: Converts the interface of a class into another interface clients expect. Allows incompatible interfaces to work together.

**When to Use**: When you want to use an existing class but its interface doesn't match what you need.

**Java Example**:
```java
// Target interface
interface MediaPlayer {
    void play(String audioType, String fileName);
}

// Adaptee
interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

class VlcPlayer implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        System.out.println("Playing vlc file: " + fileName);
    }
    
    @Override
    public void playMp4(String fileName) {
        // Do nothing
    }
}

class Mp4Player implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        // Do nothing
    }
    
    @Override
    public void playMp4(String fileName) {
        System.out.println("Playing mp4 file: " + fileName);
    }
}

// Adapter
class MediaAdapter implements MediaPlayer {
    AdvancedMediaPlayer advancedMusicPlayer;
    
    public MediaAdapter(String audioType) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedMusicPlayer = new VlcPlayer();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedMusicPlayer = new Mp4Player();
        }
    }
    
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedMusicPlayer.playVlc(fileName);
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedMusicPlayer.playMp4(fileName);
        }
    }
}

// Client
class AudioPlayer implements MediaPlayer {
    MediaAdapter mediaAdapter;
    
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("Playing mp3 file: " + fileName);
        } else if (audioType.equalsIgnoreCase("vlc") || 
                   audioType.equalsIgnoreCase("mp4")) {
            mediaAdapter = new MediaAdapter(audioType);
            mediaAdapter.play(audioType, fileName);
        } else {
            System.out.println("Invalid media type");
        }
    }
}
```

---

### 2. Decorator Pattern

**Purpose**: Adds new functionality to an object dynamically without altering its structure.

**When to Use**: When you want to add responsibilities to objects dynamically and transparently.

**Java Example**:
```java
// Component interface
interface Coffee {
    String getDescription();
    double getCost();
}

// Concrete component
class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() {
        return "Simple Coffee";
    }
    
    @Override
    public double getCost() {
        return 2.0;
    }
}

// Decorator abstract class
abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription();
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost();
    }
}

// Concrete decorators
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Milk";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.5;
    }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Sugar";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.2;
    }
}

// Usage
public class DecoratorDemo {
    public static void main(String[] args) {
        Coffee coffee = new SimpleCoffee();
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
        
        coffee = new MilkDecorator(coffee);
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
        
        coffee = new SugarDecorator(coffee);
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
    }
}
```

---

### 3. Facade Pattern

**Purpose**: Provides a simplified interface to a complex subsystem.

**When to Use**: When you want to provide a simple interface to a complex system.

**Java Example**:
```java
// Complex subsystem classes
class CPU {
    public void freeze() {
        System.out.println("CPU: Freezing...");
    }
    
    public void jump(long position) {
        System.out.println("CPU: Jumping to " + position);
    }
    
    public void execute() {
        System.out.println("CPU: Executing...");
    }
}

class Memory {
    public void load(long position, byte[] data) {
        System.out.println("Memory: Loading data at " + position);
    }
}

class HardDrive {
    public byte[] read(long lba, int size) {
        System.out.println("HardDrive: Reading " + size + " bytes from " + lba);
        return new byte[size];
    }
}

// Facade
class ComputerFacade {
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
    }
    
    public void start() {
        System.out.println("Starting computer...");
        cpu.freeze();
        memory.load(0, hardDrive.read(0, 1024));
        cpu.jump(0);
        cpu.execute();
        System.out.println("Computer started!");
    }
}

// Usage
public class FacadeDemo {
    public static void main(String[] args) {
        ComputerFacade computer = new ComputerFacade();
        computer.start();
    }
}
```

---

### 4. Proxy Pattern

**Purpose**: Provides a placeholder or surrogate for another object to control access to it.

**When to Use**:
- Virtual Proxy - lazy initialization
- Protection Proxy - access control
- Remote Proxy - represents object in different address space

**Java Example**:
```java
// Subject interface
interface Image {
    void display();
}

// Real subject
class RealImage implements Image {
    private String fileName;
    
    public RealImage(String fileName) {
        this.fileName = fileName;
        loadFromDisk();
    }
    
    private void loadFromDisk() {
        System.out.println("Loading " + fileName);
    }
    
    @Override
    public void display() {
        System.out.println("Displaying " + fileName);
    }
}

// Proxy
class ProxyImage implements Image {
    private RealImage realImage;
    private String fileName;
    
    public ProxyImage(String fileName) {
        this.fileName = fileName;
    }
    
    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(fileName);
        }
        realImage.display();
    }
}

// Usage
public class ProxyDemo {
    public static void main(String[] args) {
        Image image = new ProxyImage("test.jpg");
        
        // Image will be loaded from disk
        image.display();
        
        // Image will not be loaded from disk
        image.display();
    }
}
```

---

## Behavioral Patterns

### 1. Observer Pattern

**Purpose**: Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified.

**When to Use**: When changes to one object require changing others, and you don't know how many objects need to change.

**Java Example**:
```java
import java.util.ArrayList;
import java.util.List;

// Subject
class Subject {
    private List<Observer> observers = new ArrayList<>();
    private int state;
    
    public int getState() {
        return state;
    }
    
    public void setState(int state) {
        this.state = state;
        notifyAllObservers();
    }
    
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    public void notifyAllObservers() {
        for (Observer observer : observers) {
            observer.update();
        }
    }
}

// Observer
abstract class Observer {
    protected Subject subject;
    public abstract void update();
}

// Concrete Observers
class BinaryObserver extends Observer {
    public BinaryObserver(Subject subject) {
        this.subject = subject;
        this.subject.attach(this);
    }
    
    @Override
    public void update() {
        System.out.println("Binary: " + 
            Integer.toBinaryString(subject.getState()));
    }
}

class HexObserver extends Observer {
    public HexObserver(Subject subject) {
        this.subject = subject;
        this.subject.attach(this);
    }
    
    @Override
    public void update() {
        System.out.println("Hex: " + 
            Integer.toHexString(subject.getState()).toUpperCase());
    }
}

// Usage
public class ObserverDemo {
    public static void main(String[] args) {
        Subject subject = new Subject();
        
        new HexObserver(subject);
        new BinaryObserver(subject);
        
        System.out.println("First state change: 15");
        subject.setState(15);
        
        System.out.println("Second state change: 10");
        subject.setState(10);
    }
}
```

---

### 2. Strategy Pattern

**Purpose**: Defines a family of algorithms, encapsulates each one, and makes them interchangeable.

**When to Use**: When you have multiple algorithms for a specific task and want to switch between them at runtime.

**Java Example**:
```java
// Strategy interface
interface PaymentStrategy {
    void pay(int amount);
}

// Concrete strategies
class CreditCardStrategy implements PaymentStrategy {
    private String cardNumber;
    private String cvv;
    
    public CreditCardStrategy(String cardNumber, String cvv) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using Credit Card");
    }
}

class PayPalStrategy implements PaymentStrategy {
    private String email;
    
    public PayPalStrategy(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using PayPal");
    }
}

class BitcoinStrategy implements PaymentStrategy {
    private String walletAddress;
    
    public BitcoinStrategy(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using Bitcoin");
    }
}

// Context
class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout(int amount) {
        paymentStrategy.pay(amount);
    }
}

// Usage
public class StrategyDemo {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        
        cart.setPaymentStrategy(new CreditCardStrategy("1234-5678", "123"));
        cart.checkout(100);
        
        cart.setPaymentStrategy(new PayPalStrategy("user@example.com"));
        cart.checkout(200);
    }
}
```

---

### 3. Command Pattern

**Purpose**: Encapsulates a request as an object, allowing you to parameterize clients with different requests, queue requests, and support undoable operations.

**When to Use**: When you want to parameterize objects with operations, queue operations, or implement undo/redo.

**Java Example**:
```java
// Command interface
interface Command {
    void execute();
    void undo();
}

// Receiver
class Light {
    public void on() {
        System.out.println("Light is ON");
    }
    
    public void off() {
        System.out.println("Light is OFF");
    }
}

// Concrete commands
class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.on();
    }
    
    @Override
    public void undo() {
        light.off();
    }
}

class LightOffCommand implements Command {
    private Light light;
    
    public LightOffCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.off();
    }
    
    @Override
    public void undo() {
        light.on();
    }
}

// Invoker
class RemoteControl {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
    }
    
    public void pressUndo() {
        command.undo();
    }
}

// Usage
public class CommandDemo {
    public static void main(String[] args) {
        Light light = new Light();
        Command lightOn = new LightOnCommand(light);
        Command lightOff = new LightOffCommand(light);
        
        RemoteControl remote = new RemoteControl();
        
        remote.setCommand(lightOn);
        remote.pressButton();
        remote.pressUndo();
        
        remote.setCommand(lightOff);
        remote.pressButton();
        remote.pressUndo();
    }
}
```

---

### 4. Template Method Pattern

**Purpose**: Defines the skeleton of an algorithm in a method, deferring some steps to subclasses.

**When to Use**: When you want to let subclasses redefine certain steps of an algorithm without changing its structure.

**Java Example**:
```java
// Abstract class
abstract class DataProcessor {
    // Template method
    public final void process() {
        readData();
        processData();
        saveData();
    }
    
    abstract void readData();
    abstract void processData();
    
    // Hook method with default implementation
    void saveData() {
        System.out.println("Data saved");
    }
}

// Concrete classes
class CSVDataProcessor extends DataProcessor {
    @Override
    void readData() {
        System.out.println("Reading data from CSV file");
    }
    
    @Override
    void processData() {
        System.out.println("Processing CSV data");
    }
}

class XMLDataProcessor extends DataProcessor {
    @Override
    void readData() {
        System.out.println("Reading data from XML file");
    }
    
    @Override
    void processData() {
        System.out.println("Processing XML data");
    }
    
    @Override
    void saveData() {
        System.out.println("Saving XML data to database");
    }
}

// Usage
public class TemplateMethodDemo {
    public static void main(String[] args) {
        DataProcessor csvProcessor = new CSVDataProcessor();
        csvProcessor.process();
        
        System.out.println();
        
        DataProcessor xmlProcessor = new XMLDataProcessor();
        xmlProcessor.process();
    }
}
```

---

### 5. State Pattern

**Purpose**: Allows an object to alter its behavior when its internal state changes.

**When to Use**: When an object's behavior depends on its state and it must change its behavior at runtime.

**Java Example**:
```java
// State interface
interface State {
    void insertCoin();
    void ejectCoin();
    void dispense();
}

// Context
class VendingMachine {
    private State noCoinState;
    private State hasCoinState;
    private State soldState;
    private State currentState;
    
    public VendingMachine() {
        noCoinState = new NoCoinState(this);
        hasCoinState = new HasCoinState(this);
        soldState = new SoldState(this);
        currentState = noCoinState;
    }
    
    public void insertCoin() {
        currentState.insertCoin();
    }
    
    public void ejectCoin() {
        currentState.ejectCoin();
    }
    
    public void dispense() {
        currentState.dispense();
    }
    
    public void setState(State state) {
        this.currentState = state;
    }
    
    public State getNoCoinState() { return noCoinState; }
    public State getHasCoinState() { return hasCoinState; }
    public State getSoldState() { return soldState; }
}

// Concrete states
class NoCoinState implements State {
    private VendingMachine machine;
    
    public NoCoinState(VendingMachine machine) {
        this.machine = machine;
    }
    
    @Override
    public void insertCoin() {
        System.out.println("Coin inserted");
        machine.setState(machine.getHasCoinState());
    }
    
    @Override
    public void ejectCoin() {
        System.out.println("No coin to eject");
    }
    
    @Override
    public void dispense() {
        System.out.println("Insert coin first");
    }
}

class HasCoinState implements State {
    private VendingMachine machine;
    
    public HasCoinState(VendingMachine machine) {
        this.machine = machine;
    }
    
    @Override
    public void insertCoin() {
        System.out.println("Coin already inserted");
    }
    
    @Override
    public void ejectCoin() {
        System.out.println("Coin ejected");
        machine.setState(machine.getNoCoinState());
    }
    
    @Override
    public void dispense() {
        System.out.println("Dispensing product");
        machine.setState(machine.getSoldState());
    }
}

class SoldState implements State {
    private VendingMachine machine;
    
    public SoldState(VendingMachine machine) {
        this.machine = machine;
    }
    
    @Override
    public void insertCoin() {
        System.out.println("Please wait, dispensing product");
    }
    
    @Override
    public void ejectCoin() {
        System.out.println("Cannot eject, already dispensing");
    }
    
    @Override
    public void dispense() {
        System.out.println("Product dispensed");
        machine.setState(machine.getNoCoinState());
    }
}
```

---

## Interview Questions

### 1. What are Design Patterns? Why are they important?

**Answer**: Design patterns are reusable solutions to commonly occurring problems in software design. They represent best practices and proven solutions that have been tested and refined over time.

**Importance**:
- **Reusability**: Solve common problems without reinventing the wheel
- **Communication**: Provide a common vocabulary for developers
- **Maintainability**: Make code easier to understand and maintain
- **Best Practices**: Incorporate proven design principles
- **Scalability**: Help create flexible and extensible systems

---

### 2. Explain the difference between Factory and Abstract Factory patterns.

**Answer**:

**Factory Pattern**:
- Creates objects of a single type/family
- Uses a single factory class with a method that returns different objects based on input
- Example: ShapeFactory creates Circle, Rectangle, or Square

**Abstract Factory Pattern**:
- Creates families of related objects
- Uses multiple factory classes, each creating a family of related objects
- Example: GUIFactory creates different UI components (Button, Checkbox) for different platforms (Windows, Mac)

**Key Difference**: Factory creates one product type, Abstract Factory creates multiple related product types.

---

### 3. When would you use the Singleton pattern? What are its disadvantages?

**Answer**:

**Use Cases**:
- Database connections
- Configuration managers
- Logging services
- Thread pools
- Caches

**Disadvantages**:
- **Testing**: Hard to unit test due to global state
- **Tight Coupling**: Creates hidden dependencies
- **Thread Safety**: Requires careful implementation for multi-threading
- **Single Responsibility Violation**: Controls both its creation and functionality
- **Hidden Dependencies**: Makes dependencies less explicit

**Best Practice**: Consider dependency injection instead of Singleton when possible.

---

### 4. Explain the Decorator pattern with a real-world example.

**Answer**:

The Decorator pattern allows adding new functionality to objects dynamically without altering their structure.

**Real-World Example**: Coffee shop ordering system
- Base: Simple Coffee ($2)
- Decorators: Milk (+$0.50), Sugar (+$0.20), Whipped Cream (+$0.70)
- You can combine decorators: Coffee + Milk + Sugar = $2.70

**Benefits**:
- Flexible alternative to subclassing
- Add responsibilities dynamically
- Avoid class explosion (instead of MilkCoffee, SugarCoffee, MilkSugarCoffee classes)
- Follow Open/Closed Principle

**Java Example**: `java.io` package (BufferedReader wraps FileReader wraps InputStreamReader)

---

### 5. What is the Observer pattern? Give an example.

**Answer**:

The Observer pattern defines a one-to-many dependency where when one object (subject) changes state, all its dependents (observers) are automatically notified.

**Components**:
- **Subject**: Maintains list of observers and notifies them of changes
- **Observer**: Interface for objects that should be notified
- **ConcreteObserver**: Implements Observer interface

**Real-World Examples**:
- Event handling systems (GUI buttons)
- News subscriptions
- Stock price updates
- Social media notifications
- MVC pattern (Model notifies View)

**Java Implementation**: `java.util.Observable` and `java.util.Observer` (deprecated in Java 9)

---

### 6. Difference between Strategy and State patterns?

**Answer**:

Both patterns appear similar but serve different purposes:

**Strategy Pattern**:
- **Purpose**: Choose algorithm/behavior from a family of algorithms
- **Client**: Knows about different strategies and chooses one
- **Example**: Payment methods (CreditCard, PayPal, Bitcoin)
- **Transitions**: Client changes strategy explicitly

**State Pattern**:
- **Purpose**: Change behavior based on internal state
- **Client**: Doesn't know about states; object manages its own state
- **Example**: Vending machine states (NoCoin, HasCoin, Sold)
- **Transitions**: Object transitions between states automatically

**Key Difference**: Strategy is about choosing different algorithms; State is about object behavior changing based on state.

---

### 7. What is the difference between Adapter and Facade patterns?

**Answer**:

**Adapter Pattern**:
- **Purpose**: Convert one interface to another
- **Scope**: Works with a single class
- **Intent**: Make incompatible interfaces compatible
- **Example**: MediaAdapter converts AdvancedMediaPlayer to MediaPlayer interface

**Facade Pattern**:
- **Purpose**: Provide simplified interface to complex subsystem
- **Scope**: Works with multiple classes/subsystems
- **Intent**: Simplify complex system interaction
- **Example**: ComputerFacade simplifies starting computer (CPU, Memory, HardDrive)

**Key Difference**: Adapter makes two interfaces work together; Facade simplifies multiple interfaces.

---

### 8. Explain the Builder pattern. When should you use it?

**Answer**:

The Builder pattern constructs complex objects step by step, separating construction from representation.

**When to Use**:
- Object has many parameters (especially optional ones)
- Want immutable objects
- Construction process is complex
- Avoid telescoping constructors

**Benefits**:
- Fluent interface (method chaining)
- Immutable objects
- Clear separation of construction
- Flexible object creation

**Example**:
```java
User user = new User.UserBuilder("John", "Doe")
    .age(30)
    .phone("123-456-7890")
    .address("123 Main St")
    .build();
```

**Real-World**: StringBuilder, Stream builders, Lombok @Builder annotation

---

### 9. What is the Template Method pattern?

**Answer**:

The Template Method pattern defines the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the algorithm's structure.

**Components**:
- **Abstract Class**: Defines template method and abstract steps
- **Concrete Classes**: Implement the abstract steps
- **Template Method**: Final method that defines algorithm structure

**Example**: Data processing pipeline
1. Read data (varies by source)
2. Process data (varies by format)
3. Save data (common or can vary)

**Benefits**:
- Code reuse
- Control algorithm structure
- Follow Hollywood Principle ("Don't call us, we'll call you")

**Real-World**: Java Collections `sort()`, Spring Template classes (JdbcTemplate, RestTemplate)

---

### 10. Explain the Proxy pattern and its types.

**Answer**:

The Proxy pattern provides a surrogate or placeholder for another object to control access to it.

**Types**:

1. **Virtual Proxy**: Lazy initialization
   - Example: Loading large images only when needed

2. **Protection Proxy**: Access control
   - Example: Checking permissions before method execution

3. **Remote Proxy**: Represents object in different address space
   - Example: RMI stubs, REST API clients

4. **Caching Proxy**: Cache results
   - Example: Database query results

5. **Smart Proxy**: Additional functionality
   - Example: Reference counting, logging

**Real-World**: Spring AOP proxies, Hibernate lazy loading, Java RMI

---

### 11. What are the SOLID principles? How do design patterns relate to them?

**Answer**:

**SOLID Principles**:

1. **S**ingle Responsibility Principle: Class should have one reason to change
   - Patterns: Decorator, Facade

2. **O**pen/Closed Principle: Open for extension, closed for modification
   - Patterns: Strategy, Decorator, Template Method

3. **L**iskov Substitution Principle: Subtypes must be substitutable for base types
   - Patterns: Most patterns follow this

4. **I**nterface Segregation Principle: Many specific interfaces are better than one general
   - Patterns: Adapter, Bridge

5. **D**ependency Inversion Principle: Depend on abstractions, not concretions
   - Patterns: Factory, Abstract Factory, Strategy

**Relationship**: Design patterns are practical applications of SOLID principles, providing concrete implementations that follow these principles.

---

### 12. What is the Command pattern? Give a practical example.

**Answer**:

The Command pattern encapsulates a request as an object, allowing parameterization of clients with different requests, queuing, logging, and undo/redo operations.

**Components**:
- **Command**: Interface with execute() method
- **ConcreteCommand**: Implements command and holds receiver
- **Invoker**: Triggers command
- **Receiver**: Knows how to perform operations

**Practical Example**: Text Editor
- Commands: Cut, Copy, Paste, Undo, Redo
- Each command is an object that can be stored in history
- Undo stack maintains command history

**Benefits**:
- Decouples sender and receiver
- Supports undo/redo
- Command queuing
- Macro commands (composite commands)

**Real-World**: GUI buttons, transaction systems, job schedulers

---

### 13. How does the Prototype pattern differ from cloning?

**Answer**:

**Prototype Pattern**:
- Design pattern for creating new objects by copying existing ones
- Uses cloning as implementation mechanism
- Involves pattern structure (prototype registry, concrete prototypes)
- Focuses on object creation strategy

**Cloning**:
- Just the mechanism (shallow or deep copy)
- Java's `Cloneable` interface and `clone()` method
- Technical implementation detail

**Key Point**: Prototype is the pattern; cloning is the technique. The pattern adds structure like prototype caches, factories, and management of prototypes.

**Use Case**: When object creation is expensive (database entities, configuration objects)

---

### 14. Explain the difference between Composition and Inheritance in design patterns.

**Answer**:

**Inheritance** ("is-a" relationship):
- Tight coupling
- Defined at compile time
- Cannot change at runtime
- Example: Dog is-a Animal

**Composition** ("has-a" relationship):
- Loose coupling
- Can change at runtime
- More flexible
- Example: Car has-a Engine

**Design Pattern Perspective**:
- **Favor Composition over Inheritance**
- Patterns using composition: Strategy, Decorator, Proxy, Bridge
- Better for: Flexibility, testing, avoiding fragile base class problem

**Example**:
```java
// Inheritance (rigid)
class SportsCar extends Car { }

// Composition (flexible)
class Car {
    private Engine engine;
    public void setEngine(Engine engine) {
        this.engine = engine;
    }
}
```

---

### 15. What is the difference between Abstract Factory and Factory Method patterns?

**Answer**:

**Factory Method**:
- Uses inheritance
- Defines an interface for creating object but lets subclasses decide
- Single product creation
- One factory method
- Example: Creator class with factoryMethod() overridden by subclasses

**Abstract Factory**:
- Uses composition
- Provides interface for creating families of related objects
- Multiple product creation
- Multiple factory methods
- Example: GUIFactory with createButton(), createCheckbox()

**When to Use**:
- **Factory Method**: When you need to defer instantiation to subclasses
- **Abstract Factory**: When you need to create families of related objects

---

### 16. How would you implement thread-safe Singleton?

**Answer**:

**Options**:

1. **Eager Initialization** (Thread-safe but not lazy):
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

2. **Double-Checked Locking** (Lazy and thread-safe):
```java
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

3. **Bill Pugh Singleton** (Best practice):
```java
public class Singleton {
    private Singleton() {}
    
    private static class SingletonHelper {
        private static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}
```

4. **Enum Singleton** (Simplest and safest):
```java
public enum Singleton {
    INSTANCE;
}
```

**Best Choice**: Enum for simplicity; Bill Pugh for traditional approach.

---

### 17. Can you break the Singleton pattern? How?

**Answer**:

Yes, Singleton can be broken in several ways:

1. **Reflection**:
```java
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton instance = constructor.newInstance();
```

2. **Serialization**:
```java
// Serialize and deserialize creates new instance
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
out.writeObject(Singleton.getInstance());

ObjectInputStream in = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton instance = (Singleton) in.readObject(); // Different instance!
```

3. **Cloning**: If Cloneable is implemented

**Prevention**:

1. **Against Reflection**: Throw exception in constructor if instance exists
2. **Against Serialization**: Implement readResolve()
```java
protected Object readResolve() {
    return getInstance();
}
```
3. **Against Cloning**: Don't implement Cloneable or override clone() to throw exception
4. **Best Solution**: Use Enum (immune to all attacks)

---

### 18. What are Anti-patterns? Give examples.

**Answer**:

Anti-patterns are common responses to recurring problems that are ineffective and counterproductive.

**Examples**:

1. **God Object**: One class does everything
   - Solution: Single Responsibility Principle

2. **Spaghetti Code**: No structure, hard to follow
   - Solution: Apply proper design patterns

3. **Golden Hammer**: Using same solution for every problem
   - Solution: Choose appropriate pattern for each problem

4. **Lava Flow**: Dead code that nobody dares to remove
   - Solution: Regular refactoring and code reviews

5. **Copy-Paste Programming**: Duplicating code instead of reusing
   - Solution: DRY principle, extract methods/classes

6. **Premature Optimization**: Optimizing before it's necessary
   - Solution: "Make it work, make it right, make it fast"

---

### 19. What is Dependency Injection? Is it a design pattern?

**Answer**:

**Dependency Injection (DI)** is a technique where an object receives its dependencies from external sources rather than creating them itself.

**Is it a Pattern?**: It's a specialized form of the **Strategy** or **Inversion of Control** pattern, but often considered a principle rather than a pattern.

**Types**:

1. **Constructor Injection**:
```java
public class UserService {
    private UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

2. **Setter Injection**:
```java
public class UserService {
    private UserRepository repository;
    
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}
```

3. **Interface Injection**: Less common

**Benefits**:
- Loose coupling
- Easier testing (mock dependencies)
- Better maintainability
- Follows Dependency Inversion Principle

**Frameworks**: Spring, Guice, CDI

---

### 20. How do you choose which design pattern to use?

**Answer**:

**Decision Process**:

1. **Identify the Problem**:
   - Is it about object creation? → Creational patterns
   - Is it about object composition? → Structural patterns
   - Is it about object interaction? → Behavioral patterns

2. **Consider Specific Needs**:
   - Need flexible object creation? → Factory/Builder
   - Need single instance? → Singleton
   - Need to add functionality dynamically? → Decorator
   - Need to simplify complex system? → Facade
   - Need to notify multiple objects? → Observer
   - Need interchangeable algorithms? → Strategy

3. **Evaluate Trade-offs**:
   - Complexity vs. benefits
   - Current needs vs. future flexibility
   - Team familiarity

4. **Don't Force Patterns**:
   - Solve problem first, apply pattern if it helps
   - Avoid over-engineering
   - Simple solution is often better

5. **Consider Existing Code**:
   - Consistency with codebase
   - Team conventions
   - Framework patterns

**Golden Rule**: Use patterns to solve problems, not to show you know patterns.

---

## Quick Reference Table

| Pattern | Category | Use Case |
|---------|----------|----------|
| Singleton | Creational | Single instance needed |
| Factory | Creational | Create objects without specifying exact class |
| Abstract Factory | Creational | Create families of related objects |
| Builder | Creational | Construct complex objects step-by-step |
| Prototype | Creational | Clone objects instead of creating new |
| Adapter | Structural | Make incompatible interfaces work together |
| Decorator | Structural | Add functionality dynamically |
| Facade | Structural | Simplify complex subsystem |
| Proxy | Structural | Control access to another object |
| Observer | Behavioral | One-to-many notification |
| Strategy | Behavioral | Interchangeable algorithms |
| Command | Behavioral | Encapsulate requests as objects |
| Template Method | Behavioral | Define algorithm skeleton |
| State | Behavioral | Alter behavior based on state |

---

## Key Takeaways

1. **Design patterns are tools, not rules** - Use them when they solve your problem
2. **SOLID principles** guide pattern usage
3. **Composition over inheritance** in most cases
4. **Keep it simple** - Don't over-engineer
5. **Patterns work together** - Often combine multiple patterns
6. **Context matters** - Same pattern can be implemented differently
7. **Learn by doing** - Practice implementing patterns in real projects