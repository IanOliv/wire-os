# Mapa do wire-os para quem quer deixar o Vector mais inteligente

- Base: `wire-os` @ `a6e4c3a88`, submódulo `anki/victor` (wire-os-victor) @ `505eb84c`.
- Só leitura. Nada foi alterado, compilado ou rodado no robô.
- **FATO** = visto no código, com caminho. **HIPÓTESE** = inferência a confirmar no robô (comando incluído).
- `victor/` abrevia `anki/victor/`. Para a auditoria de estabilidade (supervisão, watchdog, webviz), ver `docs/analise/analise-wire-os.md`.

---

## 1. Visão geral

### 1.1 Pastas

| Pasta | O que é |
|---|---|
| `build/` | Scripts de build em Docker: `build.sh` (entrada principal), `Dockerfile`, `shell.sh`, `clean.sh`, `upload.sh` |
| `poky/` | Yocto. `bitbake`, `openembedded-core`, `meta-openembedded` são submódulos upstream; `poky/build/conf/` tem `local.conf` e `set_bb_env.sh` |
| `poky/victor/meta-anki/` | Camada da Anki/WireOS: receitas dos `vic-*`, `fault-code`, `rebooter`, `wired`, imagem do robô |
| `poky/victor/meta-vicos/`, `meta-vicos-mods/` | Ajustes de sistema: systemd, connman, firewall, montagem de `/data`, câmera, áudio, update-engine |
| `poky/victor/meta-qcom/`, `meta-qcom-prop/` | BSP Qualcomm: máquina `apq8009-robot`, kernel, wifi, Bluetooth, HALs de áudio/câmera/GPU |
| `kernel/msm-4.9/` | Kernel Linux 4.9 da Qualcomm (device tree do Vector em `arch/arm64/boot/dts/qcom/apq8009-robot-anki-vector.dts`) |
| `anki/victor/` | **Submódulo wire-os-victor**: código de todos os `vic-*` (o "cérebro") |
| `anki/wired/` | Painel web de ajustes do WireOS (porta 8080): frequência da CPU, palavra de ativação, sensibilidade |
| `anki/vic-cloudless/` | Alternativa ao `vic-cloud` com reconhecimento de fala rodando no próprio robô (sherpa-onnx) |
| `anki/fault-code/`, `anki/rebooter/`, `anki/rampost/`, `anki/anki-crash-log/` | Tratamento de falhas, reboot noturno, teste do corpo no boot, log de crash |
| `prebuilt_HY11/` | Binários Qualcomm prontos (câmera, Bluetooth, GPU, áudio) |
| `system/`, `external/`, `frameworks/`, `hardware/`, `qmi*/`, `mdm-*`, `vendor/` | Código Android/Qualcomm de apoio (adb, logd, logwrapper, QMI) |
| `ota/` | `Makefile` que empacota a imagem em `.ota`, chaves de assinatura |
| `android_compat/` | Configs de dispositivo Qualcomm (inclui `device/qcom/msm8909/WCNSS_qcom_cfg.ini`, wifi) |

### 1.2 Como o sistema é construído

1. `./build/build.sh -bt <dev|oskr|devcloudless|prod> -bp <senha-boot> -v <número>` (`build/build.sh:17,165`).
2. O script cria a imagem Docker `vic-yocto-builder-7` (`build/build.sh:13,242-247`) e roda tudo dentro dela, com caches em `build/cache`, `build/gocache`, `build/usercache`.
3. Dentro do container: `source poky/build/conf/set_bb_env.sh` e `build-dev` (ou `build-oskr`...), que chama `bitbake machine-robot-image` com `MACHINE=apq8009-robot`, `DISTRO=msm-perf` (`set_bb_env.sh:93-120,209-223`).
4. Depois do bitbake: `cd ota && make devsign && make` assina o boot e gera o `.ota` em `_build/` (`build/build.sh:262-271`).

Toolchain: Yocto "whinlatter", glibc, ARM 32 bits (`armv7`, softfp segundo `anki/vic-cloudless/README.md`). O victor usa o próprio sistema CMake/ninja, chamado pela receita (abaixo).

Instalar no robô: HIPÓTESE, o caminho usual do WireOS é servir o `.ota` e apontar o robô para ele (o `update-engine` aceita uma URL como argumento, `victor/platform/update-engine/update-engine.cpp:470-480`). Com SSH: `/anki/bin/update-engine http://<pc>:8000/arquivo.ota`. Ver `README.md` do repositório para o procedimento oficial.

