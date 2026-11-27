# Ответы на домашнее задание: Интерфейсы и инверсия зависимостей

---

## Часть 1. Задачи на понимание

### Задача 1.1. "Кто виноват и что делать?"

**1. Почему `ProcessDamage` принимает `List<IDamageable>`, а не `List<Player>` или `List<Trap>`?**

Метод принимает `List<IDamageable>`, потому что это позволяет обрабатывать одной системой объекты из совершенно разных ветвей наследования. `Player` и `Trap` не имеют общего базового класса (кроме `object`), но оба реализуют `IDamageable`. С точки зрения полиморфизма это означает: вызывающий код взаимодействует с объектами единообразно, через вызов `target.TakeDamage(damage)`, не зная и не заботясь о их конкретном классе. Если бы метод принимал `List<Player>`, то ловушки пришлось бы обрабатывать отдельно, а при добавлении нового типа (например, `Barrel`) пришлось бы писать новый метод или перегрузку. Интерфейс же гарантирует наличие нужного метода у любого объекта, который его реализует, и система остаётся открытой для расширения без изменения.

**2. Нарушает ли `Trap` контракт `IDamageable`?**

Формально `Trap` не нарушает контракт: он реализует все объявленные члены с совпадающими сигнатурами, код компилируется. Однако семантически поведение странное: метод называется `TakeDamage`, а ловушка не получает урон и не меняет `HP`, возвращая `false`. С точки зрения вызывающего кода это допустимо — `GameSystem` просто вызывает метод и не обязан знать, что внутри. Но это пример «лживого» контракта: интерфейс обещает реакцию на урон, а объект реагирует иначе. Такое поведение не является ошибкой компиляции, но может привести к логическим багам, если другие системы ожидают, что `HP` действительно уменьшится. Правильнее было бы либо не реализовывать `IDamageable` у ловушки, либо переименовать/переосмыслить контракт (например, `IReactToDamage`).

**3. Что было бы, если бы зелье лечили через `ProcessDamage`?**

`HealthPotion` не реализует `IDamageable`, поэтому его нельзя передать в `List<IDamageable>` — код просто не скомпилируется. Это и есть защита на уровне типов: интерфейс гарантирует, что объект умеет реагировать на вызов `TakeDamage`. Если бы такой защиты не было (например, использовался `object`), пришлось бы делать приведение типа и проверять его в рантайме, что привело бы к `InvalidCastException` или `NullReferenceException`. Именно поэтому интерфейсы ценны: они переносят проверку корректности использования объекта из рантайма в компайл-тайм.

**4. Что произойдёт при добавлении `IsDead` в `IDamageable`?**

Придётся изменить все классы, реализующие `IDamageable`: `Player` и `Trap`. Если этого не сделать, код не скомпилируется, потому что реализация интерфейса требует объявления всех его членов с совпадающими именами и сигнатурами. Это называется нарушением принципа контракта: интерфейс — это обещание, и любой класс, его реализующий, обязан выполнить все обещания. Добавление нового члена в интерфейс — это breaking change для всех реализаций. Именно поэтому в C# появились default interface members: они позволяют добавлять члены с реализацией по умолчанию, чтобы не ломать существующие классы.

---

### Задача 1.2. "Abstract или Interface — вот в чём вопрос"

| Утверждение | Abstract class | Interface | Комментарий |
| :--- | :---: | :---: | :--- |
| Может содержать поля с конкретными значениями | + | − | Абстрактный класс может объявлять поля и инициализировать их; интерфейс не может содержать полей экземпляра. |
| Может иметь полноценный конструктор | + | − | Абстрактный класс может иметь конструктор для инициализации полей; интерфейс конструктора не имеет. |
| Один класс может наследовать/реализовывать несколько таких типов | − | + | Класс может наследовать только одному базовому классу, но реализовывать сколько угодно интерфейсов. |
| Может содержать реализацию метода по умолчанию | + | + | Абстрактный класс может содержать обычные методы с реализацией; интерфейс с C# 8 поддерживает default interface members. |
| Может объявлять свойства без реализации | + | + | Абстрактный класс может объявлять абстрактные свойства; интерфейс объявляет свойства без реализации по умолчанию. |
| Может быть использован как тип поля или переменной | + | + | И абстрактный класс, и интерфейс являются ссылочными типами и могут использоваться как тип переменной, поля, параметра и возвращаемого значения. |
| Может наследовать классы | + | − | Абстрактный класс может наследовать другой класс; интерфейс может наследовать только другие интерфейсы. |
| Может содержать приватные методы | + | + | Абстрактный класс может иметь private методы; интерфейс с default-реализацией может содержать private методы (они обязаны иметь реализацию). |

