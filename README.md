                                                Нестеренко А. Е., Пескова О.Ю.

                                    Институт компьютерных технологий и информационной

                                        безопасности южного федерального университета

                                                    NesTerrSaSha20@yandex.ru



# LINUX KERNEL ROOTKIT

    В работе приведен пример реализации rootkit – программы для linux систем. Реализован собственный реально функционирующий многопоточный драйвер, производящий скрытие и защиту указанных данных, а также дампинг и отсылку информации по сети о процессах запускаемых пользователем. В заключении представлены результаты выполнения тестового задания с использованием программы.


Ключевые слова: rootkit, linux, драйвер, многопоточность, slab, ядро операционной системы.



## Введение.

    В моей работе будет рассмотрен такой аспект программирования, как rootkit технология, для ядра операционных систем linux версии 2.6.x.x и выше. Здесь будет описано, что такое rootkit, какими они бывают, как их использовать и приведен пример rootkit’а, который скрывает дампинг процессов и отсылку пакетов по сети.
    Основное назначение rootkit-программ — замаскировать присутствие постороннего лица и его инструментальных средств на машине. Данная программа предназначена для скрытной слежки за действиями пользователя. Она может быть полезна для системных администраторов, аналитиков DLP систем при контроле за пользователями на предприятии, мониторинге активности целевой системы, либо судебно – техническим экспертам при расследовании инцидентов.
    Для внедрения в систему потребуется Makefile для компиляции под конкретное ядро ОС, файл с программой и файл с шеллкодом, который собственно будет внедрять.

## Основные задачи программы:

Представлять собой полноценную и используемую практически программу.
Иметь возможность встраиваться в ядро операционной системы.
Иметь возможность скрывать информацию о себе.
Иметь возможность получать информацию о процессах.
Иметь возможность сохранять информацию о процессах в файл.
Иметь возможность отсылать udp-пакеты на указанный адрес.

