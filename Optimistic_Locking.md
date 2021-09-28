---


---

<h1 id="о-синхронизации-потоков-через-субд"><em>О синхронизации потоков через СУБД</em>Есть целый класс задач, когда необходимо выполнить какую-то обработку данных на нескольких узлах (экземплярах микросервиса) при этом необходимо обеспечить синхронизацию этих независимых узлов так, чтобы только один обрабатывал порцию данных в один момент времени. Kafka из коробки предлагает механизм партицирования, но к сожалению такое решение не всегда применимо</p>
<h2 id="формулировка-задачи"><strong>Формулировка задачи</strong></h2>
<p>В таблицу Tasks помещаются строки для обработки, каждая содержит ID, статус, дата создания. Необходимо сделать так, чтобы каждый из экземпляров микросервиса обрабатывал только одну запись, и каждая уникальная запись обрабатывалась только одним микросервисом.</p>
Задачи выполняются по мере их поступления в порядке FIFO (т.е. сортирует по дате создания).</p>
<p>Первое требование выполняется элементарно: достаточно лишь  обработку строк таблицы выполнять в одном потоке. А вот второе требование чуть сложнее. Рассмотрим варианты:</p>
<h3 id="пессимистчная-блокировка"><strong>Пессимистчная блокировка</strong></h3>
<p>Тут все очень просто: многие БД предоставляют доступ к инструментам блокировки строк и таблиц, например [<a href=" семантику select for uphttps://www.postgresql.org/docs/9.1/explicit-locking.html">postgres предоставляет date</a>](<a href=")]([https://www.postgresql.org/docs/9.1/explicit-locking.htmlhttps://www.postgresql.org/docs/9.1/explicit-locking.html</a>)</p>
<p>Этот способ вполне ок, до тех пор пока вы можете обеспечить минимальный период блокировки.</p>
Наш пример может выглядеть так:</p>
<pre class=" language-sql"><code class="prism  language-sql">
<span class="token keyword">select</span> <span class="token operator">*</span>

<span class="token keyword">from</span> Tasks

<span class="token keyword">where</span>  <span class="token keyword">status</span><span class="token operator">=</span><span class="token string">'READY'</span>

<span class="token keyword">order</span> <span class="token keyword">by</span> created_date <span class="token keyword">asc</span>

<span class="token keyword">limit</span>  <span class="token number">1</span>

<span class="token keyword">FOR</span>  <span class="token keyword">UPDATE</span>  SKIP  LOCKED<span class="token punctuation">;</span>

</code></pre>
<p>Тут два ключевых момента. С помощью <code>FOR UPDATE</code>` мы указываем, что необходимо использовать блокировку строки в EXCLUSIVE режиме. С помощью <code>SKIP LOCKED</code> мы указываем на необходимость пропустить заблокированные строки.</p>
<p>Таким образом, полная обработка в нашем случае будет выглядеть как-то так:</p>
<pre class=" language-sql"><code class="prism  language-sql">
<span class="token keyword">select</span> <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span> <span class="token keyword">FOR</span>  <span class="token keyword">UPDATE</span>  SKIP  LOCKED<span class="token punctuation">;</span>

  

<span class="token comment">-- some row processing</span>

<span class="token comment">--</span>

<span class="token comment">-- end processing</span>

  

<span class="token keyword">update</span> Tasks <span class="token keyword">set</span>  <span class="token keyword">status</span><span class="token operator">=</span><span class="token string">'COMPLETED'</span>  <span class="token keyword">where</span>  id<span class="token operator">=</span>?<span class="token punctuation">;</span>

<span class="token keyword">commit</span><span class="token punctuation">;</span>

</code></pre>
<p>Несложно понять, что граница блокировки здесь охватывает весь блок, в т.ч. обработку записей</p>
<h3 id="оптимистичная-блокировка"><strong>Оптимистичная блокировка</strong></h3>
<p>**

В отличие от пессимистичной (которая исходит из предположения, что эту строку точно кто-то кроме нас попытается обработать) оптимистичная блокировка рассчитана на более редкие коллизии. На то она и оптимистичная. Вариантов реализации оптимистичной блокировки масса, покажу лишь общий принцип.</p>
<p>

Суть заключается в том, что мы не используем эксклюзивные блокировки, предоставляя БД решать какая транзакция может накатиться, а какая нет. Самый простой и, пожалуй, распространненый способ реализации оптимистичной блокировки - колонка с версией записи. В качестве версии может выступать как число-счетчик изменений, так и timestamp последнего изменения записи.</p>
<pre><code>
select  id, version

from Tasks

where  status='READY'

order by created_date asc

limit  1;

  

-- some row processing

--

-- end processing

  

update Tasks set  status='COMPLETED', version=version+1

where  id=? and  version=?;

commit;

</code></pre>
<p>На что тут обратить внимание:</p>
<ul>
<li>
<p>

- мы выбираем не только идентификатор строки, но и ее версию</p>
</li>
<li>
<p>после обработки строки мы в условие дописываем критерий version=? куда передаем значение версии строки, вычитанное на первом этапе</p>
</li>
</ul>
<p>Таким образом, если какая-то транзакция обновила строку, пока мы ее обрабатывали update не выполнит никаких изменений, т.к. строка просто не подойдет по условию</p>
<p>Тут важный момент: все обновления таблицы должны использовать версионированную колонку.</p>
<p>

Как несложно заметить это решение не создает экслюзивной блокировки, но и не помогает в достижении нашей цели. Более того, в этом примере результат обработки, фактически, оказывается потеряным, если параллельная транзакция обновила строку.</p>
<p>Но, вернемся к нашей цели. Напомню, нам нужно позволить двум экземплярам сервиса одновременно обрабатывать разные строки.</p>
<p>

Тут возможен следующий алгоритм действий</p>
<ol>
<li>Каждый из микросервисов отбирает набор строк для обработки:</li>
</ol>
<pre><code>
select  id, version

from Tasks

where  status='READY'

order by created_date asc

limit N;

</code></pre>
<p>Значение N должно определяться количеством потоков, которые могут одновременно обрабатывать строки, т.к. при использовании этого подхода часть отобранных строк может успеть попасть в обработку и нам придется их пропустить</p>
<ol start="2">
<li>По каждой строке пытаемся обновить статус</li>
</ol>
<pre><code>
update Tasks

set  version=X, status='PROCESSING'

where id=? and version=X

commit;

</code></pre>
<p>Если в результате апдейта не была изменена ни одна строка (это можно выяснить, например, использую семантику [<a href="https://postgrespro.ru/docs/postgresql/9.5/dml-returning">returning</a>](<a href=")]([https://postgrespro.ru/docs/postgresql/9.5/dml-returninghttps://postgrespro.ru/docs/postgresql/9.5/dml-returning</a>)) то необходимо перейти к следующей строке</p>
<h4 id="что-важно-понимать"><strong>Что важно понимать</strong></h4>
<p>В этом примере нет долгоживущих эксклюзивных блокировок и долгих транзакций. Вместо этого тут короткие и частые транзакции. Кроме того, транзакция перестает содержать <code>unit of work</code>` а используется для присвоения промежуточного статуса. Тут важно понимать, что нужен какой-то механизм recovery на тот случай, если микросервис не закончит обработку строки по какой-то причине (например, если процесс будет убит через kill -9).</p>
<p>

Таким образом, этот вариант подходит для случая высокой конкуренции за ресурсы БД (быстрее освобождаем коннекты, более короткие транзакции, нет эксклюзивной блокировки).</p>

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5MTk4MzQ2M119
-->