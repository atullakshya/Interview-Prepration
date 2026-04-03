# C# OOP Intermediate to Senior Interview Questions

## Encapsulation
1. **What is encapsulation in C#?**  
   Encapsulation is the mechanism of restricting access to certain details of an object, enabling a controlled interface to the user's interaction with the object.

2. **How do you implement encapsulation in C#?**  
   Use access modifiers (private, public, protected, internal) to restrict access to class members and expose necessary functionalities through methods.

## Inheritance
1. **What is inheritance?**  
   Inheritance is a fundamental principle of OOP that allows a class to inherit properties and behaviors from another class.

2. **What is the difference between a class and an interface in C#?**  
   A class can provide implementation for its methods while an interface can only declare methods (aside from default interface methods in C# 8.0).

## Polymorphism
1. **What is polymorphism and how is it achieved in C#?**  
   Polymorphism allows methods to do different things based on the object it is acting upon, achieved through method overloading and overriding.

2. **Explain method overriding in C#.**  
   Method overriding allows a derived class to provide a specific implementation of a method that is already defined in its base class using the `override` keyword.

## Abstraction
1. **What is abstraction in C#?**  
   Abstraction is the concept of hiding the complex reality while exposing only the necessary parts of an object, typically implemented using abstract classes and interfaces.

2. **Can you give an example of an abstract class in C#?**  
   ```csharp  
   public abstract class Shape  
   {  
       public abstract double Area();  
   }  
   ```

## SOLID Principles
1. **What does SOLID stand for?**  
   - S: Single Responsibility Principle  
   - O: Open/Closed Principle  
   - L: Liskov Substitution Principle  
   - I: Interface Segregation Principle  
   - D: Dependency Inversion Principle

2. **Explain the Single Responsibility Principle.**  
   A class should have one and only one reason to change, meaning it should have only one job or responsibility.

## Design Patterns
1. **What is a design pattern?**  
   A design pattern is a standard solution to a common problem in software design.

2. **Can you explain the Singleton pattern?**  
   The Singleton pattern ensures a class has only one instance and provides a global point of access to it.  
   ```csharp  
   public sealed class Singleton  
   {  
       private static readonly Singleton instance = new Singleton();  
       private Singleton() {}  
       public static Singleton Instance => instance;  
   }  
   ```

## Real-World Examples
1. **How is encapsulation used in a banking system?**  
   In a banking application, sensitive data like an account balance can be encapsulated within a class and manipulated through methods like `Deposit` and `Withdraw` that ensure data integrity.

2. **Provide an example of inheritance using vehicles.**  
   ```csharp  
   public class Vehicle  
   {  
       public void Start() { ... }  
   }  
   public class Car : Vehicle  
   {  
       public void OpenTrunk() { ... }  
   }  
   ```  
   Here `Car` inherits the `Start` method from `Vehicle`.
