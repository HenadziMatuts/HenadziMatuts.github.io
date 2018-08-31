---
layout: post
title: "RtN. Part 3: Making the Game (RU)"

desc: The article is devoted to implementing the main menu and level loading functions.

author: Henadzi Matuts
date: 2018-07-05
link: https://habr.com/post/415555/
---

# Реверсим «Нейроманта». Часть 3: Добили рендеринг, делаем игру

<img src="https://habrastorage.org/webt/wn/4k/jm/wn4kjm-p2fyta0emos5ufhkop_0.png" />

Привет, это уже третья часть из серии моих публикаций, посвящённых обратной разработке «Нейроманта» — видеоигрового воплощения одноимённого романа Уильяма Гибсона.

> [Реверсим «Нейроманта». Часть 1: Спрайты][1]<br>
> [Реверсим «Нейроманта». Часть 2: Рендерим шрифт][2]

Эта часть может показаться несколько сумбурной. Дело в том, что большая часть того, о чём здесь рассказано, было готово ещё во время написания [предыдущей][2]. Поскольку с того момента прошло уже два месяца, а у меня, к сожалению, нет привычки вести рабочие заметки, некоторые детали я попросту забыл. Но уж как есть, поехали.

<cut />

---

_[После того, как я научился печатать строки, логично было бы продолжить реверсить построение диалог-боксов. Но, по каким-то ускользающим от меня причинам, вместо этого я целиком ушёл в разбор системы рендеринга.]_ В очередной раз прогуливаясь по `main`-у, удалось локализовать вызов, впервые выводящий что-либо на экран: `seg000:0159: call sub_1D0B2`. "Что-либо", в данном случае - это курсор и фоновое изображение главного меню:

<img src="https://habrastorage.org/webt/9l/9h/yq/9l9hyqeu9n7m8vyt8iizlz5wnze.png" /><br>

Примечательно, что функция `sub_1D0B2` _[далее - `render`]_ не имеет аргументов, однако её первому вызову предшествую два, практически одинаковых участка кода:

````asm
loc_100E5:                      loc_10123:
    mov     ax, 2                   mov     ax, 2
    mov     dx, seg seg009          mov     dx, seg seg010
    push    dx                      push    dx
    push    ax                      push    ax
    mov     ax, 506Ah               mov     ax, 5076h   ; "cursors.imh", "title.imh"
    push    ax                      push    ax
    call    load_imh                call    load_imh    ; load_imh(res, offt, seg)
    add     sp, 6                   add     sp, 6
    sub     ax, ax                  sub     ax, 0Ah
    push    ax                      push    ax
    call    sub_123F8               call    sub_123F8   ; sub_123F8(0), sub_123F8(10)
    add     sp, 2                   add     sp, 2
    cmp     word_5AA92, 0           mov     ax, 1
    jz      short loc_10123         push    ax
    sub     ax, ax                  mov     ax, 2
    push    ax                      mov     dx, seg seg010
    mov     ax, 2                   push    dx
    mov     dx, seg seg009          push    ax
    push    dx                      sub     ax, ax
    push    ax                      push    ax
    mov     ax, 64h                 push    ax
    push    ax                      mov     ax 0Ah
    mov     ax, 0A0h                push    ax
    push    ax                      call    sub_1CF5B   ; sub_1CF5B(10, 0, 0, 2, seg010, 1)
    sub     ax, ax                  add     sp, 0Ch
    push    ax                      call    render
    call    sub_1CF5B   ; sub_1CF5B(0, 160, 100, 2, seg009, 0)
    add     sp, 0Ch
````

Перед вызовом `render`, курсоры (`cursors.imh`) и фон (`title.imh`) распаковываются в память (`load_imh` - это переименованная `sub_126CB` из [первой части][1]), в девятый и десятый сегмент соответственно. Поверхностное изучение функции `sub_123F8` не принесло мне никакой новой информации, но зато, только глядя на аргументы `sub_1CF5B`, я сделал следующие выводы:
* аргументы 4 и 5, в совокупности, представляют собой адрес распакованного спрайта (`segment:offset`);
* аргументы 2 и 3, вероятно, координаты, поскольку эти числа коррелируют с изображением, выводимым после вызова `render`;
* последний аргумент может быть флагом непрозрачности фона спрайта, ведь распакованные спрайты имеют чёрный фон, а курсор на экране мы видим без него.