### 1.3 Onde o wire-os-victor entra

- Receita `poky/victor/meta-anki/recipes/anki-robot/victor.bb`.
- `inherit externalsrc` com `EXTERNALSRC = "${WORKSPACE}/anki/victor"` (`victor.bb:13-15`): o bitbake compila direto da pasta do submódulo.
- `do_compile` escolhe o script por tipo de build (`victor.bb:182-231`). Em `dev`: `project/victor/scripts/victor_build_release.sh` (CMake `Release`). Em `oskr`: `victor_build_oskr.sh`.
- `do_install` roda `project/victor/scripts/install.sh` e coloca tudo em **`/anki`** (`victor.bb:233-242`): binários em `/anki/bin`, bibliotecas em `/anki/lib`, assets em `/anki/data`, configs em `/anki/etc`.
- Compatibilidade checada no build: `VICTOR_COMPAT_VERSION` dos dois repositórios precisa bater (`build/build.sh:106-123`).
- `.gitmodules` aponta `anki/victor` para `os-vector/wire-os-victor`. Se você trabalhar no seu fork, troque a URL do submódulo.

---

## 2. Inicialização e serviços

### 2.1 Ordem de subida (FATO)

```
kernel (zImage-dtb, cmdline em poky/victor/meta-qcom/conf/machine/apq8009-robot.conf:20)
 └─ systemd
     ├─ sysinit: rampost.service (testa o corpo, rampost-init.sh) → vic-bootAnim.service (animação de boot)
     ├─ local-fs: mount-data.service (abre LUKS e monta /data)
     ├─ multi-user.target
     │   ├─ connman, enable-wifi, avahi, logd, init_post_boot (governor), anki-audio-init
     │   ├─ mm-qcamera-daemon → mm-anki-camera
     │   ├─ ankiinit → wired (porta 8080, +3 s)
     │   ├─ multi-user-done.service (espera mount-data, câmera, leprop, Bluetooth, qtid)
     │   └─ start-anki.service → systemctl start anki-robot.target
     └─ anki-robot.target
         vic-init → vic-robot → vic-anim → vic-engine → vic-cloud, vic-switchboard → vic-gateway-cert
```

Fontes: `poky/victor/meta-anki/recipes/anki-robot/files/{start-anki,multi-user-done,vic-*}.service`, `anki-robot.target`; `poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/services/*.service`; `poky/victor/meta-vicos/recipes-core/initscript-anki/files/rampost.service`; `anki/wired/wired.service`.

### 2.2 Processos do robô

| Processo | O que faz | Unit | Config de ambiente |
|---|---|---|---|
| `vic-robot` | Fala com o corpo (STM32) pela UART `/dev/ttyHS0`; motores, sensores, **microfones** | `vic-robot.service` (Nice -20, CPUs 1-3) | `/anki/etc/vic-robot.env` |
| `vic-anim` | Rosto, som, **processamento de microfone, palavra de ativação** | `vic-anim.service` (Nice -2) | `/anki/etc/vic-anim.env` |
| `vic-engine` | Comportamentos, visão, mapa, decisão | `vic-engine.service` | `/anki/etc/vic-engine.env` |
| `vic-cloud` (inclui `vic-gateway`) | Envia voz ao WirePod (chipper), jdocs, token; gRPC do SDK em 443 | `vic-cloud.service` | `/anki/etc/vic-cloud.env`, `/data/data/server_config.json` |
| `vic-switchboard` | Bluetooth LE para configuração | `vic-switchboard.service` | `/anki/etc/vic-switchboard.env` |
| `mm-anki-camera` | Daemon da câmera | `mm-anki-camera.service` | argumentos na unit |
| `wired` | Painel de ajustes do WireOS | `wired.service` | `/data/...` (salva preferências) |

Todos os `.service` dos `vic-*` ficam em `poky/victor/meta-anki/recipes/anki-robot/files/`. Os `.env` vêm de `victor/platform/config/etc/` e, por tipo de build, de `victor/platform/config/{development,oskr,shipping}/etc/` (`victor/platform/config/CMakeLists.txt:40-99`). Os parâmetros finos de comportamento ficam nos JSON de `victor/resources/` (instalados em `/anki/data/assets/cozmo_resources/`).

