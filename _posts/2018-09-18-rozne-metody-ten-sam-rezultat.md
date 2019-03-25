---
layout: post
title: "Różne metody, ten sam rezultat"
date: 2018-09-18T20:36:58+02:00
draft: false
tags: "ruby, rubyonrails"
---

Jak wiadomo, elastyczność Rubiego pozwala nam na wiele rozwiązań tego samego problemu. Programuję najczęsciej i najwięcej w Javascripcie, więc niestety przekładam na Rubiego niektóre nawyki z JSa. Czasem ze świadomością, że rozwiązanie, które stosuję jest typowo JSowe, a czasem (co gorsze) nie jestem świadomy, że mogę zrobić coś lepiej. W tym poście zawrę kilka metod, o których wiem, że powinno się stosować, jeśli ma się na uwadze wydajność swojej aplikacji. Mam stuprocentową świadomość tego, że temat nie zostanie wyczerpany, ponieważ sam wciąż się uczę i na pewno jest wiele innych sposobów, które można wykorzystywać w podanych tutaj przykładach.

### Select, map, where... Pluck?

Rozpatrzmy następujący przypadek: w tabeli `Like` zawieramy informację o użytkowniku, który go (`like`) stworzył (tj.`user_id`) oraz ID filmu, który polubił. Standardowo, znajduje się tam też `id` obiektu oraz pola `created_at` oraz `updated_at`. Wygląda to mniej więcej tak:
```ruby
=> #<Like:0x0000557e171804d0
 id: 2,
 movie_id: 4,
 user_id: 21,
 created_at: Fri, 10 Aug 2018 12:43:12 UTC +00:00,
 updated_at: Fri, 10 Aug 2018 12:43:12 UTC +00:00>
```
Załóżmy, że mamy tablicę zawierającą dziesięć obiektów `Like` i chcemy wybrać tylko te, których `user_id == 22`. Pierwszym pomysłem jaki przyszedł mi do głowy (powtórzę tutaj, jestem człowiekiem przychodzącym z JavaScriptu:) ) było zastosowanie metody `.select`, która przypomina mi JSową funkcję `filter()`. A zatem:
```ruby
Like.select { |like| like.user_id == 22 }
```
I mamy wynik. Jednak, jeśli zgłębimy się w działanie selecta, dowiemy się, iż pobiera on wszystkie wartości z bazy danych i iteruje po każdym elemencie tablicy w poszukiwaniu tych, które spełniają określony przez nas warunek. Nie ma w tym nic złego, jeśli zdajemy sobie z tego sprawę i używamy go świadomie. Jednak w naszym przypadku jest on ekstremalnie nieekonomiczny. Wyobraźmy sobie, że tabela `Like` zawiera pięćset tysięcy rekordów. Jeśli język Ruby miałby iterować po pięćset tysięcznoelementowej tablicy, użytkownicy serwisu mogliby szybko zmienić naszą aplikację na inną, szybszą.  

#### Where przychodzi z pomocą
Na szczęście mamy lepsze narzędzia niż `.select` do operowania na tak dużych zbiorach danych. Dzięki użyciu metody `.where`, nasze zapytanie zostanie "przekonwertowane" do języka SQL i wykonane bezpośrednio w bazie, która przecież została stworzona do operowania na ogromnych ilościach danych. Zamieniając selecta na where otrzymamy taki kawałek kodu:
```ruby
Like.where(user_id: 22)
```
I już. Możemy też użyć innych wersji:
```ruby
user_id = 22
Like.where("user_id = ?", user_id) # podstawiamy zmienną do warunku
Like.where("user_id = #{user_id}") # j.w., jednak niezbyt bezpieczne, lepsza wersja wyżej
```
Dlaczego sposób drugi jest mniej bezpieczny niż pierwszy? W pierwszym przypadku Railsy dbaja o to, by uniknąć ataku SQL injection i "sprawdzają" zmienną przed jej podstawieniem do warunku w bazie. W drugim przypadku zmienna jest bezpośrednio "wklejana" do zapytania, przez co omija ten swego rodzaju walidator.  

