---
title: 'Dealing with FTP #4'
layout: post
tags: ruby, rubyonrails, ftp, files
---

W naszej mini-aplikacji brakuje jeszcze funkcjonalności usuwania plików. Tym razem stworzymy dwa serwisy:
- usuwający plik z serwera,
- wywołujący serwis nr.1 i po pomyślnym jego wykonaniu, usuwający rekord z bazy danych.

Chcemy mieć pewność, że usuniemy dane z bazy danych dopiero wtedy, kiedy pliku już nie będzie na serwerze. Nie chcielibyśmy dopuścić do sytuacji, kiedy na przykład tracimy połączenie z internetem, plik zostaje, a rekord w bazie danych został usunięty. Co prawda, mamy [możliwość synchronizacji](/2019/dealing-with-ftp-2/) w takim przypadku, jednak nie chcemy mnożyć czynności, które użytkownik musi wykonywać w aplikacji.

Rozpocznijmy więc od usuwania pliku. Tak jak przy uploadzie, do usunięcia potrzebujemy jednej, prostej metody: `delete`.
```ruby
require 'net/ftp'

module Media
  class DeleteFromFTP < ApplicationService
    def call(medium_name)
      Net::FTP.open(host) do |ftp|
        ftp.login(username, password)
        ftp.delete(medium_name)
      end
    rescue StandardError
      'failed'
    end
  end
end
```
Mając do dyspozycji taki serwis, usunięcie pliku może wyglądać w ten sposób:
```ruby
medium = Medium.find_by(name: 'raport_out_of_date.xls')
Media::DeleteFromFTP.new.call(medium.name)
```
Jeżeli wykonanie powyższych linijek kodu nie zwróciło błędu, plik powininen zostać usunięty z serwera FTP.

Opakujmy zatem powstały serwis w jeszcze jeden, który będzie usuwał rekord z bazy danych. W pierwszej kolejności będzie wywoływał serwis `DeleteFromFTP`, a następnie, jeżeli nie pojawią się żadne błędy, usunie powiązany z plikiem rekord.
```ruby
module Media
  class Destroy
    def call(medium)
      service = DeleteFromFTP.call(medium.name)
      return 'Failed' if service == 'failed'

      medium.destroy
    end
  end
end
```
W tym serwisie nie potrzebujemy modułu `net/ftp`, ani danych logowania więc pomijamy `require 'net/ftp` oraz `ApplicationService`. Dzięki temu, że wykonując `Media::Destroy#call` jesteśmy 'wewnątrz' modułu `Media`, wezwanie serwisu `DeleteFromFTP` może zostać wykonane bez opisywania całej ścieżki wiodącej do niego (tzn. możemy ominąć `Media::`). Zgodnie z założeniem serwis zwraca komunikat "Failed" jeżeli usunięcie pliku z serwera się nie powiedzie oraz przerywa działanie.



#### Koniec z FTP
To by było na tyle jeśli chodzi o serię 'Dealing with FTP'. Kod zamieszczony tutaj jest mocno uproszczony i napisany w nienajlepszym stylu, ale założeniem było przedstawienie podstawowych narzędzi jakie oferuje język Ruby do obsługi plików przez protokół FTP, a nie budowanie pełnoprawnej aplikacji webowej. Dzięki za uwagę :)