Todos os `vic-*` têm `Restart=no`. Quem reinicia é `anki/fault-code/fault-code-handler` (detalhes na auditoria).

---

## 3. Áudio e microfones

### 3.1 O caminho do som até o WirePod (FATO)

Os microfones **não passam pelo ALSA nem pelo kernel**. O ALSA do SoC só cuida do alto-falante.

| Etapa | Onde | Detalhe |
|---|---|---|
| 1. Captura | Corpo (STM32), `victor/robot/syscon/src/mics.cpp` | 4 microfones PDM lidos por SPI1/SPI2 com DMA (`mics.cpp:26-90`). Clock PDM 3 MHz, decimação 96 (`victor/robot/syscon/src/timer.h:14-18`) |
| 2. Transporte | UART `/dev/ttyHS0` ("spine") | 80 amostras por microfone por quadro de 5 ms (`victor/robot/syscon/schema/messages.h:227-228`) = **16 kHz, 16 bits, 4 canais** |
| 3. Recepção | `vic-robot` | repassa os quadros ao `vic-anim` |
| 4. Processamento | `vic-anim`, `victor/animProcess/src/cozmoAnim/micData/micDataProcessor.cpp` | Signal Essence (beamforming, supressão de ruído) e VAD; taxa `kSampleRate_hz = 16000` (`victor/lib/util/source/anki/audioUtil/audioDataTypes.h:30`) |
| 5. Palavra de ativação | `vic-anim`, `victor/animProcess/src/cozmoAnim/speechRecognizer/speechRecognizerPicovoice.cpp` | Picovoice Porcupine 1.5.0, modelo `hey_vector.ppn`, sensibilidade padrão **0.60** (`:33-35`) |
| 6. Envio | `vic-cloud`, `victor/cloud/internal/voice/process.go` | Opus, 66 kbps, complexidade 0, quadros de 60 ms (`process.go:367-375`); blocos de 120 ms (`victor/cloud/cloud/main.go:199-202`); timeout de 9 s por pedido (`process.go:33`) |
| 7. Destino | WirePod | gRPC "chipper" no endereço de `/data/data/server_config.json` (seção 5) |

### 3.2 Alto-falante (ALSA)

- `anki-audio-init` liga o caminho do alto-falante e fixa `RX3 Digital Volume = 74` (`poky/victor/meta-anki/recipes/anki-robot/files/anki-audio-init.sh` e `poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/initscripts/anki-audio-init`).
- **FATO:** existem duas versões desse script, instaladas no mesmo caminho `/etc/initscripts/anki-audio-init` por receitas diferentes (`anki-robot-target.bb` e `extra-conf.bb`). Uma usa `numid=`, a outra usa nomes de controle. HIPÓTESE: a que vence depende da ordem de instalação. Confirmar: `cat /etc/initscripts/anki-audio-init` no robô.
- Não há ganho de microfone para ajustar no ALSA, porque os microfones não estão nele.

### 3.3 O que pode degradar o áudio enviado ao WirePod

| # | Ponto | Tipo | Onde | Por que importa |
|---|---|---|---|---|
| 1 | **Modo calmo usa um microfone só, com ganho ×8 sem saturação** | FATO no código; HIPÓTESE no efeito | `micDataProcessor.cpp:84,567-572,626-643` | Quando o engine pede economia de energia (`CALM_POWER_MODE`, `:493`), o anim desliga o beamforming e faz `bufferOut[i] <<= 3` em amostras de 16 bits. Sem limitar o valor, voz alta passa do limite e "dá a volta" (estouro de inteiro), virando distorção forte. Economia de energia é pedida por vários comportamentos (modo silencioso, por exemplo: `behaviorQuietModeCoordinator.cpp`) |
| 2 | **Sem cancelamento de eco** | FATO | `micDataProcessor.cpp:648-653` | O sinal do alto-falante passado ao Signal Essence é um vetor de zeros (`dummySpeakerOut`). Falar enquanto o Vector toca som ou fala piora muito |
| 3 | Beamforming desligado com o robô em movimento | FATO | `micDataProcessor.cpp:567-569` | Com motor ou alto-falante ativos, usa `SigEsBeamformingOff` |
| 4 | Sensibilidade da palavra de ativação | FATO | `speechRecognizerPicovoice.cpp:34-35`; `anki/wired/mods/sensitivity-pv.go:11-12` | O anim usa 0.60 se o arquivo `/data/data/com.anki.victor/persistent/picovoice/sensitivity2` não existe; o painel `wired` mostra 0.45 como padrão, mas só grava quando você salva. O valor exibido pode não ser o valor em uso |
| 5 | Opus com complexidade 0 | FATO; efeito HIPÓTESE | `process.go:371-373` | A taxa (66 kbps) é boa, mas a complexidade mínima reduz a qualidade do codificador. Efeito pequeno no reconhecimento, a medir |
| 6 | CPU limitada a 533 MHz por padrão | FATO no limite; HIPÓTESE no efeito | `anki/wired/mods/freq.go:13-14,30-43,84-88` | Sem arquivo salvo, o `wired` aplica `FreqPreset = 0` no boot: teto de 533 MHz (a mensagem de log diz "preset 1", mas a variável é 0). Signal Essence + Porcupine + engine competem por pouca CPU; se o anim atrasar, quadros de áudio podem ser perdidos. Teste: preset 1 ou 2 no painel e comparar a taxa de acerto |
| 7 | VAD | FATO | `micDataProcessor.cpp:159-160` | Limiar absoluto 250 e hangover de 10 blocos; ajusta quando o processamento "acorda" |
| 8 | Timeout de 9 s | FATO | `process.go:33` | Frases longas ou WirePod lento acabam cortadas |