#### Bardziej złożone warunki
Posłużę się w tej sekcji przykładami z książki [Rails Tutorial](https://www.railstutorial.org/book/following_users).
Mamy model `Micropost` oraz model `User`. Chcemy stworzyć zapytanie, które umożliwi nam wyświetlanie mikropostów, które stworzył `current_user` lub userzy, których "śledzi". A więc powstanie mniej więcej coś takiego:
```ruby
Micropost.where("user_id IN (?) OR user_id = ?", following_ids, current_user.id)
```
gdzie `following_ids` to ID użytkowników, których `current_user` followuje, a `current_user.id` to oczywiście jego ID. Dopóki tablica `following_ids` nie jest zbyt duża, zapytanie działa w porządku, jednak przy większej ilości pojawia nam się podobny problem co przy `.select`cie z sekcji powyżej. Już nawet przy trzydziestu idkach zapytanie do bazy wygląda tak:
```ruby
SELECT "likes".* FROM "likes" WHERE (user_id IN (0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29) OR user_id = 1)
```
Wciąż trwa to w mgnieniu oka, jednak co jeśli byłoby ich tysiąc, a równocześnie wywołałoby tą metodę trzystu użytkowników? Mogłoby to spowodować poważne spowolnienie aplikacji, a przez to utratę użytkowników. Zatem z pomocą przychodzi...   
#### Subselect
Aby oszczędzić Rubiemu obsługiwania długich tablic z idkami przeniesiemy jej wywołanie do SQLa.  
Zaczniemy od zmiany powyższego kodu - pytajniki zamienimy na konkretne nazwy zmiennych, tak aby móc wykorzystać je w wielu miejscach - zaraz zobaczysz o co mi chodzi.
```ruby
Like.where("user_id IN (:following_ids) OR user_id = :user_id)", following_ids: following_ids, user_id: current_user.id)
```
Dzięki temu, w miejscu :following_ids możemy "wepchnąć" jeszcze jednego selecta, który zwolni Rubiego z wykonywania tego obliczenia:
```ruby
following_ids = "SELECT followed_id FROM relationships WHERE follower_id = :user_id"
Like.where("user_id IN (#{following_ids}) OR user_id = :user_id", :user_id = current_user.id)
```
Teraz nie jest istotne czym jest `relationships`, po prostu załóż, że z tej tabeli pobierani są użytkownicy obserwowani przez wybranego przez nas użytkownika. Jak widać - pod zmienną `following_ids` zapisaliśmy stringa odpowiadającego zapytaniu do SQLa, dzięki czemu wszelkie obliczenia zostaną wykonane przez bazę danych, a my dostaniemy porządany przez nas wynik.  

### Map, czy pluck?
Ostatnim przedstawionym przykładem, również opierającym się na podobnym problemie co poprzednie, jest użycie metody map. Załóżmy, że mamy standardową tabelę `User` i chcemy pobrać z niej wszystkie adresy email użytkowników. Możemy zrobić to przy pomocy metody `.map`:
```ruby
User.all.map(&:email)
User Load (0.7ms)  SELECT "users".* FROM "users"
```
Zwraca nam to tablicę emaili i jest pięknie. Zwróć jednak uwagę na zapytanie, które zostało wykonane do bazy. Nasza instrukcja spowodowała że pobralismy **wszystkie** informacje z tablicy `User`. I dopiero po ich pobraniu, wyświetlilismy tylko to co nam potrzebne - emaile. Jeśli, tak jak w tym przypadku, jedyne czego potrzebujemy to prosta lista z emailami, to możemy skorzystać z metody `pluck`:
```ruby
User.pluck(:email)
(0.1ms)  SELECT "users"."email" FROM "users"
```
Zwróć uwagę na różnicę w czasach wykonywania instrukcji. W momencie ich wykonywania w tablicy są tylko 22 rekordy, a już jest widoczna ogromna różnica! A w outpucie również otrzymujemy tablicę z emailami wszystkich userów. Oczywiście korzystając z `pluck` dostajemy "czystą" tablicę ze stringami, nie możemy na pobranych wartościach wykonywać żadnych metod zdefiniowanych w modelu `User`.

###

To by było na tyle.  
Pozdrawiam,  
**R**