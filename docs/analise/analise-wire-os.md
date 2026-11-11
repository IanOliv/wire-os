# Auditoria técnica do wire-os (WireOS / Anki Vector)

- Base analisada: `wire-os` @ `a6e4c3a88`, submódulo `anki/victor` (wire-os-victor) @ `505eb84c`, `anki/wired`, `anki/vic-cloudless`, `external/purplpkg`.
- Não analisado: `poky/openembedded-core`, `poky/meta-openembedded`, `poky/bitbake` (submódulos não baixados; onde o padrão upstream importa, está marcado como HIPÓTESE).
- Nada foi compilado nem executado no robô. Toda validação é roteiro para rodar via SSH.
- Convenção: **FATO** = visto no código, com `arquivo:linha`. **HIPÓTESE** = inferência a confirmar com dado do robô.
- Caminhos são relativos à raiz do repositório. `victor/` abrevia `anki/victor/`. `defconfig` abrevia `poky/victor/meta-qcom/recipes-kernel/linux-msm/linux-msm-4.9/apq8009-robot/defconfig`.

## Sumário

1. [Hardware e build confirmados](#0-hardware-e-build-confirmados)
2. [Respostas diretas aos incidentes](#1-respostas-diretas-aos-incidentes)
3. [Comece por aqui](#2-comece-por-aqui)
4. [Achados priorizados](#3-achados-priorizados)
5. [Extensões para a arquitetura do dono](#4-extensões-de-funcionalidade)
6. [Hipóteses abertas e roteiro de coleta](#5-hipóteses-abertas-e-roteiro-de-coleta)
7. [Pontos de contato com o wire-os-victor](#6-pontos-de-contato-com-o-wire-os-victor)

---

## 0. Hardware e build confirmados

| Item | Valor | Fonte (FATO) |
|---|---|---|
| Máquina Yocto | `apq8009-robot`, DTB `qcom/apq8009-robot-anki-vector` | `poky/victor/meta-qcom/conf/machine/apq8009-robot.conf:11` |
| Modelo no DT | "Anki Vector (APQ8009 Robot)" | `kernel/msm-4.9/arch/arm64/boot/dts/qcom/apq8009-robot-anki-vector.dts:5` |
| Kernel | msm-4.9, `CONFIG_NR_CPUS=4`, `CONFIG_PREEMPT=y`, `HZ=100`, `NO_HZ_IDLE` | `defconfig:479,485,494,86` |
| RAM visível | `mem=511M` na linha de comando | `apq8009-robot.conf:20` |
| Tabela de clock da CPU | 200 / 400 / 533 / 729,6 / 800 / 998,4 / 1094,4 / 1190,4 / 1248 / 1267,2 MHz | `apq8009-robot-anki-vector.dts:30-42` |
| Userspace | Yocto "whinlatter", systemd 257, connman 1.36, wpa-supplicant 2.6 | `poky/build/conf/local.conf` (`PREFERRED_VERSION_*`), `.gitmodules` |
| rootfs | ext4 somente leitura, `noatime` | `poky/victor/meta-qcom/recipes-core/base-files/base-files-3.0.14/fstab:4` |
| `/data` | LUKS (dm-crypt) + ext4, `defaults,exec,nosuid,nodev,noatime` | `poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/initscripts/mount-data:37` |
| UART do body board | `/dev/ttyHS0` (dono: usuário `robot`) e `/dev/spidev0.0` | `poky/victor/meta-anki/recipes/anki-robot/files/vic-robot.service:12` |
| Wifi | WCNSS/Pronto (`CONFIG_WCNSS_CORE_PRONTO=y`) | `defconfig:3684` |

Config de kernel efetiva: **HIPÓTESE forte** de que é o `defconfig` da receita (`linux-msm_4.9.bb:13` faz `SRC_URI += "file://defconfig"`, `linux-msm.inc:48` usa `KCONFIG_MODE = "--alldefconfig"`), e não o `msm8909_defconfig` da árvore do kernel, que tem mais opções de debug. Confirmar no robô: `zcat /proc/config.gz | grep -E 'CPU_THERMAL|MEMCG|WATCHDOG|ZRAM'`.

Tipos de build (`build/build.sh:165`): `dev`, `oskr`, `devcloudless`, `prod`. Em `dev` e `oskr` o victor é compilado em `Release` sem `ANKI_RESOURCE_SHIPPING` (`victor/project/victor/scripts/victor_build_release.sh`, `victor_build_oskr.sh`), o que instala a configuração **`development`** de `victor/platform/config/` (`victor/platform/config/CMakeLists.txt:95-99`).

Processos e units confirmados na imagem (`poky/victor/meta-anki/recipes-products/images/apq8009/apq8009-anki-robot-image.inc`): `vic-robot`, `vic-anim`, `vic-engine`, `vic-cloud` (ou `vic-cloudless`), `vic-switchboard`, `vic-gateway-cert`, `vic-init`, `vic-bootanim`, `wired`, `rebooter`, `fault-code`, `anki-crash-log`, `animfail`. O `vic-gateway` roda **dentro do `vic-cloud`** (`victor/cloud/cloud/ipc_manager.go:41-43`).

**Não estão no `IMAGE_INSTALL`**: `vic-webserver`, `vic-dasmgr`, `vic-crashuploader`, `vic-log-uploader`. As receitas existem (`poky/victor/meta-anki/recipes/anki-robot/vic-*.bb`), mas não achei quem as puxe. HIPÓTESE: esses quatro não rodam no WireOS. Confirmar: `systemctl list-units 'vic-*' --all`.

---

## 1. Respostas diretas aos incidentes

### 1.1 Quem reinicia o vic-engine, qual o limite, o que acontece quando esgota

**FATO. O systemd não reinicia nada.** Todas as units `vic-*` têm `Restart=no` (`poky/victor/meta-anki/recipes/anki-robot/files/vic-engine.service:24`, idem `vic-anim.service:12`, `vic-robot.service:16`, `vic-cloud.service:19`, `vic-switchboard.service:17`).

**FATO. O supervisor é um script bash: `anki/fault-code/fault-code-handler`**, ativado por socket FIFO (`poky/victor/meta-anki/recipes/fault-code/files/fault-code.socket` → `ListenFIFO=/run/fault_code`). O fluxo:

1. Uma falha escreve um código em `/run/fault_code`. Duas origens para o engine:
   - **914** (`NO_ENGINE_PROCESS`): o `ExecStopPost=/anki/bin/vic-on-exit` escreve `VIC_ONFAILURE_FAULTCODE` quando o serviço termina com resultado diferente de `success` (`victor/platform/config/bin/vic-on-exit:36-64`, `victor/platform/config/etc/vic-engine.env:3`).
   - **915** (`NO_ENGINE_COMMS`): o `vic-anim` escreve quando fica **5 s** sem conexão com o engine (`victor/animProcess/src/cozmoAnim/animProcessMessages.cpp:882,913`). Ou seja, **um engine travado por mais de 5 s também vira "vic-engine crashed"**. Nesse caso o handler manda `kill -ILL` no engine (`fault-code-handler:144-148`).
2. O handler manda `SIGTERM` em **todos** os serviços de `anki-robot.target`, espera até 3 s e manda `SIGQUIT` nos que sobrarem (`fault-code-handler:168-204,242-245`).
3. Mata com `kill -9` e reinicia `mm-anki-camera` e `mm-qcamera-daemon` (`fault-code-handler:348-354`).
4. Se pode reiniciar: dorme `TIMEOUT_RESTART_SEC` e roda `systemctl restart anki-robot.target` (`fault-code-handler:370-388`).

**FATO. O limite:** `FAULT_RESTART_LIMIT=2` e `FAULT_RESTART_LIMIT_SEC=60` (`fault-code-handler:25-26`). O contador fica em `/run/fault_restart_count` e o instante do último reinício em `/run/fault_restart_uptime`. O contador zera se passaram 60 s ou mais desde o último reinício (`fault-code-handler:262-270`). Só esgota quando acontecem **três falhas encadeadas, cada uma a menos de 60 s do reinício anterior** (`fault-code-handler:274-277`). Na configuração `development`, `TIMEOUT_RESTART_SEC` não é definido e vale o padrão de 5 s (`fault-code-handler:23`; `victor/platform/config/development/etc/fault-code-handler.env`).

**FATO. Quando esgota:** `ON_FAULT_RESTART=0`, a tela mostra o texto fixo " Restarts exhausted." (`victor/platform/faultCodeDisplay/faultCodeDisplay.cpp:74`), os `vic-*` ficam parados e **nada mais tenta recuperar**. O contador vive em `/run` (tmpfs), então só um reboot limpa.

**FATO. O reboot de manutenção também fica bloqueado.** O `rebooter` só reinicia se o governor da CPU for `powersave` ou `userspace` (`anki/rebooter/rebooter.sh:56-61`). Quem põe `userspace` é o engine em modo de economia (`victor/osState/osState_vicos.cpp:125-126`, `victor/engine/components/powerStateManager.cpp:52-54,195,202`). Com o engine morto em `interactive`, o inibidor nunca some e o script desiste às 05:00 (`rebooter.sh:44-46,96-99`).

Consequência para o incidente: 08:03:24 foi o começo de uma cadeia de três falhas em menos de ~2 min. A pergunta que resta é **por que o engine falhou de novo logo depois de reiniciar** (ver 5.1).

### 1.2 Existe watchdog?

**FATO. Não existe nenhum, em nenhuma camada.**

| Camada | Estado | Fonte |
|---|---|---|
| Watchdog de hardware do SoC | desligado na linha de comando: `watchdog_v2.enable=0` | `apq8009-robot.conf:20` |
| `/dev/watchdog` | não existe: `# CONFIG_WATCHDOG is not set` | `defconfig:2272` |
| Detector de lockup do kernel | `# CONFIG_LOCKUP_DETECTOR is not set`, `# CONFIG_WQ_WATCHDOG is not set` | `defconfig:4331,4340` |
| systemd `RuntimeWatchdogSec` | não configurado; `system.conf` só tem `DefaultMemoryAccounting=yes` | `poky/victor/meta-vicos/recipes-core/systemd/systemd-conf/system.conf:1-2` |
| `WatchdogSec=` / `sd_notify` nas units | nenhuma ocorrência no repositório | busca global |
| Aplicação | só o anim → engine (5 s) descrito acima | `animProcessMessages.cpp:882` |

Agravante: `CONFIG_QCOM_DLOAD_MODE=y` (`defconfig:2052`) com `download_mode = 1` por padrão (`kernel/msm-4.9/drivers/power/reset/msm-poweroff.c:67`). HIPÓTESE: um kernel panic deixa o SoC em modo de download (esperando ramdump por USB) em vez de reiniciar, apesar de `CONFIG_PANIC_TIMEOUT=5` (`defconfig:4343`). Isso combina com "sumiu da rede e não voltou".

### 1.3 Para onde vão core dumps e logs, e sobrevivem ao reboot?

| Artefato | Destino | Sobrevive? | Fonte |
|---|---|---|---|
| Minidump Breakpad dos `vic-*` | `/data/data/com.anki.victor/cache/crashDumps/*.dmp` | **Sim** | `victor/platform/victorCrashReports/google_breakpad.cpp:81` (opção `USE_GOOGLE_BREAKPAD` ligada por padrão, `CMakeLists.txt:8`) |
| Últimas 500 linhas de log por crash | `<dump>.log`, ao lado do `.dmp` | **Sim** | `anki/anki-crash-log/anki-crash-log:11-12,44-45` |
| Fault report (`messages.log`) | `/data/fault-reports/fault-<código>-<data>/`, só se existir `/run/das_allow_upload` | Sim, condicional | `fault-code-handler:37,296-302,321-325` |
| Core dump do kernel | `/var/tmp/core.%e.%p.%s.%t` → tmpfs | **Não**, e consome RAM | `poky/victor/meta-qcom/recipes-core/initscripts/initscripts-1.0/set_core_pattern.sh:3`; `fstab:10` |
| journald | `/run/log/journal` (tmpfs), até 64 MB | **Não** | `systemd-conf/journald.conf:2-3` |
| syslog | `/var/log/messages` (tmpfs) | **Não** | `poky/victor/meta-vicos-mods/recipes-support/syslog-ng/files/syslog-ng.conf.systemd` |
| dmesg do boot | `/data/boot.log`, sobrescrito a cada boot | Só o boot atual | `poky/victor/meta-vicos/recipes-extended/update-engine/files/boot-successful.sh` (última linha) |
| Panic do kernel | nada grava em `/data/panics`; pstore comentado no fstab | **Não** | `fstab:12`; `CONFIG_PSTORE_RAM=m`, `# CONFIG_PSTORE_CONSOLE is not set` (`defconfig:4205,4202`) |

O `vic-crashuploader` está desativado no próprio script: a primeira instrução é `exit 0` (`victor/platform/config/bin/vic-crashuploader:15`). Com isso também não roda a limpeza `KEEP_LATEST`, então os `.dmp` se acumulam em `/data`.

**Ação imediata:** os minidumps das duas noites provavelmente estão no robô. Copie antes de qualquer coisa (roteiro em 5.0).

### 1.4 As quedas de hora em hora no minuto :03

**FATO.** O servidor web embutido (civetweb 1.9.1) é iniciado com `"websocket_timeout_ms", "3600000"  // 1 hour` (`victor/webServerProcess/src/webService.cpp:979-980`). No civetweb esse valor é o tempo máximo de espera por **dados vindos do cliente** no laço de leitura do websocket (`victor/lib/util/source/3rd/civetweb/src/civetweb.c:9467-9468`, usado nas chamadas `pull(..., timeout)`). Um cliente webviz que só assina módulos e depois só recebe fica 1 h sem enviar nada e é desconectado pelo servidor.

**HIPÓTESE forte.** As quedas de 05:02, 06:03, 07:03 e 08:03 são esse timeout. Todos os clientes do vector-bridge conectam juntos, expiram juntos 1 h depois e reconectam; o atraso de reconexão explica o deslize de 05:02 para 06:03. O mesmo valor vale para 8888 (engine) e 8889 (anim), porque os dois usam `webService.cpp`.

Correção sem tocar no robô: o vector-bridge enviar um ping de websocket (ou qualquer frame) a cada 30–60 s. Teste: com o ping ligado, as quedas horárias somem.

Outros candidatos horários, avaliados e descartados como causa direta:

| Candidato | O que faz | Por que não explica o :03 fixo |
|---|---|---|
| `update-engine.timer` | `OnCalendar=*-*-* *:00:00` (`update-engine.timer:9`) | O serviço dorme `RANDOM % 3600` s antes de agir (`update-engine.service`, `ExecStartPre` do sleep), então o instante real varia a cada hora |
| `fake-hwclock-tick.timer` | a cada 15 min grava a hora em `/data` (`external/fake-hwclock/fake-hwclock-tick.timer:5-6`) | período de 15 min, não de 1 h |
| `rebooter.timer` | diário às 01:00 (`anki/rebooter/rebooter.timer:9`) | diário |
| cron / logrotate horário | não achei cron; `logrotate` está instalado mas a configuração vem do upstream não baixado | não confirmado |

### 1.5 A queda das 01:38

**FATO.** O `rebooter` dispara às 01:00 e reinicia o robô em um instante aleatório entre 01:00 e 05:00, exigindo 4 h de uptime e engine em economia de energia (`anki/rebooter/rebooter.sh:11-13,74-101`). Ele grava `1` em `/data/maintenance_reboot` antes de reiniciar (`rebooter.sh:32`).

**HIPÓTESE.** 01:38 foi o reboot de manutenção, e não um crash. Confirmar: `ls -la --time-style=full-iso /data/maintenance_reboot` e comparar o uptime.

Atenção ao fuso: o padrão é `America/Chicago` quando `/data/etc/localtime` não existe (`mount-data:188-191`). Se o fuso não foi ajustado, a janela "01:00–05:00" cai em outro horário local.

### 1.6 "Seguiu na rede recusando conexões por ~2 h e depois sumiu"

**FATO.** Depois do esgotamento, os `vic-*` estão parados, então 8888, 8889 e 443 recusam conexão. O Linux, o wifi, o `sshd` e o `wired` (8080) continuam de pé.

**FATO.** Sem frames do `vic-robot`, o microcontrolador do corpo entra em `POWER_CALM` após 200 frames perdidos (~1 s) (`victor/robot/syscon/src/comms.cpp:72,196-199`). Ele não desliga a cabeça por isso.

**Não achei** no código do SO nada que desligue o robô após N horas nesse estado. O sumiço fica como hipótese aberta (5.2): bateria, panic sem watchdog, ou wifi.

### 1.7 Temperatura na tela

**FATO.** O `vic-anim` desenha a temperatura em vermelho quando a zona térmica passa de **90 °C** (`victor/animProcess/src/cozmoAnim/animation/animationStreamer.cpp:195,1247-1254`), lendo `/sys/devices/virtual/thermal/thermal_zone3/temp` (`victor/osState/osState_vicos.cpp:108`).

**FATO. O kernel não tem como reduzir o clock por temperatura.** As zonas `cpu0-2-step` e `cpu1-3-step` têm trip passivo em 85 °C e mapeiam as CPUs como cooling device (`kernel/msm-4.9/arch/arm64/boot/dts/qcom/msm8909-thermal.dtsi:178-240`), mas `# CONFIG_CPU_THERMAL is not set` (`defconfig:2248`). Sem essa opção, `of_cpufreq_cooling_register()` é um stub que devolve `-ENOSYS` (`kernel/msm-4.9/include/linux/cpu_cooling.h`, ramo `#else`), e o driver apenas registra "running cpufreq for CPU%d without cooling dev" (`kernel/msm-4.9/drivers/cpufreq/qcom-cpufreq.c:346-352`). Também não há `thermal-engine` da Qualcomm na imagem (não achei receita).

**FATO. Três agentes disputam o governor, sem coordenação:**

| Quem | Quando | O que escreve | Fonte |
|---|---|---|---|
| Kernel | boot | governor padrão `performance` | `defconfig:588` |
| `init_post_boot` | boot | `interactive`, `scaling_min_freq=400000`, parâmetros do interactive | `system/core/rootdir/etc/init.qcom.post_boot.sh:590-612` |
| `wired` | boot + 3 s, e a cada mudança na UI web (porta 8080) | **preset 0 (padrão, `FreqPreset = 0`)**: máx 533 MHz; preset 1: máx 729,6 MHz, `interactive`; **preset 2**: máx 1267,2 MHz, **`performance`** | `anki/wired/mods/freq.go:13-14,30-43,84-98,101,108` |
| `vic-engine` | início e trocas de modo | normal: `interactive`; economia: `userspace` a 200 ou 400 MHz | `osState_vicos.cpp:125-126,314-353`; `powerStateManager.cpp:52-54` |

**HIPÓTESE.** A temperatura aparece quando o preset 2 do `wired` está ativo, ou quando há carga contínua, porque não existe throttling abaixo dos 90 °C. Verificar o preset: `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_{governor,max_freq}`.

### 1.8 Webviz aceita ~4 clientes e trava com o mapa de navegação

**FATO. Quatro clientes.** `"num_threads", "4"` (`webService.cpp:968-969`). No civetweb 1.9.1 cada conexão ocupa uma thread de trabalho enquanto durar, e um websocket dura indefinidamente (`civetweb.c:13628`, `ctx->worker_connections + thread_args->index`). Quatro websockets abertos ocupam as quatro threads; o quinto cliente e **qualquer pedido HTTP** ficam na fila.

**FATO. O mapa é reenviado inteiro, para todos, a cada pedido.** Qualquer mensagem de qualquer cliente para o módulo `navmap` marca `_webMessageDirty = true` (`victor/engine/navMap/mapComponent.cpp:311-313`), e o envio seguinte vai para **todos** os inscritos (`webService.cpp:1567-1581`).

**FATO. Fila sem limite e escrita bloqueante.** Cada envio copia o `Json::Value` por valor para uma lambda numa fila única (`"WebsocketSender"`, `webService.cpp:1034,1664`), serializa e chama `mg_websocket_write` bloqueante (`webService.cpp:1665-1669`). Não há limite de fila, descarte nem timeout de envio. Um cliente lento segura a fila; os JSONs copiados se acumulam na memória do `vic-engine`.

**HIPÓTESE.** N clientes × M pedidos de mapa de ~146 KB produzem N×M cópias + serializações. Com cliente lento isso faz o RSS do engine crescer até matar por falta de memória, ou atrasa o engine o suficiente para o anim declarar 915 (5 s).

**Não achei** nenhum `sysctl` de rede no repositório: valem os padrões do kernel 4.9 para ~511 MB. Não vejo limite de socket ou buffer do kernel como causa; o gargalo é a aplicação.

---

## 2. Comece por aqui

Cinco itens, na ordem. Os três primeiros não exigem gerar imagem.

1. **Coletar o que já está no robô** (10 min). Rodar o roteiro 5.0 antes do próximo reboot. Os minidumps em `/data/data/com.anki.victor/cache/crashDumps` dizem por que o engine morreu.
2. **Ping de websocket no vector-bridge** (30 min, só no lado do dono). Um frame a cada 30–60 s elimina a queda horária (achado A3) e a reconexão em rajada às :03.
3. **Um cliente webviz por porta e um pedido de `navmap` por vez** no vector-bridge (1 h). Tira o robô do limite de 4 threads e da fila sem limite (A3).
4. **Recuperação automática quando os reinícios esgotam** (2 h, receita `fault-code`). Trocar "desistir" por "reboot depois de N minutos" e tirar o inibidor do `rebooter` quando o engine está morto (A1). Sozinho, isso faz o robô amanhecer funcionando.
5. **Watchdog de hardware + `CONFIG_CPU_THERMAL`** (meio dia, exige imagem nova e teste em bancada). Cobre travamento de kernel e superaquecimento (A2, A4).

---

## 3. Achados priorizados

Escala: impacto / esforço / risco em B (baixo), M (médio), A (alto).

| ID | Achado | Impacto | Esforço | Risco |
|---|---|---|---|---|
| A1 | Supervisão desiste para sempre após 2 reinícios; rebooter fica inibido | A | B | B |
| A2 | Nenhum watchdog; panic pode parar em download mode | A | M | M |
| A3 | Webviz: timeout de 1 h, 4 threads, fila sem limite, mapa em broadcast | A | B (bridge) / M (victor) | B |
| A4 | Sem throttling térmico; três donos do governor | A | M | M |
| A5 | RAM consumida por tmpfs (journal, syslog, cores); sem zram, sem memcg, sem prioridade de OOM | A | M | B |
| A6 | Diagnóstico pós-morte incompleto: logs voláteis, sem pstore, dumps sem limpeza | M | B | B |
| A7 | Acesso root exposto por chaves públicas no repositório; rsync e wired sem autenticação | A | M | B |
| A8 | `/data` é reformatado em qualquer falha de montagem | M | B | M |
| A9 | OTA sem verificação de assinatura; boot marcado como bom sem checar saúde | M | M | M |
| A10 | Tarefas periódicas desnecessárias para o modo noturno | B | B | B |
| A11 | Kernel: opções de debug e drivers sem uso; sem overlayfs | B | M | M |
| A12 | Boot com esperas fixas e cadeia serial | B | B | B |
| A13 | Build frágil: AUTOREV, rede no compile, checksums desligados | M | M | B |
| A14 | Higiene de units: sem limites, sem prioridade, câmera sempre ligada | M | B | B |

### A1. Supervisão desiste para sempre

- **Local:** `anki/fault-code/fault-code-handler:23-26,262-277,370-388`; `anki/rebooter/rebooter.sh:52-66`; `poky/victor/meta-anki/recipes/anki-robot/files/vic-*.service` (`Restart=no`).
- **Atual (FATO):** descrito em 1.1. Limite de 2 reinícios; a janela de 60 s é medida desde o último reinício; pausa de 5 s antes de reiniciar na config `development`.
- **Problema:**
  - Uma causa transitória que dure mais de ~2 min (memória cheia, câmera presa) derruba o robô até intervenção manual.
  - 5 s de pausa não dão tempo de a causa passar.
  - O handler reinicia o alvo inteiro mesmo quando só um processo caiu.
  - O `rebooter` não age com o engine morto (governor fora de `powersave`/`userspace`).
- **Mudança sugerida:**
  1. Em `fault-code-handler`, no ramo "No more restarts allowed" (linha 275), agendar a recuperação em vez de desistir:
     ```bash
     if [ ${FAULT_RESTART_COUNT} -ge ${FAULT_RESTART_LIMIT} ]; then
       log "No more restarts allowed; scheduling recovery reboot"
       ON_FAULT_RESTART=0
       # Guarda contra boot-loop: no máximo 3 reboots de recuperação por dia, contados em /data
       n=$(cat /data/fault_recovery_count 2>/dev/null || echo 0)
       if [ "$n" -lt 3 ]; then
         echo $((n+1)) > /data/fault_recovery_count
         systemd-run --on-active=${FAULT_RECOVERY_REBOOT_SEC:-600} --unit=fault-recovery-reboot /sbin/reboot
       fi
     fi
     ```
     Zerar `/data/fault_recovery_count` no `boot-successful` quando o uptime passar de 1 h com os `vic-*` ativos.
  2. Backoff: `TIMEOUT_RESTART_SEC` de 5 → 30 e `FAULT_RESTART_LIMIT` de 2 → 4, via `/anki/etc/fault-code-handler.env` (muda no wire-os-victor, ver seção 6).
  3. Em `rebooter.sh`, tratar "engine inativo" como condição que **libera** o reboot:
     ```bash
     systemctl is-active --quiet vic-engine || { echo "${list[@]}"; return; }
     ```
     antes do teste do governor (linha 56).
- **Validar no robô:**
  ```bash
  # simula três falhas encadeadas
  for i in 1 2 3; do systemctl kill -s SEGV vic-engine; sleep 25; done
  journalctl -t fault-code-handler -n 40 --no-pager
  cat /run/fault_restart_count /run/fault_restart_uptime
  systemctl list-timers | grep fault-recovery     # depois da mudança: deve existir
  ```
  Métrica: tempo entre a terceira falha e o robô voltar a responder em 443, sem intervenção.

### A2. Nenhum watchdog

- **Local:** `poky/victor/meta-qcom/conf/machine/apq8009-robot.conf:20`; `defconfig:2052,2272,4331,4340,4341,4343`; `poky/victor/meta-vicos/recipes-core/systemd/systemd-conf/system.conf`; `kernel/msm-4.9/drivers/soc/qcom/watchdog_v2.c:101-105`; `kernel/msm-4.9/drivers/power/reset/msm-poweroff.c:67`.
- **Atual (FATO):** tabela em 1.2.
- **Problema:** travamento de kernel, de driver (câmera, wifi) ou do PID 1 deixa o robô morto até alguém tirar da base. Um panic pode parar em download mode.
- **Mudança sugerida, em três degraus:**
  1. Kernel: remover `watchdog_v2.enable=0` de `KERNEL_CMD_PARAMS`. O `qcom,wdt@b017000` existe no DT e o driver alimenta o watchdog por uma thread do kernel, então cobre travamento de kernel sem depender de userspace. **Risco:** não sei por que foi desligado (pode ter sido por bite durante o boot ou o `rampost`). Testar em bancada com serial.
  2. Panic → reboot: adicionar no boot `echo 0 > /sys/module/msm_poweroff/parameters/download_mode`, ou desligar `CONFIG_QCOM_DLOAD_MODE`. Manter `CONFIG_PANIC_TIMEOUT=5`. Considerar `CONFIG_PANIC_ON_OOPS=y`.
  3. Saúde de aplicação: um `vic-health.service` (esboço em 4.2) que verifica os `vic-*` e reinicia o robô se ficarem inativos por X minutos. Isso substitui o que `RuntimeWatchdogSec` faria se existisse `/dev/watchdog`.
- **Validar no robô:**
  ```bash
  cat /proc/cmdline | tr ' ' '\n' | grep -i watchdog
  cat /sys/module/msm_poweroff/parameters/download_mode
  ls /dev/watchdog* 2>/dev/null
  dmesg | grep -i -E 'wdog|watchdog'
  # teste destrutivo, em bancada: deve reiniciar sozinho em menos de 30 s
  echo c > /proc/sysrq-trigger
  ```

### A3. Webviz (8888/8889)

- **Local:** `victor/webServerProcess/src/webService.cpp:964-985,1034,1567-1581,1660-1670`; `victor/engine/navMap/mapComponent.cpp:311-313,758-801`; `victor/lib/util/source/3rd/civetweb/src/civetweb.c:9467-9468,13628`.
- **Atual (FATO):** descrito em 1.4 e 1.8. Sem autenticação (as linhas de `global_auth_file` estão comentadas, `webService.cpp:974-984`).
- **Problema:** queda de todos os clientes a cada hora; teto de 4 conexões somando websocket e HTTP; memória e latência do engine dependem do cliente mais lento.
- **Mudança sugerida:**
  - Lado do dono, agora: ping a cada 30–60 s; uma conexão por porta, multiplexando no bridge; nunca pedir `navmap` em paralelo; cache do mapa no Redis.
  - No wire-os-victor (seção 6):
    ```cpp
    "num_threads", "8",
    "websocket_timeout_ms", "86400000",
    "enable_keep_alive", "yes",
    ```
    e, em `SendToWebSocket`, descartar o envio quando a fila do `WebsocketSender` passar de um limite (por exemplo 8 itens por conexão), serializando o JSON **uma vez** por broadcast em vez de uma vez por cliente.
  - Estrutural: parar de usar o webviz como canal de produção (4.3).
- **Validar no robô:**
  ```bash
  ss -tnp | grep -E ':(8888|8889)'                 # quantas conexões por porta
  P=$(pidof vic-engine); grep -E 'VmRSS|Threads' /proc/$P/status
  # repetir durante uma rajada de pedidos de navmap e ver o RSS subir
  while :; do date +%T; grep VmRSS /proc/$P/status; sleep 2; done
  ```
  Métrica: RSS do engine antes, durante e depois da rajada; número de quedas por hora no bridge.

### A4. Térmico e governor

- **Local:** `defconfig:588,2248`; `msm8909-thermal.dtsi:178-240`; `qcom-cpufreq.c:346-352`; `anki/wired/mods/freq.go:84-110`; `system/core/rootdir/etc/init.qcom.post_boot.sh:590-612`; `osState_vicos.cpp:108,125-126`; `animationStreamer.cpp:195`.
- **Atual (FATO):** descrito em 1.7.
- **Problema:** não há mitigação entre "normal" e 90 °C. O preset 2 do `wired` fixa `performance` a 1267 MHz. O alerta usa `thermal_zone3` por número, e a numeração depende da ordem de registro.
- **Mudança sugerida:**
  1. Fragmento de kernel:
     ```
     CONFIG_CPU_THERMAL=y
     ```
     Com isso os trips de 85 °C do DT passam a reduzir o clock. Se quiser começar antes, baixar os trips `cpu0_2_step_trip` e `cpu1_3_step_trip` para 75000 no `apq8009-robot-anki-vector.dts`.
  2. No `wired`, trocar `performance` por `interactive` no preset 2 (`freq.go:97`), mantendo o teto de 1267 MHz. O `performance` mantém as quatro CPUs no clock máximo mesmo paradas.
  3. Decidir um único dono do governor. Sugestão: o `wired` só escreve `scaling_max_freq`, e o engine continua escolhendo `interactive`/`userspace`.
  4. No victor: localizar a zona por `type` (`cpu0-2-usr`) em vez do índice 3 (seção 6).
- **Validar no robô:**
  ```bash
  for z in /sys/class/thermal/thermal_zone*; do echo "$(basename $z) $(cat $z/type) $(cat $z/temp)"; done
  ls /sys/class/thermal/ | grep cooling_device      # hoje: esperado nenhum de CPU
  dmesg | grep -i 'without cooling dev'
  cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_{governor,cur_freq,min_freq,max_freq}
  cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
  cat /data/wired/* 2>/dev/null | head            # preset salvo do wired (caminho a confirmar em anki/wired/vars)
  ```
  Métrica: temperatura máxima em 30 min de atividade e em 30 min na base, antes e depois.

Ponto a verificar junto: a linha de comando tem `lpm_levels.sleep_disabled=1` (`apq8009-robot.conf:20`), mas o driver compilado é o legado (`CONFIG_MSM_PM_LEGACY=y`, `defconfig:3664`; `kernel/msm-4.9/drivers/cpuidle/Makefile:31`), cujo parâmetro o post-boot escreve em `/sys/module/lpm_levels_legacy/parameters/sleep_disabled` (`init.qcom.post_boot.sh`, ramo robot). HIPÓTESE: os estados de baixo consumo estão habilitados depois do boot. Confirmar: `cat /sys/module/lpm_levels*/parameters/sleep_disabled` (esperado `N`).

### A5. Memória: tmpfs, swap, OOM

- **Local:** `fstab:9-11`; `systemd-conf/journald.conf:1-3`; `syslog-ng.conf.systemd` (destino `/var/log/messages`); `set_core_pattern.sh:3`; `defconfig:142,103,1281,3501`; `system.conf:2`.
- **Atual (FATO):**
  - `/run`, `/var/volatile` e `/dev/shm` são tmpfs **sem `size=`**, então cada um pode crescer até 50 % da RAM.
  - journald guarda até 64 MB em `/run` e ainda encaminha tudo ao syslog-ng (`ForwardToSyslog=yes`), que grava de novo em `/var/log/messages`. O mesmo log ocupa RAM duas vezes.
  - Cores do kernel vão para `/var/tmp` (tmpfs).
  - `CONFIG_ZRAM=y` existe no kernel, mas nada o configura: o único código que faz `swapon` exige `ro.config.zram=true` (`init.qcom.post_boot.sh:104-108`), que não é definido em lugar nenhum.
  - `# CONFIG_MEMCG is not set`: `MemoryMax=`/`MemoryHigh=` não funcionam e `DefaultMemoryAccounting=yes` não tem efeito.
  - `CONFIG_ANDROID_LOW_MEMORY_KILLER=y`, e nenhuma unit define `OOMScoreAdjust=`: todos os processos têm a mesma prioridade de morte.
- **Problema:** em ~511 MB, log e core em tmpfs competem com o `vic-engine`. HIPÓTESE: a rotação de `/var/log/messages` depende de uma configuração de `logrotate` que não está neste repositório; se não existir, o arquivo cresce até encher o tmpfs. HIPÓTESE: um core de centenas de MB do engine em `/var/tmp` é suficiente para fazer o reinício seguinte falhar por memória, o que produz exatamente a cadeia "três falhas em menos de 60 s".
- **Mudança sugerida:**
  ```
  # journald.conf
  [Journal]
  Storage=volatile
  RuntimeMaxUse=8M
  ForwardToSyslog=yes
  RateLimitIntervalSec=30s
  RateLimitBurst=2000
  ```
  ```
  # fstab
  tmpfs /run          tmpfs mode=0755,nodev,nosuid,noexec,strictatime,size=48m 0 0
  tmpfs /var/volatile tmpfs defaults,nodev,nosuid,noexec,size=48m           0 0
  tmpfs /dev/shm      tmpfs defaults,nodev,nosuid,noexec,size=32m           0 0
  ```
  ```
  # /etc/logrotate.d/messages
  /var/log/messages { size 8M  rotate 1  copytruncate  missingok  notifempty }
  ```
  ```
  # core: não gravar em RAM. Os minidumps do Breakpad já cobrem o diagnóstico.
  sysctl -w kernel.core_pattern='|/bin/false'
  ```
  ```
  # zram.service (novo), 128 MB com lz4
  [Service]
  Type=oneshot
  RemainAfterExit=yes
  ExecStart=/bin/sh -c 'echo lz4 > /sys/block/zram0/comp_algorithm; echo 134217728 > /sys/block/zram0/disksize; mkswap /dev/zram0; swapon -p 100 /dev/zram0; sysctl -w vm.swappiness=100'
  ```
  ```
  # prioridade de morte (drop-ins)
  vic-robot.service   OOMScoreAdjust=-900
  vic-anim.service    OOMScoreAdjust=-600
  vic-engine.service  OOMScoreAdjust=-300
  wired.service, avahi, rsyncd  OOMScoreAdjust=500
  ```
  O ganho do zram precisa ser medido: comprimir custa CPU num Cortex-A7.
- **Validar no robô:**
  ```bash
  free -m; cat /proc/meminfo | head -8
  df -h /run /var/volatile /dev/shm
  ls -la /var/log/ /var/tmp/ | head -30
  journalctl --disk-usage
  ls /etc/logrotate.d/; systemctl list-timers --all | grep -i logrotate
  cat /proc/sys/kernel/core_pattern; grep -i core /proc/$(pidof vic-engine)/limits
  dmesg | grep -i -E 'lowmemorykiller|out of memory|killed process'
  cat /proc/swaps
  ps -eo rss,comm --sort=-rss | head -15
  ```
  Métrica: `MemAvailable` às 20:00, 02:00 e 08:00; tamanho de `/var/log/messages` nas mesmas horas.

### A6. Diagnóstico pós-morte

- **Local:** `victor/platform/config/bin/vic-crashuploader:15`; `anki/anki-crash-log/anki-crash-log:11-12`; `fault-code-handler:296-302`; `fstab:12`; `defconfig:4198-4205`; `boot-successful.sh`.
- **Atual (FATO):** tabela em 1.3.
- **Problema:** depois de um reboot ou de um panic não sobra journal nem dmesg. O fault report com log só é gravado quando existe `/run/das_allow_upload`, que depende do fluxo DAS. Os `.dmp` nunca são limpos.
- **Mudança sugerida:**
  1. No `fault-code-handler`, capturar sempre, sem depender do DAS, e limitar a quantidade:
     ```bash
     d=/data/fault-reports/fault-${FAULT_CODE}-$(date +%Y%m%d-%H%M%S); mkdir -p "$d"
     tail -n 2000 /var/log/messages > "$d/messages.log"
     dmesg > "$d/dmesg.log"
     cat /proc/meminfo > "$d/meminfo"; ps -eo pid,rss,comm --sort=-rss | head -30 > "$d/ps"
     df -h > "$d/df"; cat /sys/class/thermal/thermal_zone*/temp > "$d/temps"
     ls -1dt /data/fault-reports/fault-* | tail -n +21 | xargs -r rm -rf   # mantém 20
     ```
  2. Limpeza dos minidumps: manter os 30 mais recentes (um `ExecStartPre` no `vic-init`).
  3. pstore/ramoops: `CONFIG_PSTORE_RAM=y`, `CONFIG_PSTORE_CONSOLE=y`, um nó `ramoops` em `reserved-memory` (1 MB) e descomentar `fstab:12`. Depois, um serviço de boot copia `/sys/fs/pstore/*` para `/data/panics/`, que o `vic-init.sh` já sabe processar (`victor/platform/config/bin/vic-init.sh`, bloco `PANICS_DIR`). Risco médio: mexe no mapa de memória.
  4. `boot-successful.sh`: gravar `/data/boot-$(date +%s).log` e manter os 5 últimos, em vez de sobrescrever.
- **Validar no robô:** provocar `systemctl kill -s SEGV vic-engine`, depois `ls -la /data/fault-reports/ /data/data/com.anki.victor/cache/crashDumps/`.

### A7. Segurança básica

- **Local e fatos (só caminhos, sem conteúdo):**
  - Chave **privada** de root SSH versionada: `poky/victor/meta-qcom/recipes-connectivity/openssh/files/ssh_root_key`. A pública correspondente é instalada como `/etc/ssh/authorized_keys` (`openssh_%.bbappend:42`), com `PermitRootLogin yes` (`openssh_%.bbappend:22`). Cópias da privada: `anki/vic-cloudless/ssh_root_key`, `external/purplpkg/go/ssh_root_key`, `victor/robot_sshkey`.
  - Chaves privadas de host SSH versionadas no mesmo diretório (`ssh_host_*_key`).
  - Chaves de assinatura no repositório: `ota/ota_prod.key`, `ota/ota_test.key` (com `ota/ota_test.pass`), `ota/qtipri.encrypted.key`, `ota/qtioskrpri.encrypted.key`, `ota/qticozmopri.encrypted.key`, `ota/vble-qti.key`, `victor/robot/syscon/tools/victorSysConSigningKeyPvt.pem`.
  - Token de upload embutido em `victor/platform/config/development/etc/vic-crashuploader.env`.
  - Firewall dos builds `dev`/`oskr` (`iptables-eng.rules`, escolhido por `iptables-persistent.bb:14`; em `oskr` o nome resultante é `engoskr`) abre: 22, 443, 8080, 5555 (adb), 1873 (rsync), 5103, 8887–8890, 24024–24026 (`iptables-eng.rules:35-56`).
  - `rsyncd`: módulo `anki_root` em `/anki`, `read only = false`, `uid = root`, sem autenticação (`poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/rsync/rsyncd-victor.conf`). A unit é instalada mas não é habilitada no boot (`extra-conf.bb:37-38`).
  - `wired` escuta em `:8080` sem autenticação (`anki/wired/main.go:38`) e executa comandos como root (`anki/wired/mods/freq.go:113-114`). A porta 8080 está aberta em **todas** as variantes de firewall, inclusive `iptables-user.rules`.
  - Webviz sem autenticação, com variáveis e funções de console (`-DREMOTE_CONSOLE_ENABLED=1` em `victor_build_release.sh`).
- **Problema:** qualquer máquina na mesma rede com o repositório público tem root no robô. Para um robô com câmera e microfone dentro de casa, isso é o achado de segurança principal.
- **Mudança sugerida:**
  1. Gerar um par de chaves próprio e instalar só a pública (`/data/ssh/authorized_keys`, já que `SYSCONFDIR=/data/ssh`, `machine-robot-image.bbappend`, hook `read_only_robot_rootfs_hook`). Remover a pública padrão do build do dono.
  2. Gerar as chaves de host no primeiro boot (o `sshdgenkeys.service` já existe na receita).
  3. Restringir por origem no firewall:
     ```
     -A INPUT -p tcp -s <IP-do-vector-bridge> -m multiport --dports 22,443,8080,8888,8889 -j ACCEPT
     ```
     e remover 5555, 1873, 5103 e 24024–24026 do conjunto padrão.
  4. `wired`: escutar só em `127.0.0.1` por padrão, ou exigir token.
- **Validar no robô:** `ss -tlnp`; `iptables -S INPUT`; de outra máquina, `nmap -p 22,443,1873,5555,8080,8887-8890 <ip>`.

### A8. `/data` é reformatado em qualquer falha de montagem

- **Local:** `poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/initscripts/mount-data:165-177`.
- **Atual (FATO):** se `mount_userdata` falhar por qualquer motivo (LUKS não reconhecido, falha do `user-data-locker`, erro do ext4), o script chama `make_and_mount_cryptdata`, que faz `luksFormat` + `mkfs.ext4`. Não há `fsck` antes do `mount`. O log da formatação vai para `/data/format-data.log`.
- **Problema:** um corte de energia que deixe o ext4 inconsistente, ou uma falha transitória do QSEE no boot, apaga pareamento, wifi, fotos e todos os dumps.
- **Mudança sugerida:** entre `luksOpen` e `mount`, rodar `e2fsck -p /dev/mapper/userdata`; se o `mount` falhar, tentar de novo após 2 s; só formatar quando `cryptsetup isLuks` disser que não é LUKS. Adicionar `errors=remount-ro,commit=30` às opções.
- **Validar no robô:** `cat /data/format-data.log 2>/dev/null` (se existir, já houve formatação); `tune2fs -l /dev/mapper/userdata | grep -E 'Mount count|state|Last checked'`.

### A9. OTA e A/B

- **Local:** `victor/platform/update-engine/update-engine.cpp:215-256,313-320,504-548,581,741,771-780`; `ota/Makefile:78,90-91`; `poky/victor/meta-vicos/recipes-extended/update-engine/files/boot-successful.{service,sh}`; `poky/victor/meta-vicos/recipes-extended/update-engine/update-engine.bb:37`.
- **Atual (FATO):**
  - O cliente OTA é um binário C++ que baixa com `curl` e grava direto no slot inativo em streaming (bom para RAM), depois chama `bootctl-anki <slot> set_active`.
  - O `manifest.sha256` (assinatura) é **lido e descartado** (`update-engine.cpp:581-590,741`). Não achei nenhuma chamada de verificação de assinatura no arquivo. A proteção restante é a cifra AES com senha (`openssl enc -d -aes-256-ctr`, `update-engine.cpp:677,726`).
  - A URL automática inclui o número de série: `emresn=...&ankiversion=...` (`update-engine.cpp:219`).
  - Auto-update é inibido por `/etc/do-not-auto-update`, criado em todo build sem `AUTO_UPDATE=1` (`update-engine.bb:37`), ou por `/data/data/user-do-not-auto-update` (`update-engine.cpp:315`), que o `wired` alterna (`anki/wired/mods/auto-update.go:47-51`).
  - `if (url == "auto") system("/sbin/reboot")` (`update-engine.cpp:778-780`) nunca é verdadeiro, porque `url` já foi substituída pela URL construída (linha 493). O reboot pós-OTA depende do `rebooter`.
  - `boot-successful.service` marca o slot como bom assim que o `ankibluetoothd` sobe (`boot-successful.service`: `After=`/`Requires=ankibluetoothd.service`), sem olhar os `vic-*`.
- **Problema:** uma imagem que dá boot mas cujo `vic-engine` não sobe é marcada como boa, e o A/B perde a função de rollback. Sem verificação de assinatura, a integridade depende só do TLS e da senha.
- **Mudança sugerida:**
  1. `boot-successful`: `After=anki-robot.target` e condicionar a `systemctl is-active vic-robot vic-anim vic-engine` estáveis por 60 s.
  2. Verificar `manifest.sha256` com `ota/ota_prod.pub` embarcada (`openssl dgst -sha256 -verify`) antes de gravar; verificar o `sha256` de cada imagem contra o manifest.
  3. Corrigir o teste do reboot guardando um `bool is_auto` antes de reescrever `url`.
- **Validar no robô:** `bootctl-anki a status; bootctl-anki b status`; `tr ' ' '\n' < /proc/cmdline | grep slot_suffix`; `ls /etc/do-not-auto-update /data/data/user-do-not-auto-update`.

### A10. Tarefas periódicas

- **Local:** `update-engine.timer:9`; `external/fake-hwclock/fake-hwclock-tick.timer:5-6`; `anki/rebooter/rebooter.timer:9`.
- **Atual (FATO):** o timer do update-engine acorda de hora em hora mesmo com auto-update inibido (o serviço sobe, dorme até 1 h e sai com código 200). O `fake-hwclock` grava em `/data` a cada 15 min (96 escritas por dia). A imagem tem `chrony` (`apq8009-anki-robot-image.inc`) e o connman tem NTP desligado por patch (`1002-Disable-NTP-by-default.patch`).
- **Problema:** pequeno em custo, mas contraria a meta "zero atividade à noite". A escrita do `fake-hwclock` é desgaste pequeno e constante da eMMC.
- **Mudança sugerida:** `ConditionPathExists=!/etc/do-not-auto-update` no `update-engine.timer`; `fake-hwclock-tick` de 15 min → 1 h e gravação no shutdown.
- **Validar no robô:** `systemctl list-timers --all`; `cat /sys/block/mmcblk0/stat` em dois instantes separados por 1 h na base (setores escritos = campo 7).

### A11. Kernel: sobras

- **Local:** `defconfig` (linhas entre parênteses).
- **Atual (FATO):**
  - Ligados sem uso aparente no robô: `CONFIG_IPA=y`, `CONFIG_RMNET_IPA=y` (3524-3525, dados de modem); `CONFIG_AUDIT=y` (55); `CONFIG_SECURITY_SELINUX=y` (4520); `CONFIG_BT=y` (1125, enquanto o nó `bluetooth` do DT está `disabled`, `apq8009-robot-anki-vector.dts:45-47`); `CONFIG_DEBUG_FS=y` (4297); `CONFIG_DYNAMIC_DEBUG=y` (4279); `CONFIG_FTRACE=y` com tracers padrão (4413-4416); `CONFIG_KALLSYMS_ALL=y` (184); `CONFIG_MAGIC_SYSRQ=y` (4303).
  - `CONFIG_DEFAULT_IOSCHED="cfq"` (339).
  - `# CONFIG_OVERLAY_FS is not set` (4138).
  - `# CONFIG_PSI is not set` (103).
  - Console serial ativo em produção: `console=ttyMSM0,115200n8 loglevel=4` (`apq8009-robot.conf:20`).
  - `CONFIG_ANDROID_PARANOID_NETWORK=y` (759): só processos no grupo 3003 abrem socket (`victor.bb:56-58`). Um serviço novo de telemetria precisa de `SupplementaryGroups=3003`.
- **Atenção:** o `wired` depende de debugfs (`/sys/kernel/debug/msm-bus-dbg`, `rpm_send_msg`, `freq.go:102-109`). Desligar `DEBUG_FS` quebra a troca de frequência de RAM.
- **Mudança sugerida:** `CONFIG_OVERLAY_FS=y` (habilita 4.6); `deadline` como escalonador de I/O (pode ser em runtime: `echo deadline > /sys/block/mmcblk0/queue/scheduler`); desligar `IPA`/`RMNET_IPA` e medir. O resto tem ganho pequeno e risco de regressão: deixar para depois.
- **Validar no robô:** `zcat /proc/config.gz | grep -E 'OVERLAY_FS|IOSCHED|IPA'`; `cat /sys/block/mmcblk0/queue/scheduler`; `grep -E 'Slab|KernelStack|PageTables' /proc/meminfo`.

### A12. Boot

- **Local:** `anki/wired/wired.service` (`ExecStartPre=/usr/bin/sleep 3`); `extra-conf/initscripts/logd:45-47` (laços com `sleep 1`); `extra-conf/initscripts/enable-wifi:16` (até 15 × 1 s); `extra-conf/initscripts/ankiinit:46`; `vic-engine.service:21` (`sleep 0.1`); `multi-user-done.service`; `start-anki.service`.
- **Atual (FATO):** cadeia `multi-user.target` → `multi-user-done` (espera `mount-data`, câmera, `leprop`, `btproperty`, `ankibluetoothd`, `qtid`, `qti_system_daemon`) → `start-anki` → `anki-robot.target` → `vic-init` → `vic-robot` → `vic-anim` → `vic-engine` → `vic-cloud`/`vic-switchboard` → `vic-gateway-cert`. Os `vic-*` são ordenados por `After=` em série.
- **Problema:** o `logd` espera os sockets em passos de 1 s; o `vic-robot` espera Bluetooth e daemons Qualcomm de que não depende.
- **Mudança sugerida:** trocar `sleep 1` por `sleep 0.05` nos laços do `logd`; remover `ankibluetoothd`, `qtid` e `qti_system_daemon` do `After=` de `multi-user-done` (manter em `Wants=`); `vic-switchboard` já declara a dependência de `ankibluetoothd` sozinho.
- **Validar no robô:** `systemd-analyze`; `systemd-analyze blame | head -25`; `systemd-analyze critical-chain anki-robot.target vic-engine.service`.

### A13. Build

- **Local e fatos:**
  - `SRCREV = "${AUTOREV}"` com `inherit externalsrc` (`victor.bb:8,13`) e `do_compile[network] = "1"` (`victor.bb:109`): o victor baixa dependências durante o compile, fora do controle do bitbake.
  - `BB_STRICT_CHECKSUM:forcevariable = "0"` e `wget --no-check-certificate` (`poky/build/conf/local.conf`): downloads sem verificação de integridade nem de TLS.
  - `do_rootfs[nostamp] = "1"` e `do_build[nostamp] = "1"` (`poky/victor/meta-vicos-mods/recipes-products/images/machine-robot-image.bbappend`): o rootfs é refeito em todo build.
  - `wire-clean` apaga `sstate-cache`, `tmp-glibc` e `downloads` com `sudo rm -rf` quando detecta um layout antigo (`poky/build/conf/set_bb_env.sh:225-232`).
  - Docker: `--build-arg GID=$(id -u $USER)` usa o UID como GID (`build/build.sh:245`).
  - Submódulos `openembedded-core`, `meta-openembedded` e `bitbake` seguem branch (`.gitmodules`), fixados só pelo commit do superprojeto.
  - systemd 257 sobre kernel 4.9: a base mínima documentada do systemd 257 é mais nova que 4.9. HIPÓTESE: funciona com recursos ausentes (cgroup v2 parcial, sem memcg). Vale registrar o que o systemd reclama no boot: `journalctl -b -p warning -u init.scope`.
  - Não achei CI (`.github/workflows` não existe).
- **Mudança sugerida:** espelho local de `downloads` e `sstate`; religar checksums; fixar a revisão do victor num arquivo versionado; corrigir o `GID`.

### A14. Higiene das units

- **Local:** `poky/victor/meta-anki/recipes/anki-robot/files/*.service`; `extra-conf/services/mm-anki-camera.service`.
- **Atual (FATO):**
  - Só `vic-robot` tem `Nice=-20`, `CPUAffinity=1 2 3` e `TimeoutStopSec=5` (`vic-robot.service:22,25,28`). `vic-anim` tem `Nice=-2`. Os demais não têm limite algum (`LimitNOFILE`, `OOMScoreAdjust`, `TimeoutStopSec`).
  - `mm-anki-camera` e `mm-qcamera-daemon` têm `Restart=always` e sobem no boot, com ou sem visão em uso.
  - Daemons Qualcomm com `Restart=always` na imagem: `QCMAP_ConnectionManagerd`, `qtid`, `rmt_storage`, `mdsprpcd`, `btproperty`, `leprop`, `reboot-daemon`.
  - `vic-engine.service` faz `chown` de três arquivos de cpufreq para o usuário `engine` a cada início (`vic-engine.service:12-14`).
- **Problema:** o engine não tem `TimeoutStopSec`, então a parada usa o padrão de 90 s do systemd se o `SIGTERM` não for atendido fora do fluxo do handler.
- **Mudança sugerida:** `TimeoutStopSec=10` em `vic-engine` e `vic-anim`; `OOMScoreAdjust` conforme A5; medir a RSS dos daemons Qualcomm e desligar os que não fazem falta (candidatos: `QCMAP_ConnectionManagerd`, `qtid`; **não** mexer em `rmt_storage` sem teste, pois o firmware do wifi pode depender dele).
- **Validar no robô:** `systemctl list-units --type=service --state=running`; `ps -eo rss,nice,psr,comm --sort=-rss | head -25`; `systemd-cgtop -n 1`.

---

## 4. Extensões de funcionalidade

### 4.1 Telemetria barata para fora

Um serviço pequeno, fora do victor, que publica uma linha JSON por UDP a cada 10 s (60 s em modo noturno). Custo: um processo shell acordando a cada 10 s.

- Receita nova: `poky/victor/meta-anki/recipes/vic-telemetry/` (script + unit), adicionada ao `apq8009-anki-robot-image.inc`.
- Destino configurável sem regerar imagem: `EnvironmentFile=-/data/etc/vic-telemetry.env` (mesmo padrão já usado em `anki/rebooter/rebooter.service`, `EnvironmentFile=-/data/etc/rebooter.env`).

```ini
# vic-telemetry.service
[Unit]
Description=Telemetria leve do robô
After=network-online.target

[Service]
EnvironmentFile=-/data/etc/vic-telemetry.env
ExecStart=/usr/bin/vic-telemetry.sh
Restart=always
RestartSec=10
Nice=10
OOMScoreAdjust=500
User=net
SupplementaryGroups=3003

[Install]
WantedBy=multi-user.target
```

```bash
#!/bin/bash
# vic-telemetry.sh  (TELEM_HOST, TELEM_PORT, TELEM_PERIOD em /data/etc/vic-telemetry.env)
: "${TELEM_PORT:=9870}" "${TELEM_PERIOD:=10}"
[ -z "$TELEM_HOST" ] && exit 0
while :; do
  read -r l1 l5 l15 _ < /proc/loadavg
  ma=$(awk '/MemAvailable/{print $2}' /proc/meminfo)
  t=$(cat /sys/class/thermal/thermal_zone3/temp 2>/dev/null)
  f=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq)
  g=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor)
  st=""; for u in vic-robot vic-anim vic-engine vic-cloud vic-switchboard; do
    st="$st\"$u\":\"$(systemctl is-active $u)\","; done
  fc=$(cat /run/fault_code.showing 2>/dev/null || echo 0)
  rc=$(cat /run/fault_restart_count 2>/dev/null || echo 0)
  up=$(cut -d. -f1 /proc/uptime)
  echo "{\"up\":$up,\"load\":[$l1,$l5,$l15],\"mem_avail_kb\":$ma,\"temp\":$t,\"freq\":$f,\"gov\":\"$g\",${st}\"fault\":$fc,\"restarts\":$rc}" \
    > /dev/udp/$TELEM_HOST/$TELEM_PORT
  sleep "$TELEM_PERIOD"
done
```

O vector-bridge recebe o UDP e publica no Redis. A ausência de pacotes por 30 s vira o alarme "robô caiu". Bateria: o valor está no `vic-robot`/engine; pelo SO, o caminho barato é o gRPC `BatteryState` (4.3).

### 4.2 Supervisão 24/7

Combina A1, A2 e A6. Um verificador de saúde independente do `fault-code-handler`:

```ini
# vic-health.timer
[Timer]
OnBootSec=3min
OnUnitActiveSec=1min
```

```bash
#!/bin/bash
# vic-health.sh: reboot se o trio essencial ficar inativo por 10 verificações seguidas
S=/run/vic-health.bad
if systemctl is-active --quiet vic-robot vic-anim vic-engine; then rm -f $S; exit 0; fi
n=$(( $(cat $S 2>/dev/null || echo 0) + 1 )); echo $n > $S
[ $n -ge 10 ] || exit 0
d=/data/fault-reports/health-$(date +%Y%m%d-%H%M%S); mkdir -p $d
tail -n 2000 /var/log/messages > $d/messages.log; dmesg > $d/dmesg.log
logger -t vic-health "vic-* inativos ha $n min: reboot"
sync; /sbin/reboot
```

Guarda contra boot-loop: o mesmo contador diário em `/data` de A1.

### 4.3 Canal oficial de pose e estado, em vez do webviz

**FATO.** O `vic-gateway` (dentro do `vic-cloud`) expõe gRPC em 443 com:

- `EventStream` (`victor/tools/protobuf/gateway/public/external_interface.proto:106`), que carrega o evento periódico de estado do robô (pose, velocidades, bateria, status);
- `BatteryState` (`external_interface.proto:365`);
- `NavMapFeed` (`external_interface.proto:512`), streaming do mapa com frequência escolhida pelo cliente.

O gateway conversa com o engine por sockets de domínio em `/dev/socket/` (`_engine_gateway_server_`, `_engine_gateway_proto_server_`, `victor/cloud/cloud/ipc_manager.go:41-43`).

Caminhos, do menor esforço para o maior:

1. **vector-bridge como cliente gRPC único** (SDK). Uma conexão TLS, um `EventStream`, um `NavMapFeed`. Sem o limite de 4 threads, sem JSON, sem timeout de 1 h. Depende de autenticação (token do wire-pod ou do `vic-cloudless`). Não muda o robô.
2. **Publicador local UDP** no robô: um processo pequeno que consome o `EventStream` em `localhost:443` e reenvia pose e estado por UDP a 10–20 Hz para o bridge. Mantém o robô com zero conexões TCP de entrada.
3. **Canal nativo no engine**: emitir um datagrama compacto (pose, estado, bateria) direto do engine. Exige mudança no wire-os-victor (seção 6).

### 4.4 Modo noturno

**FATO.** Já existe a base: o engine entra em "power save" e baixa a CPU para `userspace` a 200/400 MHz, com variáveis `kPowerSave_CalmMode`, `kPowerSave_Camera`, `kPowerSave_LCDBacklight`, `kPowerSave_ThrottleCPU` (`victor/engine/components/powerStateManager.cpp:40-45`). O wifi tem economia ligada (`gEnableBmps=1`, `android_compat/device/qcom/msm8909/WCNSS_qcom_cfg.ini:21`). O connman não faz checagem online (`OnlineCheckMode = none`, `poky/victor/meta-vicos-mods/recipes-connectivity/connman/connman/main.conf`).

Esboço: um par de units `night-mode.service` / `night-mode.timer` (`OnCalendar=*-*-* 23:00` e desligamento às 07:00), que só age se o robô estiver na base:

```bash
# night-on
systemctl stop wired avahi-daemon vhostname-publish rsyncd 2>/dev/null
systemctl stop update-engine.timer fake-hwclock-tick.timer
systemctl stop mm-anki-camera mm-qcamera-daemon        # exige o engine tolerar a câmera ausente (seção 6)
iptables -I INPUT 1 -p tcp -m multiport --dports 8887:8890 -j REJECT
echo 60 > /run/vic-telemetry.period
```

O vector-bridge fecha as conexões ao receber o aviso (ou por horário). O estado "na base" e o pedido de power save precisam vir do engine: hoje o power save pode ser pedido pelo webviz (`powerStateManager.cpp:84-87`, `enablePowerSave`), o que serve para um primeiro teste.

Meta mensurável: `ss -tn state established | wc -l` igual a 0 e a temperatura na base estabilizada abaixo do valor diurno.

### 4.5 Atualização segura

Ver A9. Sequência sugerida: boot-successful condicionado à saúde → verificação de assinatura → servidor OTA do dono (`UPDATE_ENGINE_URL` já é aceito por variável de ambiente, `update-engine.cpp:478`, e `EnvironmentFile=-/run/vic-switchboard/update-engine.env` permite sobrepor sem regerar imagem).

### 4.6 Trocar um binário `vic-*` sem regerar a imagem

Três caminhos, do disponível hoje ao mais limpo:

1. **Hoje, sem mudar nada:** `mount -o remount,rw /` e copiar o binário para `/anki/bin` por `scp` ou pelo `rsyncd` (`systemctl start rsyncd`, módulo `anki_root`, porta 1873). Funciona porque o rootfs é ext4 `ro` simples (`fstab:4`); `dm-verity` não está em `DISTRO_FEATURES` (`linux-msm.inc:36` só inclui `verity.cfg` com essa feature). Confirmar: `grep -c verity /proc/cmdline`. Desvantagens: o slot A/B fica diferente da imagem e a próxima OTA apaga a troca.
2. **Drop-in em `/data`, pequeno e reversível.** Em cada unit:
   ```ini
   ExecStart=/bin/sh -c 'B=/data/dev/bin/vic-engine; [ -x $B ] || B=/anki/bin/vic-engine; exec /usr/bin/logwrapper $B $VIC_ENGINE_OPTS'
   ```
   Para desfazer, apagar o arquivo em `/data/dev/bin`. Exige `/data` montado com `exec`, o que já é o caso (`mount-data:37`). Sobrevive à OTA.
3. **overlayfs em `/anki`** com camada superior em `/data/overlay/anki`. Exige `CONFIG_OVERLAY_FS=y` (A11).

Para compilar só o victor: o `victor.bb` já usa `externalsrc` e chama `project/victor/scripts/victor_build_release.sh`; o histórico cita `populate_sdk`. O ciclo mínimo é `bitbake victor -c compile` e copiar de `anki/victor/_build/vicos/Release/bin/`.

### 4.7 Pontos de extensão que já existem

| Mecanismo | Onde | Uso |
|---|---|---|
| `EnvironmentFile=-/data/etc/rebooter.env` | `anki/rebooter/rebooter.service` | mudar janela, uptime mínimo e inibidor do reboot sem imagem nova |
| `/data/inhibit_reboot`, `/run/inhibit_reboot` | `rebooter.sh:5` | bloquear o reboot de manutenção |
| `/data/data/user-do-not-auto-update` | `update-engine.cpp:315` | bloquear auto-update |
| `/run/vic-switchboard/update-engine.env` | `update-engine.service` | apontar a OTA para outra URL |
| `wired` (HTTP 8080) | `anki/wired/mods/*.go` | presets de frequência, jdocs, wake word |
| `purplpkg` | `external/purplpkg` | instalar binários ARM empacotados (`.ppkg`) |
| gRPC 443 | `external_interface.proto` | controle e estado oficiais |
| Variáveis de console | webviz 8888/8889 | ajuste em runtime (depuração) |

---

## 5. Hipóteses abertas e roteiro de coleta

### 5.0 Coleta única (rodar agora e depois de cada incidente, antes de reiniciar)

```bash
ssh root@<robo> '
D=/data/coleta-$(date +%Y%m%d-%H%M%S); mkdir -p $D; cd $D
uptime > uptime; cat /proc/cmdline > cmdline; cat /etc/os-version /anki/etc/version > versao 2>&1
dmesg > dmesg.log
cp /var/log/messages messages.log 2>/dev/null
journalctl -b --no-pager > journal.log 2>&1
journalctl -t fault-code-handler --no-pager > fault-handler.log 2>&1
systemctl list-units --all --no-pager > units; systemctl list-timers --all --no-pager > timers
systemctl status "vic-*" mm-anki-camera --no-pager > vic-status 2>&1
cat /proc/meminfo > meminfo; free -m > free; cat /proc/swaps > swaps
ps -eo pid,ppid,rss,vsz,nice,psr,etime,comm --sort=-rss > ps
df -h > df; ls -laR /var/log /var/tmp /run/update-engine > tmpfs-ls 2>&1
ls -la --time-style=full-iso /data/maintenance_reboot /data/boot.log /data/format-data.log > marcos 2>&1
ls -la --time-style=full-iso /data/data/com.anki.victor/cache/crashDumps /data/fault-reports /data/panics > dumps-ls 2>&1
cat /run/fault_code.showing /run/fault_restart_count /run/fault_restart_uptime > fault-estado 2>&1
for z in /sys/class/thermal/thermal_zone*; do echo "$(basename $z) $(cat $z/type) $(cat $z/temp)"; done > termico
ls /sys/class/thermal > termico-ls
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_{governor,cur_freq,min_freq,max_freq} > cpufreq
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state > time_in_state 2>&1
cat /proc/sys/kernel/core_pattern > core_pattern
cat /sys/module/msm_poweroff/parameters/download_mode > dload 2>&1
zcat /proc/config.gz > kconfig 2>/dev/null
ss -tanp > sockets 2>&1; cat /proc/net/sockstat > sockstat
iptables -S > iptables 2>&1; connmanctl services > connman 2>&1
cat /sys/class/power_supply/*/uevent > bateria 2>&1
tar czf $D.tgz -C /data $(basename $D)
'
scp root@<robo>:/data/coleta-*.tgz .
scp -r root@<robo>:/data/data/com.anki.victor/cache/crashDumps ./crashDumps
scp -r root@<robo>:/data/fault-reports ./fault-reports
```

Para o `scp` funcionar após um "Restarts exhausted", o `sshd` continua de pé (1.6).

### 5.1 Por que o engine caiu três vezes seguidas às 08:03

| Hipótese | Dado que confirma |
|---|---|
| H1. Falta de memória (tmpfs cheio de log ou core; LMK mata o engine no reinício) | `dmesg.log` com `lowmemorykiller` ou `Out of memory`; `df` com `/var/volatile` ou `/run` cheios; `tmpfs-ls` com `core.vic-engine.*`; `MemAvailable` baixo |
| H2. Rajada de reconexão do webviz logo após o timeout de 1 h (N clientes pedindo `navmap` ao mesmo tempo) | `fault-handler.log` com código **915** (travou) em vez de 914 (morreu); `messages.log` com `AnimProcessMessages.MonitorConnectionState.DisconnectedFromEngine`; horário colado nas reconexões do bridge |
| H3. Defeito no engine (segfault repetível) | `.dmp` em `crashDumps` com a mesma pilha nas três ocorrências; `SERVICE_RESULT=signal` no journal |
| H4. Câmera presa: o handler faz `kill -9` no `mm-anki-camera` a cada falha, e o engine reinicia em 5 s contra uma câmera ainda subindo | `fault-handler.log` com 980/981 na sequência; `journalctl -u mm-anki-camera` |

Como decidir: o campo "Handle fault code N" e "Restart count X/2" em `fault-handler.log` (`fault-code-handler:93,272`) dá o código e os intervalos exatos. Os minidumps dão a pilha. Para ler um `.dmp` é preciso o binário com símbolos do mesmo build (`anki/victor/_build/vicos/Release/bin/*.full`) e o `minidump_stackwalk` do Breakpad.

Reprodução controlada de H2: com o bridge conectado, derrubar e reconectar todos os clientes webviz de uma vez pedindo `navmap`, observando `grep VmRSS /proc/$(pidof vic-engine)/status` a cada 1 s.

### 5.2 Por que sumiu da rede ~2 h depois

| Hipótese | Dado que confirma |
|---|---|
| H5. Bateria: em `POWER_CALM` sem `vic-robot`, a carga não se sustenta | arquivo `bateria` da coleta em intervalos de 10 min com os `vic-*` parados (`systemctl stop anki-robot.target` na base) |
| H6. Panic ou travamento do kernel, sem watchdog, parado em download mode | robô com tela congelada ou apagada e sem resposta a ping, mas voltando ao tirar e pôr na base; `dload` = 1; após A6, conteúdo em `/sys/fs/pstore` |
| H7. Wifi caiu e o connman não reconectou | LED e tela normais, sem ping; `journalctl -u connman` e `dmesg \| grep -i wcnss` na coleta seguinte |

Teste direto de H5 e H7, de dia: `systemctl stop anki-robot.target` com o robô na base, e um laço de `ping` + `ssh ... cat /sys/class/power_supply/*/uevent` a cada 5 min por 3 h.

### 5.3 Outras hipóteses a fechar

| Hipótese | Comando |
|---|---|
| A config de kernel efetiva é o `defconfig` da receita | `zcat /proc/config.gz \| grep -E 'CPU_THERMAL\|MEMCG\|WATCHDOG\|DEBUG_SPINLOCK'` |
| `vic-webserver`, `vic-dasmgr`, `vic-crashuploader` não rodam | `systemctl list-units 'vic-*' --all` |
| `/var/log/messages` não é rotacionado | `ls /etc/logrotate.d; ls -la /var/log` em dois dias seguidos |
| `thermal_zone3` é a CPU | arquivo `termico` da coleta |
| O preset do `wired` em uso | arquivo `cpufreq` da coleta |
| 01:38 foi o reboot de manutenção | arquivo `marcos` da coleta; `uptime` |
| Estados de baixo consumo da CPU ativos | `cat /sys/module/lpm_levels*/parameters/sleep_disabled` |
| rootfs sem verity | `grep -c verity /proc/cmdline; mount \| grep " / "` |
| O core dump do kernel está ativo para os `vic-*` | `grep -i core /proc/$(pidof vic-engine)/limits; cat /proc/sys/kernel/core_pattern` |
| O fuso do robô | `date; readlink -f /etc/localtime; cat /data/etc/timezone` |

---

## 6. Pontos de contato com o wire-os-victor

Mudanças que **dependem do repositório wire-os-victor** (`anki/victor`):

| Tema | Arquivo no victor | Mudança | Achado |
|---|---|---|---|
| Timeout e threads do webviz | `webServerProcess/src/webService.cpp:964-985` | `num_threads` maior; `websocket_timeout_ms` de 24 h ou ping do servidor | A3 |
| Fila de envio do websocket | `webServerProcess/src/webService.cpp:1567-1581,1660-1670` | limite de fila, descarte, serializar uma vez por broadcast, não capturar `conn` cru em lambda | A3 |
| Mapa por pedido | `engine/navMap/mapComponent.cpp:311-313,758-801` | responder só a quem pediu (o callback já recebe `sendFunc`), com limite de taxa | A3 |
| Limites do supervisor | `platform/config/development/etc/fault-code-handler.env` (e `oskr`, `shipping`) | `TIMEOUT_RESTART_SEC=30`, `FAULT_RESTART_LIMIT=4` | A1 |
| Detecção de engine travado | `animProcess/src/cozmoAnim/animProcessMessages.cpp:882` | avaliar 5 s → 10–15 s, ou distinguir "ocupado" de "morto" | A1, 5.1 |
| Zona térmica por índice | `osState/osState_vicos.cpp:108` | procurar pelo `type` da zona | A4 |
| Limite de alerta | `animProcess/src/cozmoAnim/animation/animationStreamer.cpp:195` | alerta em 80 °C e pedido de power save antes de 90 °C | A4 |
| Dono do governor | `osState/osState_vicos.cpp:314-353`, `engine/components/powerStateManager.cpp:52-54` | combinar com o `wired` quem escreve governor e quem escreve teto | A4 |
| Limpeza de minidumps | `platform/config/bin/vic-crashuploader:15` (`exit 0`) | reativar só a parte `KEEP_LATEST`, sem upload | A6 |
| Verificação de OTA | `platform/update-engine/update-engine.cpp:581,741,778` | verificar `manifest.sha256`; corrigir o teste `url == "auto"` | A9 |
| Privacidade da OTA | `platform/update-engine/update-engine.cpp:219` | não enviar o ESN na URL | A9 |
| Modo noturno | `engine/components/powerStateManager.cpp:40-45,84-87` | gatilho externo oficial (gRPC ou arquivo em `/run`) e tolerar a câmera parada | 4.4 |
| Canal de pose nativo | engine (novo) | datagrama compacto de pose/estado, ou garantir `EventStream` estável para um cliente único | 4.3 |
| Texto do erro | `platform/faultCodeDisplay/faultCodeDisplay.cpp:66-76` | mostrar "reboot em N min" quando A1 estiver aplicado | A1 |

Mudanças que ficam **só no wire-os** (este repositório):

- `anki/fault-code/fault-code-handler` e `anki/rebooter/rebooter.sh` (A1, A6).
- `poky/victor/meta-qcom/conf/machine/apq8009-robot.conf` e o `defconfig` (A2, A4, A11).
- `poky/victor/meta-vicos/recipes-core/systemd/systemd-conf/*`, `fstab`, receita de `logrotate`, zram (A5).
- `poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/initscripts/mount-data` (A8).
- `poky/victor/meta-vicos/recipes-extended/update-engine/files/boot-successful.*` (A9).
- `poky/victor/meta-anki/recipes/anki-robot/files/*.service` (A12, A14, 4.6).
- `poky/victor/meta-vicos/recipes-connectivity/iptables-persistent/files/*.rules` e `openssh_%.bbappend` (A7).
- Receitas novas: `vic-telemetry`, `vic-health`, `night-mode` (4.1, 4.2, 4.4).

Mudança no `anki/wired` (terceiro repositório, `os-vector/wired`): preset 2 sem `performance`, escuta em `127.0.0.1` ou com token (A4, A7).

Observação sobre a origem do submódulo: `.gitmodules` aponta `anki/victor` para `os-vector/wire-os-victor`, e não para o fork `IanOliv/wire-os-victor`. Mudanças feitas no fork só entram na imagem se a URL do submódulo for trocada.

---

## Limites desta auditoria

- Os cinco subagentes por área foram iniciados e interrompidos por limite de uso antes de entregar relatório; a análise foi refeita em sequência, com menos profundidade em: drivers de câmera e UART, parâmetros de eMMC, código Go do `vic-cloud` (intervalos de token e jdocs), e `shellcheck` dos scripts.
- Nenhum valor de consumo de memória ou CPU foi medido. Onde há estimativa, está marcada como HIPÓTESE com o comando de medição.
- Comportamentos que dependem de `openembedded-core` e `meta-openembedded` (logrotate padrão, limites padrão do systemd, receita do chrony) não foram lidos.
