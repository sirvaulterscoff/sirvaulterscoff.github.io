---


---

<hr>
<hr>
<p>#articles #sql #programming #patterns</p>
<h1 id="о-синхронизации-потоков-через-субд">О синхронизации потоков через СУБД</h1>
<p>Есть целый класс задач, когда необходимо выполнить какую-то обработку данных на нескольких узлах (экземплярах микросервиса) при этом необходимо обеспечить синхронизацию этих независимых узлов так, чтобы только один обрабатывал порцию данных в один момент времени. Kafka из коробки предлагает механизм партицирования, но к сожалению такое решение не всегда применимо</p>
<h2 id="формулировка-задачи">Формулировка задачи</h2>
<p>В таблицу Tasks помещаются строки для обработки, каждая содержит ID, статус, дата создания. Необходимо сделать так, чтобы каждый из экземпляров микросервиса обрабатывал только одну запись, и каждая уникальная запись обрабатывалась только одним микросервисом.</p>
Задачи выполняются по мере их поступления в порядке FIFO (т.е. сортирует по дате создания).
<p>Первое требование выполняется элементарно: достаточно лишь  обработку строк таблицы выполнять в одном потоке. А вот второе требование чуть сложнее. Рассмотрим варианты:</p>
<h3 id="пессимистчная-блокировка">Пессимистчная блокировкаТут все очень просто: многие БД предоставляют доступ к инструментам блокировки строк и таблиц, например [<a href=" семантику select for uphttps://www.postgresql.org/docs/9.1/explicit-locking.html">postgres предоставляет date</a>]
<p>)]
</p></h3><p>Этот способ вполне ок, до тех пор пока вы можете обеспечить минимальный период блокировки.</p><br>
Наш пример может выглядеть так:
<pre class="  language-sql"><code class="prism  language-sql">
<span class="token keyword">select</span> <span class="token operator">*</span>

<span class="token keyword">from</span> Tasks

<span class="token keyword">where</span>  <span class="token keyword">status</span><span class="token operator">=</span><span class="token string">'READY'</span>

<span class="token keyword">order</span> <span class="token keyword">by</span> created_date <span class="token keyword">asc</span>

<span class="token keyword">limit</span>  <span class="token number">1</span>

<span class="token keyword">FOR</span>  <span class="token keyword">UPDATE</span>  SKIP  LOCKED<span class="token punctuation">;</span>

</code></pre>
<p>Тут два ключевых момента. С помощью <code>FOR UPDATE</code> мы указываем, что необходимо использовать блокировку строки в EXCLUSIVE режиме. С помощью <code>SKIP LOCKED</code> мы указываем на необходимость пропустить заблокированные строки.</p>
<p>Таким образом, полная обработка в нашем случае будет выглядеть как-то так:</p>
<pre class="  language-sql"><code class="prism  language-sql">
<span class="token keyword">select</span> <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span> <span class="token keyword">FOR</span>  <span class="token keyword">UPDATE</span>  SKIP  LOCKED<span class="token punctuation">;</span>
</code></pre><p><span class="token comment">– some row processing</span></p>
<p><span class="token comment">–</span></p>
<p><span class="token comment">– end processing</span></p>
<p><span class="token keyword">update</span> Tasks <span class="token keyword">set</span>  <span class="token keyword">status</span><span class="token operator">=</span><span class="token string">‘COMPLETED’</span>  <span class="token keyword">where</span>  id<span class="token operator">=</span>?<span class="token punctuation">;</span></p>
<p><span class="token keyword">commit</span><span class="token punctuation">;</span></p>
<p></p>
<p>Несложно понять, что граница блокировки здесь охватывает весь блок, в т.ч. обработку записей</p>
<h3 id="оптимистичная-блокировка">Оптимистичная блокировка</h3>
<p>
</p><p>В отличие от пессимистичной (которая исходит из предположения, что эту строку точно кто-то кроме нас попытается обработать) оптимистичная блокировка рассчитана на более редкие коллизии. На то она и оптимистичная. Вариантов реализации оптимистичной блокировки масса, покажу лишь общий принцип.</p>
<p>
</p><p>Суть заключается в том, что мы не используем эксклюзивные блокировки, предоставляя БД решать какая транзакция может накатиться, а какая нет. Самый простой и, пожалуй, распространненый способ реализации оптимистичной блокировки - колонка с версией записи. В качестве версии может выступать как число-счетчик изменений, так и timestamp последнего изменения записи.</p>
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
<p>мы выбираем не только идентификатор строки, но и ее версию</p>
</li>
<li>
<p>
</p></li></ul><ul>
<li>после обработки строки мы в условие дописываем критерий version=? куда передаем значение версии строки, вычитанное на первом этапе</li>
</ul>


<p>Таким образом, если какая-то транзакция обновила строку, пока мы ее обрабатывали update не выполнит никаких изменений, т.к. строка просто не подойдет по условию</p>
<p>
</p><p>Тут важный момент: все обновления таблицы должны использовать версионированную колонку.</p>
<p>
</p><p>Как несложно заметить это решение не создает экслюзивной блокировки, но и не помогает в достижении нашей цели. Более того, в этом примере результат обработки, фактически, оказывается потеряным, если параллельная транзакция обновила строку.</p>
<p>
</p><p>Но, вернемся к нашей цели. Напомню, нам нужно позволить двум экземплярам сервиса одновременно обрабатывать разные строки.</p>
<p>
</p><p>Тут возможен следующий алгоритм действий</p>
<ol>
<li>
</li></ol><ol>
<li>Каждый из микросервисов отбирает набор строк для обработки:</li>
</ol>

<pre><code>
</code></pre><pre><code>
select  id, version

from Tasks

where  status='READY'

order by created_date asc

limit N;

&lt;/code&gt;&lt;/pre&gt;
&lt;p&gt;```

Значение N должно определяться количеством потоков, которые могут одновременно обрабатывать строки, т.к. при использовании этого подхода часть отобранных строк может успеть попасть в обработку и нам придется их пропустить&lt;/p&gt;
&lt;ol start="2"&gt;
&lt;li&gt;По каждой строке пытаемся обновить статус&lt;/li&gt;
&lt;/ol&gt;
&lt;pre&gt;&lt;code&gt;
update Tasks

set  version=X, status='PROCESSING'

where id=? and version=X

commit;

&lt;/code&gt;&lt;/pre&gt;
&lt;p&gt;Если в результате апдейта не была изменена ни одна строка (это можно выяснить, например, использую семантику &lt;a hhttps://postgrespro.ru/docs/postgresql/9.5/dml-returning"&gt;returning&lt;/a&gt;)) то необходимо перейти к следующей строке&lt;/p&gt;
&lt;h4 id="что-важно-понимать"&gt;Что важно понимать&lt;/h4&gt;
&lt;p&gt;

В этом примере нет долгоживущих эксклюзивных блокировок и долгих транзакций. Вместо этого тут короткие и частые транзакции. Кроме того, транзакция перестает содержать &lt;code&gt;`unit of work&lt;/code&gt;` а используется для присвоения промежуточного статуса. Тут важно понимать, что нужен какой-то механизм recovery на тот случай, если микросервис не закончит обработку строки по какой-то причине (например, если процесс будет убит через kill -9).&lt;/p&gt;
&lt;p&gt;

Таким образом, этот вариант подходит для случая высокой конкуренции за ресурсы БД (быстрее освобождаем коннекты, более короткие транзакции, нет эксклюзивной блокировки).&lt;/p&gt;

</code></pre>

