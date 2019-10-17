# Późna inicjalizacja w C++

Późna inicjalizacja (ang. *lazy initialization*) to wzorzec projektowy wykorzystywany w praktycznie każdym języku. Jego zaletą jest przesunięcie w czasie konstrukcji jakiegoś obiektu, która jest w jakiś sposób kosztowna, lub jej całkowite uniknięcie, jeżeli akurat tak się złoży, że dany obiekt w ogóle nie będzie potrzebny. Spójrzmy jak zaimplementować ten wzorzec poprawnie w C++.

## Opis problemu

Stwórzmy sobie przykład z życia wzięty. Załóżmy, że naszym obiektem jest plik na dysku, który jest otwierany wraz z konstruktorem. Specjalnie nie używam klasy `std::ifstream`, ponieważ ona sama w sobie ma możliwość późniejszego otwarcia pliku, przez co użycie późnej inicjalizacji mijałoby się z celem. Dlatego też załóżmy, że mamy do dyspozycji klasę przedstawioną poniżej:

```c++
class File
{
public:
	File(std::string_view fileName)
	{
		std::cout << "Opening file " << fileName << std::endl;
	}
	~File()
	{
		std::cout << "Closing file" << std::endl;
	}
	File(const File&) = delete;
	File(File&&) = default;
	File& operator=(const File&) = delete;
	File& operator=(File&&) = default;

	void write(std::string_view str)
	{
		std::cout << "Writing to file: " << str << std::endl;
	}
};
```
Jak widzimy, plik jest otwierany już w konstruktorze, przez co nie da się odwlec jego otwarcia. Użyjemy tej klasy do zapisu jakiegoś pliku konfiguracyjnego:

```c++
class Config
{
	File file;
public:
	Config() : file{"config.txt"}
	{
		std::cout << "Config object created" << std::endl;
	}
	
	void addOption(std::string_view name, std::string_view value)
	{
		file.write(name);
		file.write(" = ");
		file.write(value);
		file.write("\n");
	}
};
```

Dodajmy do tego prosty przykład użycia:

```c++
int main()
{
	Config c;
	std::cout << "Some operations..." << std::endl;
	c.addOption("dark_mode", "true");
	c.addOption("font", "DejaVu Sans Mono");
}
```