Distinguir "não ouviu a palavra" de "ouviu e o WirePod não entendeu":

```bash
# no robô
journalctl -f | grep -i -E 'hotword|wakeword|trigger|voice|chipper|intent'
cat /data/data/com.anki.victor/persistent/picovoice/sensitivity2 2>/dev/null || echo "padrão 0.60"
# no WirePod: ative o log de depuração e veja o texto transcrito de cada pedido
```

Se o WirePod recebe o texto errado, o problema é áudio ou STT (itens 1, 2, 5 e o modelo do WirePod). Se nada chega ao WirePod, é a palavra de ativação (item 4) ou a rede.

O `vic-cloud` envia `SaveAudio = true` ao servidor (`victor/cloud/cloud/main.go:199`, `victor/cloud/internal/voice/process.go:207`): é um pedido para o **WirePod** guardar o áudio recebido. HIPÓTESE: se a sua versão do WirePod honra esse campo, o áudio de cada pedido fica no servidor; ouvir esse arquivo é a forma mais direta de avaliar a qualidade.

---

## 4. Câmera

| Item | Valor | Fonte |
|---|---|---|
| Sensor | MIPI CSI no `i2c_3`, `cell-index 1`, `mount-angle 90`, MCLK 24 MHz, GPIO 26/35/34 | `kernel/msm-4.9/arch/arm64/boot/dts/qcom/msm8909-pm8916-camera-sensor-robot-pronto.dtsi:18-56` |
| Driver do kernel | `msm_camera` v2 da Qualcomm | `kernel/msm-4.9/drivers/media/platform/msm/camera_v2/` |
| Pilha de usuário | `mm-qcamera-daemon` + `mm-anki-camera`, **binários prontos** (sem código-fonte) | `prebuilt_HY11/apq8009-robot/mm-camera/usr/bin/` |
| Bibliotecas de sensor | vários `libmmcamera_*.so` e `libchromatix_*.so` (ajuste de imagem por sensor) | `prebuilt_HY11/apq8009-robot/mm-camera/usr/lib/` |
| Unit | `mm-anki-camera-wrapper -v 0 -r 1` (`LD_PRELOAD=/usr/lib/libshimcsu.so`) | `poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/services/mm-anki-camera.service` |
| Taxa | `-r <N>` reduz para 30/N fps; a unit usa `-r 1` = **30 fps** do daemon | ajuda do binário (`strings`) |
| Formatos | Bayer BGGR10 (RAW), RGB888, YUV, e variantes "2MP" | `victor/platform/camera/vicos/camera_client/camera_client.h:98-104` |

Como o quadro chega ao `vic-engine` (FATO, `victor/platform/camera/vicos/camera_client/camera_client.c`):

