<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Docker_lecture_1</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><h1 id="docker">Docker</h1>
<h2 id="cgroups">Cgroups</h2>
<p>Фича ядра linux, позволяющая задавать лимиты процессам на различные ресурсы системы, такие как CPU, RAM, Disk и тд. Сам по себе cgroups не имеет занимается изоляцией процессов, этим занимается linux namespaces, который позволяет создавать независимые деревья процессов, не имеющие доступа к соседним namespaces. Кстати, chrome использует namespaces для изоляции процессов вкладок. Таким образом docker это cgroups+namespaces</p>
<h2 id="основные-понятия">Основные понятия</h2>
<h3 id="image">Image</h3>
<p>Это слепок файловой системы в которой будет запущен процесс. Если утрировать, то это JSON + tar.gz, внутри которого по нужным путям разложены файлы. На самом деле image композитные и состоят из нескольких последовательных layer.</p>
<h4 id="layer">Layer</h4>
<p>Layer это один слой файловой системы, содержащий набор изменений, отличающих его от предыдущего слоя. ВАЖНО: layer-ы могут шариться между разными image.</p>
<p>Если совсем просто, то Image это набор .tar.gz архивов, которые последовательно в строгом порядке распаковываются. То что получилось после распаковки и есть результирующая файловая система с которой будет работать процесс контейнера. Каждый image обязательно имеет <strong>ID</strong> и может иметь <strong>tag</strong> (об этом дальше)</p>
<h3 id="container">Container</h3>
<p>Container это процесс/процессы, запущенные на основе ФС какого-то image. Процессы могут завершаться, тогда завершается и контейнер, а могут выполняться бесконечно долго. Если завершаются все процессы, наследники корневого (PID 1) то завершает работу контейнер и наоборот. Важная особенность - контейнер (и все его процессы) может быть <strong>приостановлен</strong>, а затем исполнение может быть возобновлено.<br>
Важно понимать, что контейнер - это image + запущенные в нем процессы + все изменения на ФС. Т.е. при удалении контейнера все данные теряются</p>
<h3 id="volume">Volume</h3>
<p>Volume - том, подключаемый к container. Можно рассматривать как внешний диск. Есть масса возможностей/сценариев. Два основных:</p>
<ul>
<li>подмонтировать локальную папку внутрь container. В этом случае процессы могут читать из локальной папки host OS и могут писать в нее. Соответственно так изменения могут переживать пересоздание контейнера</li>
<li>подмонтировать папку из другого контейнера. Обычно используется в сложных сценариях развертывания (например kubernetes), когда один контейнер готовит данные (например клонирует git-репозиторий) для второго.</li>
</ul>
<h2 id="базовые-команды">Базовые команды</h2>
<p><em><strong>docker pull</strong></em> - скачивает на локальную машину image<br>
<em><strong>docker run</strong></em> - запускает контейнер<br>
<em><strong>docker stop</strong></em> - останавливает контейнер<br>
<em><strong>docker start</strong></em> - возобновляет исполнение контейнера<br>
<em><strong>docker restart</strong></em> - перезапускает контейнер<br>
<em><strong>docker rm</strong></em> - удаляет запущенный или остановленный контейнер</p>
<ol>
<li>Запуск контейнера - docker run. Имя имейджа всегда последнее! После него могут идти только аргументы для запускаемого процесса. Ну и всегда стоит задавать --name для контейнера</li>
</ol>
<pre class=" language-bash"><code class="prism  language-bash">docker run --rm -p 54321:5432 --name db -e POSTGRES_PASSWORD<span class="token operator">=</span>password postgres
</code></pre>
<p><img src="https://sirvaulterscoff.github.io/images/docker_run.png" alt="docker run example"></p>
<ol start="2">
<li>Посмотреть что запущено. ps покажет работающие контнейнеры, ps -a все</li>
</ol>
<pre class=" language-bash"><code class="prism  language-bash">docker <span class="token function">ps</span>
</code></pre>
<p><img src="https://sirvaulterscoff.github.io/images/docker_ps_example.png" alt="docker ps"><br>
3. Остановить, удалить, перезапустить - docker stop/docker rm (docker rm -f удалит запущенный) /docker start / docker restart. В качестве аргумента передаем или имя (–name аргумент для docker run) или container id</p>
<pre><code> docker rm -f db
</code></pre>
<h2 id="dockerfile">Dockerfile</h2>
<p>Dockerfile описывает какие изменения нужно внести в ФС, чтобы получить какой-то новый образ.</p>
<pre class=" language-dockerfile"><code class="prism  language-dockerfile"><span class="token keyword">FROM</span> jdk<span class="token punctuation">:</span>openjre11<span class="token punctuation">-</span>alpine  

<span class="token keyword">ENV</span> APP_ARGS  
<span class="token keyword">WORKDIR</span> /opt/app 
<span class="token keyword">EXPOSE</span> 8080  
<span class="token keyword">COPY</span> build/libs/app.jar .
<span class="token keyword">RUN</span> apk add fontconfig
  