[Uruchom na Wandbox](https://wandbox.org/permlink/hc0jKV9JdZypQ2IU)

Problemem w tej implementacji jest to, że otwieramy plik, być może na długo przed koniecznością pisania do niego. Może to zablokować możliwość manipulacji tym plikiem przez inne procesy, co jest efektem niepożądanym. Chcielibyśmy, aby plik został otwarty dopiero przy pierwszym wywołaniu funkcji `addOption`. Możemy to osiągnąć na kilka sposobów...

## Sposób pierwszy - niezainicjalizowany wskaźnik:

Wskaźniki wydają się być na pierwszy rzut oka rozwiązaniem – mogą one wskazywać na jakąś wartość lub nie wskazywać na nic (*nullptr*). Wróćmy do przykładu, a później omówimy, dlaczego jest to raczej kiepski sposób.

```c++
class Config
{
	File* file{nullptr};
public:
	Config()
	{
		std::cout << "Config object created" << std::endl;
	}

	~Config()
	{
		if (file)
			delete file;
	}
	
	void addOption(std::string_view name, std::string_view value)
	{
		if (!file)
			file = new File{"config.txt"};
		file->write(name);
		file->write(" = ");
		file->write(value);
		file->write("\n");
	}
};
```

[Uruchom na Wandbox](https://wandbox.org/permlink/8fB8ZdfFX9Pirtml)

Trzymanie pamięci zaalokowanej na stercie pod „gołym wskaźnikiem” jest w nowoczesnym C++ uważane za złą praktykę. Przede wszystkim przez to, że mieszanie ich z mechanizmem wyjątków może prowadzić do wycieków pamięci. Wymagają one także manualnego zwolnienia pamięci, co można obejść, posługując się wygodnym i lekkim wzorcem projektowym RAII.

## Sposób drugi – smart pointer

```c++
class Config
{
	std::unique_ptr<File> file{};
public:
	Config()
	{
		std::cout << "Config object created" << std::endl;
	}
	
	void addOption(std::string_view name, std::string_view value)
	{
		if (!file)
			file = std::make_unique<File>("config.txt");
		file->write(name);
		file->write(" = ");
		file->write(value);
		file->write("\n");
	}
};
```

[Uruchom na Wandbox](https://wandbox.org/permlink/7BzF8RbCo6juykiR)

Problem rozwiązany w dużo bardziej elegancki sposób. W porównaniu do oryginalnej implementacji ten sposób ma jednak jedną wadę – obiekt jest zaalokowany na stercie. Alokacja na stercie wymaga wywołania systemowego (ang. *syscall*), a liczbę wywołań systemowych powinniśmy raczej ograniczać. Odwołanie do obiektu spod wskaźnika wiąże się również z mniejszą możliwością optymalizacji programu niż odwołanie do pamięci ze stosu. Możemy więc zastanowić się nad kolejnym rozwiązaniem...

## Sposób trzeci – std::optional (C++17)

```c++
class Config
{
	std::optional<File> file{};
public:
	Config()
	{
		std::cout << "Config object created" << std::endl;
	}
	
	void addOption(std::string_view name, std::string_view value)
	{
		if (!file)
			file.emplace("config.txt");
		file->write(name);
		file->write(" = ");
		file->write(value);
		file->write("\n");
	}
};
```

[Uruchom na Wandbox](https://wandbox.org/permlink/cCBguHZn1cQVk1ki)

Ten kod niewiele różni się od poprzedniego. Interfejsy klas `unique_ptr` i `optional` są podobne, natomiast implementacja oraz przeznaczenie różni się w sposób istotny. Przede wszystkim w tym przypadku nasz obiekt znajduje się na stosie. Warto wspomnieć, że jeżeli nie używasz C++17, tylko jakiejś starszej wersji języka, możesz sięgnąć do biblioteki [Boost.Optional](https://www.boost.org/doc/libs/1_71_0/libs/optional/doc/html/index.html), która implementuje prawie bliźniaczą klasę w porównaniu do `std::optional`.

## Smart pointery a optional

* `unique_ptr` jest (jak wskazuje nazwa) opakowaniem wskaźnika, natomiast obiekty klasy `optional` zawierają bezpośrednio w sobie pamięć potrzebną do rezerwacji obiektu.
* **Domyślny konstruktor** klasy `unique_ptr` po prostu ustawia opakowany wskaźnik na wartość `nullptr`, podczas gdy `optional` od razu rezerwuje na stosie pamięć, która będzie użyta później do konstrukcji obiektu.
* **Funkcja make_unique**, czyli funkcja pomocnicza dla klasy `unique_ptr` robi dwie rzeczy – rezerwuje na stercie pamięć, która będzie użyta do przechowywania obiektu i od razu konstruuje obiekt z użyciem tej pamięci. Działa więc jak zwykły *operator new*. W klasie `optional` metoda `emplace`, która może być rozpatrywana jako ekwiwalent, wywołuje jedynie konstrukcję, na wcześniej zarezerwowanym na stosie fragmencie pamięci – działa więc jak mniej znany *placement new operator*.

Powyżej opisałem sposoby implementacji omawianych klas opakowujących. Warto wymienić, jakie konsekwencje ze sobą niosą:

* **Konstruktor kopiujący** klasy `unique_ptr` nie istnieje. Gdybyśmy zastąpili `unique_ptr` innym wskaźnikiem `shared_ptr`, to moglibyśmy taki wskaźnik skopiować, ale nadal oba wskazywałyby na jeden obiekt. Klasa `optional` wykonuje całkowitą kopię (ang. `deep copy`) zarządzanego obiektu. Analogicznie sytuacja wygląda w przypadku operatora `=`.
* **Konstruktor przenoszący** klasy `unique_ptr` również nie wykonuje pełnej kopii obiektu. Po prostu przenosi zarządzanie obiektem do innego wskaźnika. Konstruktor przenoszący klasy `optional` działa natomiast tak, jakbyśmy przenosili zarządzany obiekt.
* **Destruktor** klasy `unique_ptr` nie tylko niszczy obiekt, ale też zwalnia pamięć (działa jak *operator delete*). `optional` wywołuje destruktor zarządzanego obiektu, ale nie musi już zwalniać pamięci – będzie ona dostępna dla kolejnych obiektów pojawiających się na stosie.

## Której opcji używać do lazy initialization?

Trzeba przyznać, że przedstawione wcześniej zastosowanie klasy `optional` nie jest pierwszym, które przychodzi na myśl osobom, które się nią posługują. Jest to raczej klasa, która wyraża, że jakiś obiekt *jest* lub go *nie ma*. Tutaj wyraziliśmy fakt, że obiektu *jeszcze nie ma, ale pewnie będzie w przyszłości*. Jest to jednak jak najbardziej poprawne zastosowanie tej klasy.

Odpowiedź na pytanie, którego sposobu użyć do wyrażenia późnej inicjalizacji nie jest wcale trywialna. Początkującym radziłbym, żeby domyślnie używali właśnie klasy `optional` (czy to z *std*, czy z *boost*). Jeśli jednak przeanalizujemy tę kwestię bardziej szczegółowo, to możemy wysnuć następujące wnioski:

* **Smart pointerów** używajmy przede wszystkim wtedy, gdy chcemy odwlec rezerwację jakiejś dużej ilości pamięci, np. przeznaczonej do przechowywania zawartości pliku graficznego.
* `optional` preferujmy, kiedy nie ważna jest pamięć (jej ilość), tylko rezerwacja zasobów innego typu (jak uchwyty plików, gniazda sieciowe, wątki, procesy). Warto go też użyć, gdy konstrukcja obiektu nie jest możliwa od razu, tylko zależy od jakiegoś parametru, którego wartość nie jest jeszcze znana. Dodatkowo zazwyczaj użycie tej klasy będzie wydajniejsze – zwłaszcza gdy dysponujemy na przykład dużym wektorem takich obiektów i chcemy po nich iterować.

Nie zapominajmy również o właściwościach omawianych klas, a szczególnie o tym, jak są one kopiowane i przenoszone.