С первым аргументом _[а заодно и с рендерингом в целом]_ всё стало ясно после трассировки `sub_1CF5B`. Дело в том, что в сегменте данных, начиная с адреса `0x3BD4`, расположен массив из 11-ти структур следующего вида:
````cpp
typedef struct sprite_layer_t {
    uint8_t flags;
    uint8_t update;

    uint16_t left;
    uint16_t top;
    uint16_t dleft;
    uint16_t dtop;

    imh_hdr_t sprite_hdr;
    uint16_t sprite_segment;
    uint16_t sprite_pixels;

    imh_hdr_t _sprite_hdr;
    uint16_t _sprite_segment;
    uint16_t _sprite_pixels;
} sprite_layer_t;
````

Эту концепцию я называю "спрайт-чейн". В действительности, функция `sub_1CF5B` (далее - `add_sprite_to_chain`) добавляет выбранный спрайт в цепочку. На 16-битной машине она имела бы примерно следующую сигнатуру:
````cpp
sprite_layer_t g_sprite_chain[11];

void add_sprite_to_chain(int index,
    uint16_t left, uint16_t top,
    uint16_t offset, uint16_t segment,
    uint8_t opaque);
````

Работает это вот так:
* первый аргумент - это индекс в массиве `g_sprite_chain`;
* аргументы `left` и `top` записываются в поля `g_sprite_chain[index].left` и `g_sprite_chain[index].top` соответственно;
* заголовок спрайта (первые 8 байт, расположенные по адресу `segment:offset`) копируется в поле `g_sprite_chain[index].sprite_hdr`, типа `imh_hdr_t` (переименованная `rle_hdr_t` из первой части):

````cpp
typedef struct imh_hdr_t {
    uint32_t unknown;
    uint16_t width;
    uint16_t height;
} imh_hdr_t;
````

* в поле `g_sprite_chain[index].sprite_segment` записывается значение `segment`;
* в поле `g_sprite_chain[index].sprite_pixels` записывается значение, равное `offset + 8`, таким образом `sprite_segment:sprite_pixels` - это адрес битмапа добавляемого спрайта;
* поля `sprite_hdr`, `sprite_segment`, и `sprite_pixels` дублируются в `_sprite_hdr`, `_sprite_segment`, и `_sprite_pixels` соответственно _[зачем? - понятия не имею, и это не единственный случай такого дублирования полей]_;
* в поле `g_sprite_chain[index].flags` записывается значение, равное `1 + (opaque << 4)`. Эта запись означает, что первый бит значения `flags` указывает на "активность" текущего "слоя", а пятый бит - на **непрозрачность** его фона. _[Мои сомнения по поводу флага прозрачности развеялись после того, как я экспериментально проверил его влияние на выводимое изображение. Изменяя значение пятого бита во время выполнения, мы можем наблюдать вот такие артефакты]:_

<img src="https://habrastorage.org/webt/9z/i8/ui/9zi8uizaa3s9fwsqke4rwa_3jqa.png" /><br>

Как я уже упоминал, функция `render` не имеет аргументов, но ей и не надо - она напрямую работает с массивом `g_sprite_chain`, поочерёдно перенося "слои" в _VGA_-память, от последнего (`g_sprite_chain[10]` - задний фон) к первому (`g_sprite_chain[0]` - передний план). Структура `sprite_layer_t` имеет для этого всё необходимое и даже больше. Я говорю о нерассмотренных полях `update`, `dleft` и `dtop`.

На самом деле, функция `render` перерисовывает **НЕ ВСЕ** спрайты в каждом кадре. На то, что текущий спрайт необходимо перерисовать указывает ненулевое значение поля `g_sprite_chain.update`. Допустим, мы перемещаем курсор (`g_sprite_chain[0]`), тогда в обработчике движения мыши произойдёт что-то наподобие этого:
````cpp
void mouse_move_handler(...)
{
    ...
    g_sprite_chain[0].update = 1;
    g_sprite_chain[0].dleft = mouse_x - g_sprite_chain[0].left;
    g_sprite_chain[0].dtop = mouse_y - g_sprite_chain[0].top;
}
````

