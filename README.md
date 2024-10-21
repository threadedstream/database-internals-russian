## Распределенные данные от Алекса Петрова

Изначально я планировал выкладывать конспекты по каждой главе у себя в tg канале, но вскоре понял, что телега - не лучшее место для конспектов по книгам, поэтому я перенес его сюда. 
Сказать, что это конспект в привычном понимании этого слова было бы неправильным, я бы назвал это смесью конспекта с щепотками собственного мнения под соусом странной манеры письма. 

## Table of contents
- [Глава 2. Введение в B-деревья](#chapter2)
- [Глава 3. Форматы файлов](#chapter3)
- [Глава 4. Реализация B-деревьев в БД](#chapter4)
- [Глава 5. Обработка транзакций и восстановление](#chapter5)
- [Глава 6. Варианты B-деревьев](#chapter6)

<a name="chapter2"></a> 
### Глава 2. Введение в B-деревья
Эта глава посвящена теме, с которой я познакомился еще будучи студентом колледжа, и к которой я возвращаюсь по сей день - это B-Tree. Да, та самая структура данных, которую умные люди в области создания этих сложных СХД взяли за основу построения реляционных баз данных типа PostgreSQL или MySQL. Для начала хорошо бы понять, а почему вообще упарываться с такой сложной структурой данных, когда можно использовать стандартное двоичное дерево. Ну да, может быть проблема с балансированием таких деревьев, но есть же сбалансированные варианты типа AVL, Red-Black Tree и т.д. Секрет кроется в возможностях сегодняшних жестких дисков и SSD. Если не знаете, как устроен диск, то вкратце скажу, что состоит он из движущейся головки, которая считывает информацию с пластин. Причем, головка читает данные последовательно и читает она эти данные секторами размером от 512Б до 4КБ - не побайтово. Аналогия с механизмом выделения памяти в ОС, где память отдается приложению в виде страниц фиксированного размера. Т.е чтобы прочесть какие-либо данные, должно произойти позиционирование головки, что является достаточно затратной операцией, а далее наступает процесс последовательного чтения, что происходит достаточно быстро. SSD же устроены несколько иначе: есть ячейки памяти, которые объединяются в строки (обычно от 32 до 64 ячеек на строку), строки объединяются в массивы, массивы - в страницы, а страницы - в блоки. Блоки объединяются в пластины, а пластины уже образуют кристалл. Причем, размер ячейки памяти зависит от используемой технологии. Соответственно, размер одной страницы может варьироваться от 2 до 16КБ. В SSD минимальной единицей, которую можно прочитать или записать, является страница. Но стирание данных осуществляется поблочно, этим занимается отдельный компонент внутри SSD, называемый FTL (Flash Translation Layer). Но самое главное, что в отличие от тех же жестких дисков, с произвольным вводом-выводом у SSD все гораздо лучше. На этом этапе вы, возможно, задаете себе вопрос: "Ну и зачем мне это щас нужно было", ну а я отвечу, что благодаря этой инфе теперь можно подобрать структуру данных, которая обладает следующими свойствами: 

A. У структуры данных высокая степень ветвления (т.е возможность для одного узла иметь большое кол-во потомков), таким образом можно было бы обеспечить лучшую локальность соседних ключей (держать соседние ключи ближе друг к другу).\
Б. Структура данных не предусматривает наличие большой высоты, что сокращает количество операций дискового поиска во время обхода. 

Что ж, ну и умные люди по имени Байер и МкКрейт в 1972 и создали такую структуру данных как B-Tree. Вообще если хотите на интуитивном уровне понять, как она работает, то лучше всего посмотреть видео. Я читал достаточно много литературы (сюда входят и статьи), где описывают принцип ее работы, но ничего простого не нашел. А лучше всего посмотреть видео, а потом почитать статьи. Ну и нет ничего лучше идеи реализовать это на каком-нибудь ЯП. 
Для начала посоветую вот это [видео](https://www.youtube.com/watch?v=K1a2Bk8NrYQ).  
Есть еще вот такой [сайт](https://www.cs.usfca.edu/~galles/visualization/BTree.html), где можно интерактивно изучить принцип работы разных структур данных, в том числе и B-Tree. 

<a name="chapter3"></a> 
### Глава 3. Форматы файлов
Новая глава - новый пост. Сегодня поговорю о способе организации данных на диске, а именно о том, как представить узлы B-Tree на каком-либо дисковом носителе. Во-первых, нужно сказать, что для простоты разработчики СУБД решили выделять на один узел дерева одну страницу. Например, в оригинальной статье с описанием B-Tree описывался простейший способ организации этих страниц для записи данных, где это выглядит как-то вот так

| p0 | k1 | v1 | p1 | k2 | v2 | p2| .... |kn| |vn| |pn| 
где:

p - указатель на дочерние страницы
k - ключ
v - значение

Но такой подход обладает недостатками. 
1. Если новая запись добавляется не справа, то возникает необходимость перемешивать элементы
2. Данный способ подходит только для данных фиксированного размера
3. Если хранить в таком виде данные переменного размера, возникает проблема с фрагментацией, т.е когда у нас есть много свободных кусков разного размера, появившихся в результате освобождения ненужных ячеек. При такой организации достаточно сложно дефрагментировать память. 

Для устранения этих проблем была придумана структура слотированных страниц (slotted page). Выглядит она как-то так 

<img width="755" alt="pic" src="https://github.com/user-attachments/assets/1bbbce6c-12c1-495b-a415-41b4d33460c7">

В данной схеме у нас есть: 

Заголовок, содержащий в себе метаинформацию, типа id страницы, количество свободного места, указатели на начало свободного и конец свободной памяти, различные флаги, тип узла (ROOT, INTERIOR, LEAF).

Указатели, указывающие на ячейки. Причем, указатели отсортированы для возможности двоичного поиска

Ну и сами ячейки, хранящие пары ключ/запись. 

Такая структура позволяет достаточно гибко как вставлять, так и удалять данные. В случае удаления указатель на ячейку может быть помечен как "not used", ну а на странице может быть выставлен флаг CAN_COMPACT. Через какое-то время гарбедж коллектор просто скомпактит страницу, освободив место для будущих вставок. 

Вот тут более подробно рассказывается про слотированные страницы 
https://siemens.blog/posts/database-page-layout/
Еще и код есть, что вообще неплохо. 

<a name="chapter4"></a> 
### Глава 4. Реализация B-деревьев в БД
Продолжаем говорить о B-деревьях. Это уже 3 глава о них, да. Вот такие они сложные и важные. Во-первых, надо повторить, что каждому узлу в дереве соответствует одна страница. Одна такая страница содержит заголовок, отсортированные смещения (или указатели на ячейки) и сами ячейки, содержащие пары ключ/запись. Каждый файл, содержащий данные, начинается с магического числа (Magic number). Обычно это набор каких-то байтов, которые помогают понять, с каким файлом мы имеем дело.
 
**Линки между узлами** \
Представим дерево с одним корневым узлом и 2 дочерними узлами child1 и child2. Путь от одного дочернего узла к другому дочернему узлу содержит посещение корневого узла. Некоторые реализации избегают последнего путем создания линков между одноуровневыми узлами. Учитывая, что все ключи на уровне отсортированы, это достаточно хороший подход, экономящий время поиска. Минусом такого подхода является экстра оверхед при апдейте линков при разделении и слиянии. 

**Крайние правые указатели и высокие узлы ключа** \
Дисклеймер: здесь я буду описывать подходы, значение которых я пока совсем не догоняю, но для чего-то разработчики СУБД их применяют. 
Подход крайнего правого указателя подразумевает хранение крайнего правого указателя в узле в заголовке страницы. Этот указатель не связан с каким-либо ключом. 
Подход с высоким узлом ключа подразумевает добавление еще одного ключа, который задает верхнюю границу в узле. Данный ключ является парой для крайнего правого указателя. 
Оба подхода используются в СУБД sqlite и PostgreSQL соответственно. 

**Страницы переполнения** \
Об этом я должен был, возможно, рассказать в предыдущем посте, но я скинул ссылку на статью, которая содержала в себе описание этого механизма. Лучше всего этот процесс можно понять, представив себе ситуацию. Представим, что в один из столбцов базы вы пишете целое полотно текста, полотно это настолько большое, что место в странице заканчивается уже после после половины записанного текста. В таком случае создается доп. страница переполнения, на которую ссылается страница с началом вашего полотна. Таких страниц может быть несколько в зависимости от ваших записей. Все они представляют собой связанный список. 

**Распространение операций разделения и слияния** \
Как вы уже знаете, узлы в B-Tree могут делиться и склеиваться. В таком случае нам нужно распространить эти операции в рамках всего дерева, т.е переназначить указатели. Для нахождения целевого узла нам необходимо проделать путь до корня дерева, а затем спуститься обратно рекурсивно вниз. 

**Навигационная цепочка** \
Вместо того чтобы хранить и поддерживать указатели на родительские узлы, можно хранить путь к узлу в какой-нибудь структуре данных, ну и в случае каскадного разделения или слияния можно пройти 

**Перебалансировка** \
Чтобы улучшить распределение нагрузки, некоторые реализации производят операцию перебалансировки, т.е переносят ключи и более загруженных узлов в менее загруженные узлы. 

**Добавление только справа** \
Многие СУБД используют в качестве первичного ключа авто-инкрементирующиеся значения. Это открывает возможность для оптимизации. Если вставляемый ключ больше первого ключа в крайнем правом узле, то мы можем положить ключ в закешированную крайнюю правую страницу, исключая необходимость обхода дерева в поисках нужного места. 

**Сжатие** \
Еще один необходимый элемент для эффективного хранения данных. С ростом количества данных растут и требования к объему носителей. Чтобы замедлить этот процесс, разработчики прибегают к сжатию. Алгоритмов сжатия достаточно много, но при выборе нам нужно обращать внимание на такие параметры как степень сжатия, производительность и издержки памяти.  Например, Clickhouse по дефолту использует zstd. После выбора алгоритма встает вопрос о том, что сжимать. Сжимать целый файл - как-то не комильфо. Это означает, что при каждом доступе к какой-либо из страниц нам придется вернуть весь файл в начальное состояние, достать нужную страницу и т.д. Поэтому появилась идея сжимать данные постранично. 

**Очистка и обслуживание** \
Записи в базе периодически удаляются. Конечно же, СУБД не удаляет их сразу, так как это очень дорого. Вместо этого данные помечаются нулями. Точнее, на эти данные нет никаких указателей, соответственно их можно считать мусором. Для этого в ряде СУБД существует фоновый процесс чистки, своего рода гарбэдж коллектор, который эту работу и делает. В качестве доп. чтения привожу 2 полезные статьи:
https://www.lucavall.in/blog/how-databases-store-and-retrieve-data-with-b-trees
https://fly.io/blog/sqlite-internals-btree

P.S 
Я все больше склоняюсь к тому, что на чтение одной главы этой книги нужно уделять неделю, ибо полностью понять все за сутки без наличия каких-либо знаний по проектированию СХД достаточно проблематично.

<a name="chapter5"></a> 
### Глава 5. Обработка транзакций и восстановление

Да, сегодня поговорим про транзакции, но не те транзакции, про которые я говорил в постах по битку, а про транзакции в контексте СУБД. Наряду с этим термином часто упоминают еще ACID или Atomicity, Consistency, Isolation и Durability. По порядку: 


***Атомарность***. Другими словами неделимость. Это свойство транзакций, при котором не допускается частичное выполнение команд в ее пределах. Если еще проще, то транзакция либо выполняется полностью, либо вообще не выполняется. Тут можно провести параллель с атомиками в языках программирования.

***Согласованность***. Вообще этот термин очень перегружен в среде CS, и, соответственно, может означать разное в зависимости от контекста. В данном случае означает, что база должна переходить из одного допустимого состояния в другое допустимое состояние, сохраняя все инварианты базы (включая различные ограничения, ссылочную целостность и т.д)

***Изоляция***. В современных бдшках несколько транзакций могут выполняться одновременно, ну и хорошо, если они не будут мешать друг другу. Изоляция определяет, когда изменения, сделанные одной транзакцией, должны стать видимыми другим транзакциям. Для этого люди вывели несколько уровней изоляции, о которых мы поговорим ниже. А пока только скажу, что чем сильнее уровень изоляции, тем менее производительна бдшка. 

***Долговечность***. После коммита все транзакции должны храниться на персистентном носителе, i.е диске и сохраняться в случае перебоев, отказов и т.д

Конечно же, для поддержания этих свойств в СУБД есть отдельные компоненты. Они перечислены ниже: 

***Диспетчер транзакций***. Отвечает за планирование и координацию транзакций. 

***Диспетчер блокировок***. Контролирует доступ к данным, не допуская при этом параллельных обращений, способных нарушить целостность данных. 

***Кэш страниц***. Кеширует страницы с персистентного носителя. Все операции производятся с страницами в кеше.
 
***Диспетчер журналов***. Хранит историю операций (записи журнала), примененных к кэшированным страницам, но еще не синхронизированных с персистентным носителем,
чтобы изменения не были потеряны в случае сбоя. 

#### Организация буферизации данных
Обычно в СУБД используется двухуровневая система буферизации данных. Обычно СУБД работает со страницами в кэше. Но нужна гарантия, что другие процессы не будут менять страницы на диске. Когда страница становится грязной, т.е она подверглась изменению находясь в ОЗУ, она выгружается на диск. Подобная система присутствует в ОС. Они используют незанятые сегменты памяти, чтобы прозрачно кешировать содержимое диска для повышения производительности системных вызовов ввода-вывода.   

**ОБХОД СТРАНИЧНОГО КЭША ЯДРА** 

Многие СУБД открывают файлы с помощью флага O_DIRECT. Этот флаг позволяет системным вызовам ввода-вывода обходить кэш страниц ядра, обращаясь к диску напрямую и используя управление буфером конкретной базы данных. К этому порой неодобрительно
относятся разработчики операционных систем. Линус Торвальдс не одобряет (https://databass.dev/links/32) использование флага O_DIRECT по той причине, что такой подход не асинхронен и не предусматривает упреждающее чтение или другие средства для информирования ядра о схемах доступа. Однако до тех пор,
пока операционные системы не начнут предлагать лучшие механизмы, флаг O_DIRECT по-прежнему будет полезен.
Мы можем получить некоторый контроль над процессом вытеснения страниц из кэша операционной системы, используя функцию fadvise (https://databass.dev/links/33), но такой подход позволяет лишь попросить ядро учесть наше мнение и не гарантирует, что оно так
и сделает. Чтобы отказаться от системных вызовов при вводе-выводе, мы можем использовать отображение в память, но тогда мы теряем контроль над кэшированием. Более подробно про проблему отображения в память можно узнать вот здесь https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf


#### Вытеснение из кэша 

Нужно стремиться к тому, чтобы кэш всегда был заполнен полностью. Но все страницы в кэш мы положить не можем, соответственно нам нужно удалять из кэша страницы, которые "не нужны". Беру последнее в кавычки, ибо значение этого выражения зависит от алгоритма вытеснения. В случае грязных страниц СУБД также обязан выгрузить их на диск перед удалением из кэша. Но выгружать грязные страницы при каждой операции вытеснения дорого, поэтому этим может заниматься отдельный процесс. Пример такого фонового процесса представлен в PostgreSQL. Подробнее он описан вот здесь https://www.interdb.jp/pg/pgsql08/06.html

Также важно сохранять свойство долговечности. Для этого должен существовать фоновый процесс, который будет выгружать изменения на персистентный носитель. Причем, для этого будут существовать чекпоинты. В PostgreSQL такой процесс называется checkpointer. 

#### Политики вытеснения
По политикам вытеснения напишу только, что существует всеми любимый LRU, LFU, FIFO, а также придуманный дядей Таненбаумом CLOCK. 
TODO: написать про алгоритм работы TinyLFU. 

#### Восстановление

Мы все должны признать, что мир распределенных систем нестабилен. Соответственно, нам нужно как-то уметь восстанавливать состояние СУБД до сбоя. Для этого люди придумали WAL (Write-Ahead-Logging) - append-only структура, которая выполняет следующие функции

1. Позволяет кэшу страниц буферизировать обновления размещенных на диске страниц с обеспечением долговечности в общем контексте СУБД.
1. Производит персистентное сохранение всех операций на диске до тех пор, пока кэшированные копии страниц, затронутых этими операциями, не будут синхронизированы с копиями на диске. Каждая операция, которая изменяет состояние базы данных, должна быть записана в журнале на диске до изменения содержимого соответствующих страниц.
1. Позволяет в случае сбоя восстановить из журнала операций потерянные изменения, которые были произведены в памяти


Кроме того WAL также играет немаловажную роль в обработке транзакций. Даже если данные не были зафиксированы, они будут доступны даже после сбоя. 

#### Семантика журнала 
Журнал упреждающей записи состоит из записей. Каждая запись имеет уникальный, монотонно увеличивающийся порядковый номер журнала (log sequence number, LSN). Обычно этот номер представляет собой внутренний счетчик или временную метку. Поскольку записи журнала не всегда занимают целый дисковый блок, их содержимое кэшируется в буфере журнала и выгружается на диск в ходе принудительной выгрузки. Принудительная выгрузка выполняется по мере заполнения буферов журнала и может запрашиваться диспетчером транзакций или кэшем страниц. Все записи
журнала должны выгружаться на диск согласно порядковому номеру журнала LSN.

#### Использование операций и журнала данных 
Тут автор начал с описания техники shadow paging, ну а потом перепрыгнул на описание логического и физического журнала. Не совсем понял этот момент, но опишу только разницу между логическим и физическим журналом. Логический журнал представляет собой историю операций, которые были произведены СУБД. Например, там могут быть записи типа *вставка записи данных X для ключа Y*. Физический журнал, в свою очередь, хранит историю изменений на физическом носителе, например, изменения в какой-либо странице и т.д. Современные СУБД комбинируют оба подхода. 

#### Политики кражи и принуждения
Для определения подходящего времени выгрузки на диск произведенных в памяти изменений СУБД используют политики "кражи"/"без кражи" и "принуждения"/"без принуждения".
Если коротко, то политика "кражи" позволяет СУБД выгружать измененные транзакцией страницы на диск еще до фиксации для того чтобы освободить место в кеше страниц, тогда как политика "без кражи" не позволяет делать это. Политика "принуждения" требует, чтобы все измененные транзакцией страницы были выгружены на диск еще до фиксации тогда как политика "без принуждения" позволяет транзакции зафиксироваться даже если измененные страницы не были выгружены на диск. 

#### ARIES 
Одним из примеров алгоритмов для восстановления после сбоя в СУБД служит алгоритм ARIES. Он принадлежит к группе алгоритмов с типом кражи без принуждения (steal/no force). Для своей работы он полагается на WAL. Согласно этому алгоритму в ходе перезапуска после сбоя восстановление происходит в 3 этапа

1. На этапе анализа выявляются "грязные" страницы в кэше страниц и транзакции, выполнявшиеся во время сбоя. Информация о "грязных" страницах используется для определения начальной точки для этапа повтора. Список незавершенных транзакций используется на этапе отмены для их отката
2. Этап повтора вновь применяет всю историю вплоть до момента сбоя и восстанавливает прежнее состояние БД. Этот этап выполняется для незавершенных транзакций, а также для транзакций, которые были зафиксированы без выгрузки их содержимого на персистентный носитель.
3. Этап отмены откатывает все незавершенные транзакции и восстанавливает последнее согласованное состояние БД. Все операции откатываются в обратном хронологическом порядке. На случай повторного сбоя БД во время восстановления все операции, отменяющие транзакции, также регистрируются в журнале во избежание их повторного выполнения.

Алгоритм полагается на физический лог для повтора и логический лог для отмены транзакций. 

#### Управление параллелизмом
Управление параллелизмом - это набор методов, обеспечивающих взаимодействие параллельно выполняющихся транзакций. Эти методы можно разделить на следующие категории: 

***Оптимистичное управление параллелизмом (Optimistic concurrency control, OCC)***
Позволяет транзакциям производить параллельное чтение и запись данных, ну а перед фиксацией транзакции проверяются на возможные конфликты. В случае возникновения таких конфликтов одна из транзакций откатывается

***Управление параллелизмом с несколькими версиями (multiversion concurrency control, MVCC)***
Оперирует версиями данных. Таким образом, каждая запись имеет временную метку (или версию). Этот подход может быть реализован с помощью методов проверки, позволяющих выиграть только одной из обновляющихся или фиксирующихся транзакций, а также с помощью методов без блокировки, таких как сортировка по временным меткам, или же методов на основе блокировки, таких как двухфазная блокировка (2PL). Используется в PostgreSQL btw

***Пессимистичное управление параллелизмом (pessimistic concurrency control, PCC)***
Существуют как блокирующие, так и неблокирующие консервативные методы управления паралеллизмом. Методы на основе блокировок требуют, чтобы транзакции не получали доступ к данным, если эти на этих данных уже стоит блокировка. Неблокирующие методы оперируют списками чтения и записи и ограничивают выполнение в зависимости от планирования незавершенных транзакций. Блокирующие методы могут приводить к дедлокам. 


**Сериализуемость (Serializable)** 
Под сериализуемостью в области СУБД понимают свойство, при котором параллельное выполнение операций можно свести к последовательному. Сериализуемость также является самым сильным уровнем изоляции (о них поговорим ниже).  

***Уровни изоляции***
Наверное, самый любимый вопрос интервьюверов в секции БД. Ниже разберем виды аномалий, уровни изоляций, а в конце я приведу очень наглядную таблицу, где изображено наличие аномалий в каком-либо из уровней. Начнем с аномалии чтения. 

***Грязное чтение (Dirty read)*** - аномалия, при которой транзакции могут читать незафиксированные данные 

***Неповторяемое чтение (Nonrepeatable read)*** - аномалия, при которой транзакция запрашивает одну и ту же строку дважды, и при этом получает разные данные

***Фантомное чтение (Phantom read)*** - То же самое, что неповторяемое чтение, но тут речь идет уже о целом диапазоне данных

Наряду с аномалиями чтения существуют еще и аномалии записи

***Потерянное обновление (Lost update)*** - аномалия, при которой обе транзакции меняют значение какой-либо записи, и при этом одна из записей не учитывается (или теряется)

***Грязная запись (Dirty write)*** - ситуация, при которой транзакция осуществляет "грязное" чтение, изменяет целевую запись и фиксирует это изменение. 

***Искажение записи (write skew)*** - происходит, когда транзакции соблюдают требуемые инварианты, но комбинация их действий не удовлятворяет общему результату. В книге приводится следующий пример 
```
Например, транзакции T1 и T2 изменяют значения двух счетов A1 и A2.
Вначале A1 содержит 100$, а A2 — 150$. Значение счета может быть отрицательным,
если сумма двух счетов неотрицательна: А1 + А2 >= 0. Транзакция T1 пытается снять
200$ со счета A1, а транзакция T2 — со счета A2. Так как в момент начала этих транзак-
ций A1 + A2 = 250$, то в сумме доступно 250$. Обе транзакции предполагают, что они
сохраняют инвариант и могут быть зафиксированы. Однако после фиксации счет A1
будет содержать 100$, а счет A2 — –50$, что явно нарушает требование о том, чтобы
сумма счетов всегда была положительной
```
Все понятно до момента с остатком на первом счете 100$, а на втором - -50$. 

**Уровни изоляции**

***Read uncommitted***. Является самым слабым уровнем изоляции, который допускает "грязное" чтение.
***Read committed***. Посильнее read uncommitted, так как не допускает "грязного" чтения, но допускает аномалию неповторяемого чтения. 
***Repeatable read***. Решает проблему неповторяемого чтения у read committed. 
***Serializable***. Как уже было сказано, является самым сильным уровнем изоляции. Если дядюшка Бен был говорил про сериализуемость, то фраза звучала бы что-то типа "With great power comes great perfomance impact". Да, работая в таком уровне не ожидайте хорошей производительности от СУБД. 

Ну а теперь приведу обещанную красивую таблицу из книги

<img width="787" alt="isolation-anomaly-pic.jpg" src="https://github.com/user-attachments/assets/20cebfb7-4338-48ea-845e-edbe77774378">


Дальше в книге идет описание методов контроля управления параллелизмом, как СУБД решают проблему дедлоков, также уделяется место видам блокировок (обычные блокировки, защелки (latches), блокировки чтения-записи (аля rw mutex)) и даже говорится про B-link деревья. Это я решил пока не писать, дабы уменьшить размер конспекта. Возможно, я сделаю это когда-нибудь в будущем, но это не точно. 

<a name="chapter6"></a>
### Глава 6. Варианты B-деревьев
TODO: вернуться к этой главе, на данный момент она вообще doesn't make sense с моими текущими знаниями в области СУБД-строения. 

<a name="chapter7"></a>
### Глава 7. Журналированное хранилище

