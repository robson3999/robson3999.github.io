---
layout: post
title: "Rozpoznawanie obrazów w Rubim? Hackyeah 2018"
date: 2018-11-26T21:26:11+01:00
draft: false
keywords: "hackathon, aws, rekognition, ruby, rubyonrails"
---

W miniony weekend (24 - 25 listopada) w Warszawie miał miejsce hackathon [HackYeah](https://hackyeah.pl). Ja i mój team realizowaliśmy bardzo ambitny pomysł na aplikację dla osób cierpiących na niewydolność nerek lub cukrzycę. Jak później się okazało, właściwie apka mogłaby być używana przez każdego.  

TL;DR: Pomaga śledzić co jemy, ile ma to wartości odżywczych i wody. Dla osób z takimi problemami zdrowotnymi, jakie wymieniłem wyżej, śledzenie przyjmowanych pokarmów czasem może być kwestią nie tylko odpowiedniego samopoczucia, ale nawet życia. Dlatego stworzyliśmy aplikację.

## Idea
Użytkownik pobiera naszą aplikację, by zacząć kontrolować swoje nawyki żywieniowe. Nadchodzi czas pierwszego posiłku, wyjmuje telefon, odpala apkę. Robi zdjęcie jedzenia, wysyła. Po chwili dostaje odpowiedź - 'zupa pomidorowa'. Użytkownik dopisuje 300 gramów wagi posiłku i zapisuje w bazie danych. Później może wybrać go z listy i sprawdzić jakie składniki odżywcze i ile wody znajdowało się w zjedzonym daniu, przekąsce lub napoju.

## Rzeczywistość
Mając 24 godziny na zrealizowanie tak złożonego pomysłu nie udało się ukończyć go w stu procentach. Mimo to, w niedzielę rano byliśmy pod wrażeniem jak dużo udało się osiągnać! Co prawda, gdy aplikacja wysyłała zdjęcie pomarańczy, to najczęściej otrzymywała odpowiedź 'Fruit' lub "Citrus fruit", ale hej! TO DZIAŁA!

## Jak zacząłem rozpoznawać obrazy w Rubim
Coś tam słyszałem o Tensorflow, ale nie za wiele. Zgłębiłem się zatem w temat. Szybko znalazłem repozytorium [tensorflow.rb](https://github.com/somaticio/tensorflow.rb). Pomyślałem 'wow! świetnie! wszystko odpalę u siebie na komputerze i nie będzie problemu!'. Gdy próbowałem zainstalować Tensorflow na moim lapku i po raz trzeci system został zamrożony, tak, że przestała odpowiadać nawet myszka, stwierdziłem, że musi istnieć inne rozwiązanie.  Znalazłem AWS Rekognition.  

Amazon Web Services udostępnia usługę rozpoznawania obrazów za darmo do 5000 requestów na miesiąc. Idealny plan na hackathon. Konto na AWS już miałem, więc musiałem tylko dodać tą usługę do użytkownika. Po drodze znalazłem całkiem przydatny artykuł na [Medium](https://medium.com/statuscode/how-to-detect-image-contents-from-ruby-with-amazon-rekognition-46a962cb040f).

Dodałem wygenerowane klucze do `rails credentials` (moim zdaniem super rozwiązanie), zainstalowałem gem `aws-sdk` i byłem niemal gotów do rozpoznawania obiektów na obrazach wysyłanych z aplikacji!

Ustaliliśmy, że będę otrzymywał obrazki zakodowane w base64. Dzięki temu mogłem zrezygnować z przechowywania obrazów w formie plików na serwerze, więc deploy na Heroku był o wiele szybszy. Tak się złożyło, że Ruby świetnie sobie poradził z przekonwertowaniem base64 do bajcików, które chciał otrzymać AWS. A więc jedyne co musiałem zrobić to:
```ruby
# app/controllers/concerns/file_converter.rb
module FileConverter
  def convert_file(base64)
    Base64.decode64(base64)
  end
end
```

Taki moduł includowałem w kontrolerze dla plików i wysyłałem na AWS (chamsko skopiowany z artykułu na medium):
```ruby
# app/controllers/concerns/aws_caller.rb
require 'aws-sdk'

module AwsCaller
  def process_image(image)
    client = Aws::Rekognition::Client.new(
      access_key_id: Rails.application.credentials.aws_access_key,
      secret_access_key: Rails.application.credentials.aws_secret_key,
      region: 'eu-west-1'
    )
    resp = client.detect_labels(
         image: { bytes: image }
       )
    # resp.labels.each do |label|
    #   puts "#{label.name}-#{label.confidence.to_i}"
    # end
    return resp.labels.first.name
  end
end
```
Jak widać w ostatnich linijkach zakomentowałem listowanie wyników i ich prawdopodobieństwa wystąpienia na obrazie i zwracałem tylko pierwszy rekord (z największym prawdopodobieństwa). Dla zdjęcia z pomarańczą `resp.labels` wyglądał mniej więcej tak:
```
Citrus Fruit-99
Lemon-98
Fruit-98
Plant-98
...
```
Cały kontroler odpowiedzialny za obsługę zdjęć wyglądał tak:
```
# app/controllers/api/files_controller.rb
class Api::FilesController < ApplicationController
  include Api::AwsCaller
  include Api::FileConverter

  def create
      picture = convert_file(files_params[:base])
      resp = process_image(picture)
      json_response("Successfully created file", true, resp, :ok)
  end

  private

  def files_params
    params.require(:file)
  end
end
```
`json_response` jest tutaj funkcją pomocniczą includowaną w ApplicationControllerze. To taki formater do zwracania ładnego JSONa.  

Zdaję sobie sprawę, że kod tutaj napisany mógłby być lepiej uporządkowany, wrzucony pewnie do katalogu `/lib`, ale z racji tego, że powstał w 24h i rozpoznaje obrazy z dowolnego zdjęcia - jestem z niego zadowolony! :D

## Mamy nazwę i co dalej?
Stwierdziliśmy, że nie ma co kombinować i zostaliśmy przy języku angielskim. Ułatwiło to bardzo sprawę, ponieważ znaleźliśmy [NutritionixAPI](https://developer.nutritionix.com/), który jest fantastyczną usługą z ogromną bazą danych jedzenia i jego wartości odżywczych. Zapytania do API można pisać w naturalnym języku(!), więc (niestety dopiero po hackathonie to do mnie dotarło) użytkownik zamiast pisać wagi posiłku w gramach w osobnym polu, mógłby pisać po prostu 'a cup of tea' albo 'bowl of tomato soup'. I to rzeczywiście działa! Oczywiście jeśli podamy gramy, wszystkie wartości odżywcze zostają przeliczone ile ich jest rzeczywiście w takiej ilości danego pokarmu. Rozwiązanie fenomenalne - wystarczyło tylko wysyłać do niego wynik rozpoznanego obrazu.

Krótka uwaga - wynik z AWS Rekognition wysyłałem spowrotem do aplikacji, tak aby użytkownik mógł skorygować, to co zostało wygenerowane - czasem zdarzało się, że wynikiem było po prostu "Food", wtedy Nutritionix nie miał za wiele do powiedzenia. W takim wypadku użytkownik mógł manualnie wprowadzić nazwę posiłku i zatwierdzić.

## Faraday
Do odpytywania zewnętrzengo API wykorzystałem gem `Faraday`. I tak, tutaj pozwoliłem już sobie na mały refactor po weekendzie, stworzyłem moduł `Nutritionix`:
```ruby
# lib/nutritionix.rb
require 'faraday'
require 'json'

module Nutritionix
  module Caller
    def nutritionix_connection
      Faraday.new(:url => 'https://trackapi.nutritionix.com')
    end

    def search_for(meal)
      query = "#{meal.name} + #{meal.weight}g"
      conn = nutritionix_connection

      request = conn.post do |req|
        req.url "/v2/natural/nutrients"
        req.body = { 'query': query }
        req.headers['x-app-id'] = Rails.application.credentials.x_app_id
        req.headers['x-app-key'] = Rails.application.credentials.x_app_key
        req.headers['x-remote-user-id'] = '0'
      end

      response = JSON.parse(request.body)
      response
    end
  end

  module Parser
    def parse_nutritionix_object(nutri_obj)
      data = nutri_obj['foods'][0]
      result = {
          calories: data['nf_calories'],
          total_fat: data['nf_total_fat'],
          cholesterol: data['nf_cholesterol'],
          sodium: data['nf_sodium'],
          total_carbohydrate: data['nf_total_carbohydrate'],
          dietary_fiber: data['nf_dietary_fiber'],
          sugar: data['nf_sugars'],
          protein: data['nf_protein'],
          potasium: data['nf_potassium'],
          water: data['full_nutrients'].select 
                 {|data| data['attr_id'] == 255}[0]['value']
        }
    end
  end
end
```
Trochę sporo się dzieje, ale nie ma tu nic skomplikowanego. Moje metody są nawet dość prymitywne. Za pozyskiwanie ilości wody trochę nawet mi wstyd. Ale działało, więc nie przejmowałem się tym wtedy. (attr_id == 255 dlatego, że takie id miała właśnie woda)
W `Nutritionix::Caller` najpierw 'ustanawiam' połączenie z API i zwracam wynik zapytania (połączenie nazwy posiłku z jego wagą), a w `Nutritionix::Parser` porządkuję odpowiedź, żeby była bardziej przystępna dla aplikacji mobilnej. Nutritionix zwraca bardzo dużo informacji, których nie do końca potrzebowaliśmy w tym momencie, więc po prostu nie puszczałem ich dalej.

W kontrolerze wywoływałem tylko metody z modelu, które odwoływały się do modułu powyżej.
```ruby
# controller meals_controller.rb
  def show
    json_response(@meal.name, true, 
                 { meal: @meal, nutrients: @meal.view_nutrients }, :ok)
  end
```
```ruby
# model meal.rb
  include Nutritionix::Caller
  include Nutritionix::Parser

  def view_nutrients
    parse_nutritionix_object(get_nutrients)
  end

  def get_nutrients
    search_for(self)
  end
```
Tym oto sposobem do apki leciały wartości odżywcze naszego posiłku :)


- [Link do repo backendu](https://github.com/robson3999/hackyeah2018)  
- [Bardzo prymitywny, ale zawsze jakiś, klient webowy](http://hckyea18.surge.sh/)  