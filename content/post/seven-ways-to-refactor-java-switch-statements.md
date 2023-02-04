---
title: "Seven Ways to Refactor Java Switch Statements"
date: 2023-02-04T21:16:50+08:00
draft: false
---
最近在浏览V2ex的时候发现一个帖子很有趣，[Java 代码 switch 分支过多，怎么改写比较优雅呢？](https://www.v2ex.com/t/911506)，在留言里面发现一篇文章，觉得挺好，转载在这。

Is a well-known fact that switch statements and SOLID principles—Single Responsibility Principle and Open-Closed Principle—are not good friends and usually we can choose better alternatives than switch. This is especially true when we deal with switch nested in large methods, interdependent switches and large switches (with many instructions under cases and/or many case branches).

In this article, we will pick up some switch examples, and we will try to provide several alternatives that eliminate or hide the switch statements.

## 1. Implementing the Strategy Pattern via Java Enum

### Application Name: [SwitchToStrategyEnum](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToStrategyEnum)

A typical case involves the existence of a Java enum and one (or more) switch statements based on it. Let’s suppose that we have the following simple Java enum:

```java
public enum PlayerTypes { TENNIS,
   FOOTBALL, SNOOKER
}
```

And, the following switch statement that is used to create different types of players:

```java
public class PlayerCreator {
   public Player createPlayer(PlayerTypes playerType)
         { switch (playerType) {
      case TENNIS:
         return new TennisPlayer();
      case FOOTBALL:
         return new FootballPlayer();
      case SNOOKER:
         return new SnookerPlayer();

      default:
         throw new IllegalArgumentException("Invalid player type: "
            + playerType);
      }
   }
}
```

Creating a tennis player, PlayerTypes.TENNIS:

```java
PlayerCreator playerCreator = new PlayerCreator();
Player tennisPlayer =
   playerCreator.createPlayer(PlayerTypes.TENNIS);
```

### Rely on Enum Type with Constant-specific Method Implementation

But, there is a better way to link a behavior with each enum constant. This way is known as “enum type with constant-specific method implementation,” and it is described in Effective Java, 2nd Edition, by Joshua Bloch. Following the idea, we can enrich our PlayerTypes with an abstract method, and for each value, we provide an implementation, as follows:

```java
public enum PlayerTypes { TENNIS {
   @Override
   public Player createPlayer() {
      return new TennisPlayer();
   }
},
FOOTBALL {
   @Override
   public Player createPlayer() {
      return new FootballPlayer();
   }
},
SNOOKER {
   @Override
      public Player createPlayer() {
         return new SnookerPlayer();
      }
   };

   public abstract Player createPlayer();
}
```

Creating a football player, PlayerTypes.FOOTBALL:

```java
Player footballPlayer =
   PlayerTypes.valueOf("FOOTBALL").createPlayer();
```

## 2. Implementing the Command Pattern

### Application Name: [SwitchToCommand](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToCommand)

This time, let’s write the same switch logic, but based on a string value. In Java 7+, we can use a String object in the expression of a switch statement. After all, most Java compilers will generate more efficient bytecode for this implementation than for an if-else-if chain. Well, this is not so bad 🙂 But, a switch is still involved, and the drawbacks remain the same.

```java
public class PlayerCreator {

   public Player createPlayer(String playerType) {
      switch (playerType) {
         case "TENNIS":
            return new TennisPlayer();
         case "FOOTBALL":
            return new FootballPlayer();
         case "SNOOKER":
            return new SnookerPlayer();

         default:
            throw new IllegalArgumentException
               ("Invalid player type: " + playerType);
      }
   }

}
```

Creating a tennis player, "TENNIS":

```java
PlayerCreator playerCreator = new PlayerCreator();
Player tennisPlayer = playerCreator.createPlayer("TENNIS");
```

### Implementing the Command Design Pattern

Alternatively, we can rely on the “Command” design pattern. We can shape this design pattern in two steps. First, we define an interface:

```java
public interface Command {

   Player create();
}
```

Second, we provide implementations of this interface for each player type:

```java
public class CreatePlayerCommand {

   private static final Map<String, Command> PLAYERS;

   static {
      final Map<String, Command> players = new HashMap<>();
      players.put("TENNIS", new Command() {
         @Override
         public Player create() {
            return new TennisPlayer();
         }
      });

      players.put("FOOTBALL", new Command() {
         @Override
         public Player create() {
            return new FootballPlayer();
         }
      });

      players.put("SNOOKER", new Command() {
         @Override
         public Player create() {
            return new SnookerPlayer();
         }
      });

      PLAYERS = Collections.unmodifiableMap(players);
   }

   public Player createPlayer(String playerType) {
      Command command = PLAYERS.get(playerType);

      if (command == null) {
         throw new IllegalArgumentException("Invalid player type: "
            + playerType);
      }

      return command.create();
   }

}
```

Creating a snooker player:

```java
CreatePlayerCommand createCommand = new CreatePlayerCommand();
Player snookerPlayer = createCommand.createPlayer("SNOOKER");
```

## 3. Using the Java 8+ Supplier

### Application Name: [SwitchToSupplier](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToSupplier)

Transforming a switch statement into a Map is a common approach. The idea is to place each case as a value in the Map and use the case‘s condition as a key. Starting with Java 8+, we can take advantage of a Supplier and constructor reference as well. So, let’s create a Map containing references to constructors using their names and the keyword new:

```java
public class PlayerSupplier {
   private static final Map<String, Supplier<Player>>
      PLAYER_SUPPLIER;

   static {
      final Map<String, Supplier<Player>>
         players = new HashMap<>();
      players.put("TENNIS", TennisPlayer::new);
      players.put("FOOTBALL", FootballPlayer::new);
      players.put("SNOOKER", SnookerPlayer::new);

      PLAYER_SUPPLIER = Collections.unmodifiableMap(players);
   }

   public Player supplyPlayer(String playerType) {
      Supplier<Player> player = PLAYER_SUPPLIER.get(playerType);

      if (player == null) {
         throw new IllegalArgumentException("Invalid player type: "
            + playerType);
      }
      return player.get();
   }
}
```

Creating a snooker player:

```java
PlayerSupplier playerSupplier = new PlayerSupplier();
Player snookerPlayer = playerSupplier.supplyPlayer("SNOOKER");
```

## 4. Defining a Custom Functional Interface

### Application Name: [SwitchToTriFunction](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToTriFunction)

An implementation almost similar with the one from Point 3 can be used for cases as shown next:

```java
public final class ComputeTennisPlayerStatistics {

   private ComputeTennisPlayerStatistics() {
      throw new AssertionError();
   }

   public static String computeTrend(TennisPlayer tennisPlayer,
         Period period, String owner, String trend) {
      switch (trend) {
         case "SERVE":
            return Statistics.computeServeTrend(tennisPlayer,
               period, owner);
         case "FOREHAND":
            return Statistics.computeForehandTrend(tennisPlayer,
               period, owner);
         case "BACKHAND":
            return Statistics.computeBackhandTrend(tennisPlayer,
               period, owner);

         default:
            throw new IllegalArgumentException
               ("Invalid trend attribute: " + trend);
      }
   }

}
```

Getting the SERVE trend (we use dummy arguments, because they are not relevant):

```java
String serveTrend = ComputeTennisPlayerStatistics.
computeTrend(new TennisPlayer(), Period.ZERO, "TENNIS MAGAZINE",
   "SERVE");
```

### Implementing the TriFunction Functional Interface

This time, each case invokes a static method that receives three arguments and returns a String. In such cases, a Supplier is not helpful. Because we have more than two arguments, we cannot rely on BiFunction either. An approach here will consist in defining our own functional interface and, afterward, use a Map as we did earlier.

```java
@FunctionalInterface
public interface TriFunction<T, U, V, R> {

   R apply(T t, U u, V v);
}

public final class FunctionalStatistics {

   private FunctionalStatistics() {
      throw new AssertionError();
   }

   private static final Map<String, TriFunction<TennisPlayer,
         Period, String, String>>
      STATISTICS = new HashMap<>();

   static {
      STATISTICS.put("SERVE", Statistics::computeServeTrend);
      STATISTICS.put("FOREHAND", Statistics::computeForehandTrend);
      STATISTICS.put("BACKHAND", Statistics::computeBackhandTrend);
   }

   public static String computeTrend(TennisPlayer tennisPlayer,
      Period period, String owner, String trend) {
      TriFunction<TennisPlayer, Period, String, String>
            function = STATISTICS.get(trend);

      if (function == null) {
         throw new IllegalArgumentException("Invalid trend type: "
            + trend);
         }

      return function.apply(tennisPlayer, period, owner);
   }
}
```

Getting the FOREHAND trend (we use dummy arguments, because they are not relevant):

```java
String forehandTrend = FunctionalStatistics.
computeTrend(new TennisPlayer(), Period.ZERO, "SPORT TV",
   "FOREHAND");
```

## 5. Relying on Abstract Factory

### Application Name: [SwitchToAbstractFactory](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToAbstractFactory), [SwitchToPolymorphism](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToPolymorphism)

In this section, let’s follow the Clean Code book by Robert C. Martin. We start with a switch that can be “hidden” via the Abstract Factory design pattern.

```java
public class ClassicPlayer {

   private final Type type;
   private final int delta;

   public ClassicPlayer(Type type, int delta) {
      this.type = type;
      this.delta = delta;
   }

   public Type getType() {
      return type;
   }

   public int getDelta() {
      return delta;
   }

}

public class Statistics {

   public int playerEndurance(ClassicPlayer player) {

      int delta = player.getDelta();

      switch (player.getType()) {
         case TENNIS:
            return ComputeEnduranceAlgorithm.basicEndurance(delta)
               + ComputeEnduranceAlgorithm.hardEndurance(delta);
         case FOOTBALL:
            return ComputeEnduranceAlgorithm.hardEndurance(delta)
               * ComputeEnduranceAlgorithm.factorEndurance(delta);
         case SNOOKER:
            return ComputeEnduranceAlgorithm.basicEndurance(delta);

         default:
            throw new IllegalArgumentException
               ("Invalid player type: " + player.getType());
      }
   }

}
```

Computing endurance for a tennis player:

```java
Statistics statistics = new Statistics();

ClassicPlayer player = new ClassicPlayer(Type.TENNIS, 54);
int tennisPlayerEndurance = statistics.playerEndurance(player);
```

Well, this is a messy implementation! Imagine that you need to add another method for computing the reaction speed for a player. This will require another switch statement, or probably you’ll nest more ifs in each case and rename the above method something as playerStatistics. Each time a new method will need to be added, this code needs to be adapted accordingly.

### Implementing the Abstract Factory Pattern

We can re-write this code by using polymorphism and an implementation of Abstract Factory design pattern. To start, we drop the ClasssicPlayer class and create an abstract Player, as follows:

```java
public abstract class Player {

   private final Type type;
   private final int delta;

   public Player(Type type, int delta) {
      this.type = type;
      this.delta = delta;
   }

   public Type getType() {
      return type;
   }

   public int getDelta() {
      return delta;
   }

   public abstract int playerEndurance();

   // More similar methods
}
```

Now, we can implement a football, tennis, and snooker player. For example, the SnookerPlayer can be the following:

```java
public class SnookerPlayer extends Player {

   public SnookerPlayer(Type type, int delta) {
      super(type, delta);
   }

   @Override
   public int playerEndurance() {
      return ComputeEnduranceAlgorithm.basicEndurance
         (this.getDelta());
   }
}
```

Further, we define the PlayerFactory interface:

```java
public interface AbstractPlayerFactory {

   public Player createPlayer(Type type, int delta);
}
```

Finally, we “bury” the switch in the implementation of this interface, as follows:

```java
public class PlayerFactory implements AbstractPlayerFactory {

   @Override
   public Player createPlayer(Type type, int delta) {
      switch (type) {
         case TENNIS:
            return new TennisPlayer(type, delta);
         case FOOTBALL:
            return new FootballPlayer(type, delta);
         case SNOOKER:
            return new SnookerPlayer(type, delta);

         default:
         throw new IllegalArgumentException("Invalid player type: "
            + type);
      }
   }

}
```

Computing endurance for a snooker player:

```java
PlayerFactory playerFactory = new PlayerFactory();
Player snookerPlayer = playerFactory.createPlayer(Type.SNOOKER, 8);
int snookerPlayerEndurance = snookerPlayer.playerEndurance();
```

Or, we can instantiate the right Player class directly, and drop the switch:

```java
SnookerPlayer snookerPlayer = new SnookerPlayer(7);
int snookerPlayerEndurance  = snookerPlayer.playerEndurance();
```

## 6. Implementing a State Pattern

### Application Name: [SwitchToState](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToState)

Let’s suppose that we have the following two interdependent switches:

```java
public class ClassicPlayer {

   private int state;

   public void register() {
      switch (state) {
         case 0:
         state = 1; System.out.println("Registering ...");
         break;
      default:
         System.out.println("Aready Registered ...");
      }
   }

   public void unregister() {
      switch (state) {
         case 1:
            state = 0;
            System.out.println("Un-registering ...");
            break;
         default:
            System.out.println("Aready Unregistered ...");
      }
   }
}
```

For brevity, in this example, we have only two interdependent switch structures, and we can call them as in the following example:

```java
ClassicPlayer classicPlayer = new ClassicPlayer();
classicPlayer.register();
classicPlayer.unregister();
// Causes "Already Unregistered ..." message
classicPlayer.unregister();
```

### Implementing a State Pattern

Further, let’s apply the State design pattern to eliminate these switch statements. We start with a simple interface meant to define a contract for our states (actions), register and unregister:

```java
public interface PlayerState {

   void register();
   void unregister();
}
```

Next, we define a Player class that implements the PlayerState interface:

```java
public class Player implements PlayerState {

   private PlayerState registered;
   private PlayerState unregistered;

   private PlayerState state;

   public Player() {
      this.registered = new PlayerRegister(this);
      this.unregistered = new PlayerUnregister(this);

      this.state = this.unregistered;
   }

   @Override
   public void register() {
      state.register();
   }

   @Override
   public void unregister() {
      state.unregister();
   }

   public PlayerState getRegistered() {
      return registered;
   }

   public void setRegistered(PlayerState registered) {
      this.registered = registered;
   }

   public PlayerState getUnregistered() {
      return unregistered;
   }

   public void setUnregistered(PlayerState unregistered) {
      this.unregistered = unregistered;
   }

   public PlayerState getState() {
      return state;
   }

   public void setState(PlayerState state) {
      this.state = state;
   }

}
```

The PlayerRegister and PlayerUnregister code are listed below:

```java
public class PlayerRegister implements PlayerState {

   private final Player player;

   public PlayerRegister(Player player) {
      this.player = player;
   }

   @Override
   public void register() {
      System.out.println("Already Registered ...");
   }

   @Override
   public void unregister() {
      System.out.println("Unregistering ...");
      player.setState(player.getUnregistered());
   }

}

public class PlayerUnregister implements PlayerState {

   private final Player player;

   public PlayerUnregister(Player player) {
      this.player = player;
   }

   @Override
   public void register() {
      System.out.println("Registering ...");
      player.setState(player.getRegistered());
   }

   @Override
   public void unregister() {
      System.out.println("Already Unregistered ...");
   }

}
```

Now, we can create a Player and “play” with the state:

```java
Player player = new Player();
player.register();
player.unregister();
// Causes an "Already Unregistered ..." message
player.unregister();
```

## Implementing via Predicate

### Application Name: [SwitchToPredicate](https://github.com/AnghelLeonard/Refactor/tree/master/Switch/SwitchToPredicate)

In this final example, let’s suppose a switch that contains if branches in cases, as below:

```java
public class PlayerCreator {

   public Player createPlayer(String playerType, int rank) {
      switch (playerType) {
         case "TENNIS":
            if (rank == 1) {
               return new TennisPlayer("Rafael Nadal");
            }
            if (rank > 1 && rank < 5) {
               return new TennisPlayer("Roger Federer");
            }
            if (rank >= 5 && rank <= 10) {
               return new TennisPlayer("Andy Murray");
            }

         case "FOOTBALL":
            if (rank == 1 || rank == 2) {
               return new FootballPlayer("Lionel Messi");
            }
            if (rank > 2 && rank <= 10) {
               return new FootballPlayer("Cristiano Ronaldo");
            }

         case "SNOOKER":
            if (rank == 1) {
               return new SnookerPlayer("Ronnie O'Sullivan");
            }
            if (rank == 2) {
               return new SnookerPlayer("Mark Selby");
            }
            if (rank > 3 && rank < 7) {
               return new SnookerPlayer("John Higgins");
            }
            if (rank >= 7 && rank <= 10) {
               return new SnookerPlayer("Neil Robertson");
            }

         default:
            throw new IllegalArgumentException
               ("Invalid player type: " + playerType);
      }
   }

}
```

Obtaining the output, Tennis player: Andy Murray:

```java
PlayerCreator playerCreator = new PlayerCreator();
Player tennisPlayer = playerCreator.createPlayer("TENNIS", 5);
```

If we assume that the if statements can be considered Predicate<Integer>, we can reshape this in an utility class, as follows:

```java
public final class PlayerSupplier {

   private PlayerSupplier() {
      throw new AssertionError();
   }

   private static final Map<String,
      Map<Predicate<Integer>,
      Supplier<Player>>> PLAYER_CREATOR;

   static {
      final Map<String, Map<Predicate<Integer>,
         Supplier<Player>>> playerCreator = new HashMap<>();

      final Map<Predicate<Integer>,
         Supplier<Player>> tennisPlayers = new HashMap<>();
      tennisPlayers.put(rank -> rank == 1, () ->
         new TennisPlayer("Rafael Nadal"));
      tennisPlayers.put(rank -> rank > 1 && rank < 5, () ->
         new TennisPlayer("Roger Federer"));
      tennisPlayers.put(rank -> rank >= 5 && rank <= 10, () ->
         new TennisPlayer("Andy Murray"));

      final Map<Predicate<Integer>, Supplier<Player>>
         footballPlayers = new HashMap<>();
      footballPlayers.put(rank -> rank == 1 || rank == 2,()->
         new TennisPlayer("Lionel Messsi"));
      footballPlayers.put(rank -> rank > 2 && rank <= 10,()->
         new TennisPlayer("Cristiano Ronaldo"));

      final Map<Predicate<Integer>, Supplier<Player>>
         snookerPlayers = new HashMap<>();
      snookerPlayers.put(rank -> rank == 1, () ->
         new TennisPlayer("Ronnie O'Sullivan"));
      snookerPlayers.put(rank -> rank == 2, () ->
         new TennisPlayer("Mark Selby"));
      snookerPlayers.put(rank -> rank > 3 && rank < 7, () ->
         new TennisPlayer("John Higgins"));
      snookerPlayers.put(rank -> rank >= 7 && rank <= 10, () ->
         new TennisPlayer("Neil Robertson"));

      playerCreator.put("TENNIS", tennisPlayers);
      playerCreator.put("FOOTBALL", footballPlayers);
      playerCreator.put("SNOOKER", snookerPlayers);

      PLAYER_CREATOR = Collections.unmodifiableMap(playerCreator);
   }

   public static final Player supplyPlayer(String playerType,
         int rank) {

      if (rank < 1 || rank > 10) {

         throw new IllegalArgumentException("Invalid rank: " +
            rank);
      }

      if (!PLAYER_CREATOR.containsKey(playerType)) {
         throw new IllegalArgumentException("Invalid player type: "
            + playerType);
      }

      Map<Predicate<Integer>, Supplier<Player>> players =
         PLAYER_CREATOR.get(playerType);
      for (Entry<Predicate<Integer>, Supplier<Player>>
            entry : players.entrySet()) {
         if (entry.getKey().test(rank)) {
            return entry.getValue().get();
         }
      }

      throw new IllegalStateException("The players map is
         corrupted");
   }

}
```

The usage for obtaining the Football player: Cristiano Ronaldo result:

```java
Player footballPlayer = PlayerSupplier.supplyPlayer("FOOTBALL", 6);
```

Another approach will consist in using an enum, as follows:

```java
public enum PlayerTypes {

   TENNIS(Collections.unmodifiableList(Arrays.asList(
      () -> new TennisPlayer("Rafael Nadal"),
      () -> new TennisPlayer("Roger Federer"),
      () -> new TennisPlayer("Andy Murray"))
   ),
   Collections.unmodifiableList(Arrays.asList(
      rank -> rank == 1, rank -> rank > 1 &&
      rank < 5, rank -> rank >= 5 && rank <= 10))
   ),
   FOOTBALL(Collections.unmodifiableList(Arrays.asList(
      () -> new FootballPlayer("Lionel Messi"),
      () -> new FootballPlayer("Cristiano Ronaldo"))
   ),
   Collections.unmodifiableList(Arrays.asList(
      rank -> rank == 1 || rank == 2,
      rank -> rank > 2 && rank <= 10))
   ),
   SNOOKER(Collections.unmodifiableList(Arrays.asList(
      () -> new SnookerPlayer("Ronnie O'Sullivan"),
      () -> new SnookerPlayer("Mark Selby"),
      () -> new SnookerPlayer("John Higgins"),
      () -> new SnookerPlayer("Neil Robertson"))
   ),
   Collections.unmodifiableList(Arrays.asList(
      rank -> rank == 1, rank -> rank == 2,
      rank -> rank > 3 && rank < 7,
      rank -> rank >= 7 && rank <= 10))
   );

   private final List<Supplier<Player>> names;
   private final List<Predicate<Integer>> conditions;

   private PlayerTypes(List<Supplier<Player>> names,
         List<Predicate<Integer>> conditions) {
      this.names = names;
      this.conditions = conditions;
   }

   public static final Player supplyPlayer(String playerType,
         int rank) {

      if (rank < 1 || rank > 10) {
         throw new IllegalArgumentException("Invalid rank: " +
            rank);
      }

      List<Predicate<Integer>> selectors =
         PlayerTypes.valueOf(playerType).conditions;

      for (int i = 0; i < selectors.size(); i++) {
         if (selectors.get(i).test(rank)) {
            return PlayerTypes.valueOf(playerType)
               .names.get(i).get();
         }
      }

      throw new IllegalStateException("The enum is corrupted");
   }

}
```

Obtaining the output, Snooker player: Neil Robertson:

```java
Player snookerPlayer = PlayerTypes.supplyPlayer("SNOOKER", 10);
```

## Summary
In this article, you saw seven ways to deal with switch structures, as follows:

+ Implementing the Strategy Pattern via Java Enum
+ Implementing the Command Pattern
+ Using the Java 8+ Supplier
+ Defining a Custom Functional Interface
+ Relying on Abstract Factory
+ Implementing State Pattern
+ Implementing via Predicate




Ref:

+ [【译】重构Java switch语句的七种方法](https://juejin.cn/post/7149189951895617550)
+ [Seven Ways to Refactor Java switch Statements](https://www.developer.com/design/seven-ways-to-refactor-java-switch-statements/)

