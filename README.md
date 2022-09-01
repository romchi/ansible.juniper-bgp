# juniper-bpg

Керування BGP. Піринг та магістральні підключення.

## Requirements

Для використання даної ролі, вам потрібно встановити коллекцію `junipernetworks.junos`

```bash
ansible-galaxy collection install junipernetworks.junos
```

### Instalation

```bash
ansible-galaxy role install git+https://github.com/romchi/ansible.juniper-bgp.git,latest,juniper-bgp
```


## Role Variables


### Required

* `origin_name` - ідентифікатор для префіксів, анонсів і комьюніті який буде використовуватись для вашого піру, за замовчуванням *dummy*
* `origin_asn` - номер автономної системи до якої належить цей маршрутизатор, на основі цієї автономної системи будуть будуватись комьюніті, за замовчуванням *12345*
* `prefix_lists.{{ origin_name }}-v4` для керуванням префіксами які будуть анонсуватись по ipv4 потрібно вказати змінну
* `prefix_lists.{{ origin_name }}-v6` для керуванням префіксами які будуть анонсуватись по ipv6 потрібно вказати змінну

### Optional

* `juniper_build` - каталог де будуть створюватись конфіги для завантаження на маршрутизатор, за замовчуванням *\~/.ansible-build/{{ inventory_hostname }}*
* `community_index_group_inet` - цифра комьюніті яка буде використовуватись для групи INET, за замовчуванням *25000*
* `community_index_group_ix` - цифра комьюніті яка буде використовуватись для групи IX, за замовчуванням *25100*
* `community_index_group_peer` - цифра комьюніті яка буде використовуватись для групи PEER, за замовчуванням *25200*
* `community_index_group_customer` - цифра комьюніті яка буде використовуватись для групи CUSTOMER, за замовчуванням *25300*
* `community_index_bh` - цифра комьюніті яка буде використовуватись позначення блекхолу, за замовчуванням *666*
* `community_index_dna` - цифра комьюніті яка буде використовуватись позначення не анонсувати, за замовчуванням *65000*
* `community_index_pre1` - цифра комьюніті яка буде використовуватись позначення препенд 1, за замовчуванням *65001*
* `community_index_pre2` - цифра комьюніті яка буде використовуватись позначення препенд 2, за замовчуванням *65002*
* `community_index_pre3` - цифра комьюніті яка буде використовуватись позначення препенд 3, за замовчуванням *65003*
* `community_index_pre4` - цифра комьюніті яка буде використовуватись позначення препенд 4, за замовчуванням *65004*
* `lp_inet` - локальний преференс для групи INET, за замовчуванням *130*
* `lp_ix` - локальний преференс для групи IX, за замовчуванням *140*
* `lp_peer` - локальний преференс для групи PEER, за замовчуванням *150*
* `lp_customer` - локальний преференс для групи CUSTOMER, за замовчуванням *160*
* `unmanaged_prefixes_v4` - чи потрібно ігнорувати зміни в префікс листі **{{ origin_name }}-v4**, за замовчуванням *false*
* `unmanaged_prefixes_v6` - чи потрібно ігнорувати зміни в префікс листі **{{ origin_name }}-v6**, за замовчуванням *false*

### BGP peers

