# Plano: Stela ajustado para o meu PC

Revisão de 17/11/2026. Substitui o `C:\projetos\macos-stela\PLANO.md` de 16/11 e foi conferida contra o código da v0.5.0 e da `main`.

## Hardware

| Peça | O que significa para o Stela |
|---|---|
| Intel Core **i7-10700**: 8 núcleos / 16 threads, 2,9 GHz base, até 4,8 GHz, AVX2, sem AVX-512, 65 W de TDP | O Mac emulado (M2) tem 8 núcleos. Com 16 threads, o Stela coloca cada um deles num núcleo físico próprio, sem dividir núcleo por hyper-threading (`crates/stela-boot-core/src/topology.rs:1-3`, `crates/stela-boot-core/src/options.rs:598-628`). Sobram 8 threads irmãs. |
| 32 GB DDR4 | Sobra memória. O i7-10700 aceita DDR4-2933. Dual channel (2×16) faz diferença na banda. |
| GTX 1650 Super 4 GB (TU116) | Suportada (GTX 16). É a mesma família da issue #120. |

## O que mudou em relação ao plano de 16/11

1. **O gargalo é o processador, não a GPU.** Na issue #120 (GTX 1660 Super, mesma TU116, com um i5-12400F de 6 núcleos), a versão em desenvolvimento reduziu a espera mediana da GPU de 20,5 ms para 8,6 ms, mas o FPS ficou igual (14,5 → 15,2). O autor concluiu que o limite passou a ser o custo de CPU de cada quadro no macOS. Na v0.5.0 ele mediu 12,8 quadros/s na mediana, 465 s até "Starting macOS" e mais de 1 minuto para abrir um app pela primeira vez.
   - O i7-10700 tem menos desempenho por núcleo que o i5-12400F, mas tem 8 núcleos físicos, um para cada núcleo do Mac.
   - Expectativa: números na mesma faixa da #120.

2. **O limite de potência volta a ser o ajuste principal da BIOS.** Com o limite padrão da Intel (PL1 = 65 W, PL2 = 224 W por ~28 s), os 8 núcleos ocupados por muito tempo derrubam o clock bem abaixo dos ~4,6 GHz. A estimativa de ontem era perto de 3,5 GHz; a medição confirma. Isso depende do cooler e da placa-mãe (ver Fase 1).

3. **Os C-states da BIOS ajudam o Stela.** Núcleos do Mac ociosos dormem nos estados de economia (C-states) que a BIOS declara (`crates/stela-boot/src/native_idle.rs:1-4`). Núcleos dormindo gastam menos do limite de potência, e sobra mais para os que estão trabalhando. Desligar os C-states, como eu sugeri ontem como teste, provavelmente piora.

4. **O áudio não é um detalhe de configuração.** O que o código mostra:
   - O driver HD Audio do PC só é usado num teste de tom (`--audio-check`). O arquivo diz: *"not a guest audio bridge"* (`crates/stela-boot/src/audio_check.rs:1-2`).
   - Do lado do Mac, já existe um alto-falante USB virtual (USB Audio Class 2), que o macOS reconhece com o driver dele (`crates/stela-soc/src/usb/audio.rs`). Ele ainda não está ligado a nada.
   - O `PcieOwnerMissing` (`crates/stela-driver-audio/src/hda/pci.rs:568-572`) é só o primeiro obstáculo: o Stela exige que a capability PCIe do controlador esteja em 0x70. No mesmo ponto, o Linux só lê o registrador 0x78 e escreve quando precisa.
   - Para ter som, falta a ponte completa: alto-falante virtual → driver HD Audio → codec da placa, incluindo escolher a saída certa. É uma funcionalidade nova de tamanho médio.
   - O branch `future/guest-audio` vai pelo caminho de uma placa de som USB, mas está 628 commits atrás da `main` e não foi integrado.

5. **Não precisa compilar no seu PC.** O `stela.exe` tem uma opção escondida, `update --efi`, que grava no pendrive um `stela.efi` que não veio dentro dele (`crates/stela-cli/src/setup/command.rs:505-507`). A compilação vai para o GitHub Actions do fork (`.github/workflows/build-test.yml`). O WSL sai do plano.

