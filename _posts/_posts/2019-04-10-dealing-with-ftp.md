---
layout: post
title: "Dealing with FTP #1"
date: 2019-04-20T20:09:41+01:00
draft: false
keywords: "rubyonrails, ruby, ftp"
---

W tym cyklu przedstawię schemat prostej aplikacji do obsługi zdalnego serwera FTP. Będzie ona pozwalała nam na:
- pobieranie listy zapisanych na serwerze plików, (index)
- usuwanie wybranego pliku z serwera, (delete)
- pobieranie wybranego pliku na lokalny komputer, (read)
- wysyłanie lokalnego pliku na serwer. (create)

W pierwszej części zajmiemy się listowaniem plików.

Jak widać jest to prosty CRUDowy schemat, jednak będziemy poruszać się na dwóch płaszczyznach. Pierwszej - bazie danych, w której będziemy zapisywać dane n.t. rekordów na serwerze (tak, aby nie odpytywać serwera przy każdym odświeżeniu strony), oraz druga - pliki na FTP.

Rozpocznijmy utworzeniem kontrolera i modelu o nazwie Medium.
```ruby
rails g model Medium name:string bytesize:integer mimetype:string
rails db:migrate
```
Modelowi `Medium` nadajemy następujące pola:
- name - nazwa,
- bytesize - rozmiar,
- mimetype - typ.

Oraz tworzymy kontroler z jedną akcją `index`
```ruby
rails g controller media index
```

W credentialach aplikacji umieśćmy dane potrzebne do logowania do serwera FTP:
```
$ EDITOR=nano rails credentials:edit
```
```
ftp_access:
  ftp_host: 'ftp.xyz.com'
  ftp_username: 'admin'
  ftp_password: 'test123'
```
By rozpocząć pracę z modelem `Medium`, potrzebujemy informacji n.t. plików z serwera. Do komunikacji z nim wykorzystamy wbudowaną bibliotekę 'net/ftp'. Funkcjonalności podzielimy na osobne serwisy w folderze `/services/media`, tak, aby zachować porządek w kodzie.

## Listowanie plików

Najpierw stwórzmy serwis, który przechowuje metody do pobierania credentiali i będzie nam służył za "bazę" dla reszty serwisów.  
```ruby
module Media
  class ApplicationService
    protected
    def host
      Rails.application.credentials.ftp_access[:ftp_host]
    end
    
    def user
      Rails.application.credentials.ftp_access[:ftp_user]
    end
    
    def password
      Rails.application.credentials.ftp_access[:ftp_password]
    end
  end
end
```
Taki podział przyda nam się, by nie powtarzać kodu w pozostałych serwisach. Dzięki dziedziczeniu z ApplicationService, możemy korzystać z jego metod w dziedziczących serwisach. Teraz możemy pobrać listę plików.
```ruby
require 'net/ftp'

module Media
  class ListFiles < ApplicationService
    def call
      Net::FTP.open(host) do |ftp|
        ftp.login(username, password)
        files = ftp.nlst
        files.reject { |f_name| ['.', '..'].include?(f_name) }
      end
    rescue Net::FTPPermError
      puts 'Permission denied'
    end
  end
end
```
W pierwszych liniach metody `call` otwieramy połączenie z serwerem i wszystkie operacje wykonujemy w bloku. Warto zwrócić na to uwagę, ponieważ dzięki takiemu sposobowi łączenia się z serwerem, w razie wystąpienia jakiegoś wyjątku lub błędu, połączenie automatycznie się zamyka (dzięki czemu nie musimy pamiętać, by zawsze na końcu pisać ftp_conn.close()).  Pobieramy listę plików dzięki funkcji `Net::FTP#nlst`. Linijka następująca po niej, to czyszczenie listy z dwóch elementów: `.` oraz `..`. Są to niepotrzebne nam pozycje, które służą do nawigacji po serwerze.  
Poprawność kodu możemy sprawdzić odpalając konsolę i wykonując skrypt.
```ruby
$ rails c
> Media::ListFiles.new.call
```
Jeśli wszystko poszło dobrze powinniśmy otrzymać tablicę z nazwami plików, które są na serwerze. Jeżeli nic na nim nie ma, to tablica będzie pusta ;) W celach testowych możesz wrzucić jakieś pliki np. programem Filezilla.