Теоретические очновы
Руткит
Руткит (англ. rootkit, то есть «набор root'а») — набор программных средств (например, исполняемых файлов, скриптов, конфигурационных файлов), для обеспечения:
- маскировки объектов (процессов, файлов, директорий, драйверов)
- контроля (событий, происходящих в системе)
- сбора данных (параметров системы)
Термин Rootkit исторически пришёл из мира UNIX, и под этим термином понимается набор утилит или специальный модуль ядра, которые злоумышленник устанавливает на взломанной им компьютерной системе сразу после получения прав суперпользователя. Этот набор, как правило, включает в себя разнообразные утилиты для «заметания следов» вторжения в систему, делает незаметными снифферы, сканеры, кейлоггеры,троянские программы, замещающие основные утилиты UNIX (в случае неядерного руткита). Rootkit позволяет взломщику закрепиться во взломанной системе и скрыть следы своей деятельности путём сокрытия файлов, процессов, а также самого присутствия руткита в системе.
В систему руткит может быть установлен различными способами: загрузка посредством эксплойта, после получения шелл-доступа (в таком случае, может использоваться средство типа wget или исходный FTP-клиент для загрузки руткита с удаленного устройства), в исходном коде или ресурсах программного продукта.

## Классификация руткитов.
### По уровню привилегий:
Уровень пользователя (user-mode) – категория основана на перехвате функций библиотек пользовательского режима. Руткиты этой категории получают те же права, что обычное приложение, запущенное на компьютере. Руткиты исполняются в непривилегированном кольце (с точки зрения архитектуры информационной безопасности). Они используют программные расширения (например, для проводника Windows), перехват сообщений, отладчики, эксплуатируют уязвимости в безопасности, а также производят перехваты функций (function hooking) широко используемых API (в памяти каждого отдельного процесса). Они внедряются в другие запущенные процессы и используют их память. Это более распространенный вариант. Легче обнаруживается и устраняется даже стандартными средствами операционной системы. Дёшевы в приобретении.
Уровень ядра (kernel-mode) – категория основана на установке в систему драйвера, осуществляющего перехват функций уровня ядра. Руткиты этой категории работают на самом глубинном уровне ОС, получая максимальный уровень доступа на компьютере. После инсталляции такого руткита, возможности атакующего практически безграничны. Руткиты исполняются в привилегированном нулевом кольце (наивысший уровень привилегий ОС). Они могут встраиваться в драйверы устройств, проводить прямую модификацию объектов ядра (DKOM), а также влиять на взаимодействие между пользовательским режимом и режимом ядра. Руткиты уровня ядра обычно более сложны в создании, поэтому встречаются реже. Также их гораздо сложней обнаружить и удалить. Дороги в приобретении.

### Основные способы реализации в UNIX и linux
Самый распространенный метод, обеспечивающий функционирование руткита уровня ядра - это перехват системных вызовов путем подмены соответствующей записи в таблице системных вызовов sys_call_table. Детали этого метода заключаются в следующем. При обработке прерывания int 0x80 (или инструкции sysenter) управление передается обработчику системных вызовов, который после предварительных процедур передает управление на адрес, записанный по смещению %eax в sys_call_table. Таким образом, подменив адрес в таблице, мы получаем контроль над системным вызовом. Этот метод имеет свои недостатки: в частности, он легко детектируется антируткитами; таблица вызовов в современных ядрах не экспортируется; и кроме того, перехват некоторых системных вызовов (например, execve()) нетривиален.

Другим распространенным механизмом в kernel-mode руткитах является патчинг VFS (Virtual Filesystem Switch). Этот подход применяется в рутките adore-ng. Он основан на подмене адреса какой-либо из функций-обработчиков для текущей файловой системы.

Как и в Windows, широко используется сплайсинг - замена первых байтов кода системного вызова на инструкцию jmp, осуществляющую переход на адрес обработчика руткита. В коде перехвата обеспечивается выполнение проверок, возврат байтов, вызов оригинального кода системного вызова и повторная установка перехвата. Данный метод также легко детектируется.

## Дополнительные возможности.

Кроме непосредственно себя, руткит, как правило, может маскировать присутствие в системе любых описанных в его конфигурации каталогов и файлов на диске, ключей вреестре. По этой причине естественным образом появились «навесные» руткитные библиотеки. Многие руткиты устанавливают в систему свои драйверы и службы (они, естественно, также являются «невидимыми»).

## Руткиты для и против DRM.

Руткитами по сути является большинство программных средств защиты от копирования(и средств обхода этих защит — например, эмуляторы CD- и DVD-приводов).
В 2005 году корпорация Sony BMG встраивала в свои аудиодиски защиту на основе руткита, который устанавливался без ведома пользователя.


### Slab.

В реализации программы используется выделение памяти с помощью slab allocator. Что же то такое? Распределение slab – это механизм управления памятью, предназначенный для более эффективного распределения памяти и устранения значительной фрагментации. Основой этого алгоритма является сохранение выделенной памяти, содержащей объект определенного типа, и повторное использование этой памяти при следующем выделении для объекта того же типа. Этот метод был впервые введен в SunOS Джефом Бонвиком и сейчас широко используется во многих операционных системах Unix, включая FreeBSD и Linux.

Фундаментальная идея способа распределения slab основывается на результатах наблюдений, показывающих, что некоторые объекты данных ядра часто создаются и уничтожаются после того, как перестают быть нужными. Таким образом, при каждом выделении памяти для объектов такого типа затрачивается некоторое время для нахождения наиболее подходящего места для этого объекта. Кроме того, освобождения памяти после уничтожения объекта способствует фрагментации памяти, которая создает дополнительную нагрузку на ядро для реорганизации памяти.

В случае же распределением slab, при использовании программистом определенных системных вызовов, участки памяти, подходящие для размещения объектов данных определенного типа и размера, заранее предопределены. Распределитель slab хранит информацию о размещении этих участков, известных также как кэши. Таким образом, если поступает запрос на выделение памяти для объекта данных определенного размера, он может мгновенно удовлетворить запрос уже выделенным слотом. Однако, уничтожение объектов не освобождает память, а только открывает слот, который помещается в список свободных слотов распределителем slab. Следующий вызов для выделения памяти того же размера вернет слот памяти, не используемый в данный момент. Этот процесс устраняет необходимость в поиске подходящего участка памяти и значительно снижает фрагментацию памяти. В этом контексте slab – это одна или более смежных страниц в памяти, содержащих заранее выделенные участки памяти.
Понимание распределения slab требует определения следующих терминов:
Кэш: кэш представляет собой небольшой объём очень быстрой памяти. Здесь мы используем кэш как память для хранения таких объектов, как семафоры, дескрипторы процессов, объекты файлов и т. д. Каждый кэш способен хранить только один тип объектов.
Slab: slab представляет собой непрерывный участок памяти, обычно составленный из нескольких физических смежных страниц. Кэш состоит из одного или более slab’ов.

Когда программа создает кэш, она выделяет ряд объектов в него. Их количество зависит от размера связанных slab’ов. Slab может находиться в одном из следующих состояний:
пустой – все объекты в slab’e помечены как свободные
частично занятый – slab содержит как используемые, так и пустые объекты
заполненный – все объекты в slab’е помечены как используемые
Изначально система помечает каждый slab как «пустой». Когда процесс обращается за новым объектом ядра, система делает попытку найти свободное место для этого объекта в частично занятом slab’е в кэше для этого типа объектов. Если такого места не находится, система выделяет новый slab из смежных физических страниц и передает их в кэш. Новый объект размещается в этом slab’е, а это местоположение помечается как «частично занятое». Основное преимущество алгоритма slab заключается в том, что память выделяется точно в том объёме, в котором требуется. Таким образом, отсутствует внутренняя фрагментация памяти. Распределение происходит быстро, поскольку система создает объекты заранее и легко выделяет их из slab’а.
Функциональное описание программы руткита.


Словесное описание модулей, функций, классов и переменных, используемых в программе, блок - схемы основных функций драйвера.

Программа rootkit работает в пространстве ядра, перехватывает системные вызовы, генерирует и отправляет udp пакеты, сохраняет информацию в файл, скрывает своё присутствие. Для того чтобы воспользоваться готовыми модулями, функциями, структурами и переменными необходимо подключить соответствующие библиотеки.
Функция выделения памяти – позволяет динамически выделить память под переменные определённого типажа, принимает параметрами имя структуры, её размер, смещение и флаги, возвращает указатель на выделенную память.
Макрос лицензии – позволяет указывать лицензию на продукт, необходим для использования некоторых функций, принимает параметром строку с лицензией.
Директивы препроцессора позволяют создавать глобальные переменные, узнавать архитектуру системы(i386 или amd64), поиска таблицы системных вызовов по адресу в стеке.
Функция нахождения таблицы системных вызовов ищет её среди множества соответствующих адресов в системе, возвращает адрес таблицы, либо NULL, если ничего не нашла.
Функция сохранения копии оригинальной таблицы вызовов – сохраняет оригинальную таблицу системных вызовов перед её модификацией, это может пригодиться при выгрузке драйвера.
Макрос получения передаваемых драйверу параметров – в программе используется для получения времени таймаута между отправками пакетов и записью в дамп и имени процесса.
Структура описывающая перехваченный процесс, содержит поле имени процесса, имени команды, pid процесса и двусвязный список.
Структура контекста потока – содержит флаг остановки и информацию о дампе.
Функция записи в файл записывает перехваченную информацию о процессах в файл.
Функция потока сохранения обеспечивает сохранение информации в файл, мьютексную блокировку/разблокировку файла.
Функция потока генерации и отправки udp пакетов генерирует и отправляет пакет на хост.
Функция инициализации драйвера описывает инициализацию драйвера, создание потоков, его сокрытие.
Функция обхода защиты от записи в таблицу системных вызовов помогает осуществлять запись в таблиц системных вызовов, по – сути конструктор.
Функция удаления элементов драйвера из ядра подчищает память, сливает потоки, возвращает оригинальную таблицу системных вызовов и её защиту, по – сути деструктор.
Функция создания кеш – памяти – создаёт объект в кеше.
Функция выделения памяти для кеша выделяет память определенного размера в кеше.
Функция высвобождения памяти кеша – высвобождает ранее выделенную память.
Функция удаления объекта кеша – удаляет ранее созданный объект в кеше.
Макрос загрузки модуля в ядро берёт в себя конструктор и загружает в ядро.
Макрос выгрузки модуля из ядра берёт в себя деструктор и выгружает драйвер его.


Инструкция пользователю linux - системы.

Хорошей инструкцией пользователю в этом случае будет не запускать подозрительные исполняемые файлы от root’а, а если хочется поэкспериментировать, то порядок такой:

Перейти в директорию с файлами rootkit.c и Makefile.

Выполнить от root команду make, для того, чтобы собрался драйвер.

Выполнить от root команду insmod rootkit.ko dump_timeout=<время таймаута> proc_name=<имя процесса>

Попробовать выгрузить драйвер. Или хотя бы найти его в ядре.


Инструкция системному администратору linux – системы или инженеру/аналитику DLP системы.

Перейти в директорию с файлами rootkit.c и Makefile на целевой машине.

Выполнить от root команду make, для того, чтобы собрался драйвер.

Выполнить от root команду insmod rootkit.ko dump_timeout=<время таймаута> proc_name=<имя процесса>

Запустить на своей машине скипт на scapy см Приложение Д.


Экспериментальный запуск программы.

Клиентская часть


![alt tag](https://github.com/alexander121212/rootkit/blob/master/imgs/1.png)

Рисунок 4. Сборка драйвера утилитой make.

![alt tag](https://github.com/alexander121212/rootkit/blob/master/imgs/2.png)


Рисунок 5. Загрузка драйвера в ядро утилитой insmod.
![alt tag](https://github.com/alexander121212/rootkit/blob/master/imgs/3.png)



Рисунок 6. Попытки обнаружить драйвер – руткит.

Серверная часть


![alt tag](https://github.com/alexander121212/rootkit/blob/master/imgs/4.png)

Рисунок 7. Прием пакетов на сервере.

![alt tag](https://github.com/alexander121212/rootkit/blob/master/imgs/5.png)


Рисунок 8. Прием пакетов на сервер.

Заключение



В данной работе представлен многопоточный ядерный руткит, пересылающий информацию по сети. Рассмотрен пример реализации. Подобные программы часто используют как с целью скрытия вредоносного программного обеспечения, так и в целях защиты и контроля утечки данных. Отличием данного руткита от большинства других является распараллеливание задач, связанных с отправкой и дампингом данных. Также активное использование быстрой кеш – памяти, с использованием слабового аллокатора.

СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ


поиск ядерных функций. [Электронный ресурс]. – URL:http://lxr.free-electrons.com/ident. (дата обращения: 13.12.15)

Перехват системных вызовов. [Электронный ресурс].  URL: http://jbremer.org/x86-api-hooking-demystified/#ah-basic (дата обращения: 13.12.15)

Пример руткита. [Электронный ресурс].  URL: http://turbochaos.blogspot.ru/2013/09/linux-rootkits-101-1-of-3.html (дата обращения: 13.12.15)

Информация о руткитах. [Электронный ресурс].  https://ru.wikipedia.org/wiki/%D0%A0%D1%83%D1%82%D0%BA%D0%B8%D1%82 (дата обращения: 13.12.15)

Slab и slab allocator. [Электронный ресурс].  URL: https://ru.wikipedia.org/wiki/Slab (дата обращения: 13.12.15)

Классификация руткитов [Электронный ресурс]. - https://blog.kaspersky.ru/chto-takoe-rutkit-i-kak-udalit-ego-s-kompyutera/650/
