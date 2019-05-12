---
title: 'Dealing with FTP #3'
layout: post
keywords: rubyonrails, ruby, ftp, files
---

[Ostatnio](/2019/dealing-with-ftp-2/) zsynchronizowaliśmy bazę danych z plikami zdalnymi. Dziś zajmiemy się wysyłaniem i zapisywaniem plików lokalnych na zewnętrzym serwerze oraz w bazie danych.
Czynności, które musimy wykonać to:
- połaczenie z serwerem, zapisanie na nim wybranego pliku,
- zapisanie informacji n.t. nowego pliku w bazie danych.

Tak jak poprzednio, akcje te podzielimy na osobne serwisy. Rozpoczniemy od połączenia z serwerem i tak samo jak w poprzednich przykładach serwis będzie dziedziczył z `ApplicationService` w celu ułatwienia nam połączenia.
```ruby
require 'net/ftp'

module Media
  class UploadFile < ApplicationService
    def call(file)
      Net::FTP.open(host) do |ftp|
        ftp.login(username, password)
        ftp.putbinaryfile(file.tempfile, file.original_filename)
      rescue Net::FTPPermError => e
        puts e
        failure('Connection refused')
      end
    end
  end
end
```
Do zapisania pliku, po połączeniu z serwerem, wystarczy tylko jedna linijka kodu! A dokładniej metoda `putbinaryfile`. W tym przypadku korzystamy z `file.tempfile` oraz `file.original_filename`, ponieważ zakładamy, że plik, który został przekazany do funkcji `call` został wysłany do kontrolera przez formularz. Tzn. użytkownik wszedł na stronę, w formularzu wybrał plik z komputera, który chciał wysłać i go zatwierdził. Do przykładowego kodu kontrolera dojdziemy za chwilę, wróćmy jeszcze do zapisywania pliku.  

Po udanym zapisie na zewnętrznym serwerze, musimy zachować dane n.t. pliku w  bazie danych (aby nie musieć po każdym uploadzie wykonywać synchronizacji). Posłuży nam do tego napisany ostatnio serwis `Media::Create`.  Proponuję, by akcje zapisu na serwerze oraz rejestrowania w bazie danych "opakować" w jeszcze jeden serwis, który będzie wywoływany przez kontroler.
```ruby
module Media
  class UploadAndSave
    def call(file)
      Media::UploadFile.new.call(file)
      Media::Create.call(media_params: media_params(file)) 
    end

    private

    def media_params(file)
      {
        name: file.original_filename,
        bytesize: file.size,
        mimetype: file.content_type
      }
    end
  end
end
```

W kontrolerze mielibyśmy przykładowo taki kod:
```ruby
class MediaController < ApplicationController
  ...
  def perform_uploading
    Media::UploadAndSave.new.call(media_params[:file])
  end
	...
  def media_params
    params.require(:medium).permit(:file)
  end
end
```
W wyniku wysłania formularza użytkownik równocześnie wysłałby plik na serwer FTP oraz zapisał dane na jego temat w bazie danych.


Powyższy kod jest maksymalnie uproszczony, tak by pokazać jedynie funkcjonalność zapisu i nie rozpraszać innymi aspektami. W kodzie produkcyjnym na pewno powinno pojawić się wyłapywanie wyjatków, komunikowanie użytkownika o niepowodzeniu, itd...