Когда управление перейдёт к функции `render`, последняя, добравшись до слоя `g_sprite_chain[0]`, увидит, что его необходимо обновить. Тогда:
* будет вычислено и нарисовано пересечение области, занимаемой спрайтом курсора до обновления, со всеми предыдущими слоями;
* координаты спрайта обновятся:

````сpp
g_sprite_chain[0].update = 0;

g_sprite_chain[0].left += g_sprite_chain[0].dleft
g_sprite_chain[0].dleft = 0;

g_sprite_chain[0].top += g_sprite_chain[0].dtop
g_sprite_chain[0].dtop = 0;
````

* спрайт будет нарисован по обновлённым координатам.

Таким образом минимизируется количество операций, выполняемых функцией `render`.

---

Реализовать эту логику было несложно, хотя я достаточно сильно её упростил. С учётом вычислительных мощностей современных компьютеров, мы можем себе позволить перерисовывать все 11 спрайтов цепочки в каждом кадре, за счёт этого упраздняются поля `g_sprite_chain.update`, `.dleft`, `.dtop` и вся связанная с ними обработка. Ещё одно упрощение касается обработки флага непрозрачности. В оригинальном коде, для каждого прозрачного пикселя в спрайте ищется пересечение с первым непрозрачным пикселем в нижних слоях. Но я использую 32-битный видеорежим, и поэтому могу просто изменять значение байта прозрачности в _RGBA_-схеме. В итоге, у меня получились вот такие функции добавления(удаления) спрайта в(из) цепочку(и):

````cpp
typedef struct sprite_layer_t {
    uint8_t flags;
    uint16_t left;
    uint16_t top;

    imh_hdr_t sprite_hdr;
    uint8_t *sprite_pixels;

    imh_hdr_t _sprite_hdr;
    uint8_t *_sprite_pixels;
} sprite_layer_t;

sprite_layer_t g_sprite_chain[11];

void add_sprite_to_chain(int n, uint32_t left, uint32_t top,
            uint8_t *sprite, int opaque)
{
    assert(n <= 10);

    sprite_layer_t *layer = &g_sprite_chain[n];
    memset(layer, 0, sizeof(sprite_layer_t));

    layer->left = left;
    layer->top = top;

    memmove(&layer->sprite_hdr, sprite, sizeof(imh_hdr_t));
    layer->sprite_pixels = sprite + sizeof(imh_hdr_t);

    memmove(&layer->_sprite_hdr, &layer->sprite_hdr,
        sizeof(imh_hdr_t) + sizeof(uint8_t*));

    layer->flags = ((opaque << 4) & 16) | 1;
}

void remove_sprite_from_chain(int n)
{
    assert(n <= 10);

    sprite_layer_t *layer = &g_sprite_chain[n];
    memset(layer, 0, sizeof(sprite_layer_t));
}
````

Функция переноса слоя в _VGA_-буфер выглядит следующим образом:
````cpp
void draw_to_vga(int left, int top,
    uint32_t w, uint32_t h, uint8_t *pixels, int bg_transparency);

void draw_sprite_to_vga(sprite_layer_t *sprite)
{
    int32_t top = sprite->top;
    int32_t left = sprite->left;
    uint32_t w = sprite->sprite_hdr.width * 2;
    uint32_t h = sprite->sprite_hdr.height;
    uint32_t bg_transparency = ((sprite->flags >> 4) == 0);
    uint8_t *pixels = sprite->sprite_pixels;

    draw_to_vga(left, top, w, h, pixels, bg_transparency);
}
````

Функция `draw_to_vga` - это одноимённая функция описанная во [второй части][2], но с дополнительным аргументом, указывающим на прозрачность фона изображения. Добавляем вызов `draw_sprite_to_vga` в начало функции `render` (остальное её содержимое перекочевало из [второй части][2]):
````cpp
static void render()
{
    for (int i = 10; i >= 0; i--)
    {
        if (!(g_sprite_chain[i].flags & 1))
        {
            continue;
        }
        draw_sprite_to_vga(&g_sprite_chain[i]);
    }
    ...
}
````

