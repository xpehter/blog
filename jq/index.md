`jq '.images[0,1,2]' < tmp_do_json | jq -s '.' | jq -c 'sort_by(.size_gigabytes) | .[] | {size_gigabytes}'`

`jq '.images[0,1,2] | .' < tmp_do_json`

Берём всё из файла `tmp_do_json` (его сформировали из вывода `curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer TOKENNN" "https://api.digitalocean.com/v2/images"`) и кидаем на вход jq. Не делаем так `echo tmp_do_json | jq '.images[0,1,2]`, ибо это медленее. По ключу `images` находим массив из словарей, каждый элемент которого, является объектом (словарём), содержащим описание образа в DO. Для примера берём 3 первых элемента. Функция `sort_by` принимает на вход только массивы, потому нужно поместить эти 3 объекта в массив при помощи конструкции `jq -s '.'`, где ключ `s` означает, что вмсто чтения каждого объекта и запуска фильтра для него мы в начале считаем все объекты и запустим фильтр для них всех, будь-то для одного объекта.
`jq -c 'sort_by(.size_gigabytes) | .[] | {size_gigabytes}'` здесь сортируем объекты по ключу `size_gigabytes`, вынимаем их из массива и выводим только значения по ключу `size_gigabytes`

`jq -cr '.images[0,1,2,3] | [.name, .size_gigabytes] | @tsv' < tmp_do_json` вывод нескольких значений  в столбик, соотнесённых друг с другом 
