# SŁÓWKO *inline* – CZYLI JAK OSZUKAĆ LINKER?

Słowo kluczowe `inline` w języku C++ jest znane zapewne większości osób, które w co najmniej podstawowym stopniu opanowały język C++. Z mojej praktyki wynika jednak, że mniej doświadczonym osobom znane jest tylko jego jedno, i to mniej ważne zastosowanie. Skłoniło mnie to do popełnienia tego tekstu.

Na początek mam zagadkę – czy usunięcie słówka `inline` w dowolnym miejscu kodu źródłowego programu napisanego w C++ (nie zważając na wersję języka) może spowodować, że ten przestanie się – w szerokim tego słowa znaczeniu – budować? Na razie to pytanie pozostawię bez odpowiedzi, wrócę do tego później. Lubię posługiwać się przykładami, dlatego też od takiego zacznijmy.

Przykład prostego programu
===

Na początek zobaczmy przykład. Program, który liczy wartość liczby *π* oraz liczby *e* (a w zasadzie po prostu ją zwraca, ale dla nas to bez różnicy).

Do liczenia liczby *π* mamy pliki zdefiniowane następująco:

*pi.hpp*

```c++
#ifndef LICZE_SOBIE_PI
#define LICZE_SOBIE_PI
float policz_pi();
#endif LICZE_SOBIE_PI
```

*pi.cpp*

```c++
#include "pi.hpp"
#include <iostream>

inline void log(const char* cstring)
{
    std::cout << "[ PI ]: " << cstring << std::endl;
}

float policz_pi()
{
    log("zaczynam liczyc");
    //...
    log("koncze liczyc");
    return 3.1415;
}
```

Standardowo – w nagłówku deklaracja funkcji, którą wystawiamy jako API naszej biblioteki, a w pliku źródłowym implementacja. Dodatkowo mamy tutaj pomocniczą funkcję `log`, którą zrobimy `inline` – jest dość krótka.

Pójdźmy dalej – potrzebujemy podobnego kodu, liczącego wartość liczby *e*.

*e.hpp*

```c++
#ifndef LICZE_SOBIE_E
#define LICZE_SOBIE_E
float policz_e();
#endif //LICZE_SOBIE_E
```

*e.cpp*

```c++
#include "e.hpp"
#include <iostream>

inline void log(const char* cstring)
{
    std::cout << "[ e ]: " << cstring << std::endl;
}

float policz_e()
{
    log("zaczynam liczyc");
    //...
    log("koncze liczyc");
    return 2.7183;
}
```

Sytuacja bardzo podobna – to właściwie kalka poprzednich plików. Dodatkowo napiszmy funkcję `main` w osobnym pliku, aby przetestować nasze funkcje, oraz pomocniczy plik `Makefile` (dla uproszczenia używam kompilatora g++ "na sztywno").


*main.cpp*

```c++
#include "pi.hpp"
#include "e.hpp"
#include <iostream>

int main()
{
    float pi = policz_pi();
    float e = policz_e();
    std::cout << "liczba PI: " << pi << "\nliczba e: " << e << std::endl;
    return 0;
}
```


*Makefile*

```makefile
default: app.exe

pi.o: pi.cpp pi.hpp
	g++ -c pi.cpp -o pi.o

e.o: e.cpp e.hpp
	g++ -c e.cpp -o e.o

app.exe: pi.o e.o main.cpp
	g++ pi.o e.o main.cpp -o app.exe
```

Kompilacja, uruchomienie i dostajemy takie wyjście:

```
[ PI ]: zaczynam liczyc
[ PI ]: koncze liczyc
[ PI ]: zaczynam liczyc
[ PI ]: koncze liczyc
liczba PI: 3.1415
liczba e: 2.7183
```

## Co się tutaj stało?

Jak widzimy, coś niedobrego stało się z funkcją do logowania z pliku `e.cpp`. W nawiasach kwadratowych zamiast `[ e ]` wypisany został tekst `[ PI ]`. Właściwie została zawołana funkcja z pliku `pi.cpp`. Dlaczego? Lepszym pytaniem byłoby – *Dlaczego nie?*

Przyjrzyjmy się bliżej. Mamy w programie dwie funkcje o takiej samej nazwie. Nieroztropnie obie są umieszczone w globalnej przestrzeni nazw. I obie są funkcjami `inline`. Czy to ma jakieś znaczenie? Sprawdźmy – usuńmy `inline` z obu funkcji. Oto efekt uruchomienia programu `make` po takim zabiegu:

```
g++ -c pi.cpp -o pi.o
g++ -c e.cpp -o e.o
g++ pi.o e.o main.cpp -o app.exe
/opt/binutils-2.32/bin/ld: e.o: in function `log(char const*)':
e.cpp:(.text+0x0): multiple definition of `log(char const*)'; pi.o:pi.cpp:(.text+0x0): first defined here
collect2: error: ld returned 1 exit status
Makefile:10: recipe for target 'app.exe' failed
make: *** [app.exe] Error 1
```

