## Почему не [флаги Айкара](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?
Все очень просто, сбор мусора у него основан, как он говорит, на невероятно стабильном, но крайне медленном по текущим меркам алгоритме D1GC. При этом всем он максимально устарел, все что он реализовал было инновационным во времена JDK 8, сейчас - нет. Действительно, зачем менять то, что работает? А стоило бы.

На замену я предлагаю поставить ShenandoahGC - это сборщик мусора с крайне малым временем паузы, что так раз подходит для нашей любимой игры, мы же все не любим фризы. На стабильность это никак не повлияло, за все время безперебойного тестирования (а это около недели) не было выявлено ни одной проблемы.

## Отказ от ответственности
Я не призываю всех тут же менять свои свойста запуска сервера, я лишь даю понять, что ничего идеального не бывает. Так же я не отвечаю за стабильность работы моих параметров в вашем конкретном случае, все системы разные, а резулаты абсолютно индивидуальны.

## Флаги
**Внимание!** Только JDK 16, работоспособность на старых версиях не гарантирована.

#### Поддерживается:
- [x] Vanilla
- [x] Bukkit, Spigot, Paper...
- [x] Fabric
- [x] Forge

#### Готовые настройки:
```bash
java -jar -server -Xms16G -Xmx16G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 launcher-airplane.jar --nogui
```
#### А теперь внимательно разберем, что за что отвечает:

*-Xms16G* и *-Xmx16G*: устанавливает границы использования памяти вашим сервером Minecraft, рекомендую оставить 1 - 2 Гб для системы.

*-XX:+UseLargePages* и *-XX:LargePageSizeInBytes=2M*: **только для опытных пользователей**, позволяет использовать зарегистрированую память большыми страницами, ускоряет скорость запуска и отзывчевость сервера. Заставим Linux регистрировать страницы для нас. Добавляем эту строку в `/etc/sysctl.conf`:
```bash
vm.nr_hugepages = 3372
```
Как я получил это число? Допустим я хочу зарегистрировать 6 Гб большых страниц, для этого делю 6 Гб на 2.

6 * 1024 / 2 = 3072

Дальше я рекомендую оставить немного свободного места, и добавить 300 к нашему числу.

3072 + 300 = 3372

После перезагружаем систему для применения изменений. Убедиться в том, что память успешно зарегистрована можно командой `grep /proc/meminfo`.

---
*-XX:+UnlockExperimentalVMOptions*: включает возможность использования эксперементальных возможностей.

*-XX:+UseShenandoahGC*: использование в роли алгоритма сборки мусора проект Шенандоа (именно так читает это название переводчик).

*-XX:ShenandoahGCMode=iu*: включение экспераментального режима работы нашего сборщика, он являеться зеркалом режима SATB, что сделает разметку менее консервативной, особенно в отношении доступа к слабым ссылкам.

---
*-XX:+UseNUMA*: включает чередование NUMA на хостах с несколькими сокетами, в сочетании с AlwaysPreTouch он обеспечивает лучшую производительность, чем стандартная готовая конфигурация. Более подробно о данной архитектуре можно узнать [отсюда](https://en.wikipedia.org/wiki/Non-uniform_memory_access).

*-XX:+AlwaysPreTouch*: предрегистрация сразу всей выделенной памяти, уменьшает заддержки ввода.

*-XX:-UseBiasedLocking*: существует компромисс между пропускной способностью неограниченной (предвзятой) блокировки и безопасными точками, которые JVM делает, чтобы включать и выключать их по мере необходимости. Для рабочих нагрузок, в том числе сервера Minecraft, ориентированных на задержку, имеет смысл отключить предвзятую блокировку.

*-XX:+DisableExplicitGC*: вызов System.gc () из пользовательского кода заставляет ShenandoahGC выполнить дополнительный цикл сборки мусора, отключение защищает от кода злоупотребляющего этим.
## Дополнительная конфигурация:
### bukkit.yml
```yml
chunk-gc:
 period-in-ticks: 600
```
#### Рекомендованое значение `chunk-gc.period-in-ticks`:  
Я не рекомендую использовать больше 12 Гб памяти на сервере с постоянным онлайном меньше 50.

| Память / Кол-во игроков | < 10 | 10 - 25 | 25 - 50 | > 50
| :--- | :---: | :---: | :---: | :---: |
| < 8 Гб | 400 | 200 | - | - |
| 8 - 16 Гб | 600 | 600 | 400 | 200 |
| > 16 Гб | 1200 | 1200 | 800 | 600 |

### spigot.yml
```yml
world-settings:
 default:
  max-tick-time:
   tile: 10
   entity: 20
```
## На этом пока-что все
Данная страница еще будет дорабатываться, так что следите за обновлениями :)