Также я написал функцию обновляющую позицию спрайта курсора, в зависимости от текущего положения указателя мыши (`update_cursor`), и простенький менеджер ресурсов. Заставляем всё это работать вместе:
````cpp
typedef enum spite_chain_index_t {
    SCI_CURSOR = 0,
    SCI_BACKGRND = 10,
    SCI_TOTAL = 11
} spite_chain_index_t;

uint8_t g_cursors[399];         /* seg009 */
uint8_t g_background[32063];    /* seg010 */

int main(int argc, char *argv[])
{
    ...
    assert(resource_manager_load("CURSORS.IMH", g_cursors));
    add_sprite_to_chain(SCI_CURSOR, 160, 100, g_cursors, 0);

    assert(resource_manager_load("TITLE.IMH", g_background));
    add_sprite_to_chain(SCI_BACKGRND, 0, 0, g_background, 1);

    while (sfRenderWindow_isOpen(g_window))
    {
        ...
        update_cursor();
        render();
    }
    ...
}
````

<img src="https://habrastorage.org/webt/lw/ji/sm/lwjismkxi7ewlzw9m9zahx3imjy.gif" />

---

Окей, для полноценного главного меню не хватает, собственно, самого меню. Самое время вернуться к реверсингу диалог-боксов. *[В прошлый раз я разобрал функцию `draw_frame`, формирующую диалоговое окно, и, частично, функцию `draw_string`, забрав оттуда только логику рендеринга текста.]* Взглянув по новой на `draw_frame`, я увидел, что там используется функция `add_sprite_to_chain` - ничего удивительного, просто добавление диалогового окна в спрайт-чейн. Нужно было разобраться с позиционированием текста внутри диалог-бокса. Напомню, как выглядит вызов функции `draw_string`:

```asm
    sub     ax, ax
    push    ax
    mov     ax, 1
    push    ax
    mov     ax, 5098h       ; "New/Load"
    push    ax
    call    draw_string     ; draw_string("New/Load", 1, 0)
```

и структура, заполняющаяся в `draw_frame` *[здесь с небольшим опережением, поскольку большинство элементов я проименовал уже после того, как полностью разобрался с `draw_string`. Кстати, здесь, как и в случае со `sprite_layer_t`, имеет место дублирование полей]*:

```c
typedef struct neuro_dialog_t {
    uint16_t left;              // word[0x65FA]:   0x20
    uint16_t top;               // word[0x65FC]:   0x98
    uint16_t right;             // word[0x65FE]:   0x7F
    uint16_t bottom;            // word[0x6600]:   0xAF
    uint16_t inner_left;        // word[0x6602]:   0x28
    uint16_t inner_top;         // word[0x6604]:   0xA0
    uint16_t inner_right;       // word[0x6604]:   0xA0
    uint16_t inner_bottom;      // word[0x6608]:   0xA7
    uint16_t _inner_left;       // word[0x660A]:   0x28
    uint16_t _inner_top;        // word[0x660C]:   0xA0
    uint16_t _inner_right;      // word[0x660E]:   0x77
    uint16_t _inner_bottom;     // word[0x6610]:   0xA7
    uint16_t flags;             // word[0x6612]:   0x06
    uint16_t unknown;           // word[0x6614]:   0x00
    uint8_t padding[192]        // ...
    uint16_t width;             // word[0x66D6]:   0x30
    uint16_t pixels_offset;     // word[0x66D8]:   0x02
    uint16_t pixels_segment;    // word[0x66DA]:   0x22FB
} neuro_dialog_t;
```

Вместо того, что бы объяснять, что здесь, как и зачем, я просто оставлю это изображение:

<img src="https://habrastorage.org/webt/db/tf/nw/dbtfnw8fn1qx9aktubuvwojuw7y.png" />

Переменные `x_offt` и `y_offt` - это второй и третий аргументы функции `draw_string` соответственно. На основе этой информации было несложно соорудить собственные версии `draw_frame` и `draw_text`, предварительно переименовав их в `build_dialog_frame` и `build_dialog_text`:

