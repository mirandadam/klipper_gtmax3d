# Backup e migração para um cartão SD novo

Cartões SD se desgastam, e sistemas antigos deixam de receber atualizações. O jeito mais seguro de
atualizar o Raspberry Pi é montar um **cartão novo** e guardar o antigo intacto como plano B. Foi o que o
autor fez em 2026, ao sair do Raspbian buster (FluiddPi, 2021) para o Raspberry Pi OS trixie.

## O que guardar

| O quê | Onde fica (sistema atual) | Onde ficava (FluiddPi/buster) |
|---|---|---|
| Configurações (`printer.cfg`, `moonraker.conf`...) | `~/printer_data/config` | `~/klipper_config` |
| Arquivos G-code | `~/printer_data/gcodes` | `~/gcode_files` ou o caminho do `[virtual_sdcard]` |
| Banco de dados do Moonraker (histórico, totais, ajustes do Fluidd) | `~/printer_data/database` | `~/.moonraker_database` |
| Logs (opcional) | `~/printer_data/logs` | `~/klipper_logs` |
| Configurações de compilação do firmware | ex.: `~/klipper_build_configs` | `~/klipper/.config` (só a última usada) |

O Klipper mantém cópias antigas do `printer.cfg` (`printer-AAAAMMDD_HHMMSS.cfg`) a cada `SAVE_CONFIG`. Elas
são um histórico útil de calibrações: guarde junto.

## Fazer o backup

Pare o Moonraker e o Klipper antes de copiar o banco de dados, para ele não ser gravado no meio da cópia.
No sistema atual:

```bash
sudo systemctl stop moonraker klipper
cd ~ && tar cf ~/backup_impressora.tar printer_data klipper_build_configs
sudo systemctl start klipper moonraker
```

No FluiddPi/buster (troque `gcode_files` pelo caminho do `[virtual_sdcard]`, se for outro):

```bash
sudo systemctl stop moonraker klipper
cd ~ && tar cf ~/backup_impressora.tar klipper_config gcode_files .moonraker_database klipper_logs klipper/.config
sudo systemctl start klipper moonraker
```

O arquivo fica no próprio cartão: confira antes se há espaço livre (`df -h ~`), principalmente se houver
muitos G-codes.

Com o Moonraker ligado, dá para pedir a ele uma cópia consistente do banco:
`curl -s -X POST "localhost:7125/server/database/backup"`, rodado no próprio Pi (o arquivo vai para
`~/printer_data/backup/database/`). Com login obrigatório, o `curl` precisa da chave de API (ver
[instalacao.md](instalacao.md#4-moonraker)). Esse comando foi usado no sistema atual; em versões antigas do
Moonraker ele pode não existir.

Para copiar do Pi para um computador: `scp pi@impressora3d.local:backup_impressora.tar .`

No Windows, use o `scp` ou o Git Bash. **Não use o redirecionamento `>` do PowerShell** para salvar dados
binários vindos do `ssh` (como `ssh ... tar cf - ... > arquivo.tar`): ele converte o conteúdo como texto e
corrompe o arquivo. Se precisar redirecionar, use `cmd /c "ssh ... > arquivo.tar"`.

## Restaurar o histórico ("hodômetro")

O Fluidd mostra o total de horas, de impressões e de filamento com base no histórico do Moonraker. Para
mantê-lo no sistema novo:

1. Instale o sistema novo ([instalacao.md](instalacao.md)), mas deixe o Moonraker parado:
   `sudo systemctl stop moonraker`.
2. Copie os arquivos do banco antigo (`moonraker-sql.db` e, se existirem, `data.mdb` e `lock.mdb`) para
   `~/printer_data/database/`, substituindo os que existirem.
3. Ligue o Moonraker: `sudo systemctl start moonraker`.
4. Confira no Fluidd, em *Histórico*, se os totais batem com os antigos.

Na migração do autor, o banco antigo já estava no formato atual (`moonraker-sql.db`) e os totais vieram
idênticos. Bancos muito antigos, que só têm o `data.mdb`, devem ser convertidos pelo Moonraker na primeira
inicialização, mas isso o autor não testou. Guarde uma cópia do banco antigo antes, por garantia.

## Restaurar a configuração

- Copie os arquivos de `config/` para `~/printer_data/config/`.
- No `printer.cfg`, ajuste o caminho do `[virtual_sdcard]` para `~/printer_data/gcodes`.
- **Não reaproveite o `moonraker.conf` antigo sem revisar**: no layout novo ele usa
  `klippy_uds_address: ~/printer_data/comms/klippy.sock`, e opções antigas podem ter mudado de nome. É mais
  simples partir do exemplo em [instalacao.md](instalacao.md#4-moonraker) e trazer as suas personalizações.
- Copie os G-codes para `~/printer_data/gcodes`.