1. O daemon escuta no socket Unix `/var/run/mm-anki-camera/camera-server` (`:39`); o engine conecta por `/dev/socket/vic-engine-cam_client0` (`:38`).
2. O daemon aloca um buffer em **ION** (`/dev/ion`, memória compartilhada sem cópia) e passa o descritor de arquivo ao engine pelo socket (`SCM_RIGHTS`, `:159`).
3. Os quadros ficam num anel de até 6 posições (`ANKI_CAMERA_MAX_FRAME_COUNT 6`, `:55`), com trava por posição (`locked_slots`, `:100`).
4. O `vic-engine` lê, converte e passa para a visão (`victor/engine/components/visionComponent.cpp`).
5. O engine liga e desliga o sensor pelo GPIO 94 (`PWDN_PIN 94`, `camera_client.c:123`; exportado em `vic-engine.service:18`).

Para pegar imagens fora do robô sem mexer na câmera: o gRPC do SDK em 443 tem stream de câmera (`victor/tools/protobuf/gateway/public/external_interface.proto`). Isso é o caminho mais simples para mandar visão para um PC.

---

## 5. Rede e comunicação com o WirePod

### 5.1 Endereços (FATO)

O `vic-cloud` lê os endereços de, nesta ordem (`victor/cloud/internal/config/urls.go:48-66`):

1. `/data/data/server_config.json` (gravado pela configuração do WirePod)
2. `/anki/data/assets/cozmo_resources/config/server_config.json` (padrão da imagem)

Padrão da imagem (`victor/resources/development/config/server_config.json`):

```json
{ "jdocs": "vicapi.pvic.xyz:8081", "tms": "vicapi.pvic.xyz:8081",
  "chipper": "vicapi.pvic.xyz:8081", "check": "vicapi.pvic.xyz/ok", ... }
```

Com o WirePod, os quatro campos apontam para o servidor local (HIPÓTESE do formato exato: `"<ip-ou-escapepod.local>:443"`; confirmar com `cat /data/data/server_config.json`).

| Serviço | Para que serve | Código |
|---|---|---|
| `chipper` | voz → texto → intent | `victor/cloud/internal/voice/` |
| `tms` | token de autenticação | `victor/cloud/internal/token/` |
| `jdocs` | configurações do robô (nome, volume, localização) | `victor/cloud/internal/jdocs/` |
| `check` | teste de conectividade | `victor/cloud/internal/voice/stream/connect.go:25-60` |

### 5.2 Certificados (FATO)

`victor/cloud/cloud/main.go:34,91-120`:

1. Tenta `/anki/etc/wirepod-cert.crt`; se carregar, copia para `/data/data/wirepod-cert.crt`.
2. Senão, tenta `/data/data/wirepod-cert.crt`.
3. Acrescenta o certificado ao conjunto de CAs raiz (`rootcerts.ServerCertPool()`), usado no TLS do chipper, token e jdocs.

Token salvo em `/data/data/com.anki.victor/persistent/token` (`victor/cloud/internal/token/identity/getcert_vicos.go:15`). Chave do robô em `/data/etc/robot.pem`, gerada no primeiro boot (`poky/victor/meta-vicos/recipes-core/extra-conf/extra-conf/initscripts/mount-data`, `make_userdata`).

### 5.3 Rede do robô

- Wifi: connman 1.36, `OnlineCheckMode = none` (`poky/victor/meta-vicos-mods/recipes-connectivity/connman/connman/main.conf`); economia de energia do wifi ligada (`android_compat/device/qcom/msm8909/WCNSS_qcom_cfg.ini:7,21`).
- mDNS: `avahi-daemon` publica o nome do robô (`poky/victor/meta-vicos-mods/recipes-connectivity/avahi/files/`). O WirePod costuma ser achado como `escapepod.local` (referência em `anki/vic-cloudless/internal/jdocs/client.go`).
- Firewall de entrada (build `dev`/`oskr`): 22, 443, 8080, 5555, 1873, 5103, 8887–8890, 24024–24026 (`poky/victor/meta-vicos/recipes-connectivity/iptables-persistent/files/iptables-eng.rules`). Saída livre pelo `wlan0`.
- Portas que interessam a você: **443** (gRPC do SDK, controle oficial), **8080** (`wired`), **8888/8889** (webviz de depuração do engine e do anim), **22** (SSH root).

Conferir no robô:

```bash
cat /data/data/server_config.json
ls -la /anki/etc/wirepod-cert.crt /data/data/wirepod-cert.crt
journalctl -u vic-cloud -n 50 --no-pager
ss -tnp | grep vic-cloud          # conexão ativa com o WirePod
```

---

## 6. Recursos para processamento extra

### 6.1 O que o robô tem (FATO)

