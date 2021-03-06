# Домашнее задание к занятию 4.1 (модуль 3): JVM. Организация памяти, сборщики мусора, VisualVM
## Задача "Понимание JVM"
## Описание
Просмотрите код ниже и опишите (текстово или с картинками) каждую строку с точки зрения происходящего в JVM

Не забудьте упомянуть про:
- ClassLoader'ы,
- области памяти (стэк (и его фреймы), хип, метаспейс)
- сборщик мусора

## Код для исследования
```java

public class JvmComprehension {

    public static void main(String[] args) {
        int i = 1;                      // 1
        Object o = new Object();        // 2
        Integer ii = 2;                 // 3
        printAll(o, i, ii);             // 4
        System.out.println("finished"); // 7
    }

    private static void printAll(Object o, int i, Integer ii) {
        Integer uselessVar = 700;                   // 5
        System.out.println(o.toString() + i + ii);  // 6
    }
}

```
## Решение
#Загрузка классов
После запуска на выполнение нашей программы, JVM в первую очередь загружает базовые классы иp java.lang: Object, 
Integer, System. Затем JVM загружает класс, содержащий метод main (JvmComprehension) его в область памяти Metaspace 
при помощи одного из трех типов имеющихся в ней загрузчиков классов. 
Class loaders: Bootstrap, Extension ClassLoader, System ClassLoader.
1. Bootstrap Classloader – загружает стандартные классы JDK из архива rt.jar. 
2. Platform ClassLoader – загружает классы расширений, которые по умолчанию находятся в каталоге jre/lib/ext
3. Application ClassLoader – загружает классы приложения, определенные в переменной среды окружения CLASSPATH

Далее после загрузки класса JvmComprehension запускается этап связывания в ходе которого проводится проверка байткода
(Verify), затем подготовка примитивов в статических полях и статических блоков инициализации (Prepare) и затем 
разрешение ссылок на другие классы (Resolve). В нашем случае с классом JvmComprehension нет статических полей, 
статических блоков инициализации и связанных классов, поэтому в нашем случае, я полагаю, Prepare и Resolve ничего не 
делают. 

Затем начинается выполнение метода Main. JVM создает в области памяти Stack новый стэк-фрейм `main` и сохраняет туда 
аргументы метода `main` и адрес возврата из него. Далее начинается построчное выполнение метода `main`. Встречающиеся 
локальные переменные так же сохраняются в стэк-фрейм метода `main`. Сначала `int i = 1` //1. 

Затем JVM натыкается на `Object o` //2. Она размещает в области памяти Heap экземпляр класса `Оbject`, создает в 
стек-фрейме метода `main` переменную-ссылку `o` на `Оbject`, и сохраняет адрес этого куска Heap-памяти (ссылку) 
в стек-фрейме `main` в переменную `o` //2.

Затем появляется упоминание класса `Integer` //3. Далее JVM создает в куче (или в том же стэк фрейме, где
именно - для этого случая не удалось найти) экземпляр `Integer` со значением `2`, затем создает в стек-фрейме `main` 
новую переменную `ii` типа ссылки на `Integer` и записывает в нее ссылку на созданный экземпляр `Integer`.  

Затем JVM записывает в стек новый стэк-фрейм для метода `printAll` //4. В этот стэк-фрейм так же записываются все 
значения аргументов, то есть копии переменных `o`, `i`, `ii`, а так же адрес возврата в вызывающий метод `main`. 
Начинается построчное выполнение метода `printAll` //5.

Опять встречается класс `Integer` //5. Для строки `Integer uselessVar = 700;` в куче (или в стэк фрейме 
не удалось найти) создается объект класса Integer со значением 700, ссылка на него сохраняется в переменной 
`uselessVar` в стэк-фрейме `printAll`.

Затем поток выполнения возвращается к методу `println` //6. В стэке создается новый стэк-фрейм `println`. 
В него сохраняется адрес выхода в метод `printAll` и значения аргументов i и ii и я думаю, что так же создается 
некая ссылка объект типа `String`, на данном этапе видимо `null`. 

Затем создается стэк-фрейм для `toString`. В него прописывается адрес выхода в `println`, переменная типа `String` для 
передачи возвращаемого значения в `println` и ссылка `o`. Полагаю в стэк-фрейме `println` до передачи управления в
`toString` как то отмечается, что по завершению `toString`, ссылка на строку из фрейма `toString` должна быть 
скопирована в фрейм `println` до уничтожения фрейма `toString`.

Поcле того как `toString` отработал, строка сохраняется в куче (или в стэке, не уждалось найти где ) и ссылка-результат 
сохраняется в стэк-фрейме `toString`. Затем ссылка копируется в соответсвующее поле в стэк-фрейме `println` и фрейм 
`toString` уничтожается, управление передается методу `println`. Он печатает три своих аргумента, взяв их значения из 
своего стэк-фрейма, управление передается в метод `printAll`, в котором больше нет операторов, поэтому управление 
передается в метод main к строке //7. 

Начиная с данного момента (начало строки //7), если сборщик мусора запустится, он соберет переменную uselessVar, 
если она в куче создавалась. Если же она все же в самом стэк-фрейме `printAll` создавалась, то она удалится вместе 
с ним. То же самое касается строки-результата метода `toString`.   

Создается новый стэк-фрейм для метода `println`, в этом фрейме сохраняется адрес выхода в `main` и строка аргумент со 
значением `"finished"`. Тут к сожалению не удалось найти, где создается строка "finished", предположу, что 
в данном случае, поскольку она уже определена и то скорее она создается прям в стэк-фрейме метода `println`. Метод 
`println` печатает свой аргумент, передает управление в метод main и стэк-фрейм `println` уничтожается. Если "finished"
создавалась в куче, то начиная с этого момента, если сборщик мусора запустится - он ее соберет.

Метод `main` заканчивает свою работу, стэк-фрейм `main` уничтожается, 