**Дополнительный вопрос: проектирование системы типов для `Player`, `Enemy`, `NPC`.**

Схема наследования (текстом):

```cs
abstract class Entity          // общее состояние: HP, Name, базовые методы
{
    public int HP;
    public string Name;
}

interface IDamageable           // все получают урон
{
    bool TakeDamage(int amount);
}

interface IInteractable         // Player и NPC могут взаимодействовать
{
    void Interact(Player player);
}

interface IHealable             // только Player умеет лечиться
{
    void Heal(int amount);
}

class Player : Entity, IDamageable, IInteractable, IHealable { ... }
class Enemy  : Entity, IDamageable { ... }
class NPC    : Entity, IDamageable, IInteractable { ... }
```

Обоснование: абстрактный класс `Entity` используется потому, что у всех трёх сущностей есть общее состояние (`HP`, `Name`) и общая логика — это утверждение о группе сущностей с общими свойствами. Интерфейсы используются для описания возможностей взаимодействия: `IDamageable` — все получают урон; `IInteractable` — только `Player` и `NPC`; `IHealable` — только `Player`. Согласно лекции, абстрактный класс применяется, когда для множества типов есть общее состояние или конструктор, а интерфейс — когда разные сущности должны реагировать на одинаковый вызов своими способами. Более того, абстрактный класс может сам реализовывать интерфейсы, и обе концепции применяются вместе, что и показано в схеме.

---

## Часть 2. Задачи на написание кода

### Задача 2.1. "Реализация готового интерфейса"

```cs
using System;

public interface ICombatant
{
    public string Name { get; }
    public int HP { get; set; }
    public bool IsDead { get; }
    public int Damage { get; }
    public void Attack(ICombatant target);
    public void TakeDamage(int amount);
}

public class Knight : ICombatant
{
    public string Name { get; }
    public int HP { get; set; }
    public bool IsDead => HP <= 0;
    public int Damage { get; }

    public Knight(string name)
    {
        Name = name;
        HP = 150;
        Damage = 25;
    }

    public void Attack(ICombatant target)
    {
        if (IsDead)
        {
            Console.WriteLine($"Рыцарь {Name} мёртв и не может атаковать.");
            return;
        }
        if (target.IsDead)
        {
            Console.WriteLine($"Цель {target.Name} уже мертва.");
            return;
        }
        Console.WriteLine($"Рыцарь {Name} атакует!");
        target.TakeDamage(Damage);
    }

    public void TakeDamage(int amount)
    {
        HP -= amount;
        if (HP < 0) HP = 0;
        Console.WriteLine($"Рыцарь {Name} получил {amount} урона. Осталось HP: {HP}");
    }
}

public class Mage : ICombatant
{
    public string Name { get; }
    public int HP { get; set; }
    public bool IsDead => HP <= 0;
    public int Damage { get; }

    public Mage(string name)
    {
        Name = name;
        HP = 80;
        Damage = 40;
    }

    public void Attack(ICombatant target)
    {
        if (IsDead)
        {
            Console.WriteLine($"Маг {Name} мёртв и не может атаковать.");
            return;
        }
        if (target.IsDead)
        {
            Console.WriteLine($"Маг {Name} не может атаковать мёртвую цель");
            return;
        }
        Console.WriteLine($"Маг {Name} атакует!");
        target.TakeDamage(Damage);
        if (!target.IsDead)
            target.TakeDamage(Damage);
    }

    public void TakeDamage(int amount)
    {
        HP -= amount;
        if (HP < 0) HP = 0;
        Console.WriteLine($"Маг {Name} получил {amount} урона. Осталось HP: {HP}");
    }
}

class Program
{
    static void Main()
    {
        var knight = new Knight("Артур");
        var mage = new Mage("Мерлин");

        while (!knight.IsDead && !mage.IsDead)
        {
            knight.Attack(mage);
            if (mage.IsDead) break;
            mage.Attack(knight);
        }

        if (knight.IsDead && mage.IsDead)
            Console.WriteLine("Ничья!");
        else if (knight.IsDead)
            Console.WriteLine($"Победил Маг {mage.Name}!");
        else
            Console.WriteLine($"Победил Рыцарь {knight.Name}!");
    }
}
```

---

### Задача 2.2. "Разработка собственного интерфейса"