```c
void build_dialog_frame(neuro_dialog_t *dialog,
            uint16_t left, uint16_t top, uint16_t w, uint16_t h,
            uint16_t flags, uint8_t *pixels);
void build_dialog_text(neuro_dialog_t *dialog,
            char *text, uint16_t x_offt, uint16_t y_offt);

...

typedef enum spite_chain_index_t {
    SCI_CURSOR = 0,
    SCI_DIALOG = 2,
    ...
} spite_chain_index_t;

...

uint8_t *g_dialog = NULL;
neuro_dialog_t g_menu_dialog;

int main(int argc, char *argv[])
{
    ...
    assert(g_dialog = calloc(8192, 1));
    build_dialog_frame(&g_menu_dialog, 32, 152, 96, 24, 6, g_dialog);
    build_dialog_text(&g_menu_dialog, "New/Load", 8, 0);
    add_sprite_to_chain(SCI_DIALOG, 32, 152, g_dialog, 1);
    ...
}
```

<img src="https://habrastorage.org/webt/ms/xt/bg/msxtbgydwxelkcfgng8vdud10ee.png" />

Основное отличие моих версий от оригинальных заключается в том, что я использую абсолютные значения пиксельных размеров - так проще.

---

Уже тогда я был уверен в том, что за создание кнопок отвечает участок кода, следующий сразу за вызовом `build_dialog_text`:

```asm
    ...
    mov     ax, 5098h           ; "New/Load"
    push    ax
    call    build_dialog_text   ; build_dialog_text("New/Load", 1, 0)
    add     sp, 6
    mov     ax, 6Eh             ; 'n' - этот
    push    ax
    sub     ax, ax
    push    ax
    mov     ax, 3
    push    ax
    sub     ax, ax
    push    ax
    mov     ax, 1
    push    ax
    call    sub_181A3           ; sub_181A3(1, 0, 3, 0, 'n')
    add     sp, 0Ah
    mov     ax, 6Ch             ; 'l' - и этот комментарий сгенерировала Ида
    push    ax
    mov     ax, 1
    push    ax
    mov     ax, 4
    push    ax
    sub     ax, ax
    push    ax
    mov     ax, 5
    push    ax
    call    sub_181A3           ; sub_181A3(5, 0, 4, 1, 'l')
```

Всё дело в этих сгенерированных комментариях - `'n'` и `'l'`, которые, очевидно, являются первыми буквами в словах `"New"` и `"load"`. Дальше, если рассуждать по аналогии с `build_dialog_text`, то первые четыре аргумента `sub_181A3` (далее - `build_dialog_item`) могут быть множителями координат и размеров *[на самом деле первые три аргумента, четвёртый, как оказалось, про другое]*. Всё сходится, если наложить эти значения на изображение следующим образом:

<img src="https://habrastorage.org/webt/xy/gw/hj/xygwhjq2ykcj7wf9z0dbtzhzr7q.png" />

Переменные `x_offt`, `y_offt` и `width`, на изображении - это, соотвественно, первые три аргумента функции `build_dialog_item`. Высота этого прямоугольника всегда равна высоте символа - восьми. После очень пристального взгляда на `build_dialog_item`, я выяснил, что то, что в структуре `neuro_dialog_t` я обозначил как `padding` (теперь - `items`) - это массив из 16-ти структур следующего вида:

```c
typedef struct dialog_item_t {
    uint16_t left;
    uint16_t top;
    uint16_t right;
    uint16_t bottom;
    uint16_t unknown; /* index? */
    char letter;
} dialog_item_t;
```

А поле `neuro_dialog_t.unknown` (теперь - `neuro_dialog_t.items_count`) - это счётчик количества пунктов в меню:

```c
typedef struct neuro_dialog_t {
    ...
    uint16_t flags;
    uint16_t items_count;
    dialog_item_t items[16];
    ...
} neuro_dialog_t;
```

Поле `dialog_item_t.unknown` инициализируется четвёртым аргументом функции `build_dialog_item`. Возможно, это индекс элемента в массиве, но, вроде бы, это не всегда так, а поэтому - `unknown`. Поле `dialog_item_t.letter` инициализируется пятым аргументом функции `build_dialog_item`. Опять же, возможно, что в обработчике лефт-клика игра проверяет попадание координат указателя мыши в область одного из айтемов (просто перебирая их по порядку, например), и, если попадание есть, то по этому полю выбирается нужный обработчик нажатия на конкретную кнопку. *[Не знаю, как это сделано на самом деле, но у себя я реализовал именно такую логику.]*