| Recurso | Valor | Fonte |
|---|---|---|
| CPU | 4× Cortex-A7, até 1267,2 MHz | `apq8009-robot-anki-vector.dts:30-42` |
| Limite de CPU na prática | `wired` **preset 0 (padrão do código): 533 MHz**; preset 1: 729,6 MHz; preset 2: 1267 MHz | `anki/wired/mods/freq.go:13-14,30-43,84-98` |
| GPU | Adreno 304 (`kgsl-3d0`), usada pelo engine | `vic-engine.service:17` |
| RAM | `mem=511M` | `apq8009-robot.conf:20` |
| Swap/zram | não configurado (zram existe no kernel, ninguém liga) | `defconfig:1281`; `system/core/rootdir/etc/init.qcom.post_boot.sh:104` |
| Sistema | partição ext4 de 800 MiB, somente leitura | `poky/victor/meta-qcom/conf/machine/include/apq8009.inc:36` |
| Dados | `/data` cifrado, ext4 | `mount-data` |
| eMMC | 4 GB (informado; tabela de partições não está no repositório) | — |

Já rodando: Signal Essence + Porcupine (anim), visão e comportamento (engine), Opus e gRPC (cloud), câmera a 30 fps.

### 6.2 Quanto sobra (medir, não está no código)

```bash
uptime; cat /proc/loadavg
free -m; grep MemAvailable /proc/meminfo
top -b -n 1 | head -20
ps -eo rss,pcpu,comm --sort=-rss | head -12
df -h / /data
for z in /sys/class/thermal/thermal_zone*; do echo "$(cat $z/type) $(cat $z/temp)"; done
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_{governor,max_freq}
```

HIPÓTESE com base no hardware: com tudo ligado, a folga típica é de algumas dezenas a pouco mais de 100 MB de RAM e uma fração de um núcleo. Não há swap; estourar a RAM mata `vic-*` e dispara "Restarts exhausted".

### 6.3 No robô ou no PC?

| Tarefa | Onde | Motivo |
|---|---|---|
| Palavra de ativação | Robô (já está) | latência e privacidade |
| Fala → texto | PC (WirePod) | o `vic-cloudless` prova que dá no robô, mas com o modelo de menor qualidade (`anki/vic-cloudless/README.md`) |
| LLM, intents complexos | PC | impossível com 511 MB |
| Detecção de objetos (YOLO etc.) | PC, via stream de câmera gRPC | CPU e RAM |
| Filtros leves (limiar, ajuste de ganho, telemetria) | Robô | custo baixo |

Regra prática: o robô capta, executa e reage; o PC entende e decide.

---

## 7. Os 3 pontos mais fáceis para começar

1. **Sensibilidade da palavra de ativação (sem compilar nada, 5 min).**
   - Arquivo: `/data/data/com.anki.victor/persistent/picovoice/sensitivity2`, lido em `victor/animProcess/src/cozmoAnim/speechRecognizer/speechRecognizerPicovoice.cpp:34`.
   - Pelo painel: `http://<ip-do-vector>:8080`, módulo de sensibilidade (`anki/wired/mods/sensitivity-pv.go`).
   - Teste valores entre 0.55 e 0.75 e reinicie o anim: `systemctl restart vic-anim`.
   - No mesmo painel, teste o preset de CPU 1 (729 MHz) em vez do 0 (533 MHz, padrão) e compare a taxa de acerto.
2. **Intents e respostas no WirePod (sem tocar no robô, 30 min).**
   - O robô só envia áudio; transcrição e intent são decididos no WirePod. Intents personalizados e o modelo de STT do WirePod dão o maior ganho de "inteligência" com o menor risco. Ver o ponto de entrada em `victor/cloud/internal/voice/process.go` para o que o robô espera de volta (`IntentResult`).
3. **Controle e visão por gRPC do SDK, a partir de um PC (1–2 h).**
   - `victor/tools/protobuf/gateway/public/external_interface.proto`: `EventStream`, câmera, `NavMapFeed`, comandos de movimento, `SayText`.
   - Um programa no PC (Python `anki_vector` / `wirepod_vector_sdk`) pode ver pela câmera, decidir com um LLM e mandar o robô agir, sem gerar imagem nova.

Primeira mudança de código que vale a pena no wire-os-victor (para a próxima sessão): limitar o ganho no modo de um microfone (`micDataProcessor.cpp:626-643`) com saturação em vez de deslocamento puro.
