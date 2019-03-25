---
layout: post
title: "Tworzenie modeli, relacji i migracji"
date: 2018-09-15T09:40:11+02:00
draft: false
tags: "rubyonrails, basics"
---

W tym artykule przyjrzymy się projektowaniu modeli w prostej aplikacji. Przedstawię tutaj podstawowe relacje między nimi oraz komendy, dzięki którym można wygenerować modele z gotowymi migracjami do wykonania.  

Przyjmijmy, że mamy aplikację z trzema modelami: `Map`, `Tag`, `Marker` oraz `User`.  
`User` może dodawać wiele `Map`, `Map`a posiada wiele `Tag`ów oraz wiele `Marker`ów. Jeden `Tag` może być przypisany do wielu `Map`. `Marker` należy tylko do jednej mapy. Kiedy usuniemy mapę, chcemy by markery, które zostały do niej dodane również zostały usunięte.

Na podstawie tego opisu klarują nam się następujące relacje:  
```rb
class User
  has_many :maps
end

class Map
  belongs_to :user
  has_many :markers, dependent: :destroy
  has_and_belongs_to_many :tags
end

class Tag
  has_and_belongs_to_many :maps
end

class Marker
  belongs_to :map
end
```

Aby nie pisać wszystkiego ręcznie, możemy wykorzystać railsową komendę `rails generate`, by wygenerować modele i ich pola wraz z relacjami zawartymi między nimi w migracjach, które trzeba wykonać poprzez komendę `rails db:migrate`. Porządana struktura:  
- User: name, email,  
- Map: name, description,  
- Marker: name, description, lat, lng,  
- Tag: name  

Dodatkowo, by relacja `has_and_belongs_to_many` między tagami, a mapami miała rację bytu, musimy wygenerować migrację z dodatkową tabelą zawierającą kolumny `tag_id` oraz `map_id`.

Generowanie modelu User (w tym przypadku nie zawieramy żadnych relacji, musimy dodać je ręcznie):
```ruby
rails g model User name:string email:string
```
Migracja, którą otrzymaliśmy: `db/migrate/__timestamp__create_users.rb`:  
```ruby
class CreateUsers < ActiveRecord::Migration[5.2]
  def change
    create_table :users do |t|
      t.string :name

      t.timestamps
    end
  end
end
```
a w pliku `app/models/user.rb` dodajemy linię `has_many :maps`  
```ruby
class User < ApplicationRecord
  has_many :maps
end
```

Mapy:
```ruby
rails g model Map name:string description:text user:references
```
Pole `description` definiujemy za pomocą identyfikatora `text`, by mieć możliwość wpisywania dłuższych ciągów znaków niż w `name`. Dzięki `user:references`, do tabeli `maps` zostanie dodana kolumna `user_id`, na podstawie której będziemy mogli sprawdzić, który użytkownik stworzył daną mapę.  
Powyższa komenda generuje nam model `Map` z dodaną relacją `belongs_to :user`, dodajmy jeszcze pozostałe:  
```ruby
class Map < ApplicationRecord
  belongs_to :user
  has_many :markers, dependent: :destroy
  has_and_belongs_to_many :tags
end
```
Tagi:  
```ruby
rails g model Tag name:string
```
modyfikujemy, by zawierały relację z mapami.  
```ruby
class Tag < ApplicationRecord
  has_and_belongs_to_many :maps
end
```

Wspomniana dodatkowa tabela na połączenie `Map` z `Tag`ami:  
```ruby
rails g migration CreateJoinTableMapTag maps tags
```
w wyniku której otrzymujemy następującą migrację:  

```ruby
class CreateJoinTableMapTag < ActiveRecord::Migration[5.2]
  def change
    create_join_table :maps, :tags do |t|
      t.index [:map_id, :tag_id]
    end
  end
end

```
Markery:  
```ruby
rails g model Marker name:string description:text lat:float lng:float map:references
```
Tutaj pola `lat` i `lng` są typem Float. Tzn. liczbą zmiennoprzecinkową, ponieważ współrzędne geograficzne często mają rozwinięcie do kilku liczb po przecinku. Np. współrzędne Krakowa:  
lat: 50.0646501, lng: 19.9449799.  
`map:references` oczywiście dodaje nam relację `belongs_to :map` (zwróć uwagę na liczbę pojedynczą w `map` - ponieważ należy do jednej i tylko jednej mapy).  

W tym momencie mamy gotowe modele oraz migracje. Czas na komendę `rails db:migrate`!  
Po migracji możemy uruchomić railsową konsolę i sprawdzić, czy wszystko działa. Sprawdź, co zwraca konsola, kiedy wykonasz poniższe kroki:  
```bash
$ rails console
irb 1> user = User.new(name: 'Joe')
irb 2> user.save
irb 3> user.maps.create(name: 'Map1', description: 'My favourite places in Cracow')
irb 4> map = user.maps.first
irb 5> map.tags.create(name: 'cities')
irb 6> map.markers.create(name: 'Cracow', lat: 50.0646501, lng: 19.9449799, description: 'The center of Cracow')
irb 7> map.markers.count
irb 8> user.maps.create(name: 'Second map', description: 'My favourite places to camp')
irb 9> map2 = user.maps.find(2)
irb 10> map2.tags << map.tags.first
irb 11> map.destroy
irb 12> Marker.all
```
Jak widać, nasze założenia zostały spełnione:  
- nowo stworzony `User`(1, 2) może dodawać `Map`y (3),  
- `Map`a posiada `Tag`i(5) oraz `Marker`y(6, 7),  
- `Tag` może zostać dodany do więcej niż jednej `Map`y(8, 9, 10),  
- `Marker` należy tylko do jednej mapy. Jeśli wykonamy instrukcje z poprzedniego punktu, jednak zamiast `Tag` użyjemy `Marker`, marker dodany do pierwszej mapy zostanie przeniesiony do drugiej,  
- `Marker`y dodane do mapy zostają usunięte wraz z `Map`ą (11, 12).

### ActiveRecord Associations

W Railsach mamy więcej relacji, niż te, które zostały tu wymienione, a mianowicie:  
- [belongs_to](https://guides.rubyonrails.org/association_basics.html#the-belongs-to-association),  
- [has_one](https://guides.rubyonrails.org/association_basics.html#the-has-one-association),  
- [has_many](https://guides.rubyonrails.org/association_basics.html#the-has-many-association),  
- [has_and_belongs_to_many](https://guides.rubyonrails.org/association_basics.html#the-has-and-belongs-to-many-association),  
- [has_many :through](https://guides.rubyonrails.org/association_basics.html#the-has-one-through-association),  
- [has_one :through](https://guides.rubyonrails.org/association_basics.html#the-has-one-through-association),  
Oraz [polymorphic associations](https://guides.rubyonrails.org/association_basics.html#polymorphic-associations) oraz [self joins](https://guides.rubyonrails.org/association_basics.html#self-joins).  
Jest ich tyle, że nie sposób opisać ich w jednym artykule. Polecam [rails guides - ActiveRecord associations](https://guides.rubyonrails.org/association_basics.html), a także [rails guides - ActiveRecord migrations](https://guides.rubyonrails.org/active_record_migrations.html), z których wielokrotnie korzystałem pisząc powyższy tekst :)  

Pozdrawiam,  
**R**