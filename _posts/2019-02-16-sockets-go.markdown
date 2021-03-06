---
layout: post
title:  "Разбираемся с сокетами на примере стандартной библиотеки Go"
date:   2019-02-17 15:33:30 +0300
categories: technologies
tags: [technologies, sockets, network, go]
image:
  feature: 2019-02-17/sockets.png
  teaser: 2019-02-17/sockets-teaser.png
---

Сокеты являются основой почти любого сетевого приложения, даже если их использование скрыто от ваших глаз. Наша цель — разобраться, что это такое, и написать простое приложение для взаимодействия по сети. Реализуем мы это приложение на языке Go, но принципы использования сокетов одинаковы для любых языков программирования, а библиотеки имеют очень похожий API.

# Модель TCP/IP

Сетевое взаимодействие можно описать с помощью *стека протоколов*, которые его обеспечивают. **Сетевой протокол** — это набор правил, с помощью которых можно соединиться и обмениваться данными с другим устройством сети. Эта задача включает в себя множество подзадач: кодирование данных в физические сигналы, поиск получателя в сети, обеспечение надежной передачи, шифрование трафика и так далее. Поэтому и существует такое понятие как стек протоколов: протоколы разных уровней решают разные задачи. При этом протоколы одного уровня решают поставленную задачу по-разному и является взаимозаменяемыми без влияния на протоколы других уровней.

![tcp/ip]({{ site.baseurl }}/assets/img/2019-02-17/tcp_ip.png)

Существуют разные модели представления сетевого взаимодействия. Одной из самых популярных является модель **TCP/IP**, названная по двум самым популярным протоколам современной глобальной сети. Она включает в себя четыре уровня. На самом низком уровне — *уровне сетевых интерфейсов*, решается задача физического доступа и кодирования информации. *Сетевой* уровень соединяет сети всего мира в глобальную сеть. Главный протокол этого уровня — *IP*. Он определяет адресацию (IP-адреса), благодаря которой устройства могут связываться друг с другом на любых расстояниях. Какие задачи решает *транспортный* уровень мы обсудим в следующем разделе. На *прикладном* же уровне работают большинство сетевых приложений компьютера. Они определяют свои протоколы под конкретные задачи.

# Транспортный уровень

Мы не будем сегодня разбираться в маршрутизации, коммутации и других задачах, решаемых на уровнях ниже транспортного. Можно считать, что на этом уровне мы уже знаем, какой маршрут нужно выбрать, чтобы физически доставить пакет до получателя. Однако, часто бывает недостаточно отправить пакет в сеть и надеяться, что он достигнет цели. Есть множество причин, по которым данные могут потеряться или исказиться в пути. Это могут быть помехи на физическом уровне, превышение максимального размера пакета (*MTU*) на канальном уровне, переполнение буфера коммутатора и много чего еще. Кроме того, пакеты могут прийти в другом порядке по сравнению с тем, как были отправлены.

Чтобы программисту не нужно было контролировать доставку пакетов, их целостность и порядок, эти задачи возложены на транспортный уровень стека сетевых протоколов. Есть два наиболее популярных протокола на этом уровне.

Протокол **UDP** (User Datagram Protocol) противоречит той рекламе, которую я только что дал транспортному уровню. Он является тонкой оберткой над протоколом IP и не обеспечивает никакого контроля над доставкой. Программист берет данные для отправки, упаковывает их в UDP-пакет, размер которого выбирает самостоятельно, и отправляет в сеть. Если пакет не достигнет цели, ни отправитель, ни получатель об этом не узнают. Этот протокол часто применяется в задачах, когда важна скорость доставки, но не столь важно, чтобы все пакеты были доставлены. Например, в онлайн-играх или видео-конференциях.

Протокол, который способен проконтролировать доставку вашего пакета, называется **TCP** (Transmission Control Protocol). В отличие от UDP, перед передачей данных он формирует виртуальное соединение. Для этого используются пакеты специального назначения, которыми обмениваются инициатор соединения (будем считать, что это клиент) и сервер. Такая процедура называется *трехфазным рукопожатием* (3-way handshake).

Другим отличием от UDP является то, что программист при работе с TCP не заботится о размере данных, которые будет отправлять. Этот протокол опрерирует понятием потока и самостоятельно нарезает этот поток на пакеты того размера, который посчитает нужным. Главная же особенность протокола TCP в контроле доставки данных. Он будет отправлять нужные пакеты снова, пока получатель не подтвердит (тоже средствами TCP), что они доставлены и прошли контроль целостности.

# Сокеты

Транспортный уровень обеспечивает доставку пакета до нужного компьютера, но операционной системе нужно понять, какому приложению предназначается этот пакет. Для этой цели в ОС используются **сетевые порты**. Порт — это обычное 16-битное число, которое соответствует процессу и сетевому соединению, которое он использует. Извлекая порт получателя из заголовка TCP-пакета, операционная система определяет, какому процессу нужно передать данные.

