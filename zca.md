# Czy C++ jest wolniejszy od C? Kilka słów o Zero Cost Abstraction

Język C++, w przeciwieństwie do C jest językiem wieloparadygmatowym. Możemy używać go do programowania proceduralnego, strukturalnego, obiektowego, poniekąd funkcyjnego i prawdopodobnie jeszcze jakiegoś innego. W C jesteśmy ograniczeni do pierwszych dwóch. W świecie programistów można spotkać opinie, że C++ poprzez zwiększenie poziomu abstrakcji utracił na wydajności w stosunku do starego, dobrego i szybkiego C. Zbadajmy więc, czy ta opinia ma odzwierciedlenie w rzeczywistości. Porozmawiajmy jednak najpierw trochę bardziej teoretycznie.

## Czym jest abstrakcja?

To pojęcie trzeba jakoś zdefiniować. Mimo że każdy wprawiony programista ma jakąś intuicję na temat tego, czym ona jest, to zdefiniować ją trudno. Na potrzeby tego artykułu wymyśliłem tysięczną definicję słowa `abstrakcja` w kontekście języków programowania:

> **Abstrakcja** jest rozwiązaniem generycznego problemu, który pozwala programiście pominąć niskopoziomowe aspekty i napisać krótszy lub prostszy kod, który ten problem rozwiązuje.

Używanie języka programowania, nawet tak niskopoziomowego, jak czyste C, jest już w jakimś sensie posługiwaniem się abstrakcją. Wszyscy się zgadzamy, że abstrakcja jest czymś dobrym. Do pewnego momentu...

## Darmowa abstrakcja

A właściwie z angielskiego `Zero Cost Abstraction` lub `Zero Overhead Principle` (nigdy nie słyszałem żadnego polskiego tłumaczenia, wymyśliłem je przed chwilą). Jest to zasada, którą kierował się *Bjarne Stroustrup*, kiedy projektował język C++ i którą do dziś kierują się członkowie komitetu standaryzacyjnego ISO tego języka. I właściwie o tej zasadzie jest ten wpis. Wypowiedź autora języka C++ na jej temat jest jedną z najczęściej cytowanych jego wypowiedzi:

> The zero-overhead principle is a guiding principle for the design of C++. It states that: ***What you don’t use, you don’t pay for (in time or space) and further: What you do use, you couldn’t hand code any better.***
In other words, no feature should be added to C++ which would make any existing code (not using the new feature) larger or slower, nor should any feature be added for which the compiler would generate code that is not as good as a programmer would create without using the feature.

To jest jedna z tych zasad, za które ludzie kochają C++. Inni, prawdopodobnie w większości programiści Javy, C#, JavaScriptu czy PHP, uważają ją za głupią. Wynikają z niej dwie rzeczy:

* Jeżeli dodajesz do języka nowy element składni (ang. *feature*), to tylko taki, że gdybyś napisał program bez niego, to byłby on tak samo wydajny, jak z jego użyciem.
* Jeżeli jednak wymyślisz coś, co jest świetne, ale mimo wszystko ma jakiś koszt (ang. *overhead*), to nie masz prawa kazać programiście płacić tego kosztu, jeżeli nie użyje on owego świetnego rozwiązania.

Jeszcze innymi słowy – jeżeli odpowiedź na pytanie „Jak szybki jest ten kod?” jest tożsama z odpowiedzią na pytanie „Jak szybki ten kod mógłby być w najlepszej implementacji?”, to albo piszesz w assemblerze, i jesteś w tym dobry, albo używasz `darmowej abstrakcji`.

#### Czy abstrakcja w C++ rzeczywiście jest „darmowa”?
Mój wykładowca z fizyki zawsze mawiał:

> Nigdy nie wierz książkom, które nie podają dowodów zawartych w nich informacji.

Ja być może nie udowodnię tego, że każda z konstrukcji `Zero Cost` taką jest naprawdę na dowolnym kompilatorze, ale podam przykłady dla jednego. Pokazanie pojedynczych przykładów też nie jest żadnym dowodem, w sensie matematycznym. Chodzi tutaj jednak o wyrobienie sobie pewnej intuicji co do tego, czy kod, który piszemy, jest optymalny oraz jak bardzo obciążymy nim procesor.

