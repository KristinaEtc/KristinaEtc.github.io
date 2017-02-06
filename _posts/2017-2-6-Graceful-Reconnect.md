---
layout: post
title: Корректное восстановление соединения на GO
---

Не так давно О. рассказывал, что у них на работе в очередной раз хотят уволить сотрудника. "Не справляется," - ответил он на вопрос о причине. И добавил, что чел не смог написать код сохранения в базу с retry policy".

Ввиду моей феноменально "идеальной" памяти делаю заметку.

{% highlight golang %}

    var secToRecon = time.Duration(time.Second * 2) // стартовое время реконнекта
    var numOfRecon = 0 //  номер реконнекта

    for {
        ln, err = net.Listen(network, linkD.address) // пытаемся запустить ожидание сетевого подключения
        if err == nil {
            // функция выполнилаcь корректно - можно выйти из цикла и продолжить исполнение кода
            fmt.Prinln("Listen OK")
            break
        }
        
        // функция вернула ошибку! пытаемся переподключиться
        fmt.Prinln("Listen errorr!", err.Error())
        
        // создаем тикер (таймер, который сигнализирует не один раз, а каждые secToRecon)
        ticker := time.NewTicker(secToRecon)

        select {
        case _ = <-ticker.C:
            {
                /* 
                 время переподключения не меньше установленного лимита - изменяем его:
                 увеличиваем задержку экспоненциально + 20-30% от текущего времени переподключения
                */
                if secToRecon < backOffLimit {
                    //randomAdd := int(r1.Float64()*(float64(secToRecon)*0.1) + 0.2*float64(secToRecon))
                    randomAdd := secToRecon / 100 * (20 + time.Duration(r1.Int31n(10)))
                    fmt.Printfd("Random addition=%d", randomAdd/1000000)
                    secToRecon = secToRecon*2 + time.Duration(randomAdd)
                    fmt.Printfd("secToRecon=%d", secToRecon/1000000)
                    //  увеличиваем количество переподключений
                    numOfRecon++
                }
                //  инициализируем тикер с новым значением задержки
                ticker = time.NewTicker(secToRecon)
                continue
            }
        }
     }
     
{% endhighlight %}

Основные тезисы: время переподключения увеличивается по экспоненциальному закону + случайное число в пределах 10-20% от этого значения (для избежания одномоментной попытки создать подключение - здесь не так очевидно, однако в ряде случаев очень полезно).

<a href="https://github.com/KristinaEtc/bdmq/blob/v1/transport/links.go#L54">Здесь</a> можно посмотреть применение кода в проекте.