* `bgp_groups` - список BGP партнерів з якими будуть будуватись сесії, тип змінної список пірів

    * `name` - ім'я піру
    * `peer_as` - автономна система
    * `local_as` - автономна система з нашої сторони, якщо не вказано використовується `{{ origin_asn }}`
    * `type` - тип BGP сесії. Варіанти: `[ internal, external ]`
    * `authentication_key` - опціонально, якщо використовується авторизація для всіх пірів даної групи
    * `export` - перезаписати данні експорту. Варіанти: `[ default, full, drop, origin, customer ]`
        * `default` - анонсувати тільки дефолт
        * `full` - анонсувати всю таблицю маршрутизації
        * `drop` - нічого не анонсувати
        * `origin` - тільки наші мережі
        * `customer` - тільки наші мережі та кастомерів
    * `import` - перезаписати політики імпорту. Варіанти: `[ default, drop ]`
        * `default` - приймати тільки `default route`
        * `drop` - не приймати нічого
    * `group` - група до якої відноситься BGP сесія. Варіанти:
        * `inet` - інтернет, самий дорогий трафік. Самий низький `local preference`, анонсуємо блекхол, кастомерів та наші мережі
        * `ix` - обмінки, дешевий інет. Кращий контент для пошуку інтернет трафіку, `local preference` вище ніж інет, анонсуємо блекхол, кастомерів та наші мережі
        * `peer` - піринги, якщо потрібно ганяти трафік безпосередньо з ASN. `local preference` вище ніж у обмінки, анонсуємо кастомерів та наші мережі
        * `customer` - клієнти яким ми надаємо зв'язність. Самий кращий `local preference`, анонсуємо щось на вибір: всю таблицю маршрутизації, дефолт, нічого
    * `blackhole_community` - комьюніті яке потрібно проставляти на blackhole анонси
    * `neighbors` - список пірів з якими піднімаємо сесію
        * `peer_address` - IP адреса піру
        * `description` - опис
        * `local_address` - локальна адреса з якої будується сесія
        * `authentication_key` - опціонально, якщо використовується авторизація для конкретного піру


## Role created objects

### Defaults

#### communities

Глобальні:
* {{ origin_name }}-origin - комьюніті для позначення префіксів нашого піру
* blackhole-all - комьюніті для блекхолу трафіку до /32 чи /128 IP адреси у всі магістралі

Групові комьюніті:
* from-inet - анонси від пірів магістралей
* from-ix - анонси від пірів обмінок
* from-peer - анонси від пірінгових сесій 
* from-custoner - анонси від наших клієнтів
* dna-all - не анонсувати нікуди
* dna-inet - не анонсувати в магістралі
* dna-peer - не анонсувати в пірінгові сесії
* dna-ix - не анонсувати в обмінки
* prepend-all-[1-4] - добавити препендів 1-4 рази для всіх пірів
* prepend-inet-[1-4] - добавити препендів 1-4 рази для магістральних сесій
* prepend-ix-[1-4] - добавити препендів 1-4 рази для обмінок
* prepend-peer-[1-4] - добавити препендів 1-4 рази для пірінгових сесій

Для кожного піру:
* for-blackhole - список комьюніті для блекхолу у всі до всіх пірів (комьюніті не перетирається, нові значення добавляться)
* unsupported-communities - список комьюніті які приймаємо від клієнтів (комьюніті не перетирається, нові значення добавляться)
* from-{{ peer_name }} - позначаємо анонси від кожного клієнта
* preferred-{{ peer_name }} - покращити локал преференс для анонсів, які мають комьюніті з цього списку
* skip-{{ peer_name }} - погіршити локал преференс для анонсів, які мають комьюніті з цього списку

#### policy-statement
Глобальні:
* accept-all - прийняти все
* drop-all - відкинути все
* drop-default - відфільтрувати дефолтний маршрут
* drop-martians - відфільтрувати мережі мартіанс
* import-default-originate - прийняти дефолтний маршрут
* export-default-originate - проанонсувати дефолтний маршрут
* check-rpki - валідація RPKI
* drop-long-prefixes - відкинути префікси від /25 до /32 для ipv4 та префікси від /65 до /128 для ipv6

Для кожного піру:
* export-{{ peer_name }}-custom - все індивідуальне і не автоматизоване для кожного піру на експорт
* import-{{ peer_name }}-custom - все індивідуальне і не автоматизоване для кожного піру на імпорт

#### as-path-group
Для кожного піру:
* preferred-{{ peer_name }} - покращити локал преференс для анонсів які мають as-path з цього списку
* skip-{{ peer_name }} - погіршити локал преференс для анонсів які мають as-path з цього списку