6. **O cache de traduções muda a forma de medir.** O Stela guarda no pendrive o código já traduzido e reaproveita nos boots seguintes. O cache é gravado durante o uso e no desligamento ou reinício do Mac (`crates/stela-boot/src/engine/translations.rs:1-25`). Por isso:
   - Os primeiros boots depois de uma troca de versão são mais lentos. Comparar boot frio com boot quente mede o cache, não a mudança.
   - Sempre desligue pelo menu Apple → Desligar. Segurar o botão de energia perde o que ainda não foi gravado.
   - A velocidade do pendrive importa: use USB 3, numa porta USB 3 do painel traseiro.

7. **O projeto anda muito rápido.** Foram 394 commits entre a 0.4.2 e a 0.5.0, em dois dias. A v0.5.0 já saiu oficialmente, e a `main` já está 39 commits à frente. Na #120, uma `main` do dia 16 mediu 8,0 quadros/s contra 12,8 da v0.5.0. Versão mais nova pode ser pior, então tudo se mede antes de adotar.

## Expectativa honesta

"Perfeito" não é alcançável. Cada instrução ARM do macOS é traduzida para x86 em tempo real. A meta realista:

- O processador no clock máximo sustentável, sem estourar temperatura.
- O macOS sem gastar CPU com efeitos visuais e tarefas de fundo.
- O cache de traduções aquecido, para que o uso do dia a dia seja bem melhor que o primeiro boot.
- Som funcionando, quando a ponte de áudio estiver pronta.

## Como medir

**Roteiro fixo (~15 min por boot):**
1. Boot até a área de trabalho.
2. Mexer o cursor por 30 s.
3. Abrir o Finder.
4. Abrir o Safari numa página fixa.
5. Abrir os Ajustes.
6. Rodar `top -l 2` no Terminal.
7. Menu Apple → Desligar.

**Coleta:** no Windows, como administrador, `.\stela.exe log --output log.txt`, e me envie o arquivo. A análise fica do meu lado.

**Números do log** (linhas que o código realmente escreve):

| Métrica | Linha no log |
|---|---|
| Quadros por segundo (mediana, pior janela) | `DCP PACING ... per_second=` |
| Espera da GPU por quadro | `fence ... per swap` |
| Pedido de clock aos núcleos | `HOST PERFORMANCE hwp= ... legacy=` (`crates/stela-boot-core/src/host_performance.rs:372`) |
| Estados de economia aceitos | `HOST IDLE ...` (`crates/stela-boot/src/native_idle.rs:108-189`) |
| Cache de traduções | "Opening the translation cache" e `TRANSLATIONS ... loaded= absent= invalid=` |
| Tempo de boot | marcações de progresso até "Starting macOS" |

**Regras:**
- Uma mudança por vez.
- Linha de base: 2 boots de aquecimento e depois 3 boots medidos.
- Uma mudança só fica se ganhar mais que o dobro da variação natural entre os 3 boots da linha de base.
- Depois de trocar a versão do Stela, repetir o aquecimento.
- O `--profile` (`stela.exe settings --profile on`) mede o tempo de cada parte do motor, mas ele mesmo pesa. Uso só em boots de diagnóstico, nunca nos que medem FPS.

## Segurança

- Ponto de restauração e backup dos arquivos importantes do Windows antes de começar.
- A partição do Windows nunca é tocada.
- Um segundo pendrive com o Stela oficial fica guardado como resgate e não entra nos testes.
- Para voltar ao oficial no pendrive de teste: `.\stela.exe update` sem `--efi`.
- Toda mudança de limite de potência é testada antes no Windows, com monitoramento de temperatura (ver Fase 1).

## Fases

### Fase 0 — Linha de base com a v0.5.0 oficial
1. Baixar o `stela.exe` v0.5.0 e rodar `.\stela.exe update` como administrador.
2. Fazer 2 boots de aquecimento e 3 boots medidos.
3. Eu leio o log e confirmo:
   - quantos núcleos do Mac foram criados;
   - se `HOST PERFORMANCE` saiu como `hwp` ou `legacy`;
   - quais estados de economia (`HOST IDLE`) foram aceitos;
   - se o cache de traduções está sendo gravado;
   - qual é o erro de áudio exato.

### Fase 1 — BIOS (sem código; uma mudança por boot medido)

