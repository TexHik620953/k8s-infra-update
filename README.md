# k8s-update

Пошаговое обновление **ОС** на нодах Kubernetes (3 control plane + 5 workers)
с ручным подтверждением каждого шага и показом вывода предыдущего шага.

Версия Kubernetes при этом **не меняется**: `kubelet`, `kubeadm`, `kubectl` и
`kubernetes-cni` остаются на hold, `kubeadm upgrade` не выполняется. Обновляются
только пакеты ОС, затем нода перезагружается.

## Что делает

| # | Плей | Что происходит |
|---|------|----------------|
| 1 | `localhost` | `kubectl version`, все ли ноды `Ready` |
| 2 | все 8 нод | read-only аудит: версии, `apt-mark showhold`, свободное место |
| 3 | workers, `serial: 1` | preflight → drain → dist-upgrade → reboot → verify → autoremove → uncordon |
| 4 | control plane, `serial: 1` | то же самое, плюс проверка `/readyz` после reboot |

На каждом опасном шаге:

1. печатается вывод предыдущей команды;
2. появляется `pause` с вопросом;
3. нужно ввести **`yes`** — любой другой ввод (включая пустой) останавливает
   прогон, и следующие ноды не трогаются.

Пример:

```
TASK [Предыдущий шаг — вывод] **********************************
ok: [worker1] => {
    "msg": [
        "held: kubelet, kubeadm, kubectl",
        [
            "Inst containerd.io [1.7.20-1] (1.7.22-1 Debian:stable [amd64])",
            "Inst linux-image-6.8.0-45-generic [6.8.0-45.46] (6.8.0-47.47 ...)"
        ]
    ]
}

TASK [Подтверждение: apt-get dist-upgrade] *********************
[Подтверждение: apt-get dist-upgrade]
Выполнить «apt-get dist-upgrade» на worker1? Введите yes ...
```

## Требования

- `ansible-core` >= 2.14 на машине, откуда запускаешь (ноутбук).
- `kubectl` на этой же машине + рабочий `~/.kube/config`.
- SSH-доступ на все 8 нод, `sudo` без пароля (или запуск с `-K`).
- На нодах: Ubuntu/Debian. На **каждой** ноде на hold должны быть:
  `kubeadm`, `kubectl`, `kubelet`, `kubernetes-cni`.
  Дополнительные held-пакеты не мешают.
- Свободно >= 5 GiB на `/` каждой ноды.

Проверить связность:

```bash
ansible all -m ping
```

## Настройка

1. `inventory/hosts.yml` — адреса, имена, пользователь.
   Имя хоста в инвентаре должно совпадать с именем ноды в
   `kubectl get nodes`. Если не совпадает — задай `k8s_node_name` на хосте.

2. `group_vars/all.yml`:
   - `ansible_user` — SSH-пользователь;
   - `kubectl_host` — где выполнять `kubectl` (по умолчанию `localhost`);
   - `kubectl_become` — `false` для ноутбука, `true` если `kubectl_host` — CP.

3. Убедись, что на каждой ноде на hold стоят `kubeadm`, `kubectl`, `kubelet`,
   `kubernetes-cni`:

```bash
ansible all -m command -a "apt-mark showhold"
```

Если чего-то не хватает:

```bash
ansible all -m command -a "apt-mark hold kubeadm kubectl kubelet kubernetes-cni" --become
```

Дополнительные held-пакеты не мешают: проверяется только наличие четырёх.

## Запуск

```bash
cd k8s-update

# что будет выполняться
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-tasks --tags nodes
ansible-playbook site.yml --list-hosts

# 0) проверки кластера и аудит нод
ansible-playbook site.yml --tags preflight

# 1) канарейка — один worker
ansible-playbook site.yml --tags nodes --limit worker1

# 2) остальные workers
ansible-playbook site.yml --tags nodes --limit 'k8s_workers:!worker1'

# 3) control plane по одной ноде
ansible-playbook site.yml --tags nodes --limit k8s_control_plane

# всё сразу (для первого раза не рекомендуется)
# ansible-playbook site.yml
```

Если `sudo` просит пароль — добавь `-K`.
Если нет SSH-ключа — `-k`.

Запускай в `tmux`/`screen`: будут долгие паузы и перезагрузки.

## Как это устроено

- `serial: 1` — одновременно обрабатывается ровно одна нода.
- Теги: `preflight` (проверки), `nodes` (обновление нод).
- `max_fail_percentage: 0` + `any_errors_fatal: true` — любая ошибка
  останавливает весь прогон, остальные ноды не трогаются.
- `tasks/_confirm.yml` — общий блок «вывод → пауза → проверка `yes`».
- Все `command`/`shell` регистрируются в переменные и печатаются через
  `debug`, потому что Ansible по умолчанию не показывает их stdout.
- `tasks/00_preflight.yml` не ждёт `apt/dpkg` lock и не трогает таймеры
  автообновлений: если lock занят, apt-задача упадёт с `Could not get lock`.
- `tasks/20_os.yml` перед обновлением показывает симуляцию `apt-get -s dist-upgrade`
  (только строки `Inst`/`Remv`).
- `tasks/30_reboot.yml` использует модуль `reboot` (ждёт возврата по SSH),
  затем ждёт `systemctl is-active kubelet`.
- `tasks/40_verify.yml` проверяет `/lib/modules/$(uname -r)`, `Node Ready`,
  а для CP — `kubectl get --raw=/readyz`.