#### prefix-list
Для піру кастомерів та пірінгів:
* {{ peer_name }}-v4 - список префіксів ipv4 які приймати від піру (список просто створюється, його заповнювати потрібно окремо)
* {{ peer_name }}-v6 - список префіксів ipv6 які приймати від піру (список просто створюється, його заповнювати потрібно окремо)

## Dependencies

There are no dependencies on other roles.

## Example Playbook

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

*inventory file*
```bash
border-1 ansible_host=10.10.1.11      ansible_network_os=junos    ansible_connection=netconf
```

*playbook.yml*
```bash
---                                                                                                                     
- hosts: all

  vars:
    origin_asn: 65000
    origin_name: "my_company_name"
    prefix_lists:
      "{{ origin_name }}-v4":
        - 1.1.1.0/24
        - 5.5.5.0/24
      "{{ origin_name }}-v6":
        - fd12:3456:789a:1::/64
        - fd12:3456:789a:2::/64
    bgp_groups:
      - name: test-inet
        peer_as: "1111"
        local_as: "65000"
        type: "external"
        group: "inet"
        blackhole_community: "1111:60000"
        neighbors:
          - peer_address: "1.1.1.1"
            description: "ipv4"
            local_address: "1.1.1.2"
          - peer_address: "fd12:1111:789a:1::1"
            description: "ipv6"
            local_address: "fd12:1111:789a:1::2"
      - name: test-ix
        peer_as: "2222"
        type: "external"
        group: "ix"
        blackhole_community: "2222:666"
        neighbors:
          - peer_address: "2.2.2.1"
            description: "ipv4"
            local_address: "2.2.2.2"
            authentication_key: "$9$9gglAtOhclWLNEcwYg4UDtuO1IcKvLbYgIE7VYojiO1RcevXx-"
          - peer_address: "fd12:2222:789a:1::1"
            description: "ipv6"
            local_address: "fd12:2222:789a:1::2"
            authentication_key: "$9$9gglAtOhclWLNEcwYg4UDtuO1IcKvLbYgIE7VYojiO1RcevXx-"
      - name: test-customer
        peer_as: "3333"
        local_as: "65000"
        type: "external"
        group: "customer"
        export: "full"
        neighbors:
          - peer_address: "3.3.3.1"
            description: "ipv4"
            local_address: "3.3.3.2"
          - peer_address: "fd12:3333:789a:1::1"
            description: "ipv6"
            local_address: "fd12:3333:789a:1::2"

  roles:
    - juniper-bgp
```

### Pipeline and Tags

Процес налаштування роутеру виглядає наступним чином:
* налаштовуємо тимчасовий каталог для генерації конфігів
* генеруємо конфіг для групи параметрів
* завантажуємо згенерований конфіг на роутер і робимо коміт
* повторюємо 2 пункти вище для наступної групи параметрів

Роль має 2 групи тегів:
- `config` генерація конфігу в тимчасовій директорії
- `upload` завантаження конфігу з тимчасової директорії на роутер

Роль має наступні групи параметрів конфігурування:
* `community` налаштування стандартного набору community
* `default-lp` налаштування local preference для груп
* `prefix-list` створення префікс листів які прописані в змінній `prefix_lists`
* `policy-options` налаштування стандартних policy options
* `import-export` налаштування стандартних політик для імпорту та експорту
* `bpg` налаштування для BGP пірів із специфічними конфігураціями для кожної групи


### Example running

Є два варіанти доступу на маршрутизатори:
- по SSH ключам
- по паролю

Якщо ви використовуєте доступ по паролю на маршрутизатори, вам потрібно добавляти опцію `-k` до плейбуку, в цьому випадку буде запрошено пароль для SSH підключення. Приклад:
```
ansible-playbook -i hosts playbook.yml -k
```

Якщо потрібно згенерувати лише конфігураційні файли, не завантажуючи їх на роутер можна використати тег `config`:
```
ansible-playbook -i hosts playbook.yml --tags config
```

Якщо ми хочемо визвати тільки якусь певну групу параметрів, а не весь пайплайн:
```
ansible-playbook -i hosts playbook.yml --tags prefix-list
```

## License

MIT