Этого достаточно, чтобы, уже не оглядываясь на оригинальный код, а просто повторяя его поведение, наблюдаемое в игре, сделать полноценное главное меню.

<img src="https://habrastorage.org/webt/we/q8/ni/weq8ni9fy9vuusoaitjgazjt6e4.gif" />

---

Если ты досмотрел до конца предыдущую гифку, то наверняка заметил стартовый игровой экран на последних кадрах. На самом деле, у меня уже есть всё для того, чтобы его нарисовать. Только бери да загружай нужные спрайты и добавляй их в спрайт-чейн. Однако, размещая на сцене спрайт главного героя, я сделал одно важное открытие, связанное со структурой `imh_hdr_t`.

В оригинальном коде, функция `add_sprite_to_chain`, добавляющая изображение протагониста в цепочку, вызывается с координатами 156 и 110. Вот, что я увидел, повторив это у себя:

<img src="https://habrastorage.org/webt/i1/5s/yf/i15syfep7bdhuhxgsbkhr-bxev4.png" align="left" />

Разобравшись, что к чему, я получил следующий вид структуры `imh_hdr_t`:

```c
typedef struct imh_hdr_t {
    uint16_t dx;
    uint16_t dy;
    uint16_t width;
    uint16_t height;
} imh_hdr_t;
```

То, что раньше было полем `unknown`, оказалось значениями смещений, которые вычитаются из соответствующих координат (во время рендеринга), хранящихся в спрайт-чейне.

<br clear="left" />

Таким образом, реальная координата левого верхнего угла отрисовываемого спрайта вычисляется приблизительно так:

```
left = sprite_layer_t.left - sprite_layer_t.sprite_hdr.dx
top = sprite_layer_t.top - sprite_layer_t.sprite_hdr.dy
```

Применив это в своём коде, я получил правильную картинку, а после этого занялся оживлением главного героя. На самом деле, весь код связанный с управлением персонажем (мышью и с клавиатуры), его анимацией и перемещением я написал самостоятельно, не оглядываясь на оригинал.

<img src="https://habrastorage.org/webt/tw/ei/m2/tweim2zx4hrsefql82ny-elnimc.gif" />

---

Запилил текстовое интро для первого уровня. Напомню, что строковые ресурсы хранятся в `.BIH`-файлах. В распакованном виде `.BIH`-файлы состоят из заголовка переменного размера и последовательности нуль-терминированных строк. Исследуя оригинальный код, проигрывающий интро, я выяснил, что смещение начала текстовой части в `.BIH`-файле содержится в четвёртом ворде заголовка. Первая строка - это интро:

```c
typedef struct bih_hdr_t {
    uint16_t unknown[3];
    uint16_t text_offset;
} bih_hdr_t;

...

uint8_t r1_bih[12288];
assert(resource_manager_load("R1.BIH", r1_bih));

bih_hdr_t *hdr = (bih_hdr_t*)r1_bih;
char *intro = r1_bih + hdr->text_offset;
```

Дальше, опираясь на оригинал, я реализовал разбиение исходной строки на подстроки так, чтобы они вмещались в область для вывода текста, прокрутку этих строк, и ожидание ввода перед выдачей следующей порции.

<img src="https://habrastorage.org/webt/uk/oa/ag/ukoaagah17k7eegsy4unobeskbg.gif" />

---

На момент публикации, сверх того, что уже описано в трёх частях, я разобрался с воспроизведением звука. Пока это только у меня в голове и понадобится некоторое время на то, чтобы реализовать это в своём проекте. Так что четвёртая часть, вероятно, будет целиком про звук. Так же планирую рассказать немного об архитектуре проекта, но посмотрим, как пойдёт.

> [Реверсим «Нейроманта». Часть 4: Звук, анимация, Хаффман, гитхаб][3]

[1]: /2018/03/30/reversing-the-neuromancer-part-1.html
[2]: /2018/05/10/reversing-the-neuromancer-part-2.html
[3]: /2018/07/23/reversing-the-neuromancer-part-4.html