Od razu mamy odpowiedź na zagadkę ze wstępu tego tekstu. Co prawda kompilacja powiodła się, ale linkowanie już nie. I tutaj dochodzimy do sedna głównego zastosowania słowa kluczowego `inline`.

Czym więc jest inline?
===

Można wyróżnić dwie rzeczy:

* `inline` informuje linker, że funkcja (a od C++17 prawie dowolny symbol) o danej sygnaturze i danej definicji może pojawić się w kilku jednostkach kompilacji *(ang. translation units)* i w takiej sytuacji linker może wybrać dowolną z nich a resztę odrzucić (nie powinno mieć to większego znaczenia, którą). Należy tutaj też podkreślić jedną rzecz – jeżeli **w tej samej jednostce kompilacji** znajdą się definicje dwóch takich samych funkcji,  kompilator ma obowiązek zgłosić błąd.
* `inline` _sugeruje_ kompilatorowi, żeby wstawił ciało funkcji w miejsce wywołania, zamiast samego wywołania.

Właściwie z technicznego punktu widzenia te dwa znaczenia można traktować jako jedno, gdyż mają ze sobą sporo wspólnego (bo jak można wypisać błąd o wielokrotnej definicji funkcji, skoro po kompilacji i tak nie powinna ona istnieć jako funkcja, tylko jako zbiór instrukcji "wkomponowanych" w miejsce wywołania?). Odseparowałem je jednak od siebie celowo, aby pokazać która funkcjonalność słowa `inline` we współczesnych kompilatorach jest naprawdę ważna.

Dla ścisłości warto dodać, że definiowanie dwóch funkcji `inline` o takiej samej sygnaturze, ale nie identycznych, nie jest w języku zdefiniowane (ang. Undefined Behaviour, UB). Kompilator, a właściwie linker, może z tym zrobić "co chce". W praktyce, jak widać w naszym przykładzie, po prostu jedną z definicji odrzucił.

Dodatkowo warto sobie zapamiętać, że niejawne oznaczenie *inline* następuje w dwóch sytuacjach:
* funkcje zwykłe, funkcje składowe i statyczne pola klasy oznaczone przez `constexpr`.
* funkcje zdefiniowane w ciele klasy (składowe i zaprzyjaźnione).

## Kiedy używać?

Po przejrzeniu przykładu podanego na początku można by było pomyśleć: *Stosowanie `inline` jest niebezpieczne i lepiej go nie używać. Przecież lepiej jest, żeby linker poinformował, że mamy kolizję nazw.* Odpowiadając – to dlatego, że przykład był raczej demonstracją tego, w jaki sposób nie należy używać tego narzędzia. Tak naprawdę oszukaliśmy tutaj linkera, bo w pewien sposób obiecaliśmy mu, że jeżeli napotka na dwie funkcje o tej samej sygnaturze, to będą one miały identyczne ciała. A linker tego nie sprawdził (nie jest to takie proste), tylko uwierzył nam na słowo. Po co więc "obiecywać"? Rozważmy kilka przypadków, które pokazują, jak użyteczne jest to narzędzie:

### Krótkie funkcje pisane w nagłówku
Posługując się przykładem, całkowicie z życia wziętym – powiedzmy, że mamy plik nagłówkowy z następującym typem wyliczeniowym:

```c++
enum class EReadMethod : bool
{
	UNBUFFERED,
	BUFFERED
};
```

załóżmy, że chcemy w logach programu napisać, która wartość została użyta. Zakładając, że nie mamy jeszcze C++23 i nie możemy posłużyć się *statyczną refleksją*, musimy napisać sobie funkcję `to_string`. Możemy stworzyć do tego specjalny plik źródłowy lub zamieścić całą definicję w pliku nagłówkowym:

```c++
inline const char* to_string(EReadMethod val)
{
    switch(val)
    {
    case EReadMethod::UNBUFFERED:
        return  "UNBUFFERED";
    case EReadMethod::BUFFERED:
        return "BUFFERED";
    }
}
```

W podanym przypadku, według mnie, słowo `inline` powinno być obowiązkowe. Co prawda projekt bez tego mógłby się zbudować, pod warunkiem, że funkcja użyta byłaby tylko w jednej jednostce kompilacji. Z drugiej strony, jaka jest szansa, że ktoś w innym miejscu programu umieści inną definicję funkcji o takiej samej nazwie i z taką samą listą parametrów? Z tym pytaniem zostawiam czytelnika.

### Biblioteki header-only
To właściwie bardzo podobny przypadek użycia jak w poprzednim podpunkcie. Tyle że cel jest inny – używanie bibliotek składających się jedynie z nagłówka jest po prostu dużo prostsze.