Przykłady podane niżej będę kompilował kompilatorem `GCC 9.1` za pośrednictwem [platformy godbolt.org](https://godbolt.org/), co pozwoli mi przejrzeć assemblera wyprodukowanego przez ten kompilator. Nie musisz znać assemblera, aby te wyniki zweryfikować – wystarczy rzut oka i proste porównanie różnic. Każdy może te przykłady skompilować kompilatorem, którego używa na co dzień. Zanim jednak przejdziemy do omawiania konstrukcji, czuję się zobowiązany napisać coś jeszcze...

## Zasada numer 1 – zaufaj swojemu kompilatorowi
Biblioteki C++ to często wrappery na biblioteki czystego C. Wydawałoby się, że dodając *warstwę abstrakcji*, możemy sprawić, że nasz program będzie wolniejszy. Bo na przykład wołamy najpierw funkcję `some_cpp_function`, która z kolei woła funkcję `some_c_function`. Sprawdźmy na prostym przykładzie:

```c++
#include <ctime>

static int some_c_function() { return std::time(nullptr); }
static int some_cpp_function() { return some_c_function(); }

int main()
{
    some_cpp_function();
    return 0;
}
```
Skompilujmy nasz kod, nie pozwalając kompilatorowi na żadne optymalizacje (flaga `-O0`) i zobaczmy rezultat:
```asm
some_c_function():
        push    rbp
        mov     rbp, rsp
        mov     edi, 0
        call    time
        pop     rbp
        ret
some_cpp_function():
        push    rbp
        mov     rbp, rsp
        call    some_c_function()
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        call    some_cpp_function()
        mov     eax, 0
        pop     rbp
        ret
```

A teraz włączmy podstawowy, najniższy poziom optymalizacji `-O1`:
```asm
main:
        sub     rsp, 8
        mov     edi, 0
        call    time
        mov     eax, 0
        add     rsp, 8
        ret
```

Kompilator nie tylko pominął wywołanie drugiej funkcji, ale zawołał bezpośrednio funkcję z biblioteki. A teraz, drogi czytelniku, zapamiętaj sobie nieformalną zasadę:
> Jeżeli jakaś optymalizacja wydaje Ci się dobra i oczywista, to Twój kompilator prawdopodobnie zrobi optymalizację jeszcze lepszą*.

**(może nie dotyczyć jakichś egzotycznych kompilatorów, napisanych przez rozpoczynającego swoją przygodę z programowaniem gimnazjalistę.)*

Ludzie naprawdę nie ufają swoim kompilatorom. Często starają się robić jakieś proste optymalizacje, nawet kosztem zmniejszenia czytelności kodu. Zupełnie niepotrzebnie. Róbcie sobie tymczasowe zmienne, mini funkcje i wszystkie inne rzeczy, które przyczyniają się do zwiększania czytelności waszego kodu. Kompilator na pewno za was posprząta.

Dygresja ta nie jest bezpośrednio związana z `Zero Cost Abstraction`. Jest to jedynie wstęp do kolejnych przykładów i dodatkowo wytłumaczenie, dlaczego czasem będę w nich używał `gcc`'owej flagi `-O1` zamiast `-O0`.

## Przykłady darmowej abstrakcji w C++

### Abstrakcja 1 – klasa

Przykład w C++:
```c++
class Integer
{
public:
    int value;
    void increment()
    {
        ++value; //lub ++this->value
    }
};

int main()
{
    Integer i = {-1};
    i.increment();
    return i.value;
}
```

Oraz analogiczny kod w C:
```c
struct Integer
{
    int value;
};

void increment(struct Integer* this)
{
    ++this->value;
}

int main()
{
    struct Integer i = {-1};
    increment(&i);
    return i.value;
}
```

Kompilacja `g++ -O0 test.cpp` daje następujący rezultat:
```asm
Integer::increment():
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        mov     eax, DWORD PTR [rax]
        lea     edx, [rax+1]
        mov     rax, QWORD PTR [rbp-8]
        mov     DWORD PTR [rax], edx
        nop
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], -1
        lea     rax, [rbp-4]
        mov     rdi, rax
        call    Integer::increment()
        mov     eax, DWORD PTR [rbp-4]
        leave
        ret
```

A tutaj rezultat kompilacji wersji napisanej w C (`gcc -O0 test.c`):
```asm
increment:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        mov     eax, DWORD PTR [rax]
        lea     edx, [rax+1]
        mov     rax, QWORD PTR [rbp-8]
        mov     DWORD PTR [rax], edx
        nop
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], -1
        lea     rax, [rbp-4]
        mov     rdi, rax
        call    increment
        mov     eax, DWORD PTR [rbp-4]
        leave
        ret
```
Jaka jest różnica? Żadna!

Musimy uświadomić sobie jedno – po kompilacji program to zbiór zmiennych/stałych i funkcji na nich operujących. I tyle, nie ma nic innego. Przede wszystkim nie ma czegoś takiego jak klasa. Klasa to po kompilacji stałe/zmienne (tworzymy je za każdym razem, kiedy w kodzie źródłowym tworzymy obiekt) oraz funkcje, które w szczególnym przypadku dostają `this` jako pierwszy argument. Co więcej, ten `this` (jak i każda inna zmienna) po kompilacji nie ma już typu. To po prostu ***jakiś*** adres. Informacja o typie po kompilacji nie jest komputerowi do niczego potrzebna (chyba że używamy RTTI, o czym później).

#### Konstruktor, destruktor i inne funkcje specjalne
Kolejny przykład:

C++:
```c++
#include <cstdio>

class FileReader
{
    FILE* file;
public:
    FileReader(const char* fileName)
    {
        file = fopen(fileName, "r");
    }
    
    ~FileReader()
    {
        fclose(file);
    }
};

int main()
{
    FileReader f("test.txt");
    return 0;
}
```

C:
```c
#include <cstdio>

int main()
{
    FILE* f = fopen("test.txt", "r");
    fclose(f);
    return 0;
}

```

Kompilacja C++ `g++ -O1 test.cpp`:
```asm
.LC0:
        .string "r"
.LC1:
        .string "test.txt"
main:
        sub     rsp, 8
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, OFFSET FLAT:.LC1
        call    fopen
        mov     rdi, rax
        call    fclose
        mov     eax, 0
        add     rsp, 8
        ret
```

Kompilacja w C `gcc -O1 test.c`:
```asm
.LC0:
        .string "r"
.LC1:
        .string "test.txt"
main:
        sub     rsp, 8
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, OFFSET FLAT:.LC1
        call    fopen
        mov     rdi, rax
        call    fclose
        mov     eax, 0
        add     rsp, 8
        ret
```

I znów wygenerowane kody asemblerowe się od siebie nie różnią.

Wszystkie inne funkcje specjalne, jak przeciążone operatory, operatory konwersji, konstruktory kopiujące i przenoszące i im podobne to tylko zwykłe funkcje. Przed kompilacją różnica jest taka, że (jak destruktory czy konstruktory kopiujące) są one czasem wywoływane niejawnie, lub (jak operatory) mają dodatkową składnię ich wywoływania. Po kompilacji różnicy nie ma żadnej.

#### Dziedziczenie i agregacja

Dziedziczenie jest kolejnym przykładem elementu języka, który ma znaczenie tylko dla programisty i kompilatora. Po kompilacji nie ma po nim śladu. Często początkujący programiści mają problemy z decyzją, czy w przypadku konkretnego problemu lepiej jest zastosować dziedziczenie, czy agregację. Dzieje się tak, ponieważ te dwie konstrukcje dają ten sam efekt. Dla kompilatora jest to jedynie informacja, że ma skleić dwa (lub więcej) „pojemników” na dane w jeden, większy. Ponownie, zobaczmy przykład:

Dziedziczenie w C++:
```c++
struct A
{
    char aChar;
};

struct B : public A
{
    float aFloat;
};

int main()
{
    B obj;
    obj.aChar = 'a';
    obj.aFloat = 3.14;
    return 0;
}
```
Agregacja w C:
```c
struct A
{
    char aChar;
};

struct B
{
    struct A a;
    float aFloat;
};

int main()
{
    struct B obj;
    obj.a.aChar = 'a';
    obj.aFloat = 3.14;
    return 0;
}
```

Płaska struktura w C:
```c
struct B
{
    char aChar;
    float aFloat;
};

int main()
{
    struct B obj;
    obj.aChar = 'a';
    obj.aFloat = 3.14;
    return 0;
}
```

Wszystkie 3 kody źródłowe kompilują się do identycznego, co do joty, asemblera ([dziedziczenie](https://godbolt.org/z/a25lye), [agregacja](https://godbolt.org/z/f-Iz4X), [płaska struktura](https://godbolt.org/z/a1qpTK)).

### Abstrakcja 2 – statyczny polimorfizm

#### Przeciążanie funkcji

C++ pozwala na rozróżnianie funkcji na podstawie typów parametrów:
```c++
bool isPositive(int value)
{
    return value > 0;
}

bool isPositive(float value)
{
    return value > 0;
}

int main()
{
    return isPositive(-1);
}
```

Standardowo C nie pozwala na podobne zabiegi, więc funkcje muszą się różnić nazwą:
```c
int isPositiveInt(int value)
{
    return value > 0;
}

int isPositiveFloat(float value)
{
    return value > 0;
}

int main()
{
    return isPositiveInt(-1);
}
```

Kompilacja C++ – `g++ -O0 test.cpp`:
```asm
isPositive(int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        cmp     DWORD PTR [rbp-4], 0
        setg    al
        pop     rbp
        ret
isPositive(float):
        push    rbp
        mov     rbp, rsp
        movss   DWORD PTR [rbp-4], xmm0
        movss   xmm0, DWORD PTR [rbp-4]
        pxor    xmm1, xmm1
        comiss  xmm0, xmm1
        seta    al
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, -1
        call    isPositive(int)
        movzx   eax, al
        pop     rbp
        ret
```

Kompilacja C – `gcc -O0 test.c`:
```asm
isPositiveInt(int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        cmp     DWORD PTR [rbp-4], 0
        setg    al
        movzx   eax, al
        pop     rbp
        ret
isPositiveFloat(float):
        push    rbp
        mov     rbp, rsp
        movss   DWORD PTR [rbp-4], xmm0
        movss   xmm0, DWORD PTR [rbp-4]
        pxor    xmm1, xmm1
        comiss  xmm0, xmm1
        seta    al
        movzx   eax, al
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, -1
        call    isPositiveInt(int)
        nop
        pop     rbp
        ret
```

Jak widać, znów kod wynikowy jest taki sam, jak wersja z C. Warto się pokusić o kilka słów komentarza. Trochę się tutaj powtórzę, bo ponownie napiszę, że typy zmiennych przy kompilacji są usuwane. Kompilator statycznie, w trakcie działania, wybierze odpowiednie przeciążenie na podstawie typu (również na podstawie modyfikatorów `const`, `volatile` czy też [kategorii wartości](https://cpp-polska.pl/post/podzial-wyrazen-ze-wzgledu-na-kategorie-wartosci-w-c)), po czym na zawsze informacji o tym typie się pozbędzie. W przypadku języka C wybór ten nie jest automatyczny, po prostu programista musi wywołać odpowiednią funkcję.

#### Szablony
Ze statycznym polimorfizmem C++ poszedł kilka kroków dalej niż inne języki – chyba nigdzie indziej nie znajdziemy tak rozbudowanego mechanizmu szablonów, jak w C++. I to szablonów, których ewaluacja w całości odbywa się w czasie kompilacji. Spójrzmy jednak na przykład najprostszy z najprostszych – rozwinięcie poprzedniego przykładu:

```c++
template<typename T>
bool isPositive(T value)
{
    return value > 0;
}

int main()
{
    return isPositive(-1);
}
```

Po kompilacji – `g++ -O0 test.cpp`:
```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, -1
        call    bool isPositive<int>(int)
        movzx   eax, al
        pop     rbp
        ret
bool isPositive<int>(int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        cmp     DWORD PTR [rbp-4], 0
        setg    al
        pop     rbp
        ret
```

Tutaj widzimy coś, czego wcześniej nie było. Abstrakcja nie tylko zaoszczędziła nam pisania kodu. Dodatkowo pozwoliła na lepszą optymalizację kodu maszynowego. Tutaj nie tylko abstrakcja nie ma kosztu, ale potrafi dać profit. Chociaż trzeba przyznać, że jest on pozorny. Większy poziom optymalizacji usunąłby nieużywaną funkcję również w poprzednim przykładzie (pod warunkiem, że nie byłaby ona funkcją `external linkage`).

Każdy inny szablon w C++ również należy do grupy `Zero Cost Abstraction`. Dzieje się tak, ponieważ zawsze, bez wyjątku, proces instancjacji szablonu odbywa się w czasie kompilacji. Dokładnie tak, jakbyśmy napisali tyle różnych, niezwiązanych ze sobą klas/funkcji ile różnych kombinacji typów można dla danego szablonu napotkać w całym naszym kodzie. Ktoś mógłby powiedzieć, że koszt jest – każda instancjacja szablonu to kawał kodu maszynowego do wygenerowania. Programy używające dużo szablonów szybko się rozrastają. Mimo że nie musi to wpływać negatywnie na wydajność naszego programu (i zazwyczaj nie wpływa), to wpływa na wielkość binarki. Tyle że – patrząc obiektywnie – skoro konkretne instancjacje były nam potrzebne, to i tak musielibyśmy ten kod napisać – czy to w wersji szablonowej, czy nie.

Warto dodać, że istnieją języki, jak `Java` czy `C#`, w których szablony działają na całkowicie innej zasadzie i nie są pozbawione kosztów. Nie o nich jednak tutaj mowa.

### Abstrakcja 3 – lambda

Lambda jest obiektem nienazwanej klasy, ze zdefiniowanym operatorem wywołania (`operator()`). I właściwie na tej definicji moglibyśmy skończyć analizę tego elementu C++ – pokazaliśmy już przecież, że obiekty zwykłych klas mają zerowy koszt abstrakcji. Dla niedowiarków napiszemy jednak przykład.

C++:
```c++
int main(int argc, char* argv[])
{
    auto getParam = [argc, argv](int num) -> char*
    {
        if(num >= argc)
            return nullptr;
        return argv[num];
    };
    
    char* first = getParam(0);
    
    return 0;
}
```

Również w wersji C++, ale bez użycia lambdy wyglądałoby to tak:
```c++
struct ParamGetter
{
    int argc;
    char** argv;
    
    char* operator()(int num) const
    {
        if(num >= argc)
            return nullptr;
        return argv[num];
    }
};

int main(int argc, char* argv[])
{
    auto getParam = ParamGetter{argc, argv};
    
    char* first = getParam(0);
    
    return 0;
}
```

Powyższy kod przekłada się na następujący w C:
```c
struct Params
{
    int argc;
    char** argv;
};

char* getParam(const struct Params* this, int num)
{
    if(num >= this->argc)
        return 0;
    return this->argv[num];
}

int main(int argc, char* argv[])
{
    struct Params params = {argc, argv};
    
    char* first = getParam(&params, 0);
    
    return 0;
}
```

Śmiało można uznać, że powyższe kody to ekwiwalenty 1:1 ([link1](https://godbolt.org/z/K-MrdM), [link2](https://godbolt.org/z/vopEbQ), [link3](https://godbolt.org/z/T7xVqW)).

## Co w C++ nie jest darmową abstrakcją?
Takie rzeczy też istnieją. A właściwie na dzień dzisiejszy (C++17) są trzy. `Funkcje wirtualne` (polimorfizm dynamiczny), `mechanizm wyjątków` oraz `RTTI`. Nie obejmuje ich pierwsza część zasady przytoczonej we wstępie. Ma tutaj zastosowanie druga część. To znaczy – jeżeli nie używasz w swoim programie mechanizmu wyjątków bądź funkcji wirtualnych, to nie płacisz za nie żadnej ceny. Uświadomienie sobie tego prostego faktu pozwala spać spokojnie tym, którzy nie są pewni czy postąpili właściwie, wybierając do swojego projektu (w którym wydajność jest kluczowa) język C++ zamiast C. Czy jednak użycie tych elementów języka pozwala spać spokojnie? Sprawdźmy...

### Koszt obsługi wyjątków

Zobaczmy minimalny przykład:

```c++
#include <ctime>

int divide(int a, int b)
{
    if (b == 0)
        throw 0;
    return a / b;
}

void handleException()
{
    //...
}

int main()
{
    try
    {
        divide(std::time(nullptr), std::time(nullptr));
    }
    catch(int val)
    {
        handleException();
    }
    return 0;
}
```

Nie będę tutaj wklejał wygenerowanego assemblera dla tego kodu, wkleję tylko linka ([link](https://godbolt.org/z/J6UVjz)). Dla porównania kod w C bez obsługi sytuacji wyjątkowej:

```c
#include <time.h>

int divide(int a, int b)
{
    return a / b;
}

int main()
{
    divide(time(NULL), time(NULL));
    return 0;
}
```

Rzecz jasna, gdyby napisać podobny kod (czy to w C++, czy w C), który nie obsłużyłby sytuacji wyjątkowej dzielenia przez zero, to wygenerowany assembler byłby sporo krótszy. No właśnie, byłby krótszy, ale też działałby inaczej w tej wyjątkowej sytuacji. W teorii – przy dzieleniu przez zero – wygenerowałoby to niezdefiniowane zachowanie, w praktyce program prawdopodobnie po prostu by się wykrzaczył.

Mechanizm wyjątków nie ma odpowiednika 1:1 w języku C. Możemy jedynie próbować napisać kod, który realizowałby to zadanie:

```c
#include <time.h>

typedef struct
{
    int success;
    int value;
} result;

void handleException()
{
    //...
}

result divide(int a, int b)
{
    result r;
    if (b == 0)
    {
        r.success = 0;
        return r;
    }
    else
    {
        r.success = 1;
        r.value = a / b;
    }
    return r;
}

int main()
{
    result r = divide(time(NULL), time(NULL));
    if (r.success == 0)
        handleException();

    return 0;
}
```

Teraz moglibyśmy porównać [assembler wygenerowany przez wersję z wyjątkami](https://godbolt.org/z/J6UVjz) oraz [wersję napisaną w C](https://godbolt.org/z/Aup6K0). Ale czy na pewno są to dokładnie ekwiwalenty? Otóż nie! Mechanizm wyjątków dodatkowo:

* zapewnia obsługę dowolnej głębokości stosu wywołań;
* implementuje mechanizm rozwikłania stosu – wywołuje wszystkie odpowiednie destruktory;
* wywołuje funkcję z std::terminate, jeżeli wyjątek zostanie rzucony podczas rozwikływania stosu.

Nie potrzebujesz tego wszystkiego? Nie używaj więc wyjątków. Napisz kod, który wygląda tak jak powyższy w C. Możesz obsłużyć błędy na kilka różnych sposobów. Masz do dyspozycji `std::error_code`, możesz napisać kod, który zwróci instancję klasy `std::optional` lub `boost::outcome::result`. **Co najważniejsze, nie podejmuje za programistę tej decyzji twórca języka, tylko sam programista, pisząc swój kod.**

### Koszt funkcji wirtualnych

Napisanie w języku C czegoś, co działałoby jak metody wirtualne w C++ jest możliwe. Nie jest to jednak sprawa prosta, gdyż ogranicza nas silne typowanie języka C. Typ wskaźnika do klasy bazowej i klasy pochodnej nie są ze sobą kompatybilne. Możemy się jednak posłużyć jedną z metod tak zwanego `type erasure` (w wolnym tłumaczeniu „wymazywania typów”). Konkretnie faktem, że każdy wskaźnik możemy rzutować na `void*`, a funkcję, która przyjmuje wskaźnik do struktury, możemy rzutować na taką, która przyjmuje `void*`. Przedstawię tutaj implementację mechanizmu bardzo podobnego do tego, który kompilatory C++ niejawnie stosują dla klas polimorficznych (posiadających metody wirtualne), opartą na tablicach `vtable`. Od razu zaznaczam, że ta implementacja ma pewne ograniczenia – przykładowo nie zadziała poprawnie w przypadku wielodziedziczenia. Ma ona dać jedynie obraz tego, jaki koszt przychodzi nam zapłacić za używanie klas polimorficznych w C++.

Na początek kod w C++, który później spróbujemy odtworzyć w C:

```c++
#include <cstdio>

struct Interface
{
    virtual const char* fun1() = 0;
    virtual const char* fun2() { printf("this: %p\n", this); return "Interface fun2"; }
};

struct Impl1 : public Interface
{
    const char* fun1() override { printf("this: %p\n", this); return "Impl1 fun1"; }
    const char* fun2() override { printf("this: %p\n", this); return "Impl1 fun2"; }
};

struct Impl2 : public Interface
{
    const char* fun1() override { printf("this: %p\n", this); return "Impl2 fun1"; }
    const char* fun2() override { printf("this: %p\n", this); return "Impl2 fun2"; }
};

void print(Interface& obj)
{
    std::puts(obj.fun1());
}

int main()
{
    Impl2 obj;
    printf("obj: %p\n", &obj); 
    print(obj);
    return 0;
}
```

Powyższy kod zadziała mniej-więcej tak, jak następujący:

```c
#include <stdio.h>

#define ERASE_THIS_TYPE(fun) (const char* (*)(void* this))fun

/*  Interface  */
struct Interface
{    
    struct InterfaceVtable* vtable;
};

struct InterfaceVtable
{
    const char* (* fun1_ptr)(void* this);
    const char* (* fun2_ptr)(void* this);
};

const char* Interface_fun2(struct Interface* this) { printf("this: %p\n", this); return "Interface fun2"; }
struct InterfaceVtable STATIC_INTERFACE_VTABLE = {0, ERASE_THIS_TYPE(Interface_fun2)};

/*  Impl1  */
struct Impl1
{
    struct Interface base;
};
const char* Impl1_fun1(struct Impl1* this) { printf("this: %p\n", this); return "Impl1 fun1"; }
const char* Impl1_fun2(struct Impl1* this) { printf("this: %p\n", this); return "Impl1 fun2"; }
struct InterfaceVtable STATIC_INTERFACE_IMPL1_VTABLE = {ERASE_THIS_TYPE(Impl1_fun1), ERASE_THIS_TYPE(Impl1_fun2)};

/*  Impl2  */
struct Impl2
{
    struct Interface base;
};
const char* Impl2_fun1(struct Impl2* this) { printf("this: %p\n", this); return "Impl2 fun1"; }
const char* Impl2_fun2(struct Impl2* this) { printf("this: %p\n", this); return "Impl2 fun2"; }
struct InterfaceVtable STATIC_INTERFACE_IMPL2_VTABLE = {ERASE_THIS_TYPE(Impl2_fun1), ERASE_THIS_TYPE(Impl2_fun2)};

void print(struct Interface* obj)
{
    puts(obj->vtable->fun1_ptr(obj));
}

int main()
{
    struct Impl2 obj = {.base = {.vtable = &STATIC_INTERFACE_IMPL2_VTABLE}};
    printf("obj: %p\n", &obj);
    print(&obj.base);
    return 0;
}
```

Jeszcze raz zaznaczam, że powyższa implementacja mechanizmu metod wirtualnych jest uproszczona. Pokazuje jednak dość wiernie jego koszt:

* klasa polimorficzna zachowuje się tak, jakby miała jedno dodatkowe pole typu wskaźnikowego (jest to wskaźnik na `vtable`). Można sprawdzić, że ma większy rozmiar (sizeof) o rozmiar jednego wskaźnika (na obiekt `vtable`) w porównaniu z taką samą klasą bez funkcji wirtualnych plus ewentualne wyrównanie (ang. *padding*);
* gdzieś w pamięci muszą być odpowiednie tablice `vtable` dla każdej z implementacji (zauważmy, że nie byłoby sensu robić ich dla każdego obiektu);
* podczas wywoływania funkcji wirtualnej trzeba najpierw pobrać jej adres z `vtable`. Wbrew pozorom nie jest to wielki koszt – `vtable` ma ściśle określoną strukturę, przez co nie trzeba jej przeszukiwać, więc jest to jedna operacja o złożoność `O(1)`.

Na ogół koszt funkcji wirtualnych sprowadza się do powyższych 3 punktów. Jeżeli mamy do czynienia z wielokrotnym dziedziczeniem to obiektów `vtable` jest tyle, ile polimorficznych klas bazowych. Osobiście spotkałem się kilkakrotnie z kodem napisanym w C, który implementował w mniej lub bardziej podobny sposób to, co przedstawiłem powyżej. Więc, mimo że nie jest to darmowa abstrakcja, to czasami właśnie jej koszt „płacą” programiści C, mimo że pewnie często robią to bardziej świadomie.

Oczywiście można by było napisać `switch`a, `if`a, czy inną konstrukcję, która realizowałaby podobny mechanizm. Zapiszę jednak to po raz kolejny, aby utkwiło to każdemu czytelnikowi w pamięci: **Nie podejmuje za programistę tej decyzji twórca języka, tylko sam programista, pisząc swój kod.**

Wspomnę tutaj jeszcze o jednej rzeczy – istnieje pewna optymalizacja kompilatora, która ma nawet swoją nazwę: `dewirtualizacja`. Czasem kompilator może usunąć cały narzut wygenerowany przez mechanizm metod wirtualnych. Przykładowo wtedy, kiedy jest pewien, że dany interfejs będzie miał tylko jedną implementację. Jest to z pewnością dobry temat na osobny artykuł. Tutaj tylko przypomnę: zaufaj swojemu kompilatorowi.

### Koszt RTTI

Jak wcześniej wspomniałem, w ogólnym przypadku informacje o typie danych są potrzebne jedynie programiście i kompilatorowi. Kiedy jednak chcemy uzyskać informację o typie już w trakcie działania programu, możemy użyć [narzędzi RTTI](https://cpp-polska.pl/post/dynamic-cast-oraz-type-id-jako-narzedzia-rtti) (ang. *Run Time Type Information*). Taka informacja musi jednak wtedy zostać „wkomponowana” w binarkę, jako zbiór dodatkowych danych. Ponieważ RTTI działa odrobinę inaczej dla typów polimorficznych a inaczej dla zwykłych, używanie go w różnych kontekstach może mieć różny koszt:

* typy polimorficzne zazwyczaj przechowują swoje dane o typie w tablicy `vtable`, jako dodatkowe pole typu `std::type_info`.
* typy niepolimorficzne generują dodatkowe, globalne obiekty typu `std::type_info`. Są one tworzone tylko w razie wystąpienia takiej potrzeby – na przykład, gdy w kodzie źródłowym jest wywołany operator `typeid` dla danego typu lub jest rzucony wyjątek o takim typie. Konieczność wygenerowania obiektów `std::type_info` dla typów niepolimorficznych jest całkowicie weryfikowalna już w trakcie kompilacji.

Źródła do nauki języka C++ często podają, że używanie RTTI jest złą praktyką. Nie będę tutaj rozstrzygał, czy tak jest naprawdę. Widzimy tutaj jednak jeden z powodów, dlaczego nie warto nadużywać tego mechanizmu. On po prostu kosztuje.

## Podsumowanie

Przykłady `Zero Cost Abstraction` w C++ wraz z ekwiwalentami napisanymi w czystym C moglibyśmy mnożyć w nieskończoność. Zwłaszcza gdy oprócz konstrukcji czysto językowych przeanalizowalibyśmy bibliotekę standardową. Tutaj jednak chodzi o coś bardziej użytecznego – o intuicję. Odpowiadając na pytanie ze wstępu – czy C++ jest wolniejszy od C? – niestety, tutaj nie można podać konkretnej odpowiedzi. Byłoby to wysoce niesprawiedliwe. I osobiście przestrzegam przed każdym, kto twierdzi z przekonaniem, że jeden z tych języków jest szybszy. To po prostu zależy. Dobra informacja jest taka, że nie zależy to w ogóle od języka. Zależy to od programisty (w jaki sposób napisze swój kod) oraz, w małym stopniu, od kompilatora. Dlatego tak ważne jest, aby programista języków takich jak C++ wyrobił sobie intuicję co do tego, w jaki sposób kompilator potraktuje jego kod i jaki narzut powodują konstrukcje, których używa. Mam nadzieję, że lektura tego artykułu pomogła osiągnąć ten cel.
