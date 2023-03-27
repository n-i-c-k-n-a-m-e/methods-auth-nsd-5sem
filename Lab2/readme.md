Сбор и аналитическая обработка информации о сетевом трафике
================
Шабанов Р.В.

## Цель работы

1.  Развить практические навыки использования современного стека
    инструментов сбора и аналитической обработки информации о сетевом
    трафике

2.  Освоить базовые подходы блокировки нежелательного сетевого трафика

3.  Закрепить знания о современных сетевых протоколах прикладного уровня

## Исходные данные

1.  Windows 11

2.  Visual Studio Code

3.  Wireshark

4.  Zeek

## Шаги

1.С использованием Wireshark был получен дамп сетевого трафика на 200
МБ.

2.С помощью Zeek была выделена метаинформация сетевого трафика (файлы
dns.log и http.log)

3.Далее необходимо получить список нежелательных ресурсов:

``` bash
mkdir hosts
wget https://github.com/StevenBlack/hosts/raw/master/data/add.2o7Net/hosts -q -O hosts/hosts.1
wget https://raw.githubusercontent.com/StevenBlack/hosts/master/data/KADhosts/hosts -q -O hosts/hosts.2
wget https://raw.githubusercontent.com/StevenBlack/hosts/master/data/Adguard-cname/hosts -q -O hosts/hosts.3
wget https://raw.githubusercontent.com/StevenBlack/hosts/master/data/adaway.org/hosts -q -O hosts/hosts.4
wget https://winhelp2002.mvps.org/hosts.txt -q -O hosts/hosts.5
sort hosts/hosts* | grep -Eo '^([^\\"'\''#]|\\.|"([^\\"]|\\.)*"|'\''[^'\'']*'\'')*' | uniq > hosts.data
rm -rf hosts
```

В результате был получен файл hosts.data

4.Из метаинформации о трафике получим список доменов:

``` python
hosts = []
with open('dns.log') as file:
    for line in file.readlines():
        if line[0] == '#':
            continue
        try:
            hosts.append(line.split()[9])
        except IndexError:
            continue
```

5.Подготовим полученный список нежелательных ресурсов:

``` python
bad_hosts = []
with open('hosts.data') as file:
    for line in file.readlines():
        try:
            bad_hosts.append(line.split()[1])
        except IndexError:
            continue
```

6.Получим число вхождений DNS в список нежелательного трафика:

``` python
bad_count = len([host for host in hosts if host in bad_hosts])
percentile = round(bad_count/len(hosts),3)*100
print("Число вхождений DNS в список нежелательного трафика: {}.".format(str(bad_count)),
"Процент нежелательного трафика: {}%.".format(str(percentile)),sep='\n')
```

    Число вхождений DNS в список нежелательного трафика: 301.
    Процент нежелательного трафика: 16.6%.

## Оценка результата

В результате практической работы был выявлен нежелательный трафик

## Вывод

В результате проделанной практической работы, с помощью современного
стека инструментов, был проанализован трафик и выявлен нежелательный.