### Szablony klas
I tutaj – przynajmniej w mojej opinii – `inline` ma najpotężniejszą moc. Paradoksalnie, bo bardzo często w definicji szablonu nie znajdziemy słowa kluczowego `inline`, mimo że ono tam explicite jest. A to wszystko przez zasadę, że każda funkcja składowa klasy zdefiniowana w całości w jej ciele jest niejawnie `inline`. Podkreślam – każda – nie tylko taka w szablonie klasy. Jak wiemy, instancjacja szablonu wymaga pełnej jego definicji, dlatego nie możemy w jednej jednostce kompilacji podać jedynie deklaracji szablonu, a w drugiej jednostce kompilacji definicji. *(Co prawda jest to osiągalne, jeżeli wiemy dokładnie jakie instancjacje będą nam potrzebne, ale to szczególny przypadek).* Gdyby nie `inline`, za każdym razem, kiedy zinstancjonowalibyśmy tak samo szablon w różnych jednostkach kompilacji (np `std::vector<int>`), to linker odpowiadałby błędem. Gdy współpracuje się z biblioteką STL lub inną, opartą na szablonach, to sytuacja taka jest codziennością.

### Zmienne i stałe globalne (C++17)
W standardzie z 2017 roku możemy również zdefiniować stałą lub zmienną `inline`:

```c++
inline constexpr auto THREAD_POOL_SIZE{5};
inline ApplicationConfig globalAppConfig{};
```

Jest to miłe ułatwienie wprowadzone w nowym standardzie – bo skoro można tak z funkcją, to dlaczego nie z dowolnym symbolem? Ciekawostka – w starszych wersjach C++ można niektóre takie sytuacje obejść w ten sposób:

```c++
enum
{
	THREAD_POOL_SIZE = 5
};
```

### Statyczne pola klasy (C++17)

… definiowanie ich w C++ przed standardem C++17 było naprawdę brzydkie. Bo o ile deklaracja "zwykłej" składowej nie oznacza, że od razu musimy rezerwować na nią miejsce (potrzebujemy go dopiero przy tworzeniu obiektu) to dla składowej statycznej już tak. Należało więc gdzieś to miejsce w pamięci zarezerwować. Dlatego w pliku źródłowym należało to jawnie napisać, mniej więcej w taki sposób:

```c++
int NazwaKlasy::nazwaPola = 0;
```

Uciążliwe, prawda? W C++17 możemy w ciele klasy po prostu napisać:

```c++
static inline int nazwaPola = 0;
```

Gdyby się tak zdarzyło, że linker znajdzie kilka definicji tego pola statycznego, "znormalizuje" je do jednej. Wygodne.

# Co z drugim zastosowaniem inline?
Drugie zastosowanie jest dla kompilatora sugestią. Nie zmienia ono sposobu, w jaki kod zostanie wykonany. Nie zmieni znaczenia naszego kodu w sensie algorytmicznym. Jedynie co może zrobić, to zmienić sposób optymalizacji.

Nie przeglądałem ostatnio kodów źródłowych kompilatorów, ale jest duże prawdopodobieństwo, że współczesne kompilatory ignorują to znaczenie `inline`. A to dlatego, że optymalizacja to specjalność kompilatorów, i programista nie powinien się w nią za bardzo wtrącać. Jeżeli uważasz, że zoptymalizujesz coś lepiej niż współczesny kompilator, to pisz w assemblerze. To tak pół żartem, pół serio.

Jak uniknąć sytuacji z przykładu?
===

No dobrze, więc są sytuacje, kiedy `inline` ma sens, i to duży. Co jednak zrobić, żeby uniknąć sytuacji z przykładu? Kilka rad osób mądrzejszych ode mnie:

* Nigdy nie używaj `inline` w pliku źródłowym – używaj go tylko w nagłówkach;
* Funkcje, które są używane jedynie w pliku źródłowym, bez wystawiania ich jako API, definiuj w nienazwanej przestrzeni nazw lub oznaczaj słowem kluczowym `static` *(ang. internal linkage)*;
* Nie używaj `inline` tylko po to, żeby zasugerować kompilatorowi, że powinien funkcję rzeczywiście wstawić kontekst zamiast wywołania. Chyba że masz naprawdę dobry powód i jesteś świadom, dlaczego to robisz. Po pierwsze w dzisiejszych czasach kompilatory są bardzo sprytne i wiedzą jak co zoptymalizować (o czym pisałem wcześniej). Po drugie jest ogromna szansa, że współczesny kompilator i tak Twoją sugestię zignoruje.

Dodatkowo nadmienię, że w moich osobistych statystykach i tak sytuacja z przykładu nie istnieje. Ani sam się z taką nie spotkałem (nie licząc przypadku, kiedy specjalnie ją stworzyłem), ani mi się taka sytuacja nie obiła o uszy. A może po prostu mam zbyt małe doświadczenie – chętnie się dowiem, czy któryś z czytelników na coś takiego się natknął.

Źródła
===
* [C++17 standard ](https://timsong-cpp.github.io/cppwp/n4659/dcl.inline) [dostęp: 28 maja 2019]
* [cppreference.com – inline](https://en.cppreference.com/w/cpp/language/inline) [dostęp: 28 maja 2019]
* [Andrzej's C++ blog - inline functions](https://akrzemi1.wordpress.com/2014/07/14/inline-functions/) [dostęp: 28 maja 2019]
* [Simon Brand blog – Do compilers take inline as a hint?](https://blog.tartanllama.xyz/inline-hints/) [dostęp: 28 maja 2019]
