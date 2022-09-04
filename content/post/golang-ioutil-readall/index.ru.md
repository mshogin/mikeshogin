---
title: "Неоптимальность ioutil.ReadAll"
date: 2022-08-22T13:19:43+02:00
description: Рассмотрим почему использовани ioutil.ReadAll может быть не оптимальным и как с этим бороться.
tags: [производительность, профилирование, память]
categories: [ golang, разработка ]
---

Рассмотрим функцию ReadAll из пакета ioutil.
Поиск использования этой функции по github [отображает](https://github.com/search?l=Go&q=ioutil.ReadAll&type=Code&utf8=%E2%9C%93) ее популярность. Я также использую ее достаточно активно. Для небольших файлов
это очень удобный инструмент.

В один прекрасный день, мы получили уведомление от OOM killer о том, что одина из наших задач была
принудительно завершена. Это была типичная задача, с достаточно простой функциональностью:
* взять файл, конвертировать его из одного формата в другой и сохранить
![Task: Convert data](TaskConverter.jpg)

Размер исходного файла был примерно 4.5G.

После непродолжительного расследования, я нашел причину. Самая большое использование памяти было в следующем
участе кода:

```bash
go tool pprof -list problematic_package -alloc_space master.mem.pprof

         .          .    100:	}
         .          .    101:
         .       16GB    102:	data, err := ioutil.ReadAll(reader)
         .          .    103:	if err != nil {
         .          .    104:		return nil, fmt.Errorf("can't read file: %w", err)
         .          .    105:	}
```

Просматривая исходники оказалость что проблема заключается в следующем.
Если используется io.Reader, то нет простого пути получить информацию о размере файла и
соответственно нет возможности предсоздать буфер необходимого размера, для записи результата
чтения файла. Для решения этой проблемы используется стандартный подход:
* буфер создается с минимальным предопределенным размером

```go
// MinRead is the minimum slice size passed to a Read call by
// Buffer.ReadFrom. As long as the Buffer has at least MinRead bytes beyond
// what is required to hold the contents of r, ReadFrom will not grow the
// underlying buffer.
const MinRead = 512
```

Далее MinRead используется для увеличения размера буфера в процессе чтения данных из файла:
```go
	if int64(int(capacity)) == capacity {
		buf.Grow(int(capacity))
	}
```

и под капотом происходит выделение дополнительной памяти
```go
		buf := makeSlice(2*c + n)
		copy(buf, b.buf[b.off:])
		b.buf = buf
```

Это приводит к тому, что выделяется все больше и больше памяти, что в конечном итоге может привести
к ситуации, когда приложение достигнет лимита по потребляемой памяти.

В моем случае, я пофиксил эту проблему следующим образом:

```go
func readFile(r io.Reader, size int64) ([]byte, error) {
	data := make([]byte, size)
	offset := 0
	for {
		n, err := r.Read(data[offset:])
		if n == 0 {
			break
		}
		offset += n
		if err != nil {
			if err != io.EOF {
				return nil, fmt.Errorf("can't read file: %w", err)
			}
			break
		}
	}
	return data, nil
}

func parseJSON(path string) {
    ...
	r, err := os.Open(inPath)
	if err != nil {
		return fmt.Errorf("can't open file %q: %w", path, err)
	}
	defer r.Close()

	fi, err := os.Stat(path)
	if err != nil {
		return fmt.Errorf("can't get file info %q: %w", path, err)
	}

	data, err := readFile(r, fi.Size())
    ...
}
```

После этой оптимизации, профиль памяти стал показывать предсказуемые значения:
```bash
go tool pprof -list problematic_package -alloc_space branch.mem.pprof

         .          .     99:	}
         .          .    100:
    4.32GB     4.32GB    101:	data, err := readFile(r, fi.Size())
         .          .    102:	if err != nil {
```

Результат сравнения профилей до и после оптимизации:
```bash
go tool pprof -top -alloc_space -base master.mem.pprof branch.mem.pprof
File: main
Type: alloc_space
Time: Sep 22, 2020 at 12:30am (CEST)
Showing nodes accounting for -11.92GB, 28.25% of 42.22GB total
Dropped 36 nodes (cum <= 0.21GB)
      flat  flat%   sum%        cum   cum%
     -16GB 37.90% 37.90%      -16GB 37.90%  bytes.makeSlice
    4.32GB 10.24% 27.66%   -11.68GB 27.67%  component.glob..func3
   -0.25GB  0.59% 28.25%    -0.25GB   0.6%  component.CreateCache
         0     0% 28.25%      -16GB 37.90%  bytes.(*Buffer).ReadFrom
         0     0% 28.25%      -16GB 37.90%  bytes.(*Buffer).grow
         0     0% 28.25%   -11.93GB 28.27%  component.ConvertJSON2Binary
         .....
```

Если Вы заметили ошибку или знаете лучшее решение, пожалуйста сообщите мне.
