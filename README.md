# Erlang Rebar and GProc Tutorial

## Использование Rebar

Для начала создаём директорию, в которой будет находиться наш проект (`gpt`) и переходим в неё:

<pre>
$ mkdir gpt && cd gpt
</pre>

В ней создадим поддиректорию, где будем сохранять исходные файлы непосредственно нашего приложения:

<pre>
$ mkdir -p apps/gpt && cd apps/gpt
</pre>

Делаем скелет приложения:

<pre>
$ rebar create-app appid=gpt
</pre>

Параметр appid определяет имя нашего приложения и соответственно префикс исходных файлов.

<pre>
$ ls -1 src
gpt_app.erl
gpt.app.src
gpt_sup.erl
</pre>

Добавляем в заготовку для .app-файла (`src/gpt.app.src`) описание приложения и зависимость от gproc:

<pre>
  {description, "GProc tutorial"},
  ...
  {applications,
   [
    kernel,
    stdlib,
    gproc     % <--- Приложение зависит от gproc
   ]},
  ...
</pre>

Возвращаемся назад в директорию верхнего уровня, где хранится наш проект, создаём в ней поддиректорию `rel` и переходим туда:

<pre>
$ cd ../../
$ mkdir rel && cd rel
</pre>

В rel будут лежать файлы, необходимые для создания релиза — всего того, что требуется для запуска проекта, все его runtime-зависимости.

С помощью `rebar` создадим заготовку для ноды, передав её имя в параметре `nodeid`:

<pre>
$ rebar create-node nodeid=gptnode
</pre>

Редактируем файл `reltool.config`:

<pre>
  ...
  {lib_dirs, ["../deps", "../apps"]},    % <--- В этих директориях reltool будет искать зависимости и наше приложение
  {rel, "gptnode", "1",
   [
    kernel,
    stdlib,
    sasl,
    gproc,     % <--- Приложение gproc
    gpt        % <--- Наше приложение
   ]},
  ...
</pre>

Далее можно отредактировать файл `files/vm.args`, изменив, допустим, имя ноды:

<pre>
-name gptnode@127.0.0.1
</pre>

на

<pre>
-sname gptnode@localhost
</pre>

Вернёмся в директорию верхнего уровня:

<pre>
$ cd ../
</pre>

и создадим файл `rebar.config` со следующим содержанием:

<pre>
%% Здесь будут лежать зависимости
{deps_dir, ["deps"]}.

%% Поддиректории, в которые rebar должен заглядывать
{sub_dirs, ["rel", "apps/gpt"]}.

%% Опции компилятора
{erl_opts, [debug_info, fail_on_warning]}.

%% Список зависимостей
%% В директорию gproc будет клонирована ветка master соответствующего git-репозитория.
{deps,
 [
  {gproc, ".*", {git, "http://github.com/esl/gproc.git", "master"}}
 ]}.
</pre>

Теперь у нас всё готово для создания релиза. Выполним несколько команд `rebar` (вывод команд опущен):

<pre>
$ rebar get-deps
$ rebar compile
$ rebar generate
</pre>

Команда `get-deps` скачивает зависимости. В нашем случае, это приложение gproc. Команда `compile`, очевидно, вызывает компиляцию всех исходных файлов, а `generate` создаёт релиз.

Директорию `rel/gptnode` можно смело перемещать на другие хосты (разумеется, при условии бинарной совместимости, так как релиз включает в себя и виртуальную машину Erlang). После создания релиза запускаем то, что получилось:

<pre>
(cd rel/gptnode && sh bin/gptnode console)
</pre>

Убедимся в том, что все нужные приложения запущены:

<pre>
(gptnode@localhost)1> application:which_applications().
[{sasl,"SASL  CXC 138 11","2.1.9.2"},
 {gpt,"GProc tutorial","1"},
 {gproc,"GPROC","0.01"},
 {stdlib,"ERTS  CXC 138 10","1.17.2"},
 {kernel,"ERTS  CXC 138 10","2.14.2"}]
</pre>

Нас интересуют gpt и gproc. Как видно, они в этом списке присутствуют.

## Использование gproc

Итак, с `rebar` разобрались, научились создавать простенький проект и работать с ним. Приступим к gproc.

Как известно, приложения в Erlang, как правило, состоят из множества процессов, которые обмениваются сообщениями.

Чтобы процессы знали, кому какое сообщение посылать, необходимо иметь регистратор, преобразующий какие-то координаты в идентификатор процесса. По умолчанию Erlang/OTP предоставляет регистрацию процессов под именем-атомом. Это расточительно, так как атомы не собираются сборщиком мусора, и, создавшись один раз, живут до завершения работы всей ноды, что обязательно приведёт к исчерпанию всей памяти при необходимости регистрировать процессы под уникальными именами. К тому же подобный подход неудобен, так как пришлось бы преобразовывать разные термы в атом, сочинять для этого какие-то правила, кроме того, процесс можно зарегистрировать только под одним именем. Регистрация процессов под именем-атомом с помощью функции `erlang:register/2` допустима лишь для небольшого числа долгоживущих процессов, имя которых не должно меняться, аналог — глобальные переменные в императивных языках программирования.