| Ajuste | Por quê |
|---|---|
| **Intel Speed Shift (HWP) ligado** | Depois que o Stela assume a máquina, nenhum sistema pede velocidade ao processador. O Stela pede o máximo só via HWP; sem isso, o núcleo fica no clock que a BIOS deixou (`host_performance.rs:1-7`). Se o log mostrar `legacy=`, esse é o primeiro ajuste. |
| Turbo Boost ligado | Até 4,8 GHz. |
| **Limite de potência (PL1/PL2/Tau)** | Maior ganho esperado para carga sustentada nos 8 núcleos. Primeiro no Windows: rodar uma carga multinúcleo (ex.: Cinebench R23 por 10 min) com o HWiNFO aberto, com os valores atuais e depois com o PL1 maior. O PL1 sobe só até onde a temperatura do processador fica abaixo de ~85 °C e a dos VRMs (se a placa informar) fica estável. Com o cooler original da Intel, o espaço é pequeno; com um cooler melhor, dá para subir mais. Placas H410/B460 de entrada têm VRMs fracos: não usar "sem limite". |
| C-states ligados (C6/C8 e "Package C-State" no automático) | Núcleos ociosos dormem e liberam potência para os ocupados. Testar contra desligado só se o atraso do cursor piorar. |
| Memória em dual channel, na velocidade nominal (XMP/perfil até 2933) | Conferir pentes e slots. |
| Hyper-Threading desligado (teste de baixa prioridade) | As 8 threads irmãs não recebem núcleo do Mac. Hipótese: efeito neutro. Só fica se medir ganho. |
| Secure Boot desligado e CSM desligado | Requisito do Stela. |

### Fase 2 — Ajustes do macOS (um por vez, medido)
- Reduzir transparência e reduzir movimento (Ajustes → Acessibilidade → Tela).
- Resolução 1920×1080 a 60 Hz. Testar também 1280×720, como sugere a #120.
- Desligar o Spotlight depois que o sistema assentar: `sudo mdutil -a -i off`.
- Desligar itens de login, widgets e sincronização do iCloud.

### Fase 3 — Compilação na nuvem
1. Fork `Dias0202/Stela`. A `main` do fork espelha a oficial. O branch `meu-pc` guarda este plano e o workflow `build-test`, e só adiciona arquivos novos, para nunca dar conflito com a oficial.
2. O workflow roda só manualmente (Actions → build-test → Run workflow), com três entradas: repositório, ref e se embute o Stela.app. Gera dois artefatos: `stela-efi` e `stela-test-windows-x86_64` (`stela.exe` com o mesmo `stela.efi` dentro).
3. Validação obrigatória: compilar a v0.5.0 sem alterações, fazer boot e comparar com a oficial. Se os números não baterem dentro da variação, nosso build está diferente e nada do que vier depois vale.
4. Instalar no pendrive de teste com `.\stela.exe update --efi stela.efi`, usando o `stela.exe` do mesmo build.

### Fase 4 — Acompanhar a versão em desenvolvimento
Testar a `main` de tempos em tempos contra a v0.5.0. Ela só substitui a v0.5.0 se ganhar na medição.

### Fase 5 — Mudanças nossas
- **Áudio (prioridade 1):**
  - corrigir a checagem do `PcieOwnerMissing` do jeito que o Linux faz;
  - criar a ponte alto-falante virtual → HD Audio da placa;
  - testes unitários no padrão do projeto e validação com som real no seu PC.
- **Desempenho:** só onde o `--profile` mostrar gasto real.
  - Hipótese a investigar: usar as 8 threads irmãs que sobram para o trabalho dos drivers do PC (GPU, disco), que hoje divide tempo com os núcleos do Mac.
- Toda mudança que funcionar vira proposta (PR) para o projeto oficial. Com o projeto nessa velocidade, mudanças mantidas só no fork viram conflito toda semana.

### Fase 6 — Sem pendrive (investigar, não prometido)
Ainda não confirmei no código se o Stela pode dar boot pelo SSD interno. Fica para depois que as fases anteriores estiverem estáveis.

## Divisão do trabalho
- **Eu:** leitura dos logs, análise, código, testes, workflow de build, propostas de PR.
- **Você:**
  - BIOS e testes de temperatura no Windows;
  - boots com o roteiro;
  - `stela.exe log --output log.txt` e envio do arquivo;
  - disparar o workflow e baixar os artefatos.

## Pendências
1. Modelo da placa-mãe, do cooler e dos pentes de memória. No PowerShell:
   - `Get-CimInstance Win32_BaseBoard | Select-Object Manufacturer, Product`
   - `Get-CimInstance Win32_PhysicalMemory | Select-Object DeviceLocator, Capacity, ConfiguredClockSpeed`
2. Qual versão do Stela está hoje no pendrive, e se ele é USB 3.
3. Primeiro `log.txt` da v0.5.0 oficial (Fase 0).
