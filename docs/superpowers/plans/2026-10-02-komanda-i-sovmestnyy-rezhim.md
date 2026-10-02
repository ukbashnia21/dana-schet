# Команда и совместный режим — план работ

> **ВЫПОЛНЕНО 02.10.2026.** Проверено замером. По ходу работы вскрылся дефект,
> которого в плане не было: `swapTo()` менял игрока, но не переподключал базу —
> подписка оставалась на прежнем профиле и могла затереть данные нового.
> Исправлено отпиской от прежних подписок и переподключением внутри `swapTo()`.

**Цель:** появляется общая лестница на двоих и кнопка «Вдвоём», которая
проводит две минуты подряд с передачей хода.

**Подход:** командный зачёт считается из `team/days`, который уже пишется с
прошлого слоя. Совместный заход — обёртка над существующей минутой: после
финиша первого игрока профиль переключается, запускается вторая минута, затем
показывается общий итог.

**Технологии:** те же — ванильный JS в `index.html`, Firestore.

## Общие ограничения

- **Ступени не хранить счётчиком.** Считать из дней обоих игроков, иначе два
  устройства засчитают один день дважды.
- **Одиночные дни Даны копят её прогресс всегда.** Командная ступень ждёт
  второго и засчитывается задним числом.
- **Профили не смешиваются.** При переключении внутри совместного захода
  состояние первого игрока обязано сохраниться до загрузки второго.
- **Версия в двух местах** при публикации: `версия N` в `index.html` и
  `C='ustny-vN'` в `sw.js`.
- **Проверка — в браузере**, модульных тестов в проекте нет.

---

### Задача 1: Читать дни команды и считать ступени

**Файлы:** `index.html` — рядом с `pushDays()`

**Интерфейсы:**
- Отдаёт дальше: `TEAMDAYS` — объект `{dana:{дата:bool}, papa:{дата:bool}}`;
  `teamDone(k)` — выполнен ли день командой; `teamSteps()` — число ступеней;
  `teamStreak()` — серия командных дней подряд.

- [x] **Шаг 1: Хранилище и расчёты**

```js
let TEAMDAYS={dana:{},papa:{}};
const teamDone=k=>!!(TEAMDAYS.dana&&TEAMDAYS.dana[k])&&!!(TEAMDAYS.papa&&TEAMDAYS.papa[k]);
function teamSteps(){const a=Object.keys(TEAMDAYS.dana||{}),b=TEAMDAYS.papa||{};
 return a.filter(k=>TEAMDAYS.dana[k]&&b[k]).length;}
function teamStreak(){let n=0;const d=new Date();if(!teamDone(dkey(d)))d.setDate(d.getDate()-1);
 while(teamDone(dkey(d))){n++;d.setDate(d.getDate()-1);}return n;}
function myDone(k){return taskDone(k);}
function partnerDone(k){const o=WHO==='dana'?'papa':'dana';return !!(TEAMDAYS[o]&&TEAMDAYS[o][k]);}
```

- [x] **Шаг 2: Подписка на дни команды в `initDb()`**

Добавить после подписки на профиль:

```js
  db.doc('team/days').onSnapshot(sn=>{if(!sn.exists)return;
   TEAMDAYS=Object.assign({dana:{},papa:{}},JSON.parse(JSON.stringify(sn.data())));
   if(!RUN)render();});
```

- [x] **Шаг 3: Проверка**

В консоли браузера выполнить `teamSteps()` — должно вернуть число без ошибок.

---

### Задача 2: Блок команды на экране «Сегодня»

**Файлы:** `index.html` — `vToday()`, стили

- [x] **Шаг 1: Разметка блока**

Вставить в `vToday()` перед закрывающим тегом секции:

```js
 <div class="team">
  <div class="teamhead"><b>Команда</b><span class="muted small">ступеней ${teamSteps()} · серия ${teamStreak()}</span></div>
  <div class="teamrow">
   <span class="who ${myDone(k)?'ok':''}">${esc(PLAYERS[WHO].name)}: ${myDone(k)?'готово':'ещё нет'}</span>
   <span class="who ${partnerDone(k)?'ok':''}">${esc(PLAYERS[WHO==='dana'?'papa':'dana'].name)}: ${partnerDone(k)?'готово':'ещё нет'}</span>
  </div>
  ${teamDone(k)?'<p class="pen">День команды засчитан — поднялись на ступень!</p>':'<p class="muted small">Ступень даётся, когда задание выполнили оба. Твой прогресс копится в любом случае.</p>'}
 </div>
```

- [x] **Шаг 2: Стили**

```css
.team{grid-column:1/-1;border-top:1.5px dashed var(--line);padding-top:14px;display:flex;flex-direction:column;gap:8px}
.teamhead{display:flex;align-items:baseline;gap:10px}
.teamrow{display:flex;gap:10px;flex-wrap:wrap}
.teamrow .who{border:1.5px solid var(--line);border-radius:999px;padding:3px 12px;font-size:14px}
.teamrow .who.ok{border-color:var(--green);color:var(--green)}
```