Для обхода этих ограничений зачастую используется следующая схема:

1. запускается процесс-регистратор, создающий ets-таблицу и являющийся её владельцем;
2. при запуске процессов, нуждающихся в регистрации, они посылают регистратору сообщение, содержащее координаты для регистрации (любой erlang-терм) и свой идентификатор;
3. регистратор записывает это сопоставление в ets-таблицу и включает мониторинг процесса с помощью `erlang:monitor/2`;
4. регистрируемый процесс при завершении либо явно посылает сообщение о своей дерегистрации, либо регистратор получает сообщение `'DOWN'` при падении этого процесса, после чего удаляет его запись из ets-таблицы;

Схема эта очень часто используется, почти в каждом приложении есть своя реализация, со своими особенностями и багами. Разумеется, что возникает естественное желание заменить этот регистратор на что-то единственное. И решение проблемы пришло в виде разработчика Ulf Wiger и его приложения gproc (https://github.com/esl/gproc).

С API приложения можно ознакомиться на странице https://github.com/esl/gproc/blob/master/doc/gproc.md.

### Локальная регистрация

Рассмотрим простейший случай — локальная (на текущей ноде) регистрация процесса по произвольному терму.

Исходный код для примеров можно взять здесь: http://github.com/Zert/gproc-tutorial.git

Код процессов, которые мы будем регистрировать через gproc, находится в файле `gpt_proc.erl`. В `gpt_sup.erl` находится код супервизора этой группы процессов. При вызове функции `gpt_sup:start_worker/1` будет запускаться наш процесс и регистрироваться под тем именем, которое передаётся в функцию в качестве единственного аргумента. В данном случае, это число.

Запустили ноду, используя вышеупомянутую команду, и выполнили в ней серию запусков процессов с разными идентификаторами:

<pre>
(gptnode@localhost)1> [gpt_sup:start_worker(Id) || Id <- lists:seq(1,3)].
(gpt_proc:29) Start process: 1
(gpt_proc:29) Start process: 2
(gpt_proc:29) Start process: 3
[{ok,<0.61.0>},{ok,<0.62.0>},{ok,<0.63.0>}]
</pre>

Вызов функции `gproc:add_local_name(Name)` регистрирует процесс, её вызывающий, под именем `Name` (эта функция является просто обёрткой над `gproc:reg({n,l,Name})`, где `n` — `name`, `l` — `local`). После этого функция `gproc:lookup_local_name(Name)` будет возвращать идентификатор процесса.

Теперь скажем одному из процессов, чтобы он начал ждать запуска и регистрации процесса с именем 4. Код, отвечающий за это:

<pre>
handle_info({await, Id},
            #state{id = MyId} = State) ->
    gproc:await({n, l, Id}),
    ?DBG("MyId: ~p.~nNewId: ~p.", [MyId, Id]),
    {noreply, State};
</pre>

Здесь функция `gproc:await/1` вызывается с аргументом, имеющим следующий вид: `{n, l, Id}`. Почему-то она не имеет обёртки, ну да ладно.

<pre>
(gptnode@localhost)2> gproc:lookup_local_name(1) ! {await, 4}.
{await,4}
</pre>

Запустив процесс с идентификатором 4, увидим сначала сообщение от него, а затем от первого ждущего процесса:

<pre>
(gptnode@localhost)3> gpt_sup:start_worker(4).
(gpt_proc:29) Start process: 4
(gpt_proc:45) MyId: 1.
NewId: 4.
{ok,<0.66.0>}
</pre>

Сделаем остановку процесса по приёму сообщения `stop`:

<pre>
handle_info(stop, State) ->
    {stop, normal, State};
</pre>

и остановим его:

<pre>
(gptnode@localhost)4> gproc:lookup_local_name(1) ! stop.
stop
</pre>

После этого процесс автоматически удаляется из базы данных регистратора:

<pre>
(gptnode@localhost)5> gproc:lookup_local_name(1).
undefined
</pre>

### Глобальная регистрация

Общеизвестно, что Erlang является дико распределённым. Под этим подразумевается прозрачный обмен сообщениями между узлами, другими словами, имея идентификатор процесса, можно послать ему сообщение, не зная, на каком узле он находится. Локальная регистрация процесса с помощью gproc позволяет сопоставить произвольный терм идентификатору процесса в пределах одной ерланг-ноды, при этом на другой ноде нельзя получить значение идентификатора, используя этот терм.

Для того, чтобы любая нода в кластере имела возможность регистрировать свои процессы так, чтобы они были доступны и с других нод, имеется глобальная регистрация. GProc реализует вызов `gproc:add_global_name/1`, позволяющий осуществлять это действие. Рассмотрим на примере.

Сначала соорудим две ноды, объединённые в кластер, и `rebar` нам в этом поможет, так как имеет возможность создавать конфигурационные файлы по заданному шаблону. При создании кластера необходимо учесть следующие детали:

 * Установить одинаковые значения cookie у узлов
 * Задать им разные имена
 * Передать соответствующие параметры приложению обязательному `kernel` на каждом узле
 * При использовании GProc, передать ему желаемые роли узлов

Первые два пункта устанавливаются в файле `files/vm.args`:

<pre>
## Name of the node
-sname {{node}}

## Cookie for distributed erlang
-setcookie gptnode
</pre>

Здесь `{{node}}` — плейсхолдер, который будет заполняться при создании релиза. Параметр виртуальной машины `-setcookie` устанавливает значение cookie для этой ноды, в кластере у всех нод эти значения должны быть одинаковыми.

Вторые два пункта устанавливаются в файле `files/app.config`. Здесь тоже будут использоваться плейсхолдеры:

<pre>
 %% GProc
 {gproc, {{ gproc_params }} },

 %% Kernel
 {kernel, {{ kernel_params }} },
</pre>

Для заполнения плейсхолдеров укажем в файле `reltool.config`, что предыдущие два файла надо обрабатывать как шаблоны:

<pre>
  {template, "files/app.config", "etc/app.config"},
  {template, "files/vm.args", "etc/vm.args"}
</pre>

Создаём два конфигурационных файла, по одному на каждую ноду: `vars/dev1_vars.config` и `vars/dev2_vars.config`. В файле `dev1_vars.config` будут находиться следующие значения плейсхолдеров:

<pre>
%% etc/app.config
{gproc_params,
"[
  {gproc_dist, {['gpt1@localhost'],
                [{workers, ['gpt2@localhost']}]}}
 ]"}.

{kernel_params,
"[
  {sync_nodes_mandatory, ['gpt2@localhost']},
  {sync_nodes_timeout, 15000}
 ]"}.

%% etc/vm.args
{node,         "gpt1@localhost"}.
</pre>

Для файла `dev2_vars.config` параметры `sync_nodes_mandatory` и `node` поменяются местами. Разберём их подробнее.

Параметр `gproc_dist` относится к приложению gproc, он является кортежом из двух списков. Первый список — узлы, которые способны становиться лидером (master), второй список содержит key-value кортежи, нам пока достаточно лишь одного ключа — `workers`, который задаёт список нод, являющихся простыми участниками кластера (slave).

К приложению kernel относятся два параметра. Первый, `sync_nodes_mandatory` — список нод, которые обязаны присутствовать в кластере. Второй, `sync_nodes_timeout` — время в миллисекундах, которое каждый из узлов будет ожидать появления узлов из предыдущего списка. Если в течение этого времени ноды не появились, то нода остановится. Сделаем его значение 15 секунд, дабы успеть запустить их оба руками.

Значение `node` будет записано в параметры запуска виртуальной машины, это её имя.

Теперь создадим два релиза с помощью следующего правила из Makefile:

<pre>
dev1 dev2:
	mkdir -p dev
	(cd rel && rebar generate target_dir=../dev/$@ overlay_vars=vars/$@_vars.config)
</pre>

Переходим в директорию `dev/dev1`, запускаем второе окно терминала (или создаём новое окно в screen`), переходим в директорию `dev/dev2`. Запускаем в каждом шелле `./bin/gptnode console`. Посмотрим список доступных нод в первом ерланговском шелле:
<pre>
(gpt1@localhost)1> nodes().
[gpt2@localhost]
</pre>

Видим, что вторая нода нормально запустилась и подключилась к кластеру. Чтобы долго не мудрить, зарегистрируем глобально процесс текущего шелла под каким-нибудь термом:

<pre>
(gpt1@localhost)2> gproc:add_global_name({shell, 1}).
true
</pre>

В другом окне попробуем запросить идентификатор процесса по этому терму:

<pre>
(gpt2@localhost)2> gproc:lookup_global_name({shell, 1}).
<3358.70.0>
</pre>

Как видим, успешно. Послав этому процессу сообщение, мы сможем получить его на первой ноде:

<pre>
(gpt2@localhost)3> gproc:lookup_global_name({shell, 1}) ! {the,message}.
message
</pre>

Читаем его на первой ноде командой `flush()`:

<pre>
(gpt1@localhost)3> flush().
Shell got {the,message}
ok
</pre>

### Заключение

На этом всё. Статью начал писать для себя, так как документация по rebar очень скудная и постоянно забывается после очередного использования. Попутно начал использовать gproc, и чтобы два раза не вставать, разместил всё в одной статье.