```cs
using System;
using System.Collections.Generic;

public enum Rarity
{
    Common,
    Rare,
    Legendary
}

public interface IUsableItem
{
    string Name { get; }
    string Description { get; }
    Rarity Rarity { get; }
    void Use(Player player);
}

public class Player
{
    public string Name { get; }
    public int HP { get; private set; }
    public int Damage { get; private set; }

    public Player(string name, int hp, int damage)
    {
        Name = name;
        HP = hp;
        Damage = damage;
    }

    public void Heal(int amount)
    {
        HP += amount;
    }

    public void IncreaseDamage(int amount)
    {
        Damage += amount;
    }

    public void TakeDamage(int amount)
    {
        HP -= amount;
        if (HP < 0) HP = 0;
    }

    public override string ToString()
    {
        return $"{Name}: HP = {HP}, Damage = {Damage}";
    }
}

public class HealthPotion : IUsableItem
{
    public string Name => "Зелье здоровья";
    public string Description => "Восстанавливает 50 HP";
    public Rarity Rarity => Rarity.Common;

    public void Use(Player player)
    {
        player.Heal(50);
        Console.WriteLine("Вы выпили зелье и восстановили 50 HP");
    }
}

public class Sword : IUsableItem
{
    public string Name => "Меч";
    public string Description => "Увеличивает урон на 10";
    public Rarity Rarity => Rarity.Rare;

    public void Use(Player player)
    {
        player.IncreaseDamage(10);
        Console.WriteLine("Вы взяли меч и увеличили урон на 10");
    }
}

public class AncientScroll : IUsableItem
{
    public string Name => "Древний свиток";
    public string Description => "Наносит 20 урона, но увеличивает урон на 30";
    public Rarity Rarity => Rarity.Legendary;

    public void Use(Player player)
    {
        player.TakeDamage(20);
        player.IncreaseDamage(30);
        Console.WriteLine("Вы прочли свиток: получили 20 урона, но урон вырос на 30");
    }
}

class Program
{
    static void Main()
    {
        var player = new Player("Герой", 100, 10);

        var items = new List<IUsableItem>
        {
            new HealthPotion(),
            new Sword(),
            new AncientScroll()
        };

        foreach (var item in items)
        {
            Console.WriteLine($"Используем: {item.Name} ({item.Rarity}) — {item.Description}");
            item.Use(player);
        }

        Console.WriteLine($"Итог: {player}");
        Console.WriteLine(player.HP > 0 ? "Игрок жив" : "Игрок мёртв");
    }
}
```

---

### Задача 2.3. "DI на минималках: система урона в игре"

```cs
using System;

public interface IDamageCalculator
{
    int CalculateDamage(int baseDamage, int targetArmor);
}

public class PhysicalDamageCalculator : IDamageCalculator
{
    public int CalculateDamage(int baseDamage, int targetArmor)
    {
        int result = baseDamage - targetArmor / 2;
        return Math.Max(1, result);
    }
}

public class MagicDamageCalculator : IDamageCalculator
{
    public int CalculateDamage(int baseDamage, int targetArmor)
    {
        if (targetArmor == 0)
            return baseDamage * 2;
        int result = baseDamage - targetArmor / 4;
        return Math.Max(1, result);
    }
}

public class FireDamageCalculator : IDamageCalculator
{
    public int CalculateDamage(int baseDamage, int targetArmor)
    {
        int result = baseDamage - targetArmor / 10;
        if (baseDamage > 50)
            result += 10;
        return Math.Max(1, result);
    }
}

public class Unit
{
    public string Name { get; }
    public int HP { get; set; }
    public int Armor { get; }

    public Unit(string name, int hp, int armor)
    {
        Name = name;
        HP = hp;
        Armor = armor;
    }
}

public class DamageSystem
{
    private readonly IDamageCalculator _calculator;

    public DamageSystem(IDamageCalculator calculator)
    {
        _calculator = calculator;
    }

    public void ApplyDamage(Unit target, int baseDamage)
    {
        int damage = _calculator.CalculateDamage(baseDamage, target.Armor);
        target.HP -= damage;
        if (target.HP < 0) target.HP = 0;
        Console.WriteLine($"Юнит {target.Name} получил {damage} урона. Осталось HP: {target.HP}");
    }
}

class Program
{
    static void Main()
    {
        var target = new Unit("Гоблин", 100, 10);

        Console.WriteLine("== Физический урон ==");
        new DamageSystem(new PhysicalDamageCalculator()).ApplyDamage(target, 40);

        Console.WriteLine("== Магический урон ==");
        new DamageSystem(new MagicDamageCalculator()).ApplyDamage(target, 40);

        Console.WriteLine("== Огненный урон ==");
        new DamageSystem(new FireDamageCalculator()).ApplyDamage(target, 60);

        // Ответ на вопрос: чтобы добавить ледяной урон, достаточно создать новый класс
        // IceDamageCalculator : IDamageCalculator и передать его в конструктор DamageSystem.
        // Класс DamageSystem менять не нужно — это и есть инверсия зависимостей.
    }
}
```

---