Таким образом, чтобы доставить данные из процесса на одном устройстве в процесс на другом, необходимо знать IP-адрес получателя и порт, используемый этим процессом. Пара этих параметров образует понятие **сокета**. В то же время, сокетом называют ресурс операционной системы, который она выделяет, когда процесс запрашивает порт для сетевого взаимодействия. По факту, это два взгляда на одно и то же. Рассмотрим стандартную процедуру, когда клиентское приложение хочет связаться с сервером.

Допустим, клиент каким-то образом знает, какой порт нужно указать в качестве получателя в пакете. Чтобы принимать пакеты, сервер должен быть готов к этому. Это возможно, засчет *открытых* портов. Они постоянно “слушают” входящие соединения, которых может быть больше одного, в отличие от обычных (*закрытых*) портов. Как правило, службы, к которым хотят получить доступ процессы на других устройствах, имеют общеизвестные открытые порты, что решает проблему узнавания, к какому порту обращаться клиенту. Клиентские же процессы обычно могут брать произвольный незанятый порт для взаимодействия по сети.

Далее мы реализуем клиент-серверное взаимодействие с помощью сокетов на языке Go. Как уже было сказано, работа с сокетами представлена очень похожим API в разных языках программирования. Поэтому мы постараемся выделить основные моменты, не заостряясь на языковых особенностях.

# TCP

Начнем с более популярного протокола TCP. Наша задача — реализовать простейший, не очень общительный сервер, который здоровается с клиентом, узнает имя собеседника и прощается с ним. Классическая реализация TCP-сервера:

1. Создать слушающий сокет с открытым портом.
2. В бесконечном цикле принимать соединения от клиентов.
3. При соединении с клиентом создается новый сокет для взаимодействия только с этим клиентом.
4. Для каждого соединения обрабатывать клиентские запросы в отдельном потоке.

Код сервера:

{% highlight golang %}
package main

import (
   "fmt"
   "net"
)

func main() {
   listener, _ := net.Listen("tcp", "localhost:8080") // открываем слушающий сокет
   for {
      conn, err := listener.Accept() // принимаем TCP-соединение от клиента и создаем новый сокет
      if err != nil {
         continue
      }
      go handleClient(conn) // обрабатываем запросы клиента в отдельной го-рутине
   }
}

func handleClient(conn net.Conn) {
   defer conn.Close() // закрываем сокет при выходе из функции

   buf := make([]byte, 32) // буфер для чтения клиентских данных
   for {
      conn.Write([]byte("Hello, what's your name?\n")) // пишем в сокет

      readLen, err := conn.Read(buf) // читаем из сокета
      if err != nil {
         fmt.Println(err)
         break
      }

      conn.Write(append([]byte("Goodbye, "), buf[:readLen]...)) // пишем в сокет
   }
}
{% endhighlight %}

Все, что требуется от клиента, это подключиться к серверу по нужному порту и обмениваться с ним данными через сокет. Код клиента:

{% highlight golang %}
package main

import (
   "fmt"
   "io"
   "log"
   "net"
   "os"
)

func main() {
   if len(os.Args) != 2 {
      fmt.Fprintf(os.Stderr, "Usage: %s host:port ", os.Args[0])
      os.Exit(1)
   }
   serv := os.Args[1] // берем адрес сервера из аргументов командной строки
   conn, _ := net.Dial("tcp", serv) // открываем TCP-соединение к серверу
   go copyTo(os.Stdout, conn) // читаем из сокета в stdout
   copyTo(conn, os.Stdin) // пишем в сокет из stdin
}

func copyTo(dst io.Writer, src io.Reader) {
   if _, err := io.Copy(dst, src); err != nil {
      log.Fatal(err)
   }
}
{% endhighlight %}

Так выглядит взаимодействие клиента с сервером:

![client tcp]({{ site.baseurl }}/assets/img/2019-02-17/client_tcp.png)

# UDP

Взаимодействие по UDP отличается тем, что в этом протоколе нет понятия соединения. Поэтому мы реализуем совсем простой вариант, в котором сервер отвечает клиенту тем же текстом, что получил от него. При этом клиентский код отличается только именем протокала, который передается в функцию net.Dial. Код сервера:

{% highlight golang %}
package main

import (
   "fmt"
   "net"
)

func main() {
   listener, _ := net.ListenUDP("udp", &net.UDPAddr{IP: net.ParseIP("localhost"), Port: 8080 }) // открываем слушающий UDP-сокет
   for {
      handleClient(listener) // обрабатываем запрос клиента
   }
}

func handleClient(conn *net.UDPConn) {
   buf := make([]byte, 128) // буфер для чтения клиентских данных
   
   readLen, addr, err := conn.ReadFromUDP(buf) // читаем из сокета
   if err != nil {
      fmt.Println(err)
      return
   }

   conn.WriteToUDP(append([]byte("Hello, you said: "), buf[:readLen]...), addr) // пишем в сокет
}
{% endhighlight %}

![client udp]({{ site.baseurl }}/assets/img/2019-02-17/client_udp.png)

Компьютерные сети являются обширной темой, о которой нужно иметь представление любому бэкенд-разработчику. Сегодня мы рассмотрели сокеты, с помощью которых программист может обеспечить взаимодействовие с другими устройствами сети на транспортном уровне.