<span class="token keyword">ENTRYPOINT</span> <span class="token punctuation">[</span><span class="token string">"java"</span><span class="token punctuation">,</span> <span class="token string">"-jar"</span><span class="token punctuation">,</span> <span class="token string">"/opt/app/app.jar"</span><span class="token punctuation">,</span> <span class="token string">"$APP_ARGS"</span><span class="token punctuation">]</span>
</code></pre>
<p>Ключевые элементы</p>
<ol>
<li>FROM указывает образ на основании которого нужно собрать новый. Т.е. берем базовый образ, в него добавляем (или удаляем из него) файлы, получаем новый образ. Совсем пустой образ - scratch</li>
<li>ENV - декларируем, что есть переменная. Может быть использована внутри Dockerfile. Например, можно использовать в COPY, но нужно понимать, что в этом случае ее значение будет использовано один раз. Значение может быть задано при запуске через -e. Если переменная не определена через ENV не означает, что ее значение нельзя задать через -e</li>
<li>WORKDIR - просто указываем в какой директории выполнять последующие команды</li>
<li>EXPOSE - указываем какие порты открывает приложение. Опять же, больше информационный параметр. Можно будет опубликовать любой порт через аргумент -p</li>
<li>COPY - первый главный  инструмент модификации файловой системы образа. Копирует файл (файлы) из директории, где был запущен docker build в какую-то директорию внутри image (точнее layer). Каждая команда COPY создает отдельный layer</li>
<li>RUN запускает какую-то команду внутри  образа при сборке. Запустить можно только те команды, которые присутствуют в собираемом образе или в базовых. Все что команда запишет на ФС будет сохранено как отдельный layer. Для уменьшения количества слоев нужно объединять неск команд в один RUN</li>
<li>ENTRYPOINT - какую команду docker выполнит при запуске контейнера. Есть еще CMD. Разница в том, что при использовании CMD docker запустит bash и передаст ему CMD в виде аргумента.</li>
</ol>
<p>Как собрать образ</p>
<pre class=" language-bash"><code class="prism  language-bash">docker build <span class="token keyword">.</span> -t my-app:version
</code></pre>
<p><img src="https://sirvaulterscoff.github.io/images/docker_build.png" alt="enter image description here"></p>
<p>Что происходит во время выполнения команды:</p>
<ul>
<li>содержимое указанной директории упаковывается архив и передается сервису докера - это контекст. Таким образом, в Dockerfile нельзя указывать команды вида COPY …/…/somefile т.к. такого файла просто не будет в контексте</li>
<li>выполняются все команды из файла. Полученные в процессе layer объединяются в image</li>
<li>проставляется имя + тег для созданного image. Это идентично выполнению команды docker tag imageid myapp:version</li>
</ul>
<p>Для чего имена и теги:</p>
<ol>
<li>по имени проще работать с image</li>
<li>теги позволяют задавать версии. Есть специальный тег :latest - он выкачивает всегда последнюю версию. Использовать :latest во FROM крайне не рекомендуется, т.к. сборка может в один прекрасный момент поломаться.</li>
<li>имена так же указываю в какой docker registry отправить image. Например <a href="http://nexus.company.ru/docker/myapp:1.0">nexus.company.ru/docker/myapp:1.0</a> позволит затем выполнить docker push и опубликовать этот image в общем репозитории. Некоторые репозитории требуют аутентификации - для это цели docker login</li>
</ol>
<h2 id="we-need-to-go-deeper">We need to go deeper</h2>
<p>Как можно взаимодействовать с тем, что запущено ?</p>
<ol>
<li>docker exec - позволяет выполнить команду внутри запущенного container</li>
</ol>
<pre class=" language-bash"><code class="prism  language-bash">docker <span class="token function">exec</span> -it db /bin/bash
</code></pre>
<p><img src="https://sirvaulterscoff.github.io/images/docker_exec.png" alt="enter image description here"></p>
<ol start="2">
<li>docker cp - позволяет копировать файлы в/из запущенного контейнера</li>
</ol>
<pre class=" language-bash"><code class="prism  language-bash">docker <span class="token function">cp</span> README.md db:/tmp
</code></pre>
<h2 id="more-advanced-stuff">More advanced stuff</h2>
<p>Build container. Это такая штука, которая позволяет разделить сборку результирующего docker image на две фазы. В первой фазе собирается промежуточный docker image внутри которого запускаются команды. Затем собирается финальный docker image в который можно будет скопировать часть файлов из первого контейнера</p>
<pre class=" language-dockerfile"><code class="prism  language-dockerfile"><span class="token keyword">FROM</span> gradle as builder
<span class="token keyword">RUN</span> ./gradlew clean build

<span class="token keyword">FROM</span> gradle
<span class="token keyword">COPY</span> <span class="token punctuation">-</span><span class="token punctuation">-</span>from=build ~/.m2 ~/.m2
<span class="token keyword">ENTRYPOINT</span> <span class="token punctuation">[</span><span class="token string">"gradle"</span><span class="token punctuation">]</span>
</code></pre>
<p>Так, например, можно собрать image, который помимо gradle будет так же содержать и maven репозиторий со скачанными зависимостями.</p>
</div>
</body>

</html>
