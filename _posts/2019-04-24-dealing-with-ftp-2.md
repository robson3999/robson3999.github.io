---
title: 'Dealing with FTP #2'
layout: post
comments: true
keywords: rubyonrails, ruby, ftp
---

W dzisiejszym wpisie przedstawię jak zachować pobierane dane w naszej bazie danych, tak, aby ograniczyć ilość zapytań do zewnętrznego serwera do minimum.  
Mając metodę z poprzedniego wpisu `Media::ListFiles` pobieramy listę nazw wszystkich plików. Teraz zajmiemy się zapisywaniem ich w bazie.  


Ostatnio wygenerowaliśmy model `Medium`, który posłuży nam do przechowywania danych. Stwórzmy zatem serwisy do dodawania i usuwania rekordów.  
```ruby
module Media
  class Create
    def call(params)
      medium = Medium.new(params)
      return 'Failed' unless medium.save

      medium
    end
  end
end
```
```ruby
module Media
  class Destroy
    def call(medium)
      return 'Failed' unless medium.destroy

      medium
    end
  end
end
```
Wykorzystamy je w serwisie służącym do synchronizacji danych aplikacji z plikami na serwerze FTP. Zaczniemy od pobrania listy(`ftp_media`), 
następnie usuniemy z naszej bazy rekordy, które nie pojawiły się w odpowiedzi z serwera (`remove_records_not_present_on_ftp`), a 
na końcu dodamy rekordy, których nie mamy zapisanych lokalnie(`add_records_present_on_ftp`).
```ruby
require 'net/ftp'

module Media
  class SyncWithFTP < ApplicationService
    def call
      ftp_media = ListFiles.new.call

      remove_records_not_present_on_ftp(ftp_media)
      add_records_present_on_ftp(ftp_media)
    end

    private

    def remove_records_not_present_on_ftp(ftp_media)
      Media.all.reject { |m| ftp_media.include?(m.name) }.each do |medium|
        Destroy.new.call(medium: medium)
      end
    end

    def add_records_present_on_ftp(ftp_media)
      Net::FTP.open(host) do |ftp|
        ftp.login(username, password)

        ftp_media.reject { |f| Media.pluck(:name).include?(f) }.each do |medium|
          Create.new.call(media_params: medium_params(medium, ftp))
        end
      rescue Net::FTPPermError => e
        puts e
        failure('Connection refused')
      end
    end

    def medium_params(medium, ftp)
      { name: medium, bytesize: ftp.size(medium),
        mimetype: medium.split('.')[-1] }
    rescue Net::FTPPermError
      { name: medium, mimetype: medium.split('.')[-1] }
    end
  end
end
```
Przyjrzyjmy się jeszcze metodzie `add_records_present_on_ftp`. Otwieramy tutaj połączenie z serwerem, ponieważ potrzebujemy dostępu do informacji o rozmiarach plików, które zapiszemy w bazie. A więc przekazujemy połączenie w zmiennej `ftp` do metody `medium_params`, a tam, dzięki `ftp.size(#NAZWA_PLIKU)` otrzymujemy jego rozmiar w bajtach. Tak "sparsowane" parametry `Medium` przekazujemy do serwisu `Media::Create` utworzonego na początku tego wpisu.  

Po wykonaniu `Media::SyncWithFTP` powinniśmy mieć w bazie danych rekordy odwzorowujące stan plików z serwera FTP.