- `tasks/50_cleanup.yml` делает симуляцию `autoremove`, запрещает удаление
  `linux-image-*` / `linux-modules-*` / `linux-headers-*` текущего ядра
  и только после подтверждения запускает очистку.
- `tasks/60_uncordon.yml` возвращает ноду в работу **последним** шагом.
- `tasks/70_report.yml` вызывается в `always`: пишет `reports/<дата>/<нода>.md`
  и пересобирает `summary.md` по всем уже обработанным нодам.
- `tasks/80_cluster_snapshot.yml` — снимок кластера целиком: `nodes`, `pods -A`,
  PDB, `/readyz`, PVC, DaemonSets, `top`, events. Вызывается из preflight («до»)
  и из `70_report.yml` («после»).

## Отчёты

После каждой ноды на ноутбуке создаются:

```
reports/<YYYY-MM-DD>/<нода>.md
reports/<YYYY-MM-DD>/summary.md
reports/<YYYY-MM-DD>/cluster.md
reports/<YYYY-MM-DD>/cluster-before.md
reports/<YYYY-MM-DD>/cluster-after.md
reports/<YYYY-MM-DD>/.summary/<нода>.tsv
```

В `<нода>.md`:

- `apt-mark showhold` до и после (полный список held-пакетов);
- `uname -r` до и после;
- `ls -la /lib/modules/` до, после reboot и после `autoremove --purge`;
- список установленных ядер и пакетов `linux-image-*` до/после;
- `df -h` и `df -i` до/после;
- системная информация до/после: `uname -a`, `/etc/os-release`, `uptime`,
  `free -h`, `systemctl --failed`, `timedatectl`, версии kubelet/containerd;
- ошибки kubelet после reboot (`journalctl -p err`) в свёрнутом блоке;
- список обновлённых пакетов (`Inst`/`Remv`) в свёрнутом блоке;
- статус: `OK` или `FAILED: <шаг>`.

`cluster.md` — состояние кластера целиком до и после:

- `kubectl get nodes -o wide` (включая kernel version);
- `kubectl get pods -A -o wide`;
- поды не в `Running`/`Completed`;
- таблица «подов на ноду»;
- PDB, `/readyz`, PVC, DaemonSets, `kubectl top nodes`, последние events.

Снимок «до» делается один раз на первой ноде, «после» обновляется после каждой
ноды — в шапке видно `обработано N/8`. `cluster.md` собирается из
`cluster-before.md` и `cluster-after.md`.

`summary.md` — таблица по обработанным нодам: ядро до/после, свободно на `/`, статус,
ссылка на `cluster.md`. Пересобирается после каждой ноды, поэтому остаётся
актуальной, даже если прогон остановился на середине.

Отчёт пишется в `always`, поэтому появляется и при отказе на подтверждении,
и при ошибке на любом шаге. После записи отчёта прогон всё равно останавливается
(кроме случая, когда нода стала unreachable).

## Возобновление после сбоя

Плей остановился на конкретной ноде — остальные не тронуты.

Продолжить с определённой задачи:

```bash
ansible-playbook site.yml --tags nodes --limit worker3 --start-at-task "Drain"
```

Или начать с проверок после reboot:

```bash
ansible-playbook site.yml --tags nodes --limit worker3 --start-at-task "Ждём Node Ready"
```

Если нода осталась `cordoned` — верни вручную:

```bash
kubectl uncordon worker3
```

## Опасные места

1. **`kubectl drain` может упасть** на подах с `hostPath`/local storage:
   `--delete-emptydir-data` их не покрывает. Плей остановится — разберись,
   что за под, и только потом при необходимости добавляй `--force`.
2. **`--delete-emptydir-data` удаляет данные emptyDir.** Это ожидаемо,
   но поды, которые хранят там важное, потеряют данные.
3. **`autoremove --purge`** защищён только от удаления пакетов текущего ядра.
   Пакеты DKMS (nvidia, zfs) и старые headers могут быть удалены — проверяй
   список в подтверждении.
4. **Три CP — это etcd-кворум.** `serial: 1` держит живыми минимум 2 из 3,
   но если API-эндпоинт в `~/.kube/config` — это IP одной CP, то во время
   её reboot `kubectl` с ноутбука будет недоступен. Лучше VIP/haproxy.
5. **Reboot может не вернуться** (плохое ядро, диск, fsck). Поэтому по одной
   ноде и с подтверждением: упавшую ноду чинишь руками, остальные целы.
6. **`apt dist-upgrade` может обновить `containerd`** — он перезапустится,
   но нода уже drained, так что это безопасно.
7. **`kubernetes-cni` тоже на hold.** При будущем апгрейде Kubernetes его
   нужно будет unhold вместе с `kubeadm`/`kubelet`/`kubectl`.

## Файлы

```
k8s-update/
├── ansible.cfg
├── site.yml
├── inventory/hosts.yml
├── group_vars/all.yml
├── templates/
│   ├── cluster-report.md.j2
│   └── node-report.md.j2
├── reports/                # создаётся автоматически
└── tasks/
    ├── _confirm.yml
    ├── 00_preflight.yml
    ├── 10_drain.yml
    ├── 20_os.yml
    ├── 30_reboot.yml
    ├── 40_verify.yml
    ├── 50_cleanup.yml
    ├── 60_uncordon.yml
    ├── 70_report.yml
    └── 80_cluster_snapshot.yml
```