- [x] **Шаг 3: Проверка**

Открыть «Сегодня» — блок команды виден, обе метки «ещё нет». Пройти норму —
своя метка становится «готово» и зелёной.

---

### Задача 3: Переключение профиля без потери данных

**Файлы:** `index.html` — рядом с `setWho()`

**Интерфейсы:**
- Отдаёт дальше: `swapTo(id)` — сохраняет текущего игрока и загружает другого,
  возвращает промис.

- [x] **Шаг 1: Функция обмена**

```js
async function swapTo(id){
 save();                                   // текущий игрок зафиксирован
 WHO=id;try{localStorage.setItem(LS_WHO,id);}catch(e){}
 S=fresh();S.name=PLAYERS[id].name;SESS=[];
 try{const x=JSON.parse(localStorage.getItem(LS+'-'+id)||'null');if(x&&x.v)S=Object.assign(fresh(),x);}catch(e){}
 if(DB){try{const sn=await DB.doc('players/'+id).get();
  if(sn.exists&&sn.data().updatedAt>(S.updatedAt||0))S=Object.assign(fresh(),JSON.parse(JSON.stringify(sn.data())));
 }catch(e){}}
}
```

Сначала локальный профиль, затем облачный, если он свежее — порядок важен:
иначе занятия с другого устройства затрут более новые местные.

- [x] **Шаг 2: Проверка**

В консоли: `await swapTo('papa'); S.name` → «Папа»; `await swapTo('dana'); S.totalOk`
→ прежнее число Даны, не ноль.

---

### Задача 4: Совместный заход

**Файлы:** `index.html` — `vWho()`, `vToday()`, `finish()`, новые функции

**Интерфейсы:**
- Потребляет: `swapTo()` из задачи 3.
- Отдаёт дальше: `DUO` — состояние совместного захода или `null`.

- [x] **Шаг 1: Состояние и запуск**

```js
let DUO=null;
async function startDuo(){
 DUO={order:['dana','papa'],idx:0,res:[]};
 if(WHO!=='dana')await swapTo('dana');
 render();startCh(curLevel());}
async function duoNext(){
 if(!DUO)return;
 DUO.idx++;
 if(DUO.idx>=DUO.order.length){const r=DUO.res;DUO=null;showDuoResult(r);return;}
 await swapTo(DUO.order[DUO.idx]);
 render();startCh(curLevel());}
```

- [x] **Шаг 2: Кнопка на экране выбора и на «Сегодня»**

В `vWho()` добавить третью кнопку:

```js
  <button class="btn huge" onclick="setWho('dana');setTimeout(startDuo,300)">Вдвоём</button>
```

В `vToday()`, рядом с «Начать минуту»:

```js
  <button class="btn" onclick="startDuo()">Заниматься вдвоём</button>
```

- [x] **Шаг 3: Передача хода вместо «На главную»**

В `finish()`, в блоке кнопок, заменить условие: если `DUO` активен, показывать
одну кнопку передачи вместо обычных:

```js
 if(DUO){DUO.res.push({who:WHO,name:PLAYERS[WHO].name,ok,n});
  const nxt=DUO.order[DUO.idx+1];
  кнопки = nxt ? `<button class="btn primary huge" onclick="duoNext()">Передать ход: ${PLAYERS[nxt].name}</button>`
               : `<button class="btn primary huge" onclick="duoNext()">Общий итог</button>`;}
```

- [x] **Шаг 4: Общий итог**

```js
function showDuoResult(res){
 const total=res.reduce((s,r)=>s+r.ok,0);
 $('#main').innerHTML=`<section class="page result">
  <h2>Занимались вдвоём</h2>
  <p class="big">Вместе решили ${plural(total,'пример','примера','примеров')}</p>
  <ul class="reqs">${res.map(r=>`<li><span>${esc(r.name)}</span><b>${r.ok} из ${r.n}</b></li>`).join('')}</ul>
  ${teamDone(TODAY())?'<p class="pen big">День команды засчитан!</p>':'<p class="muted">Ступень засчитается, когда норму выполнят оба.</p>'}
  <div class="row"><button class="btn primary" onclick="render()">На главную</button></div></section>`;}
```

- [x] **Шаг 5: Проверка**

- Нажать «Вдвоём» → идёт минута Даны.
- По окончании — кнопка «Передать ход: Папа», профиль меняется, уровень
  папин.
- После второй минуты — общий итог с двумя строками.
- Открыть «Сегодня» за каждого: у обоих свои цифры, данные не перемешались.

---

### Задача 5: Публикация

- [x] **Шаг 1:** Поднять версию до 5 в `index.html` и `sw.js`.
- [x] **Шаг 2:** Коммит и `git push`.
- [x] **Шаг 3:** Дождаться публикации, проверить боевой адрес в браузере:
  кнопка «Вдвоём» есть, заход проходит целиком, ошибок в консоли нет.
